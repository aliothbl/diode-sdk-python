# SPEC-20261113-2201: Deadlines e retry de transporte com backoff (gRPC `Ingest` e token OAuth2)

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:01
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** timeout, deadline, retry, backoff, jitter, UNAVAILABLE, RESOURCE_EXHAUSTED, DEADLINE_EXCEEDED, requests, ConnectionError, retryPolicy, service-config
**Features:** client-transport, client-auth
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11, SPEC-20261113-2200-ingest-retry-preserva-entidades (id de request estável)
**Programa:** —
**Impacto:** alto · **Complexidade:** média
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achados S-4 e S-5).
**Resumo:** Hoje nenhuma chamada de rede do `DiodeClient` tem deadline, e só `UNAUTHENTICATED` (gRPC) e alguns HTTP status (token) são repetidos. Introduzir timeouts configuráveis e retry com backoff exponencial e jitter para falhas transitórias de transporte.

## Objetivo

Um ingester ou endpoint de auth que aceita a conexão e não responde trava o produtor **para sempre**: o construtor (`_authenticate`) e cada `ingest()` bloqueiam sem limite. Também não há resiliência a falhas transitórias comuns em k8s (rolling update do ingester → `UNAVAILABLE`, backpressure do Redis → `RESOURCE_EXHAUSTED`, `ConnectionError` no `/auth/token`). O objetivo é dar ao SDK um comportamento previsível: sempre falhar dentro de um prazo e repetir automaticamente o que é transitório.

## Escopo

**DENTRO:**
- Parâmetro `timeout: float | None = 30.0` em `DiodeClient.__init__`, com env `DIODE_INGEST_TIMEOUT`, aplicado em `self._stub.Ingest(..., timeout=...)`.
- Timeout do token: `session.post(url, ..., timeout=(connect, read))` com default `(5, 30)` e env `DIODE_AUTH_TIMEOUT`.
- Retry de transporte no `ingest()` para `UNAVAILABLE` e `RESOURCE_EXHAUSTED`. `DEADLINE_EXCEEDED` é configurável e fica desligado por default, porque o servidor pode ter processado. Usar backoff exponencial com jitter total e reaproveitar `_auth_retry_delay`/`_sleep` injetável (já existe em `_DiodeAuthentication`). O limite vem de `max_retries` (novo, env `DIODE_MAX_RETRIES`, default 3), separado de `max_auth_retries`.
- Retry de `requests.ConnectionError`/`Timeout` na obtenção do token, encadeando a causa (`raise ... from e`).
- O `id` do request fica estável entre tentativas (herdado da SPEC-2200).
- README: documentar as novas variáveis e a semântica de retry.

**FORA:**
- Retry de `INVALID_ARGUMENT`, `PERMISSION_DENIED` e `UNAUTHENTICATED` (este já tem caminho próprio).
- Retry no `DiodeOTLPClient`: ele já tem `timeout` (`client.py:703`). Retry fica fora.
- Circuit breaker.

## Invariantes
- SEMPRE: toda chamada de rede do `DiodeClient` tem um deadline finito.
- NUNCA: repetir erros não transitórios (`INVALID_ARGUMENT`, `PERMISSION_DENIED`, HTTP 4xx exceto 429).
- SEMPRE: o tempo total de `ingest()` é limitado por `max_retries × (timeout + max_backoff)`.

## Implementação

Referências **contra `upstream/develop`** (`git show upstream/develop:netboxlabs/diode/sdk/client.py`), porque a SPEC é implementada depois do sync 2131 e o #114 reescreveu `client.py`.

- `client.py:391-404` (`__init__`): novos parâmetros `timeout`, `max_retries`, `auth_timeout`, lidos com a precedência corrigida da SPEC-2202. Se a 2202 ainda não estiver feita, ler com um helper novo `_resolve_int(param, env, default)`.
- `client.py:549-573` (`ingest`): laço único com classificação:
  - `UNAUTHENTICATED` → reautentica (caminho atual).
  - `UNAVAILABLE`/`RESOURCE_EXHAUSTED` → `self._sleep(_backoff(attempt))` e repete.
  - Demais erros → `DiodeClientError`.
  - `self._stub.Ingest(request, metadata=self._metadata, timeout=self._timeout)` (linha 563).
- `client.py:1016-1025` (`_DiodeAuthentication.authenticate`): `except requests.RequestException as e: raise DiodeConfigError(...) from e`. Hoje a causa não é encadeada.
- `client.py:1069-1102` (`_request_access_token`): `session.post(url, data=..., headers=..., timeout=self._timeout)` (linha 1070). Capturar `requests.ConnectionError`/`requests.Timeout` dentro do laço e tratar como retriable com o mesmo `_auth_retry_delay`.
- `client.py:1124` (`_is_retriable_auth_http_status`): reaproveitar. Criar `_is_retriable_grpc_code(code)` irmão.
- Para o jitter: `_auth_retry_delay` hoje é determinístico. Adicionar jitter total (`random.uniform(0, delay)`) atrás de um `rng` injetável para os testes.
- Testes `tests/test_client.py` com fakes (sem rede):
  - número de tentativas por código;
  - delays via `sleep` injetado;
  - ausência de retry em `INVALID_ARGUMENT`;
  - timeout propagado ao stub;
  - auth com servidor "pendurado" (`requests` mock levantando `Timeout`) termina em `DiodeConfigError` dentro do prazo.

### Modelo de dados
—

<!-- Alternativas: (a) `retryPolicy` no service config do gRPC (`grpc.service_config`) — transparente e eficiente, mas não reautentica nem expõe hooks/logs e interage mal com o tunnel skip-verify do #114; rejeitada como mecanismo único (pode ser complementar). (b) dependência `tenacity` — rejeitada para não ampliar o grafo de dependências de uma lib. -->

## Riscos
- Retry de `UNAVAILABLE` depois de o servidor ter recebido o lote gera ingestão duplicada. Mitigação: o reconciler deduplica por hash de entidade (`diode-server/entityhash`), e o `id` estável permite dedup futura por request.
- Mudar o default de "sem timeout" para 30 s pode quebrar lotes muito grandes em links lentos. Mitigação: documentar o env e usar `create_message_chunks` (limite de 3 MB).

## Sinais de sucesso
- Rolling update do ingester no k8s não gera erro visível para produtores. Nenhum processo produtor fica pendurado indefinidamente.

## Critério de aceite
- [ ] `Ingest` sempre chamado com `timeout` e configurável por parâmetro/env | verify: `pytest -q tests/test_client.py -k "timeout"`
- [ ] `UNAVAILABLE`/`RESOURCE_EXHAUSTED` repetidos com backoff + jitter, e `INVALID_ARGUMENT` não repetido | verify: `pytest -q tests/test_client.py -k "transport_retry"`
- [ ] Token: `session.post` com timeout e `ConnectionError`/`Timeout` repetidos, com causa encadeada | verify: `pytest -q tests/test_client.py -k "auth_timeout or auth_connection_error"`
- [ ] README documenta `DIODE_INGEST_TIMEOUT`, `DIODE_AUTH_TIMEOUT` e `DIODE_MAX_RETRIES` | evidence: manual @aliothbl
- [ ] Suíte completa verde | verify: `pytest -q`
