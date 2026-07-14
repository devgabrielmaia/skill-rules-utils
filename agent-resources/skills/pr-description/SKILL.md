---
name: pr-description
description: >-
  Analisa commits e diff da branch atual e gera descrição de Pull Request em
  Markdown pronta para colar no GitHub. Use quando o usuário pedir comentário de
  PR, descrição de PR, body da PR, resumo da branch, /pr-description ou texto
  para abrir pull request.
---

# Descrição de Pull Request — branch

Gera o **corpo da PR** em Markdown para copiar e colar no GitHub (ou `gh pr create --body`). Foco em **o que mudou, por quê e como validar** — não é code review (use `branch-code-review` para isso).

## Regras invioláveis

- Basear o texto **somente** em evidências do git (commits, diff, arquivos alterados). Não inventar escopo ou motivação.
- Se algo não estiver claro no diff/commits, marcar como **(inferido)** ou omitir.
- Entregar **um único bloco Markdown** final, pronto para copiar — sem explicações longas antes/depois do bloco.
- Escrever em **português**, tom objetivo e profissional.
- **Não** criar PR, commit ou push, salvo se o usuário pedir explicitamente.
- **Não** usar TodoWrite nem Task para este fluxo.

## Workflow

### 1. Coletar contexto (paralelo quando possível)

```bash
git branch --show-current
git status -sb
git remote show origin 2>/dev/null | head -5
```

Definir a base (nessa ordem):

```bash
BASE=$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || git merge-base HEAD master)
echo "$BASE"
```

Se o usuário indicar outra base (ex.: `develop`), usar a informada.

Em paralelo:

```bash
# Commits da branch (mensagem completa)
git log --format=fuller "$BASE"..HEAD

# Lista resumida (referência rápida)
git log --oneline "$BASE"..HEAD

# Arquivos e estatísticas
git diff --name-status "$BASE"...HEAD
git diff --stat "$BASE"...HEAD

# Diff completo (limitar leitura a hunks relevantes se muito grande)
git diff "$BASE"...HEAD
```

Comandos opcionais para enriquecer contexto:

```bash
# Autores e datas
git log --format='%h %an %ad %s' --date=short "$BASE"..HEAD

# Apenas PHP / migrations / front se útil
git diff "$BASE"...HEAD -- '*.php'
git diff "$BASE"...HEAD -- 'database/migrations/*'
git diff "$BASE"...HEAD -- 'resources/**/*'
```

Se não houver commits ou diff vs base, informar o usuário e não gerar PR fictícia.

### 2. Entender a mudança

Para cada commit relevante, extrair:

- **Assunto** (`Subject` / primeira linha)
- **Corpo** (parágrafos e bullets do commit)
- **Relação** entre commits (sequência lógica: refactor → feature → fix)

Do diff, identificar:

| Sinal | O que registrar na PR |
|-------|-------------------------|
| Novos models/migrations | Entidades e impacto em dados |
| Services/Actions | Regras de negócio alteradas |
| Controllers/rotas | Superfície HTTP / API |
| Views/JS | UX ou telas afetadas |
| Seeders/SQL de teste | Dados de exemplo (mencionar se não for para produção) |
| Config/.env.example | Variáveis novas |
| Testes | Cobertura adicionada ou ausente |

### 3. Sintetizar (antes de escrever)

Responder mentalmente (não precisa mostrar ao usuário):

1. **Problema/objetivo:** que dor ou tarefa esta branch resolve?
2. **Abordagem:** decisão técnica principal em 1–2 frases.
3. **Escopo:** o que ficou **fora** de propósito (se inferível).
4. **Riscos:** migrations, breaking changes, feature flags, tenant.
5. **Validação:** passos concretos que um revisor pode executar.

Agrupar commits em **1–3 temas** na PR; não listar commit a commit salvo se forem poucos e cada um for independente.

### 4. Produzir o Markdown final

Usar o template abaixo. **Omitir seções vazias** (não escrever "N/A"). Ajustar títulos se a PR for só fix, só chore, etc.

