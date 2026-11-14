# SPEC-20261113-2202: Precedência de configuração (parâmetro × env) e correções do dry-run

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:02
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** DIODE_MAX_AUTH_RETRIES, DIODE_DRY_RUN_OUTPUT_DIR, precedencia, config, env, dry-run, load_dryrun_entities, producer_app_version, perf_counter_ns, replay
**Features:** client-config, dry-run
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** médio · **Complexidade:** baixa
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achados S-3 e S-12).
**Resumo:** `DIODE_MAX_AUTH_RETRIES` nunca é lido (o parâmetro com default sempre vence), `max_auth_retries=0` termina em `RuntimeError`, e no dry-run a env sobrepõe o argumento explícito. O dry-run também omite `producer_app_version`, gera nomes de arquivo colidíveis e o replay perde `stream`/`metadata`.

## Objetivo

Deixar a regra de configuração **única e documentada**: argumento explícito > variável de ambiente > default. Hoje cada campo segue uma regra diferente e uma variável documentada no README (`DIODE_MAX_AUTH_RETRIES`, README:30) não tem efeito. No dry-run, o arquivo gerado precisa ser fiel ao que o `DiodeClient` enviaria, para que o `diode-replay-dryrun` reproduza a ingestão sem perdas.

## Escopo

**DENTRO:**
- Helper `_resolve(param, env_name, default, cast)` com sentinela `_UNSET` para distinguir "não passado" de "passado com valor default". Aplicar a `max_auth_retries`, `cert_file` e `output_dir`.
- Validar `max_auth_retries >= 1`. Valores inválidos (não-int, ≤0) → `DiodeConfigError` com mensagem clara.
- `DiodeDryRunClient.__init__`:
  - argumento explícito vence a env;
  - aceitar `app_version` e preencher `producer_app_version` no request.
- Nome de arquivo do dry-run: `f"{app}_{time.time_ns()}_{uuid4().hex[:8]}.json"`, ordenável e único entre processos.
- `load_dryrun_entities`:
  - corrigir a docstring ("concatenated JSON" ≠ comportamento);
  - expor `load_dryrun_request(path) -> IngestRequest` para o replay preservar `stream` e `metadata`.
- `scripts/dryrun_replay.py`: usar `load_dryrun_request` e repassar `stream` e `metadata` ao `ingest()`.

**FORA:**
- Novas variáveis de timeout/retry → SPEC-2201.
- Mudança de formato do arquivo de dry-run (continua `MessageToJson` de um `IngestRequest`).

## Invariantes
- SEMPRE: argumento explícito > env > default, em todos os campos configuráveis do SDK.
- NUNCA: valor de config inválido vira comportamento silencioso (ex.: zero tentativas).

## Implementação

Referências **contra `upstream/develop`**, porque a SPEC é implementada depois do sync 2131.

- `client.py:145-151` (`_get_optional_config_value`): manter para compatibilidade e adicionar `_resolve`.
- `client.py:409-412`: hoje `_get_optional_config_value(_MAX_RETRIES_ENVVAR_NAME, str(max_auth_retries))` só lê a env quando o valor é `None`, e `str(3)` nunca é `None`. Trocar a assinatura para `max_auth_retries: int | None = None` e usar `_resolve(max_auth_retries, "DIODE_MAX_AUTH_RETRIES", 3, int)` + validação.
- `client.py:573`: `RuntimeError("Max retries exceeded")` deixa de ser alcançável após a validação. Manter como `DiodeClientError`/`AssertionError` defensivo.
- `client.py:623-626` (`DiodeDryRunClient.__init__`): `os.getenv(ENV, output_dir)` faz a env vencer. Inverter para `_resolve(output_dir, ENV, None, str)`.
- `client.py:663-670`: incluir `producer_app_version=self._app_version`.
- `client.py:677`: `time.perf_counter_ns()` → `time.time_ns()` + sufixo `uuid4`.
- `client.py:68-75` (`load_dryrun_entities`): docstring e nova função `load_dryrun_request`.
- `netboxlabs/diode/scripts/dryrun_replay.py`: usar o request completo.
- Testes: matriz parametrizada (parâmetro × env × inválido) em `tests/test_client.py`; teste de replay preservando `metadata`.

### Modelo de dados
— (formato do JSON de dry-run inalterado; só passa a conter `producer_app_version`).

<!-- Alternativas: env sempre vence (12-factor) — rejeitada: contradiz o restante do SDK (`_get_required_config_value` já faz param > env) e surpreende quem passa o argumento explicitamente. -->

## Riscos
- Quem hoje depende de `DIODE_DRY_RUN_OUTPUT_DIR` sobrepondo o argumento muda de comportamento. Mitigação: nota de breaking change no PR, já que o comportamento atual contradiz o padrão do SDK.

## Sinais de sucesso
- `DIODE_MAX_AUTH_RETRIES=5` produz 5 tentativas, e um replay de dry-run reproduz o request original byte a byte (exceto o `id`).

## Critério de aceite
- [ ] `DIODE_MAX_AUTH_RETRIES` respeitado quando o parâmetro não é passado; parâmetro explícito vence ← specctl check
- [ ] `max_auth_retries` ≤ 0 ou não numérico → `DiodeConfigError` ← specctl check
- [ ] Dry-run: argumento `output_dir` vence a env; `producer_app_version` presente; nomes de arquivo únicos entre processos ← specctl check
- [ ] `diode-replay-dryrun` preserva `stream` e `metadata` (teste) ← specctl check
