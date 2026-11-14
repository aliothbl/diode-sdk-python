# Feature: upstream-sync

**Keywords:** fork, upstream, netboxlabs, fast-forward, develop, release, tags, divergencia, customizacao
**Arquivos principais:**
  - specs/ (única customização do fork)
  - .github/workflows/release.yaml (risco de publicação a partir do fork)
**Resumo:** Manutenção do fork `aliothbl/diode-sdk-python` alinhado a `netboxlabs/diode-sdk-python`. O fork não tem commits próprios de código, e toda melhoria segue a estratégia upstream-first.

## Specs desta feature
### Concluídas
—
### Planejadas (future/)
- SPEC-20261113-2131-sync-upstream-develop-2026-11 | Fast-forward develop (+2: #114, #117) e release (+5); remover branch obsoleta `fix/enghlp-1703-skip-tls-verify-parity` | Fork atrasado e sem a correção de TLS do #114

## Estado atual
Em 2026-11-13:
- Remotes: `origin` = `git@github.com:aliothbl/diode-sdk-python.git`, `upstream` = `https://github.com/netboxlabs/diode-sdk-python.git`.
- `develop` está 0 commits à frente e 2 atrás de `upstream/develop`; `release`, 0 à frente e 5 atrás de `upstream/release`.
- A branch `origin/fix/enghlp-1703-skip-tls-verify-parity` reúne 14 commits pré-squash do trabalho que entrou upstream como #114. O upstream já apagou essa branch.
- Tags: `v1.14.1` a `v1.14.3` presentes nos dois lados.
- A única customização planejada do fork é o diretório `specs/` na raiz. O `docs/` pertence ao upstream e não é usado para specs, para evitar conflito.

## Decisões arquiteturais ativas
- DEC-20261113-2131-upstream-first [ativa] (SPEC-20261113-2131): toda correção de código vai como PR para `netboxlabs/diode-sdk-python`, e o fork carrega o patch só até o merge. Trade-off: latência de revisão upstream, em troca de zero divergência permanente.
- DEC-20261113-2131-specs-fora-de-docs [ativa] (SPEC-20261113-2131): specs vivem em `specs/`, não em `docs/`. Trade-off: desvio de path em relação ao wynk_crm (`docs/`), em troca de sync sem conflito.
- DEC-20261113-2131-sem-rewrite [ativa] (SPEC-20261113-2131): sincronização só por FF ou merge, nunca por rebase/force-push em `develop`/`release`.

## Alternativas consideradas e rejeitadas
—

## Gotchas
- SPEC-20261113-2131 | O `release.yaml` herdado publica no PyPI via trusted publishing ao receber push em `release` (2026-11-13). No fork, o publish falha por OIDC, mas o semantic-release ainda pode criar tag/release no GitHub. Desabilitar o workflow no fork (ver SPEC-20261113-2207).
- SPEC-20261113-2131 | O #114 muda o comportamento de `DIODE_SKIP_TLS_VERIFY` com `grpcs://` (2026-11-13): o canal deixa de cair para texto puro e passa a usar TLS sem verificação via tunnel local. No Windows, sem Unix socket, exige `DIODE_SKIP_TLS_VERIFY_ALLOW_LOOPBACK=true`.