---

## Template de saída (entregar assim ao usuário)

Introduzir com uma linha curta: *"Descrição pronta para colar na PR:"* e em seguida o bloco:

```markdown
## Resumo

[2–4 frases ou 3–5 bullets: o que esta PR entrega e por quê existe]

## Alterações principais

- **[Área 1]:** [mudança concreta]
- **[Área 2]:** [mudança concreta]

## Commits incluídos

| Hash | Mensagem |
|------|----------|
| `abc1234` | feat: … |
| `def5678` | fix: … |

_Omitir esta seção se houver um único commit ou se a tabela não agregar valor._

## Detalhes técnicos

[Opcional: migrations, novos endpoints, classes-chave, dependências, flags — só o que o diff/commits mostram]

## Impacto e compatibilidade

- **Banco de dados:** [sim/não — descrever migration ou "nenhuma"]
- **API / contratos:** [breaking ou não]
- **Multi-tenant / permissões:** [se aplicável ao diff]

## Plano de testes

- [ ] [Passo reproduzível 1]
- [ ] [Passo reproduzível 2]
- [ ] [Regressão: área relacionada]

## Observações para revisão

[Pontos que o revisor deve olhar com atenção; links para issues/tickets se citados nos commits]

## Screenshots / evidências

_[Placeholder se houver mudança visual; remover se não aplicável]_
```

### Variações por tipo de PR

| Tipo | Ajuste no template |
|------|------------------|
| **Bugfix** | Resumo: sintoma + causa; testes: caso que reproduzia o bug |
| **Feature** | Alterações principais por capability; plano de testes mais longo |
| **Refactor** | Deixar claro "sem mudança de comportamento" se o diff indicar |
| **Chore/deps** | Encurtar; foco em versões e comandos (`composer`, `npm`) |
| **Dados/seed** | Alertar se SQL/seed é só local ou homologação |

---

## Qualidade do texto

- **Resumo:** primeira coisa que o revisor lê; deve bastar para aprovar ou pedir contexto.
- **Bullets:** verbo no presente ou passado consistente; uma ideia por bullet.
- **Plano de testes:** comandos reais do projeto quando existirem (`make test`, `php artisan test`, rotas, telas).
- **Sem jargão vazio:** evitar "melhorias gerais", "ajustes diversos" sem listar o quê.
- **Tamanho:** preferir PR enxuta; se >15 arquivos, agrupar por pasta/domínio em vez de listar cada arquivo.

---

## Relação com outras skills

| Skill | Uso |
|-------|-----|
| `pr-description` (esta) | Texto da PR para GitHub |
| `branch-code-review` | Achados técnicos, severidade, merge blockers |
| Regra do usuário `creating-pull-requests` | Só quando o usuário pedir **criar** a PR via `gh` |

Se o usuário pedir **descrição + review**, gerar dois blocos Markdown separados, com títulos distintos.

---

## Comandos úteis (referência)

```bash
# Contagem de linhas por arquivo
git diff "$BASE"...HEAD --numstat

# Apenas arquivos novos
git diff --diff-filter=A --name-only "$BASE"...HEAD

# Mensagem do último commit (se branch = 1 commit)
git log -1 --format=fuller
```

---

## Exemplo de resumo bom vs ruim

**Ruim:** "Atualizações e correções diversas no sistema."

**Bom:** "Corrige falha ao salvar registro quando o campo obrigatório vem vazio em `StoreRequest`; adiciona validação em `FormRequest` e tratamento de erro na action `CreateRecord`. Requer `php artisan migrate` e testar criação com payload incompleto retornando 422."

---

## Entrega ao usuário

1. Uma linha indicando que o Markdown está pronto para colar.
2. O bloco completo do template preenchido.
3. Opcional (uma linha): base usada (`main` / `develop`) e quantidade de commits/arquivos.

Não repetir o diff inteiro dentro da PR — só referências a arquivos ou módulos quando ajudar o revisor.
