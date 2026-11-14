# Feature: dry-run

**Keywords:** DiodeDryRunClient, DIODE_DRY_RUN_OUTPUT_DIR, MessageToJson, load_dryrun_entities, diode-replay-dryrun, replay
**Arquivos principais:**
  - netboxlabs/diode/sdk/client.py (`DiodeDryRunClient`, `load_dryrun_entities`)
  - netboxlabs/diode/scripts/dryrun_replay.py
**Resumo:** Modo sem rede. O SDK serializa o `IngestRequest` em JSON (stdout ou um arquivo por chamada) e o CLI `diode-replay-dryrun` reenvia esses arquivos a um servidor real.

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2202-precedencia-config-dryrun | Argumento vence a env, `producer_app_version`, nomes únicos, replay fiel | Replay perde `stream`/`metadata`

## Estado atual
Referência `upstream/develop`.
- `DiodeDryRunClient.__init__` usa `os.getenv(DIODE_DRY_RUN_OUTPUT_DIR, output_dir)` (`client.py:623-626`), então a env vence o argumento.
- `ingest()` monta o request sem `producer_app_version` (`client.py:663-670`).
- O nome do arquivo é `<app>_<perf_counter_ns>.json` (`client.py:677-683`). `perf_counter_ns` não é único entre processos nem ordenável no tempo.
- `load_dryrun_entities` (`client.py:68-75`) faz um único `json.load`, apesar de a docstring falar em "concatenated JSON", e devolve só as entidades.

## Decisões arquiteturais ativas
- DEC-20261113-2202-arquivo-por-request [ativa] (SPEC-20261113-2202): cada `ingest()` gera um arquivo JSON com um `IngestRequest` completo. Trade-off: muitos arquivos pequenos, em troca de um replay trivial.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2202 | O replay atual descarta `stream` e `metadata` do request original (2026-11-13).
