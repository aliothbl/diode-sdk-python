# TAXONOMY.md — vocabulário canônico de áreas

> Toda feature de `specs/features/` DEVE corresponder a uma área daqui (R.4/R.13). Área nova = confirmação explícita do usuário.

## Áreas

- upstream-sync — Sincronização do fork com netboxlabs/diode-sdk-python (develop, release, tags)
- client-transport — Canal gRPC, TLS/skip-verify, proxy, retries, deadlines, chunking
- client-auth — OAuth2 client credentials contra diode-auth, renovação de token
- client-config — Precedência parâmetro/env, logging e Sentry do SDK
- dry-run — DiodeDryRunClient, load_dryrun_entities e diode-replay-dryrun
- packaging-ci — pyproject, dependências, tipagem distribuída, CI e release no PyPI
- codegen — Código gerado (ingester_pb2, wrappers ingester.py) e verificação de drift
