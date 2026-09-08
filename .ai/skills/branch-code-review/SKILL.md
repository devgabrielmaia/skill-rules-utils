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

### 2. Ler a skill de qualidade e as rules do projeto

Antes de analisar qualquer hunk, ler:

- **`.claude/skills/code-quality/SKILL.md`** — fonte única de verdade dos 19 critérios de qualidade, das severidades (Crítico / Relevante / Sugestão) e das subseções Clean Code (critério 8), Object Calisthenics (critério 18) e CQS (critério 19). Os critérios usados no passo 4 **são os dela**, não uma cópia — se a skill mudar, este review muda junto, sem precisar editar esta skill.
- **`.claude/rules/project-conventions.md`** (ou rule equivalente do projeto) — convenções específicas da stack, usadas nos critérios 9 (arquitetura/separação de camadas), 10 (convenções do projeto) e 12 (estrutura do projeto). Se essa rule não existir no projeto, espelhar idioms já usados no repositório.

Inferir a stack a partir de arquivos na raiz e no diff (manifestos de dependência, estrutura de pastas) **apenas para contexto** — os critérios da skill `code-quality` são agnósticos de linguagem; **não** aplicar convenções de uma stack no review de outra.

Ler **apenas** arquivos necessários para contexto das mudanças (callers, interfaces, testes relacionados) — não revisar o arquivo inteiro se só parte dele mudou.

### 3. Analisar cada hunk alterado

Para cada arquivo/trecho modificado, aplicar **os critérios e severidades de `.claude/skills/code-quality/SKILL.md`** somente ao código novo ou alterado — não redefinir critério, nome ou severidade além do que a skill já define.

Prioridade de severidade ao classificar (mesma escala da skill `code-quality`):

1. 🔴 **Crítico** — bug, falha de segurança, regra de negócio incorreta, bloqueador de merge
2. 🟡 **Relevante** — performance, design ruim, risco futuro, dívida com impacto
3. 🟢 **Sugestão** — melhoria real e não cosmética

Notas específicas deste review (contexto de diff), que complementam a skill `code-quality` sem alterá-la:

- **Critério 14 (Mudanças de schema)** só entra em jogo quando o diff realmente contém migration/alteração de schema.
- **Critérios 9, 10 e 12** — cruzar com a rule de convenções do projeto lida no passo 2.
- Itens 8, 10, 11, 18 e 19 nunca são críticos, conforme a skill `code-quality` — salvo se mascararem bug de algum dos critérios 1–7.

### 4. Produzir e salvar o relatório

**Caminho do arquivo** (nessa ordem):

1. Caminho indicado pelo usuário
2. `code-reviews/code-review-<branch>-<YYYY-MM-DD>.md`
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
| `code-quality` | Fonte dos critérios e severidades aplicados no passo 3 |
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
