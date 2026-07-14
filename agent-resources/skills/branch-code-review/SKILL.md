---
name: branch-code-review
description: >-
  Realiza code review técnico completo apenas do código alterado na branch atual
  (diff, commits e mudanças não commitadas), classifica achados por severidade e
  salva o relatório em Markdown. Use quando o usuário pedir code review, review
  da branch, revisão técnica, /code-review, /branch-code-review ou análise de
  diff antes do merge.
---

# Code Review — branch

Revisa **somente o código alterado** nesta branch. Entrega um relatório técnico em arquivo `.md` bem formatado.

**Não** é descrição de PR (use `pr-description` para isso).

## Regras invioláveis

- Analisar **apenas** código que sofreu alteração nesta branch (diff + working tree não commitado).
- **Não** comentar arquivos ou trechos intocados pelo diff.
- **Não** sugerir refatorações cosméticas (renomear variável, reordenar imports, estilo sem impacto).
- **Não** propor mudanças por preferência pessoal ou “seria melhor assim” sem risco técnico real.
- Basear achados em **evidência** (arquivo, linha, trecho do diff). Sem comentários genéricos.
- Se não houver problemas relevantes, declarar explicitamente que o código está adequado.
- Escrever o relatório em **português**, tom objetivo, técnico e pragmático.
- **Não** criar commit, push ou PR, salvo se o usuário pedir explicitamente.
- **Não** usar TodoWrite nem Task para este fluxo.

## Workflow

### 1. Coletar contexto (paralelo quando possível)

```bash
git branch --show-current
git status -sb
```

Definir a base (nessa ordem):

```bash
BASE=$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || git merge-base HEAD master)
echo "$BASE"
```

Se o usuário indicar outra base (ex.: `develop`), usar a informada.

Em paralelo:

```bash
# Commits da branch
git log --format=fuller "$BASE"..HEAD
git log --oneline "$BASE"..HEAD

# Diff commitado vs base
git diff --name-status "$BASE"...HEAD
git diff --stat "$BASE"...HEAD
git diff "$BASE"...HEAD

# Mudanças ainda não commitadas (working tree + staged)
git diff
git diff --cached
git status --porcelain
```

Comandos opcionais por tipo de arquivo:

```bash
# Diff unificado incluindo working tree (útil para visão completa)
git diff "$BASE"...HEAD
git diff HEAD  # unstaged
git diff --cached  # staged

# Foco por extensão quando o diff for grande (ajustar ao projeto)
git diff "$BASE"...HEAD -- 'src/**'
git diff "$BASE"...HEAD -- 'database/**'
git diff "$BASE"...HEAD -- '**/*.{ts,tsx,js,jsx,vue}'
git diff "$BASE"...HEAD -- '**/*.{py,go,rs,java,rb,php}'
```

Se não houver diff nem mudanças locais, informar o usuário e encerrar sem inventar achados.

### 2. Detectar stack e convenções do projeto

Inferir stack a partir de arquivos na raiz e no diff **apenas para contexto** — os critérios desta skill são agnósticos de linguagem.

| Sinal | Uso no review |
|-------|----------------|
| Manifestos de dependência (`package.json`, `composer.json`, `go.mod`, `pyproject.toml`, etc.) | Identificar stack e ler skill de convenções do projeto, se existir |
| Estrutura de pastas existente | Critério 12: aderência ao padrão **deste** repositório |
| Skill `project-conventions` (ou equivalente no projeto) | Critérios 9, 10 e 12 — regras específicas de arquitetura, linguagem e organização |

**Não** aplicar convenções de uma stack no review de outra. Detalhes de framework, linter, ORM ou estilo de código pertencem à skill de convenções do projeto.

Ler **apenas** arquivos necessários para contexto das mudanças (callers, interfaces, testes relacionados) — não revisar o arquivo inteiro se só parte dele mudou.

### 3. Analisar cada hunk alterado

Para cada arquivo/trecho modificado, aplicar os critérios abaixo **somente ao código novo ou alterado**.

Prioridade de severidade ao classificar:

