# SPEC-20261113-2205: Hierarquia de exceções, tipagem verificável e dependências declaradas corretamente

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:05
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** DiodeClientError, BaseError, RpcError, Protocol, mypy, py.typed, wrappers, __new__, protobuf, setuptools, PEP-621, python-3.12, python-3.13, python-3.14, lowest-direct
**Features:** packaging-ci, client-transport
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** médio · **Complexidade:** média
**Estratégia upstream:** upstream-first. Abrir PR em netboxlabs/diode-sdk-python e carregar no fork só até o merge. A parte dos wrappers depende do gerador (SPEC-2208).
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achados S-10, S-11 e S-13).
**Resumo:** `DiodeClientError` não herda de `BaseError` e tem `str()` vazio. O pacote distribui `py.typed` sem nenhum verificador de tipos no CI, e os wrappers `ingester.py` mentem para o type checker. O `pyproject` declara `setuptools>=43` (inválido para PEP 621), não declara `protobuf`, que o gencode exige, e não testa Python 3.12+.

## Objetivo

Dar aos consumidores do SDK três garantias hoje ausentes:
1. Um único `except BaseError` captura qualquer erro do SDK, e as mensagens são legíveis.
2. Os tipos distribuídos (`py.typed`) são verificados no CI e correspondem ao runtime.
3. O pacote instala e funciona com as versões mínimas e máximas declaradas.

## Escopo

**DENTRO:**
- Exceções (`exceptions.py`):
  - `DiodeClientError(BaseError, RpcError)` com `super().__init__(mensagem)`, incluindo status e detalhes;
  - manter as propriedades `status_code`/`details`;
  - `__str__` útil.
- `DiodeClientInterface` (`client.py:78-81`, classe vazia) vira um `typing.Protocol` com `ingest(...)`, `__enter__` e `__exit__`. Os três clientes continuam herdando, porque um `Protocol` pode ser herdado explicitamente.
- Tipagem:
  - `sentry_dsn: str | None = None`;
  - mypy (modo `strict` só em `client.py`, `chunking.py` e `exceptions.py`; demais módulos em modo básico) no CI.
- Wrappers (`ingester.py`, gerado): declarar o retorno real (`pb.Device` etc.) via stub `.pyi` gerado, ou converter as classes em funções-fábrica. A implementação fica com o gerador (SPEC-2208). Aqui entram só o contrato e o teste de tipo.
- Dependências (`pyproject.toml`):
  - `protobuf>=5.26.1,<7` (gencode "Protobuf Python Version: 5.26.1");
  - `setuptools>=61` no `build-system`;
  - remover `black` do extra `dev` (não é usado; o ruff formata);
  - classifiers 3.12, 3.13 e 3.14.
- CI: matriz Python 3.10 a 3.14 e um job `lowest-direct` (`uv pip install --resolution lowest-direct .[test]`).

**FORA:**
- Endurecimento do pipeline de release → SPEC-2207.
- Reescrita do gerador de wrappers → SPEC-2208.

## Invariantes
- SEMPRE: toda exceção levantada intencionalmente pelo SDK é instância de `netboxlabs.diode.sdk.exceptions.BaseError`.
- SEMPRE: `DiodeClientError` continua sendo `grpc.RpcError` (compatibilidade com quem captura `RpcError`).
- NUNCA: o `pyproject` declara um piso de dependência que não foi testado no CI.

## Implementação

Referências **contra `upstream/develop`**, porque a SPEC é implementada depois do sync 2131.

- `exceptions.py:23-47` (`DiodeClientError`): hoje herda só de `RpcError` e o `__init__` (linhas 30-33) não chama `super().__init__`. Mudar para `class DiodeClientError(BaseError, RpcError)` com `super().__init__(f"{code.name}: {details}")`. Seguir o modelo do `OTLPClientError` (linhas 50-86), que já monta a mensagem.
- `client.py:78-81`: `class DiodeClientInterface(Protocol)`.
- `client.py:398`: `sentry_dsn: str | None = None`.
- `pyproject.toml:15-33,66-68`: classifiers, `protobuf` e `setuptools>=61`.
- `pyproject.toml:35-37`: `dev = ["check-manifest", "ruff", "mypy", "types-requests", "grpc-stubs"]` (avaliar `grpc-stubs` × `types-grpcio`).
- Novo `[tool.mypy]` no `pyproject` com `exclude` dos pacotes gerados (`diode/`, `validate/`).
- `.github/workflows/lint-tests.yml:24`: matriz `["3.10","3.11","3.12","3.13","3.14"]`, mais o job `lowest` e o step `mypy`.
- Teste de tipo para os wrappers: `tests/typing/test_wrappers.py` com `assert_type(Device(name="x"), ingester_pb2.Device)` executado pelo mypy. Esse teste nasce marcado como xfail/skip até a SPEC-2208 entregar o stub.

### Modelo de dados
—

<!-- Alternativas: pyright em vez de mypy — melhor inferência, mas exige Node no CI; rejeitada por simplicidade (reavaliar se mypy ficar lento com ingester.py de 8.5k linhas). -->

## Riscos
- Mudar a MRO de `DiodeClientError` pode alterar a ordem de `except` em código de terceiros que captura `BaseError` antes de `RpcError`. Mitigação: nota de release e teste de compatibilidade (`isinstance` nos dois).
- O piso `protobuf>=5.26.1` pode conflitar com ambientes travados em `protobuf` 4.x. Mitigação: o gencode já exige 5.26, então isso só torna explícito o que já quebraria.

## Sinais de sucesso
- `except BaseError` em integrações captura falhas de ingestão, `str(err)` mostra o código e o detalhe, e o mypy dos consumidores enxerga `pb.Device`.

## Critério de aceite
- [ ] `DiodeClientError` é `BaseError` e `RpcError`, com `str()` não vazio | verify: `pytest -q tests/test_client.py -k "client_error"`
- [ ] mypy limpo em `client.py`, `chunking.py` e `exceptions.py` no CI | verify: `mypy netboxlabs/diode/sdk/client.py netboxlabs/diode/sdk/chunking.py netboxlabs/diode/sdk/exceptions.py`
- [ ] `protobuf` e `setuptools>=61` declarados; build OK | verify: `python -m build --sdist --wheel`
- [ ] CI verde em 3.10 a 3.14 e no job `lowest-direct` | evidence: manual @aliothbl
