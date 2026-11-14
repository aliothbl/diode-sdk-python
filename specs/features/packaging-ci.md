# Feature: packaging-ci

**Keywords:** pyproject, setuptools, wheel, sdist, semantic-release, pypi, trusted-publishing, github-actions, ruff, pytest-cov, trivy, dependabot, matriz-python
**Arquivos principais:**
  - pyproject.toml
  - .github/workflows/lint-tests.yml
  - .github/workflows/release.yaml
  - .github/workflows/security.yaml
**Resumo:** Empacotamento (`netboxlabs-diode-sdk`, versão injetada no build), pipeline de testes e lint e release automatizada (semantic-release → PyPI via OIDC).

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2205-tipagem-excecoes-dependencias | `protobuf` declarado, `setuptools>=61`, Python 3.10 a 3.14, mypy | Pisos inválidos e não testados
- SPEC-20261113-2207-ci-release-hardening | Permissões mínimas, tag → publish, wheel, pins, guard de fork | Publish antes da tag; workflow de testes não dispara em push

## Estado atual
Referência `upstream/develop` (o `.github/` é idêntico ao do fork).
- `pyproject.toml`: `version = "0.0.1"`, sobrescrita no build. Dependências: `certifi>=2026.10.19` (após o #117), `grpcio`, `grpcio-status`, `requests`, `sentry-sdk` e `opentelemetry-proto`, todas com piso aberto. Não há `protobuf` declarado. O build usa `setuptools>=43.0.0` (`:66-68`). Os classifiers vão só até 3.11.
- `lint-tests.yml`:
  - `push.branches: ["!release"]` (`:5-7`), só com padrão negativo, de modo que não dispara em push;
  - `permissions: contents: write` (`:13-16`);
  - matriz 3.10/3.11;
  - ruff com `continue-on-error: true` (`:51-54`).
- `release.yaml`:
  - semantic-release `25.0.0-beta.6`;
  - `perf*` gera major;
  - `::set-output`;
  - `checkout@v3` (`:142`);
  - build só de sdist (`:153-159`);
  - publish no PyPI (`:168`) antes do job `semantic-release` (`:172-174`).

## Decisões arquiteturais ativas
- DEC-20261113-2207-versao-injetada-no-build [ativa] (SPEC-20261113-2207, herdada): a versão vem do semantic-release e é gravada por `sed`/`toml set` no build. Trade-off: o repositório nunca contém a versão real.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2207 | O filtro `branches` do GitHub Actions só com padrões `!` não casa nada (2026-11-13). Use `branches-ignore`.
- SPEC-20261113-2207 | No fork, o `release.yaml` herdado pode criar tags e releases no GitHub (2026-11-13). Desabilite-o ou aplique o guard `github.repository`.
