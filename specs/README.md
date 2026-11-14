# specs/ — sistema SPEC-driven do fork `aliothbl/diode-sdk-python`

Este diretório adota o padrão SPEC-driven do projeto **wynk_crm** (schema `sdd-4.0`). Referência: `wynk_crm/docs/RULES.md` e `wynk_crm/docs/rules/formats.md`. A estrutura, os campos e a gramática dos critérios de aceite são os mesmos.

## Por que `specs/` e não `docs/`

`docs/` pertence ao upstream (netboxlabs). No `diode` ele é publicado via mkdocs (`mkdocs.yml`), e todo `.md` dentro dele seria publicado. Manter o sistema de specs num diretório **que não existe no upstream** garante dois efeitos: (1) nenhuma sincronização com o upstream gera conflito, e (2) nenhum documento interno é publicado por engano. Fora essa troca de raiz (`docs/` → `specs/`), o padrão é idêntico ao do wynk_crm.

## Estrutura

| Caminho | Conteúdo |
|---|---|
| `specs/future/SPEC-<YYYYMMDD-HHMM>-<slug>/main.md` | SPEC planejada (`Status: draft`). Só `main.md`. |
| `specs/active/` | SPEC em execução: `main.md` + `journal.md` (SNAPSHOT + LOG append-only). Vazio em `develop`; SPEC ativa vive em branch. |
| `specs/archive/`, `specs/discard/` | Concluídas / descartadas, com `digest.md`. Nunca deletar. |
| `specs/features/<area>.md` | Memória viva por área: estado atual, decisões (DEC-…), gotchas, SPECs da área. |
| `specs/TAXONOMY.md` | Áreas canônicas. Toda SPEC vincula ≥1 área daqui. |
| `specs/ROADMAP.md` | Ordem de execução (priorização humana) das SPECs future que não pertencem a programa. |
<!-- | `specs/programs/` | (só no `diode`) DAG de SPECs com dependência entre repos. | -->
<!-- | `specs/DIAGNOSTICO-2026-11-13.md` | (só no `diode`) diagnóstico consolidado dos 3 repos que originou as SPECs. | -->

## Regras essenciais (resumo de wynk_crm RULES R.1–R.17)

1. ID = `SPEC-YYYYMMDD-HHMM-slug`, com timestamp real. Proibido sequencial ou reuso.
2. `main.md` é o **contrato** validado por humano. Plano, progresso e decisões vão no `journal.md`, não no main.
3. Critério de aceite por porte: **P** usa `specctl check`/marcação com timestamp+commit. **M/G** usam `| verify: \`cmd\``. Humano: `| evidence: manual @user`. `[x]` só com timestamp e evidência.
4. Deferir, aceitar incompleto ou reduzir escopo **só com autorização explícita do usuário** (`[aceito-incompleto: "<citação>" ts]`).
5. Commits sob SPEC ativa levam o trailer `Spec: SPEC-<id>`.
6. Segredo em `specs/` é proibido (`<REDACTED:tipo>`).
7. Toda SPEC de melhoria segue **upstream-first**: PR em `netboxlabs/diode-sdk-python`, carregado no fork só até o merge. Isso preserva o fork como espelho e torna a sincronização trivial.

## Sincronização com upstream (runbook manual)

```bash
git remote add upstream https://github.com/netboxlabs/diode-sdk-python.git   # uma vez
git fetch upstream --tags
git rev-list --left-right --count develop...upstream/develop          # "<nossos> <deles>"
git switch develop && git merge --ff-only upstream/develop || git merge --no-ff upstream/develop
git push origin develop --follow-tags
```

Repetir para `release` quando existir. A automação está descrita em `diode/specs/future/SPEC-20261113-2140-automacao-sync-upstream`.
