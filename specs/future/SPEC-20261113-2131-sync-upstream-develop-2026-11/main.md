# SPEC-20261113-2131: Sincronizar fork `diode-sdk-python` com upstream (develop/release, nov/2026)

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-13 21:31
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** upstream, sync, fork, netboxlabs, tls, skip-verify, certifi, ENGHLP-1703
**Features:** upstream-sync, client-transport
**Branch:** _(a definir na ativação)_
**Programa:** `diode/specs/programs/diode-stack-hardening.md`
**Relacionadas:** `diode/specs/future/SPEC-20261113-2130-sync-upstream-develop-2026-11/main.md`
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, §5).
**Resumo:** Fast-forward de `develop` (+2 commits: #114 TLS mantido com `DIODE_SKIP_TLS_VERIFY`, #117 `certifi>=2026.10.19`) e de `release` (+5), mais remoção da branch obsoleta `fix/enghlp-1703-skip-tls-verify-parity`.

## Objetivo

O fork não tem commits próprios e está 2 commits atrás em `develop` e 5 em `release`. O commit #114 muda o comportamento de segurança do cliente: com `DIODE_SKIP_TLS_VERIFY` o canal continua TLS (sem verificação), em vez de cair para gRPC em texto puro. Isso precisa entrar no fork antes de qualquer melhoria no transporte.

## Escopo

**DENTRO:**
- `git merge --ff-only upstream/develop` em `develop` (`2db9b1f` #114, `44ff22f` #117).
- `git merge --ff-only upstream/release` em `release`.
- Remover `origin/fix/enghlp-1703-skip-tls-verify-parity`: são 14 commits pré-squash do mesmo trabalho que entrou como #114. Upstream já apagou a branch. O diff dela contra `upstream/develop` **remove** `tests/test_skip_verify_tunnel.py` (830 linhas), então mantê-la só gera risco de merge acidental.
- Rodar a suíte (`pytest`) e o lint (`ruff`) após o FF.
- Tags (`v1.14.x`) sincronizadas.

**FORA:**
- Revisão de fundo do `_skip_verify_tunnel.py` (338 linhas novas): vai para SPEC própria de transporte se o diagnóstico apontar problema.
- Publicação em PyPI a partir do fork (o fork consome o pacote upstream; ver Riscos).

## Invariantes
- NUNCA reescrever histórico de `develop`/`release`.
- NUNCA publicar no PyPI a partir do fork com o nome `netboxlabs-diode-sdk`.

## Implementação

```bash
git fetch upstream --tags
git switch develop  && git merge --ff-only upstream/develop
git switch release  && git merge --ff-only upstream/release
pytest -q && ruff check .
git push origin develop release --follow-tags
git push origin --delete fix/enghlp-1703-skip-tls-verify-parity   # confirmar com o owner antes
```

### Modelo de dados
—

## Riscos
- O workflow `release.yaml` herdado do upstream pode tentar publicar ao receber push em `release` se os secrets existirem no fork. Mitigação: confirmar que o fork não tem `PYPI_*` configurado ou desabilitar o workflow no fork (ver SPEC de CI do fork).
- #114 muda o comportamento com `DIODE_SKIP_TLS_VERIFY=true`: quem dependia do fallback para texto puro (ex.: servidor sem TLS escutando em 8080) precisa trocar o target para `grpc://`. Mitigação: nota no CHANGELOG interno.

## Sinais de sucesso
- Clientes do fork com `DIODE_SKIP_TLS_VERIFY` passam a negociar TLS (verificável com `tcpdump`/log do servidor).

## Critério de aceite
- [ ] `develop` e `release` sem atraso para upstream (`rev-list --left-right --count` = `N 0`) ← specctl check
- [ ] Branch `fix/enghlp-1703-skip-tls-verify-parity` removida do `origin` com OK do owner ← specctl check
- [ ] `pytest` e `ruff` verdes após o FF ← specctl check
