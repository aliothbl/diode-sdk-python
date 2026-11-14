# Feature: codegen

**Keywords:** ingester.py, ingester_pb2, ingester_pb2_grpc, validate_pb2, buf, protoc, gen-diode-sdk-python, wrappers, PRIMARY_VALUE_MAP, NetBox-v4.7.0, drift
**Arquivos principais:**
  - netboxlabs/diode/sdk/ingester.py (gerado; 110 classes wrapper)
  - netboxlabs/diode/sdk/diode/v1/ingester_pb2.py, .pyi, _grpc.py (gerados)
  - netboxlabs/diode/sdk/validate/ (gerado)
  - ../diode/Makefile (alvo `gen-diode-sdk-python`)
**Resumo:** Código gerado do SDK: os stubs protobuf e gRPC vêm do `diode-proto` (repositório `diode`, via buf), e os wrappers ergonômicos `ingester.py` vêm de um gerador da NetBox Labs que não está versionado em nenhum dos três repositórios.

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2208-verificacao-drift-codegen | Teste wrapper × descriptor e job de regeneração no CI | O gerador de wrappers está fora do repositório; o drift só é evitado à mão
- SPEC-20261113-2205-tipagem-excecoes-dependencias | Tipos reais de retorno dos wrappers (contrato) | O `__new__` devolve `pb.X`, mas o type checker infere a classe wrapper

## Estado atual
- `ingester.py` traz o cabeçalho "Generated code. DO NOT EDIT. Source: NetBox v4.7.0. Timestamp: 2026-09-02 20:52:38Z" e tem 110 classes. Cada classe é um `class X: def __new__(cls, ...) -> pb.X`, que aceita atalhos de string via `PRIMARY_VALUE_MAP`, além de `metadata` como `dict`, convertido em `Struct`.
- `ingester_pb2*` são gerados pelo buf (protobuf-python v26.1, grpc-python v1.62.1) por `diode/Makefile:6-11`, com um `sed` que reescreve os imports para `netboxlabs.diode.sdk.*`.
- Em 2026-11-13 não havia drift entre `ingester.proto` e o `.pyi` (112 messages conferidos pelo diagnóstico).

## Decisões arquiteturais ativas
- DEC-20261113-2208-proto-no-repo-diode [ativa] (SPEC-20261113-2208, herdada): o schema é dono do repositório `diode` (`diode-proto/`), e o SDK só consome os artefatos gerados. Trade-off: duas etapas de geração e dois repositórios para manter em sincronia.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2208 | Nunca editar `ingester.py` nem `*_pb2*.py` à mão (2026-11-13). Sem o gerador, a correção some na próxima regeneração upstream. (relacionado: diode-netbox-plugin SPEC-20261113-2218)
