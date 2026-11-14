# SPEC-20261113-2203: Autenticação thread-safe com renovação proativa do token

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:03
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** oauth2, client_credentials, token, expires_in, lock, thread-safety, thundering-herd, renovacao-proativa, metadata
**Features:** client-auth
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** médio · **Complexidade:** média
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achado S-6).
**Resumo:** Hoje o token só é renovado reativamente (após `UNAUTHENTICATED`) e sem lock: N threads compartilhando um `DiodeClient` disparam N requisições de token e reatribuem `self._metadata` concorrentemente. Introduzir um `_TokenManager` com lock e renovação antes do `expires_in`.

## Objetivo

Produtores multithread (ex.: coletores com `ThreadPoolExecutor`) compartilham um `DiodeClient`. Quando o token expira, todas as threads recebem `UNAUTHENTICATED` ao mesmo tempo, cada uma chama `_authenticate`, o Hydra recebe uma rajada e `self._metadata` é sobrescrito sem sincronização. O objetivo é ter **uma** renovação por expiração, feita antes de o token vencer, e uma leitura de metadata consistente por chamada.

## Escopo

**DENTRO:**
- `_TokenManager` interno com:
  - `threading.Lock`;
  - `token`, `expires_at` (de `expires_in` da resposta OAuth2) e `skew` (default 30 s ou 10 % do TTL);
  - `get()`, que renova com *double-checked locking* quando `now >= expires_at - skew`;
  - `invalidate(old_token)`, que só força renovação se o token atual ainda for o que falhou (evita renovar N vezes).
- `_DiodeAuthentication._request_access_token` passa a retornar também `expires_in` (opcional na resposta; sem ele, comportamento reativo atual).
- `DiodeClient.ingest()` monta a metadata por chamada a partir de `token_manager.get()`, sem mutar estado compartilhado.
- Testes de concorrência: 50 threads com o token expirando, 1 fetch.

**FORA:**
- Cache de token entre processos/instâncias.
- Mudanças no `DiodeOTLPClient` (não usa OAuth2).

## Invariantes
- SEMPRE: no máximo uma requisição de token em voo por `DiodeClient`.
- NUNCA: uma chamada `Ingest` envia metadata parcialmente atualizada (lista mutada por outra thread).
- SEMPRE: `UNAUTHENTICATED` continua sendo tratado (rede de segurança se o relógio local divergir).

## Implementação

Referências **contra `upstream/develop`**, porque a SPEC é implementada depois do sync 2131.

- `client.py:443-448`: `self._metadata` vira metadata **base**, uma tupla imutável sem `authorization`. O construtor cria `self._tokens = _TokenManager(self._new_auth_client)` e chama `self._tokens.get()` (preserva o fail-fast atual de credenciais inválidas no construtor).
- `client.py:563`: `self._stub.Ingest(request, metadata=self._metadata + (("authorization", f"Bearer {self._tokens.get()}"),), ...)`.
- `client.py:565-571`: em `UNAUTHENTICATED`, `self._tokens.invalidate(token_usado)` e repetir.
- `client.py:592-612` (`_authenticate`): vira fábrica `_new_auth_client(scope)` usada pelo manager. Remover a mutação de `self._metadata` (linhas 610-612).
- `client.py:1072-1079`: retornar `(access_token, response.json().get("expires_in"))`. Manter `authenticate() -> str` para compatibilidade e adicionar `authenticate_with_expiry()`.
- Relógio: `time.monotonic()` injetável no manager para teste determinístico.
- Testes novos em `tests/test_token_manager.py`:
  - expiração com skew;
  - double-check (mock do auth com `Event` para forçar a corrida);
  - `invalidate` idempotente;
  - fallback sem `expires_in`.

### Modelo de dados
— (estado só em memória: `token`, `expires_at`).

<!-- Alternativas: (a) só um lock em volta do `_authenticate` atual — evita a corrida de escrita mas mantém a rajada reativa a cada expiração; rejeitada. (b) `grpc.metadata_call_credentials` com plugin de auth — idiomático, mas roda em thread do gRPC core e complica o tunnel skip-verify do #114 e o tratamento de erro; reavaliar depois. -->

## Riscos
- `expires_in` ausente ou errado no servidor. Mitigação: fallback reativo mantido, com skew mínimo.
- Deadlock se `get()` for chamado dentro do callback de auth. Mitigação: o lock não é reentrante e o auth não chama o manager. Coberto por teste.

## Sinais de sucesso
- Métrica do Hydra mostra 1 token por cliente por janela de TTL, e produtores multithread deixam de logar "Retrying ingestion due to UNAUTHENTICATED".

## Critério de aceite
- [ ] 50 threads com token expirado geram exatamente 1 fetch de token | verify: `pytest -q tests/test_token_manager.py -k concurrent`
- [ ] Renovação proativa antes de `expires_in - skew` | verify: `pytest -q tests/test_token_manager.py -k proactive`
- [ ] Sem `expires_in`, comportamento reativo preservado | verify: `pytest -q tests/test_token_manager.py -k fallback`
- [ ] `self._metadata` não é mais mutado após o construtor | verify: `pytest -q tests/test_client.py`
