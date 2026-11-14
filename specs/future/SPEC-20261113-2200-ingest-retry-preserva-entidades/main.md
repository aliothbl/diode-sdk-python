# SPEC-20261113-2200: `ingest()` não pode perder entidades ao repetir após `UNAUTHENTICATED`

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:00
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** ingest, retry, unauthenticated, iterable, generator, perda-silenciosa, IngestRequest, request-id
**Features:** client-transport
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** alto · **Complexidade:** baixa
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achado S-2).
**Resumo:** Materializar `entities` uma única vez antes do laço de retry de `DiodeClient.ingest()`. Hoje um gerador é consumido na 1ª tentativa e a 2ª envia um `IngestRequest` vazio, que o servidor aceita com OK: perda silenciosa de dados.

## Objetivo

`DiodeClient.ingest()` aceita `Iterable[Entity]`, mas constrói o `IngestRequest(entities=entities)` **dentro** do laço de retry. Quando `entities` é um gerador ou iterador e a primeira chamada recebe `UNAUTHENTICATED`, o laço reautentica e envia uma segunda requisição sem nenhuma entidade. O ingester responde OK e o produtor acredita que ingeriu tudo. O objetivo é eliminar essa perda silenciosa sem mudar a API pública.

## Escopo

**DENTRO:**
- Construir o `IngestRequest` (incluindo `metadata`) **uma vez**, antes do laço, e reutilizar o mesmo objeto em todas as tentativas.
- Manter o mesmo `id` de request entre as tentativas: hoje é um `uuid4` por tentativa. Isso permite ao servidor deduplicar no futuro e correlacionar logs.
- Fazer o mesmo no `DiodeDryRunClient.ingest()` e no `DiodeOTLPClient.ingest()` se o mesmo padrão aparecer (no dry-run não há retry, mas o objeto é montado uma vez só).
- Testes de regressão com gerador mais `UNAUTHENTICATED` na 1ª tentativa.

**FORA:**
- Retry por `UNAVAILABLE`/`DEADLINE_EXCEEDED` → SPEC-20261113-2201.
- Correção da precedência de `DIODE_MAX_AUTH_RETRIES` → SPEC-20261113-2202.

## Invariantes
- SEMPRE: toda tentativa de `Ingest` carrega exatamente as mesmas entidades e o mesmo `id` da primeira.
- NUNCA: consumir o iterável do chamador mais de uma vez.

## Implementação

Referências de linha **contra `upstream/develop`** (`git show upstream/develop:netboxlabs/diode/sdk/client.py`), porque esta SPEC é implementada depois da SPEC de sync 2131. O #114 reescreve boa parte de `client.py`.

- `netboxlabs/diode/sdk/client.py:541-573` (`DiodeClient.ingest`): mover as linhas 551-562 (montagem do `IngestRequest` e do `metadata`) para antes do `for attempt in range(self._max_auth_retries)` (linha 549). Dentro do laço fica só `return self._stub.Ingest(request, metadata=self._metadata)` (linha 563). Como o protobuf copia o iterável para o campo repetido na construção, isso sozinho já resolve. Não é preciso `list(entities)` explícito, mas registre isso em comentário.
- `client.py:573`: trocar `raise RuntimeError("Max retries exceeded")` por um erro do SDK. Esse caminho só acontece com `max_auth_retries <= 0` (ver SPEC-2202), e a correção definitiva fica lá. Aqui basta não regredir.
- Testes em `tests/test_client.py`:
  - `test_ingest_retry_preserves_generator_entities`: stub falso cujo 1º `Ingest` levanta `RpcError(UNAUTHENTICATED)` e o 2º captura o request. Verificar `len(request.entities) == N`.
  - `test_ingest_retry_keeps_request_id`: o `id` é igual nas duas chamadas.

### Modelo de dados
—

<!-- Alternativas: (a) `entities = list(entities)` no topo — funciona, mas duplica memória (lista + campo repetido); montar o request uma vez evita a cópia extra. (b) documentar "não passe geradores" — rejeitada, a assinatura promete Iterable. -->

## Riscos
- Request com `id` estável pode, no futuro, ser deduplicado pelo servidor. Hoje o ingester não deduplica por `id`, então não há efeito colateral. Mitigação: registrar na feature `client-transport`.

## Sinais de sucesso
- Produtores que passam geradores (ex.: `yield` de inventário) não perdem lotes após rotação de token.

## Critério de aceite
- [ ] `IngestRequest` construído uma única vez em `DiodeClient.ingest()`, fora do laço de retry ← specctl check
- [ ] Teste com gerador + `UNAUTHENTICATED` na 1ª tentativa prova N entidades na 2ª chamada ← specctl check
- [ ] Teste prova `id` idêntico entre as tentativas ← specctl check
- [ ] PR aberto em netboxlabs/diode-sdk-python com o link registrado no journal ← specctl check
