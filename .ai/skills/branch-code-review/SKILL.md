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

Esta skill define o **padrão do review** (coleta de diff, severidades, estrutura do relatório) — não os critérios de qualidade de código em si. Critérios de qualidade, padrões, design e arquitetura vêm da skill `code-quality` quando ela existir; na ausência dela, usa-se o [checklist básico](#checklist-básico-fallback) definido aqui.

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

### 2. Definir os critérios de qualidade

Esta skill define **o padrão do review** (o quê coletar, como classificar severidade, como estruturar o relatório) — ela **não** redefine critérios de qualidade de código, design ou arquitetura. Para isso:

- **Se existir uma skill `code-quality`** disponível para o agente (mesma coleção de skills desta ferramenta): ler a skill inteira e usar **exatamente** os critérios, severidades e subseções dela (Clean Code, Object Calisthenics, CQS, Design Patterns GoF etc.) — fonte única de verdade. Não redefinir critério, nome ou severidade além do que ela já define; se ela mudar, este review muda junto, sem precisar editar esta skill.
- **Se não existir:** aplicar o [Checklist básico (fallback)](#checklist-básico-fallback) definido mais abaixo nesta própria skill.

Inferir a stack a partir de arquivos na raiz e no diff (manifestos de dependência, estrutura de pastas) **apenas para contexto** — os critérios de qualidade (da skill `code-quality` ou do checklist básico) são agnósticos de linguagem.

Ler **apenas** arquivos necessários para contexto das mudanças (callers, interfaces, testes relacionados) — não revisar o arquivo inteiro se só parte dele mudou.

### 3. Analisar cada hunk alterado

Para cada arquivo/trecho modificado, aplicar **os critérios definidos no passo 2** (skill `code-quality` ou checklist básico) somente ao código novo ou alterado — não redefinir critério, nome ou severidade além do que a fonte usada já define.

Prioridade de severidade ao classificar:

1. 🔴 **Crítico** — bug, falha de segurança, regra de negócio incorreta, bloqueador de merge
2. 🟡 **Relevante** — performance, design ruim, risco futuro, dívida com impacto
3. 🟢 **Sugestão** — melhoria real e não cosmética

Nota: quando a fonte de critérios for a skill `code-quality`, o critério de mudanças de schema só entra em jogo se o diff realmente contiver migration/alteração de schema, e os critérios marcados como "apenas sugestão" nunca são críticos — salvo se mascararem bug de regra de negócio, segurança, edge case ou vazamento de recurso.

### 4. Produzir e salvar o relatório

**Caminho do arquivo** (nessa ordem):

1. Caminho indicado pelo usuário
2. `code-reviews/code-review-<branch>-<YYYY-MM-DD>.md`
3. `code-review-<branch>.md` na raiz do projeto

Criar o diretório pai se não existir. Usar `<branch>` sanitizado (substituir `/` por `-`).

Preencher o template abaixo. **Omitir seções vazias** (não escrever "N/A"). Ordenar achados: 🔴 → 🟡 → 🟢.

---

## Checklist básico (fallback)

Usado **somente quando não há skill `code-quality` disponível**. Cobre o essencial sem duplicar a profundidade de uma skill dedicada (SOLID, Clean Code, Object Calisthenics, CQS, Design Patterns GoF etc. ficam de fora — se o usuário quiser esse nível de rigor recorrente, sugerir criar a skill `code-quality` no projeto).

| # | Critério | Severidade típica |
|---|----------|--------------------|
| 1 | Regra de negócio atende ao requisito, sem efeito colateral indevido em estados ou dados relacionados | 🔴 |
| 2 | Segurança: injection, validação de entrada, exposição de dados sensíveis/segredos, auth/authz | 🔴 |
| 3 | Bugs / edge cases: null, divisão por zero, limites, estados inválidos, concorrência lógica | 🔴 ou 🟡 |
| 4 | Performance óbvia: N+1, query em loop, ausência de paginação em coleção grande | 🟡 |
| 5 | Duplicação evidente de lógica ou literais mágicos repetidos | 🟡 |
| 6 | Funções/classes com responsabilidade única; nomes que revelam intenção | 🟡 |
| 7 | Erros tratados explicitamente — sem `catch`/`except` vazio, sem exceção engolida | 🟡 |
| 8 | Consistência com os idioms já usados no arquivo/módulo alterado | 🟢 |

Não aplicar itens fora dessa tabela quando o checklist básico for a fonte usada — isso evita reimplementar a skill `code-quality` de forma incompleta dentro deste review.

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
| `branch-code-review` (esta) | Padrão do review: coleta de diff, severidades, estrutura do relatório |
| `code-quality` | Fonte dos critérios de qualidade/design/arquitetura quando disponível (passo 2); ausente, usa-se o checklist básico desta skill |
| `pr-description` | Corpo da PR para GitHub |
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
