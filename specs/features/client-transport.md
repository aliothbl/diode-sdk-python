# Feature: client-transport

**Keywords:** grpc, channel, Ingest, IngestRequest, retry, timeout, keepalive, proxy, NO_PROXY, tls, skip-verify, SkipVerifyTunnel, chunking, DiodeMethodClientInterceptor, exceptions
**Arquivos principais:**
  - netboxlabs/diode/sdk/client.py
  - netboxlabs/diode/sdk/_skip_verify_tunnel.py (a partir do #114)
  - netboxlabs/diode/sdk/chunking.py
  - netboxlabs/diode/sdk/exceptions.py
  - netboxlabs/diode/sdk/diode/v1/ingester_pb2_grpc.py
**Resumo:** Camada de transporte do SDK: abertura do canal gRPC (TLS, skip-verify, proxy, keepalive, subpath), chamada unária `/diode.v1.IngesterService/Ingest`, retry, erros e divisão de lotes em chunks abaixo do limite de 4 MB do gRPC.

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2200-ingest-retry-preserva-entidades | Montar o `IngestRequest` uma vez antes do retry | Gerador consumido na 1ª tentativa → 2ª envia lote vazio (perda silenciosa)
- SPEC-20261113-2201-deadlines-retry-transporte | Timeouts e retry com backoff + jitter | Nenhuma chamada tem deadline; só `UNAUTHENTICATED` é repetido
- SPEC-20261113-2205-tipagem-excecoes-dependencias | `DiodeClientError` sob `BaseError`, mypy, deps | `str(err)` vazio; `py.typed` sem verificação
- SPEC-20261113-2206-proxy-unificado-chunking | Proxy único gRPC/auth/OTLP, IPv6, chunk exato | Semânticas de `NO_PROXY` divergentes; framing ignorado

## Estado atual
Referência `upstream/develop`, que é o estado após a SPEC-20261113-2131.
- O canal é aberto por `_open_grpc_channel` (`client.py:337-377`) e há três caminhos:
  - (a) `grpcs://` com verificação desligada: `insecure_channel` para um `SkipVerifyTunnel` local, que faz o TLS sem verificar;
  - (b) `grpc://`: `insecure_channel`, com `grpc.http_proxy` se houver proxy;
  - (c) `grpcs://`: `secure_channel` com `root_certificates` do certifi ou de `DIODE_CERT_FILE`.
- Keepalive de 30 s/10 s, alinhado à política do servidor (`client.py:58-65`).
- Um subpath no target vira prefixo do método via `DiodeMethodClientInterceptor` (`client.py:1189`).
- `ingest()` (`client.py:541-573`) só repete `UNAUTHENTICATED`, reautenticando. Os demais erros viram `DiodeClientError(err, hint=...)`. Não há timeout.
- `create_message_chunks` (`chunking.py:16-87`) faz bin-packing guloso com o default de 3 MB e materializa o iterável em lista.

## Decisões arquiteturais ativas
- DEC-20261113-2200-ingest-unario [ativa] (SPEC-20261113-2200): a ingestão é um RPC unário por lote. Não há streaming, e o tamanho é controlado pelo chamador via `create_message_chunks`. Trade-off: simplicidade e retry por lote, em troca de exigir chunking explícito.
- DEC-20261113-2200-skip-verify-tunnel [ativa] (SPEC-20261113-2131, herdada do upstream #114): o grpcio não permite desligar a verificação, então o SDK interpõe um tunnel TLS local (Unix socket em diretório `0700`). Trade-off: uma thread e um event loop por cliente, e ressalvas com Windows e `fork()`.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2200 | `IngestRequest(entities=gerador)` consome o gerador (2026-11-13). Qualquer laço de retry precisa reutilizar o request montado, nunca remontá-lo.
- SPEC-20261113-2206 | O `split(":")` usado para extrair o host quebra com IPv6 (`client.py:261`, `client.py:738`) (2026-11-13).
- SPEC-20261113-2201 | Repetir `UNAVAILABLE` pode duplicar a ingestão se o servidor já recebeu o lote (2026-11-13). O reconciler deduplica por hash de entidade, mas não por `id` de request.
