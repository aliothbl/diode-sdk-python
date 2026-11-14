# SPEC-20261113-2207: Endurecer CI e release (permissões, ordem tag→publish, wheel, pins) e bloquear publicação a partir do fork

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:07
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** github-actions, semantic-release, pypi, trusted-publishing, wheel, sdist, permissions, set-output, GITHUB_OUTPUT, pin-sha, dependabot, ruff, fork-guard, branches
**Features:** packaging-ci
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Programa:** —
**Impacto:** médio · **Complexidade:** baixa
**Estratégia upstream:** upstream-first para tudo, **exceto** o guard de fork (`if: github.repository == 'netboxlabs/diode-sdk-python'`). O guard é aceitável upstream, mas, se recusado, fica como única customização de CI do fork.
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achado S-14).
**Resumo:** Hoje:
- o lint não bloqueia;
- o workflow de testes tem `contents: write` e `push.branches` só com padrão negativo (não dispara em push);
- a release publica no PyPI **antes** de o semantic-release criar a tag;
- só gera sdist;
- usa `::set-output` (depreciado), `checkout@v3`, semantic-release beta e `perf*` como major.

O fork também pode disparar a release se receber push em `release`.

## Objetivo

Tornar o pipeline correto, mínimo em privilégios e reprodutível, e garantir que o fork **nunca** tente publicar `netboxlabs-diode-sdk` no PyPI nem criar release/tag de produto.

## Escopo

**DENTRO:**
- `.github/workflows/lint-tests.yml`:
  - `push.branches: ["!release"]` → `branches-ignore: [release]`. Um filtro só com padrões negativos não dispara.
  - `permissions` mínimas: `contents: read` no workflow, e `pull-requests: write`/`checks: write` só no job/step de comentário de cobertura.
  - Remover `continue-on-error: true` do ruff.
  - `concurrency.group` incluir `github.ref` e `cancel-in-progress: true` para PR.
- `.github/workflows/release.yaml`:
  - Ordem: semantic-release (tag + GitHub release) → build → publish. Alternativa: manter a ordem e publicar só após o job de tag com `needs`.
  - Gate `if: needs.get-next-version.outputs.new-release-published == 'true'` também no build.
  - `python -m build --sdist --wheel`.
  - `::set-output` → `>> "$GITHUB_OUTPUT"`.
  - `actions/checkout@v3` → v4 pinado por SHA. Pinar por SHA todas as actions oficiais.
  - semantic-release estável (sair de `25.0.0-beta.6`).
  - Regra `perf*` → `patch`.
  - Guard de repositório em todos os jobs de release: `if: github.repository == 'netboxlabs/diode-sdk-python'`.
- `.github/dependabot.yml` para `pip` e `github-actions`.
- No fork: desabilitar o workflow `release.yaml` via `gh workflow disable` como medida imediata, até o guard entrar.

**FORA:**
- Assinatura de artefatos (sigstore/attestations). Fica como evolução.
- Ajuste da matriz de Python → SPEC-2205.

## Invariantes
- NUNCA: o fork publica no PyPI nem cria tag `vX.Y.Z` de produto.
- SEMPRE: o artefato publicado corresponde a uma tag existente no repositório.
- SEMPRE: o workflow de PR roda com `contents: read`.

## Implementação

Os arquivos de workflow não mudam entre `develop` e `upstream/develop` (`git diff --stat develop upstream/develop -- .github` vazio), então as linhas valem para os dois.

- `.github/workflows/lint-tests.yml:5-7` (`push.branches: ["!release"]`), `9-11` (concurrency), `13-16` (permissions write) e `51-54` (ruff `continue-on-error`).
- `.github/workflows/release.yaml`:
  - `18-20` (permissions globais `id-token`/`contents: write`): mover para o job que precisa;
  - `57` e `193` (semantic-release beta);
  - `79` e `215` (`perf*` major);
  - `111` e `115` (`::set-output`);
  - `142` (`checkout@v3`);
  - `153-159` (build só sdist);
  - `168` (publish antes do job `semantic-release`, `172-174`, que depende de `build`).
- Validar com `actionlint` e `zizmor` (scanner de segurança de workflows) localmente e no CI.
- Medida imediata no fork, já na ativação: `gh workflow disable release.yaml -R aliothbl/diode-sdk-python`.

### Modelo de dados
—

<!-- Alternativas: apagar release.yaml no fork — rejeitada: vira conflito permanente em todo sync upstream; guard por `github.repository` é neutro para o upstream. -->

## Riscos
- Mudar a ordem tag → publish deixa uma tag sem pacote se o publish falhar. Mitigação: o publish é idempotente via retry do job, e o trusted publishing não depende da tag.

## Sinais de sucesso
- Push em `develop` roda testes e o lint bloqueia regressões. Toda versão no PyPI tem wheel e tag correspondente. O fork não registra nenhuma execução de release.

## Critério de aceite
- [ ] `lint-tests.yml` dispara em push para `develop`, com `contents: read` e ruff bloqueante ← specctl check
- [ ] `release.yaml`: guard de repositório, wheel+sdist, `$GITHUB_OUTPUT`, actions pinadas por SHA, publish após a tag ← specctl check
- [ ] `actionlint` e `zizmor` sem erros nos workflows ← specctl check
- [ ] Workflow de release desabilitado no fork `aliothbl/diode-sdk-python` ← specctl check
- [ ] Dependabot configurado para `pip` e `github-actions` ← specctl check
