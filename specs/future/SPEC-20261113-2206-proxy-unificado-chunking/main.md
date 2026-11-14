# SPEC-20261113-2206: Resolução de proxy unificada (gRPC × auth × OTLP) e estimativa de chunk exata

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:06
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** proxy, HTTP_PROXY, HTTPS_PROXY, NO_PROXY, IPv6, requests, proxies, ssl_target_name_override, chunking, varint, framing, oversize-entity, canal
**Features:** client-transport
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** baixo · **Complexidade:** média
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achados S-9 e S-15).
**Resumo:** Depois do #114 sobram três inconsistências: o gRPC usa um `NO_PROXY` próprio enquanto o token OAuth2 usa a semântica do `requests`; o parse de host quebra com IPv6 (`split(":")`); e `DiodeClient` e `DiodeOTLPClient` montam o canal com regras diferentes (`ssl_target_name_override` só no OTLP). No chunking, a estimativa ignora o framing de cada entidade e uma entidade maior que o limite passa sem aviso.

## Objetivo

Garantir que **uma** decisão de proxy valha para todo o tráfego do cliente (gRPC e `/auth/token`) e que hosts IPv6 funcionem. Também tornar `create_message_chunks` exato o bastante para nunca produzir um chunk acima do limite do gRPC (4 MB) quando o usuário confia no default de 3 MB.

**O que o #114 já resolveu** (não refazer):
- a escolha do proxy passou a seguir o esquema (`self._secure`), e não `tls_verify`;
- as credenciais do proxy são redigidas nos logs (`_redact_proxy_url`);
- com skip-verify, o proxy é tratado dentro do tunnel.

## Escopo

**DENTRO:**
- `_ProxyDecision = resolve_proxy(authority, secure)` como função única, usada por:
  - (a) `_open_grpc_channel`;
  - (b) `_DiodeAuthentication`, passando `session.trust_env = False` e `session.proxies = {...}` explícitos, para que o `requests` não reaplique uma semântica própria de `NO_PROXY`;
  - (c) `DiodeOTLPClient`.
- Parse de authority com suporte a IPv6 (`[::1]:443`) via `urllib.parse.urlsplit("//" + authority)` em `_should_bypass_proxy` e no OTLP.
- Extrair a montagem de canal duplicada entre `DiodeClient.__init__` e `DiodeOTLPClient.__init__` para um helper comum. Decidir com teste se `grpc.ssl_target_name_override` é necessário (o CONNECT do gRPC preserva o SNI do target) e aplicar a mesma regra nos dois clientes.
- Chunking:
  - somar o framing de campo repetido por entidade (`1 + _VarintSize(entity_size)`) em `estimate_message_size` e no laço de `create_message_chunks`;
  - `_LOGGER.warning` (ou `ValueError` opcional via `strict=True`) quando uma entidade sozinha excede o limite.

**FORA:**
- Suporte a proxy SOCKS.
- Mudar o default de 3 MB.

## Invariantes
- SEMPRE: o gRPC e o endpoint de token tomam a mesma decisão de usar ou não o proxy para o mesmo host.
- NUNCA: um chunk produzido por `create_message_chunks` com o limite L serializa acima de L, exceto quando uma única entidade já excede L, e então isso é sinalizado.

## Implementação

Referências **contra `upstream/develop`**, porque a SPEC é implementada depois do sync 2131.

- `client.py:180-186` (`_get_proxy_env_var`), `219-289` (`_matches_no_proxy_entry`/`_should_bypass_proxy`) e `292-325` (`_get_grpc_proxy_url`): consolidar em `resolve_proxy`. Corrigir `client.py:261` (`host = target_host.split(":")[0]`), que quebra IPv6.
- `client.py:1037-1053` (`_configure_auth_session`): `session.trust_env = False` e `session.proxies` a partir de `resolve_proxy(self._target, self._secure)`.
- `client.py:735-740` (OTLP, `ssl_target_name_override` com `split(":")[0]`) e `client.py:450-465` (DiodeClient): helper comum `_build_channel(...)`.
- `netboxlabs/diode/sdk/chunking.py:63-81` e `113-116`: incluir `_encoder._VarintSize`, ou um cálculo local equivalente para não depender de API privada, mais 1 byte de tag por entidade (o campo `entities` é `repeated Entity entities = 2`, com tag de 1 byte).
- Testes:
  - `tests/test_client.py` parametrizado com `NO_PROXY` × host IPv4/IPv6/hostname, mostrando gRPC e auth com a mesma decisão;
  - `tests/test_chunking.py`: `sum(len(IngestRequest(entities=c).SerializeToString()) <= L)` para cada chunk gerado sobre fixtures grandes, mais o aviso de entidade gigante.

### Modelo de dados
—

<!-- Alternativas: deixar o requests resolver proxy para tudo (urllib.request.getproxies + proxy_bypass) — rejeitada: semântica de NO_PROXY do Python difere da do Go (servidor/agentes), e o SDK documenta compatibilidade com Go. -->

## Riscos
- `trust_env=False` também desliga `.netrc` e `REQUESTS_CA_BUNDLE` no `requests`. Mitigação: só aplicar `proxies` explícito e manter `trust_env` (avaliar no teste qual mantém CA bundle) ou ler `REQUESTS_CA_BUNDLE` explicitamente.

## Sinais de sucesso
- Ambientes com proxy corporativo e `NO_PROXY` para o cluster passam a ter auth e gRPC pelo mesmo caminho, e nenhum `RESOURCE_EXHAUSTED: message larger than max` em lotes chunked.

## Critério de aceite
- [ ] Decisão de proxy idêntica entre gRPC, auth e OTLP para hosts IPv4, IPv6 e hostname ← specctl check
- [ ] Montagem de canal compartilhada entre `DiodeClient` e `DiodeOTLPClient` ← specctl check
- [ ] Todo chunk serializado ≤ limite (teste com fixture grande) e aviso para entidade acima do limite ← specctl check
