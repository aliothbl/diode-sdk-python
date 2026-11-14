# Feature: client-config

**Keywords:** env, DIODE_CLIENT_ID, DIODE_CLIENT_SECRET, DIODE_MAX_AUTH_RETRIES, DIODE_CERT_FILE, DIODE_SKIP_TLS_VERIFY, DIODE_SDK_LOG_LEVEL, DIODE_SENTRY_DSN, precedencia, logging, sentry, certifi
**Arquivos principais:**
  - netboxlabs/diode/sdk/client.py (helpers `_get_required_config_value`/`_get_optional_config_value`, `_load_certs`, `_setup_sentry`)
  - README.md (seções de variáveis e TLS)
**Resumo:** Como o SDK lê a configuração (parâmetros × variáveis de ambiente) e os efeitos colaterais de inicialização (logging, Sentry, origem dos CAs).

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2202-precedencia-config-dryrun | Regra única argumento > env > default | `DIODE_MAX_AUTH_RETRIES` nunca é lido
- SPEC-20261113-2204-sem-efeitos-globais-logging-sentry | Sem `basicConfig` e Sentry opt-in; CAs documentados corretamente | Biblioteca altera o estado global da aplicação hospedeira

## Estado atual
Referência `upstream/develop`.
- `_get_required_config_value`/`_get_optional_config_value` (`client.py:134-151`): o parâmetro vence quando não é `None`, e a env só é lida quando o parâmetro é `None`. Como `max_auth_retries` tem default `3` e é convertido em `str` antes do helper (`client.py:409-412`), a env nunca é lida.
- `logging.basicConfig(level=DIODE_SDK_LOG_LEVEL)` é chamado em `DiodeClient.__init__` (`client.py:406-407`) e em `DiodeOTLPClient.__init__` (`client.py:710`).
- `sentry_sdk.init` é global e acionado por `DIODE_SENTRY_DSN`, com `traces_sample_rate` e `profiles_sample_rate` 1.0 por default (`client.py:398-400`, `575-590`).
- CAs: `_load_certs(None)` retorna o bundle do `certifi` (`client.py:84-88`). Com verificação ativa, `certificates` nunca é `None`, então o ramo "system certificates" (`client.py:370`) é inalcançável. O README:210 diz "uses system certificates".

## Decisões arquiteturais ativas
- DEC-20261113-2202-param-vence-env [ativa] (SPEC-20261113-2202): a regra de precedência alvo é argumento explícito > env > default, já seguida por `_get_required_config_value`.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2202 | Defaults não-`None` em parâmetros desativam a leitura da env nos helpers atuais (2026-11-13). Use uma sentinela.
- SPEC-20261113-2204 | O README afirma "system certificates", mas o código usa o certifi (2026-11-13). CA corporativa exige `DIODE_CERT_FILE`.
