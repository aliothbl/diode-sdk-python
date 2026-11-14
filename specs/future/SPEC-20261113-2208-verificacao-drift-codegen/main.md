# SPEC-20261113-2208: Verificação de drift do código gerado (pb2 + wrappers `ingester.py`)

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-13 22:08
**Ativada:** —
**Pausada em:** —
**Concluída:** —
**Commit final:** —
**Keywords:** codegen, ingester.py, ingester_pb2, buf, protoc, gerador, drift, descriptor, NetBox-v4.7.0, gen-diode-sdk-python, CI
**Features:** codegen
**Branch:** _(a definir na ativação)_
**Depende de:** SPEC-20261113-2131-sync-upstream-develop-2026-11
**Relacionadas:** `diode-netbox-plugin/specs/future/SPEC-20261113-2218-gerador-plugin-utils-versionado/main.md` (mesmo problema: o gerador do `plugin_utils.py` está fora do repositório); `diode/Makefile` (alvo `gen-diode-sdk-python`)
**Programa:** —
**Impacto:** médio · **Complexidade:** média
**Estratégia upstream:** upstream-first. O teste de drift por descriptor é 100 % local e aceitável upstream. Publicar o gerador de wrappers depende da NetBox Labs (o gerador não está em nenhum dos três repositórios).
**Origem:** diagnóstico de 2026-11-13 (`diode/specs/DIAGNOSTICO-2026-11-13.md`, achado S-16).
**Resumo:** `ingester.py` (8,5 mil linhas, "Generated code. DO NOT EDIT. Source: NetBox v4.7.0") não tem gerador versionado. Os `ingester_pb2*` são gerados a partir do repositório `diode` via `make gen-diode-sdk-python`. Na data do diagnóstico não há drift (os 112 messages batem), mas só porque foram regenerados manualmente. Adicionar uma checagem no CI que falha quando proto, pb2 e wrappers divergem.

## Objetivo

Garantir que uma mudança de schema no `diode-proto` (repositório `diode`) nunca seja publicada no SDK com wrappers desatualizados: campo novo no proto sem kwarg no wrapper, campo removido ainda aceito, ou tipo divergente. Hoje a única defesa é a disciplina de quem regenera.

## Escopo

**DENTRO:**
- **Teste de drift por descriptor** (independe do gerador), em `tests/test_codegen_drift.py`. Para cada classe wrapper em `netboxlabs.diode.sdk.ingester`, comparar a assinatura de `__new__` (nomes de parâmetros) com os campos de `ingester_pb2.<Message>.DESCRIPTOR.fields`. Falha em campo faltante ou extra. Cobre também o `oneof` de `Entity`, garantindo que todo tipo do oneof tem wrapper.
- **Checagem pb2 × proto upstream** (job de CI opcional, `workflow_dispatch` e semanal):
  1. clonar `netboxlabs/diode` na tag correspondente;
  2. `make gen-diode-sdk-python`;
  3. `git diff --exit-code -- netboxlabs/diode/sdk/diode netboxlabs/diode/sdk/validate`.
- Registrar no cabeçalho de `ingester.py` a versão do `diode-proto` usada, além da versão do NetBox.
- Abrir issue upstream pedindo a publicação do gerador de wrappers (mesmo pedido feito pela SPEC-2218 no plugin). Link registrado no journal.

**FORA:**
- Reescrever o gerador de wrappers no fork. Isso só se justifica se o upstream recusar publicá-lo. Nesse caso, nova SPEC.
- Mudança do formato dos wrappers → SPEC-2205 (tipagem).

## Invariantes
- SEMPRE: todo campo de todo message de `ingester.proto` tem um parâmetro correspondente no wrapper, e vice-versa.
- NUNCA: editar `ingester.py` ou `*_pb2*.py` à mão. Mudança só por regeneração.

## Implementação

Os arquivos gerados não mudam entre `develop` e `upstream/develop`. As referências valem para os dois.

- `netboxlabs/diode/sdk/ingester.py:1-8`: cabeçalho "Generated code. DO NOT EDIT. Source: NetBox v4.7.0, Timestamp: 2026-09-02". O padrão dos wrappers é `class X: def __new__(cls, ...) -> pb.X`. O teste usa `inspect.signature(X.__new__).parameters`.
- `netboxlabs/diode/sdk/diode/v1/ingester_pb2.py`/`.pyi`: gerados pelo buf (protobuf-python v26.1, grpc-python v1.62.1) a partir de `diode/diode-proto` via `diode/Makefile:6-11` (`gen-diode-sdk-python`, que inclui o `sed` de reescrita de imports).
- Mapeamento de nomes: o wrapper aceita atalhos de string via `PRIMARY_VALUE_MAP` (`ingester.py`). O teste compara nomes de parâmetros, não tipos, e trata eventuais parâmetros de conveniência via allowlist explícita no teste (hoje nenhum é conhecido: `metadata` é campo `Struct` do próprio proto, convertido por `convert_dict_to_struct`, `ingester.py:571-593`).
- Novo workflow `.github/workflows/codegen-drift.yaml` com `schedule` semanal e `workflow_dispatch`. Passos:
  1. `actions/checkout` do SDK e de `netboxlabs/diode` (path `diode`);
  2. `buf` via `bufbuild/buf-setup-action` pinada;
  3. `make -C diode gen-diode-sdk-python`;
  4. `git diff --exit-code`.

### Modelo de dados
—

<!-- Alternativas: snapshot (golden file) da lista de campos — rejeitada: só detecta mudança, não diz qual lado está errado; a comparação direta wrapper × descriptor é autoexplicativa. -->

## Riscos
- O workflow semanal depende do layout do repositório `diode` (caminho do Makefile). Mitigação: fixar a tag do `diode` correspondente à versão do SDK e falhar com mensagem clara.
- Falsos positivos por parâmetros de conveniência dos wrappers. Mitigação: allowlist explícita e revisada no próprio teste.

## Sinais de sucesso
- Um PR que adiciona campo ao `ingester.proto` sem regenerar o SDK quebra o CI do SDK na semana seguinte (ou no PR, se o teste de descriptor pegar).

## Critério de aceite
- [ ] Teste wrapper × descriptor cobre todos os messages e o oneof de `Entity` | verify: `pytest -q tests/test_codegen_drift.py`
- [ ] Teste falha ao remover um campo de um wrapper (mutação manual registrada em `evidence/`) | evidence: manual @aliothbl
- [ ] Workflow `codegen-drift.yaml` regenera pb2 a partir de `netboxlabs/diode` e falha em diff | evidence: manual @aliothbl
- [ ] Issue upstream pedindo a publicação do gerador de wrappers, com link no journal | evidence: manual @aliothbl