1. 🔴 **Crítico** — bug, falha de segurança, regra de negócio incorreta, bloqueador de merge
2. 🟡 **Relevante** — performance, design ruim, risco futuro, dívida com impacto
3. 🟢 **Sugestão** — melhoria real e não cosmética

### 4. Critérios de avaliação

| # | Critério | Severidade padrão |
|---|----------|-------------------|
| 1 | **Correção da regra de negócio** — lógica atende o requisito? efeitos colaterais? | 🔴 se incorreto |
| 2 | **Segurança** — injeção, validação de entrada, auth/authz, exposição de dados sensíveis | 🔴 se vulnerável |
| 3 | **Performance** — N+1, query em loop, consultas pesadas, falta de paginação, Big O, alto uso de memória | 🟡 ou 🔴 |
| 4 | **Bugs potenciais / edge cases** — null, divisão por zero, limites, estados inválidos, concorrência lógica | 🔴 ou 🟡 |
| 5 | **SOLID / KISS / DRY** — apenas violações **claras** com impacto | 🟡 |
| 6 | **Design** — acoplamento excessivo, fragilidade, baixa coesão | 🟡 |
| 7 | **Race conditions / vazamento de memória** — threads, async, listeners, recursos não liberados | 🔴 ou 🟡 |
| 8 | **Clean Code** — ver [regras detalhadas](#clean-code-critério-8) abaixo | 🟢 (escala para 4, 5, 6, 13, 15, 16) |
| 9 | **Arquitetura / separação de camadas** — responsabilidades nas camadas corretas; detalhes na skill de convenções do projeto | 🟢 (🟡 se quebra grave) |
| 10 | **Convenções do projeto** — padrões documentados na skill de convenções ou idioms já usados no repo | 🟢 apenas |
| 11 | **Design Patterns (GoF)** — aplicar padrões do catálogo Gang of Four quando um problema real se encaixar (Strategy, Factory, Observer, Decorator, Adapter, etc.); não forçar onde não há problema | 🟢 apenas |
| 12 | **Estrutura padrão do projeto** — pastas, naming, organização já usada no repo | 🟡 se inconsistente |
| 13 | **Tratamento de erros e logs** — exceções engolidas, falta de log em falhas críticas | 🟡 ou 🔴 |
| 14 | **Mudanças de schema** — indexes, foreign keys, unique, tipos de coluna, rollback/reversão | 🔴 ou 🟡 *(só se houver no diff)* |
| 15 | **Complexidade cognitiva** — funções longas, muitos níveis de indentação, múltiplas responsabilidades | **🔴 crítico** |
| 16 | **Duplicação literal** — strings/números mágicos repetidos que deveriam ser constantes | **🔴 crítico** |
| 17 | **Type check** — tipos incorretos ou ausentes onde a stack do projeto exige tipagem | 🔴 ou 🟡 |
| 18 | **Object Calisthenics (Jeff Bay)** — ver [regras detalhadas](#object-calisthenics-critério-18) abaixo | 🟢 ou 🟡 |
| 19 | **CQS (Command Query Separation)** — ver [regras detalhadas](#cqs-critério-19) abaixo | 🟢 apenas |

**Itens 8, 10, 11, 18 e 19 nunca são críticos** salvo se mascararem bug dos itens 1–7.

#### Clean Code (critério 8)

Regras de legibilidade e manutenção. Várias escalam para critérios críticos/relevantes quando violadas gravemente — ver notas entre parênteses.

1. **Funções pequenas e com uma única responsabilidade** — cada função faz uma coisa; se precisar de "e" para descrever o que faz, quebrar em duas ou em métodos privados. *(critério 15 — 🔴)*
2. **Máximo de 3 parâmetros por função** — mais que isso, agrupar em objeto/struct/DTO.
3. **Nomes revelam intenção** — variáveis, funções e classes autoexplicativas, sem precisar de comentário para entender o que fazem.
4. **Sem números/strings mágicos** — valores literais soltos viram constantes nomeadas. *(critério 16 — 🔴)*
5. **Sem código morto ou comentado** — deletar, não comentar (o Git guarda o histórico).
6. **Evitar aninhamento profundo (> 2–3 níveis)** — usar early return / guard clauses em vez de if/else encadeados. *(critério 15 — 🔴)*
7. **Sem duplicação de lógica (DRY)** — se o mesmo trecho aparece 3+ vezes, extrair numa função/módulo. *(critério 5 — 🟡)*
8. **Tratamento de erro explícito** — nunca engolir exceções silenciosamente (`catch {}` vazio é proibido). *(critério 13 — 🔴/🟡)*
9. **Sem null/undefined implícito** — preferir valores default, Option/Maybe ou validação explícita de entrada. *(critérios 4 e 17)*
10. **Uma classe/módulo, uma responsabilidade (SRP)** — se o nome tem "e", "Manager" ou "Utils" genérico demais, provavelmente faz coisa demais. *(critérios 5 e 6 — 🟡)*
11. **Testes cobrem comportamento, não implementação** — testes não devem quebrar ao refatorar sem mudar comportamento.
12. **Sem efeitos colaterais escondidos** — uma função `getX()` não deve alterar estado; se altera, o nome deve deixar isso claro (`updateX`, `fetchAndCache`).

#### Object Calisthenics (critério 18)

Regras de design orientado a objetos (Jeff Bay). Aplicar com pragmatismo — DTOs e modelos de persistência na borda podem ser exceção; domínio e serviços devem seguir mais de perto.

1. **Um ponto por linha (Lei de Demeter)** — evitar encadeamentos como `pedido.getCliente().getEndereco().getCidade()`; cada objeto fala só com vizinhos diretos. Delegar: `pedido.cidadeDeEntrega()`. *(🟡 relevante — acoplamento e fragilidade; ver critério 6)*
2. **Não abrevie** — nomes completos e claros (`quantidade`, não `qtd`; `gerenciador`, não `gerenc`). *(🟢 sugestão — reforça Clean Code 8.3)*
3. **Não use a palavra-chave `else`** — preferir early return, guard clauses ou polimorfismo. *(🟢 sugestão — escala para critério 15 se aninhamento grave)*
4. **Máximo de 2 variáveis de instância por classe** — força coesão; se precisar de mais, quebrar a classe. *(🟡 relevante — ver critérios 5 e 6)*
5. **Sem getters/setters/properties públicos** — expor comportamento, não estado (Tell, Don't Ask). *(🟡 relevante — encapsulamento no domínio)*

#### CQS (critério 19)

Command Query Separation — cada método é **ou** Command **ou** Query, nunca os dois. *(🟢 sugestão — reforça Clean Code 8.12 e Object Calisthenics 18.5)*

- **Query** — retorna dados, **não altera estado** (sem efeitos colaterais). Idempotente: chamadas repetidas não mudam o resultado.
- **Command** — **altera estado** (efeito colateral), **não retorna dado** (idealmente `void` / sem valor de retorno útil).

Violação típica: método que persiste, loga ou muta cache e ainda devolve um valor — separar em query + command.

### 5. Produzir e salvar o relatório

**Caminho do arquivo** (nessa ordem):

1. Caminho indicado pelo usuário
2. `.cursor/reviews/code-review-<branch>-<YYYY-MM-DD>.md`
3. `code-review-<branch>.md` na raiz do projeto

Criar o diretório pai se não existir. Usar `<branch>` sanitizado (substituir `/` por `-`).

Preencher o template abaixo. **Omitir seções vazias** (não escrever "N/A"). Ordenar achados: 🔴 → 🟡 → 🟢.

---

## Template do arquivo Markdown

```markdown
# Code Review — `<branch>`

| Campo | Valor |
|-------|-------|
| **Data** | YYYY-MM-DD |
| **Branch** | `nome-da-branch` |
| **Base** | `main` / `develop` / `<hash>` |
| **Commits** | N commits (`abc1234` … `def5678`) |
| **Arquivos alterados** | N arquivos (+X / -Y linhas) |
| **Escopo** | Diff da branch + mudanças não commitadas *(se houver)* |

## Resumo executivo

[2–4 frases: veredito geral, riscos principais, se está apto para merge ou não]

## Estatísticas

| Severidade | Quantidade |
|------------|------------|
| 🔴 Crítico | 0 |
| 🟡 Relevante | 0 |
| 🟢 Sugestão | 0 |

## 🔴 Crítico

### [Título curto do achado]

- **Arquivo:** `caminho/arquivo.ext` (linhas X–Y)
- **Critério:** [número e nome, ex.: 15 — Complexidade cognitiva]
- **Problema:** [descrição objetiva do risco]
- **Evidência:** [trecho ou comportamento observado no diff]
- **Recomendação:** [ação concreta para corrigir]

_Repetir por achado. Se não houver: **Nenhum achado crítico.**_

## 🟡 Relevante

### [Título curto do achado]

- **Arquivo:** `caminho/arquivo.ext` (linhas X–Y)
- **Critério:** [número e nome]
- **Problema:** …
- **Evidência:** …
- **Recomendação:** …

_Se não houver: **Nenhum achado relevante.**_

## 🟢 Sugestão

### [Título curto do achado]

- **Arquivo:** `caminho/arquivo.ext` (linhas X–Y)
- **Critério:** [número e nome]
- **Problema:** …
- **Recomendação:** …

_Se não houver: **Nenhuma sugestão.**_

## Veredito

- [ ] **Apto para merge** — sem bloqueadores
- [ ] **Merge com ressalvas** — corrigir itens 🟡 antes ou logo após merge
- [ ] **Não mergear** — itens 🔴 devem ser corrigidos

[Parágrafo final: se o código estiver adequado sem ressalvas, declarar explicitamente.]

## Arquivos revisados

| Arquivo | Status |
|---------|--------|
| `path/to/file.ext` | ✅ sem achados / ⚠️ ver itens acima |

_Listar apenas arquivos que aparecem no diff analisado._

## Contexto analisado

### Commits

| Hash | Mensagem |
|------|----------|
| `abc1234` | feat: … |

### Mudanças não commitadas

[Resumo do `git status` / hunks locais, ou "Nenhuma"]
```

---

## Qualidade dos achados

Cada achado deve responder:

1. **O quê** — problema concreto
2. **Onde** — arquivo e linhas
3. **Por quê importa** — impacto (bug, segurança, custo, manutenção)
4. **Como corrigir** — ação específica, não vaga

**Ruim:** "O código poderia ser mais limpo."

**Bom:** "`OrderService::calculateTotal` (L45–89) acumula 4 níveis de `if` e mistura desconto com imposto; extrair `applyDiscount` e `applyTax` reduz risco de regressão no cálculo exibido ao cliente."

**Ruim:** "Considere usar Repository pattern."

**Bom:** "`UserHandler` (L22–30) monta query com 6 joins inline — no projeto, queries complexas ficam na camada de repositório (ver `OrderRepository`); mover evita duplicar a mesma query em `ReportHandler`."

---

## Relação com outras skills

| Skill | Uso |
|-------|-----|
| `branch-code-review` (esta) | Achados técnicos, severidade, arquivo `.md` |
| `pr-description` | Corpo da PR para GitHub |
| `project-conventions` | Arquitetura, stack e convenções específicas do repositório |
| `review-bugbot` / `review-security` | Reviews automatizados adicionais sob demanda |

Se o usuário pedir **review + descrição de PR**, gerar o arquivo `.md` do review **e** entregar o bloco Markdown da PR separadamente.

---

## Entrega ao usuário

1. Confirmar que o review foi salvo: informar **caminho absoluto** do arquivo `.md`.
2. Resumo curto na conversa: veredito (apto / ressalvas / não mergear), contagem por severidade.
3. Listar apenas achados 🔴 e 🟡 na mensagem (detalhes completos ficam no arquivo).
4. Se zero achados 🔴 e 🟡: declarar que **o código alterado está adequado** para merge.

Não colar o relatório inteiro no chat se for longo — apontar para o arquivo e destacar bloqueadores.

---

## Comandos úteis (referência)

```bash
# Diff completo incluindo staged + unstaged sobre último commit
git diff HEAD

# Apenas arquivos novos na branch
git diff --diff-filter=A --name-only "$BASE"...HEAD

# Ver contexto de linha no diff
git diff -U5 "$BASE"...HEAD -- caminho/arquivo.ext

# Contagem por arquivo
git diff "$BASE"...HEAD --numstat
```
