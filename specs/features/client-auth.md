# Feature: client-auth

**Keywords:** oauth2, client_credentials, diode:ingest, /auth/token, hydra, bearer, token, retry, Retry-After, expires_in, requests.Session, verify
**Arquivos principais:**
  - netboxlabs/diode/sdk/client.py (`_DiodeAuthentication`, `DiodeClient._authenticate`)
**Resumo:** Obtenção do token OAuth2 (client credentials, escopo `diode:ingest`) no endpoint `{scheme}://{authority}{path}/auth/token` do diode-auth (fachada sobre o Ory Hydra) e injeção como `authorization: Bearer` na metadata do gRPC.

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2201-deadlines-retry-transporte | Timeout no `session.post` e retry de `ConnectionError`/`Timeout` | O construtor pode travar indefinidamente
- SPEC-20261113-2203-auth-thread-safe-renovacao | `_TokenManager` com lock e renovação proativa | Rajada de N fetches e escrita concorrente de `_metadata`

## Estado atual
Referência `upstream/develop`.
- `DiodeClient.__init__` autentica de forma síncrona (`client.py:448`). Credenciais ausentes geram `DiodeConfigError` (`_get_required_config_value`, `client.py:134-142`).
- `_DiodeAuthentication._request_access_token` (`client.py:1055-1102`) repete os HTTP status retriáveis (429, 500, 502, 503) com backoff exponencial e respeita `Retry-After`. O `session.post` não tem timeout. `RequestException` vira `DiodeConfigError` sem `from e` (`client.py:1024-1025`).
- O esquema da URL de auth segue o esquema do target (`_secure`), independentemente da verificação TLS (`client.py:1110-1121`, #114).
- O token não tem rastreamento de expiração. A renovação é reativa, em `UNAUTHENTICATED` (`client.py:565-571`), e reatribui `self._metadata` sem lock (`client.py:610-612`).

## Decisões arquiteturais ativas
- DEC-20261113-2203-auth-sincrona-no-construtor [ativa] (SPEC-20261113-2203): o construtor obtém o token (fail-fast para credenciais inválidas). Trade-off: o construtor faz I/O de rede, o que torna necessário um timeout (SPEC-2201).
- DEC-20261113-2203-renovacao-reativa [ativa → a substituir pela SPEC-20261113-2203]: o token só é renovado após `UNAUTHENTICATED`.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2203 | `DIODE_MAX_AUTH_RETRIES` limita **ao mesmo tempo** as tentativas HTTP do token e as reautenticações do gRPC (2026-11-13). Separar os limites ao introduzir o retry de transporte (SPEC-2201).
