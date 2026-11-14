# SPEC-20261113-2204: Remover efeitos colaterais globais (logging, Sentry) e corrigir a fonte de CAs

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:04
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** logging.basicConfig, sentry_sdk.init, traces_sample_rate, profiles_sample_rate, extras, certifi, system-certificates, DIODE_CERT_FILE, biblioteca
**Features:** client-config
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** médio · **Complexidade:** baixa
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achados S-7 e S-8).
**Resumo:** Uma biblioteca não deve configurar o root logger nem iniciar o Sentry global com amostragem 100 %. Também é preciso alinhar código e README sobre os certificados: hoje o SDK sempre usa o bundle do `certifi` e nunca os CAs do sistema, apesar do README:210 dizer o contrário.

## Objetivo

Instanciar o `DiodeClient` dentro de uma aplicação hoje (a) chama `logging.basicConfig`, alterando o logging da aplicação hospedeira, e (b) inicializa o hub global do Sentry com `traces_sample_rate=1.0` e `profiles_sample_rate=1.0` se `DIODE_SENTRY_DSN` existir, o que pode sobrescrever o Sentry da própria aplicação e gerar custo. Além disso, ambientes corporativos com CA interna instalada no sistema não funcionam sem `DIODE_CERT_FILE`, porque o SDK sempre carrega o `certifi`. O objetivo é que o SDK seja um "bom cidadão" de biblioteca e que a documentação de TLS bata com o código.

## Escopo

**DENTRO:**
- Remover `logging.basicConfig` de `DiodeClient.__init__` e de `DiodeOTLPClient.__init__`. Manter `DIODE_SDK_LOG_LEVEL` aplicado **só** ao logger `netboxlabs.diode.sdk` (`_LOGGER.setLevel`), com `NullHandler` no pacote.
- Sentry:
  - defaults de `sentry_traces_sample_rate`/`sentry_profiles_sample_rate` → `0.0`;
  - mover `sentry-sdk` para o extra `[sentry]` com import tardio e erro claro se o DSN for passado sem o extra;
  - não chamar `sentry_sdk.init` se já houver client ativo (`sentry_sdk.get_client().is_active()`), só adicionar tags no escopo.
- CAs: quando `cert_file` não é informado, usar `grpc.ssl_channel_credentials()` sem `root_certificates` (o gRPC usa o bundle default) **ou** manter o `certifi` e corrigir o README. A decisão fica para a ativação (ver Alternativas). O mesmo vale para o `requests` (`session.verify=True` usa o `certifi` do `requests`).
- README: corrigir a seção de TLS (README:210) e de logging.

**FORA:**
- Instrumentação OpenTelemetry própria do SDK.
- Mudança no comportamento de `DIODE_SKIP_TLS_VERIFY` (feito pelo #114).

## Invariantes
- NUNCA: importar ou instanciar o SDK altera handlers do root logger ou o client global do Sentry de uma aplicação que já o configurou.
- SEMPRE: a documentação de TLS descreve exatamente a origem dos CAs usada pelo código.

## Implementação

Referências **contra `upstream/develop`**, porque a SPEC é implementada depois do sync 2131.

- `client.py:406-407`: remover `basicConfig` e trocar por `_LOGGER.setLevel(...)` só se a env existir. O mesmo em `client.py:710`.
- `netboxlabs/diode/sdk/__init__.py`: `logging.getLogger("netboxlabs.diode.sdk").addHandler(logging.NullHandler())`.
- `client.py:398-400`: defaults `0.0`. `client.py:575-590` (`_setup_sentry`): import tardio e checagem de client ativo.
- `pyproject.toml:26-33`: `sentry-sdk` sai de `dependencies` e vai para `[project.optional-dependencies] sentry = ["sentry-sdk>=2.8.0"]`.
- `client.py:84-88` (`_load_certs`) e `client.py:427-431`: hoje `_load_certs(None)` retorna `certifi.where()`, então `certificates` nunca é `None` com `tls_verify=True` e o ramo `grpc.ssl_channel_credentials()` (linha 370) é inalcançável. Para usar os CAs do sistema, `_load_certs` retorna `None` quando não há `cert_file`, e `_configure_auth_session` (linhas 1037-1053) deixa `session.verify` no default.
- Testes:
  - `test_client_does_not_touch_root_logger` (`logging.getLogger().handlers` inalterado);
  - `test_sentry_not_reinitialized_when_active`;
  - teste de credenciais sem `root_certificates` quando `cert_file` é ausente (se a opção "CAs do sistema" for escolhida).

### Modelo de dados
—

<!-- Alternativas (CAs): (a) manter certifi e só corrigir README — zero risco de regressão, mas CA corporativa continua exigindo DIODE_CERT_FILE; (b) usar default do gRPC/requests — o default do grpcio é o bundle embutido (não o store do SO) em muitas plataformas, então "system certificates" pode continuar falso; avaliar `truststore` como opção (c). Decisão na ativação, registrada como [decisão] no journal. -->

## Riscos
- Remover o `sentry-sdk` das dependências é breaking para quem passa `sentry_dsn` sem instalar o extra. Mitigação: erro explícito com instrução `pip install netboxlabs-diode-sdk[sentry]` e nota de release.
- Aplicações que dependiam do `basicConfig` implícito deixam de ver logs do SDK. Mitigação: documentar.

## Sinais de sucesso
- Integrações (ex.: orb-agent) param de ver logging duplicado e de ter o Sentry delas sobrescrito.

## Critério de aceite
- [ ] Instanciar `DiodeClient`/`DiodeOTLPClient` não altera handlers do root logger ← specctl check
- [ ] Sentry com sample rates default 0.0, extra `[sentry]`, sem reinit quando já ativo ← specctl check
- [ ] Origem dos CAs decidida, implementada e documentada no README (seção TLS) ← specctl check
