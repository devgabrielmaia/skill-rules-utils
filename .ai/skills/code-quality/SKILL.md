---
name: code-quality
description: >-
  Define critérios agnósticos de linguagem e framework para qualidade de
  código — regra de negócio, segurança, performance, bugs/edge cases,
  SOLID/KISS/DRY, design, Clean Code, Object Calisthenics, CQS e Design
  Patterns (GoF) — com níveis de severidade (crítico/relevante/sugestão) e
  checklist de entrega. Use sempre que escrever, alterar, revisar ou dar
  review em código, ou quando o contexto pedir padrões de qualidade, boas
  práticas, patterns, clean code, refactor, code smell ou convenções de
  código em geral.
---

# Qualidade de Código

Critérios **agnósticos de linguagem e framework**, autocontidos nesta skill. Convenções específicas de stack (estilo, ORM, camadas, linter) não são cobertas aqui — espelhar os idioms já usados no repositório.

Ao implementar, modificar ou revisar código, validar mentalmente cada critério abaixo **antes de considerar a tarefa concluída**. Corrigir problemas críticos no próprio diff; não deixar dívida conhecida sem avisar o usuário.

## Severidade

| Nível | Quando usar |
|-------|-------------|
| **Crítico** | Bloqueia merge — corrigir antes de entregar |
| **Relevante** | Risco real de bug, performance ou manutenção — corrigir ou justificar |
| **Sugestão** | Melhoria opcional — mencionar só se agregar valor |

**Itens 8, 10, 11, 18 e 19 nunca são críticos** salvo se mascararem bug dos itens 1–7.

## Critérios

### Críticos e relevantes

1. **Regra de negócio** (crítico) — Lógica atende o requisito? Efeitos colaterais em estados, transições e dados relacionados?
2. **Segurança** (crítico) — SQL/command injection, mass assignment, validação de entrada, auth/authz, exposição de PII/secrets, CSRF onde aplicável.
3. **Performance** (relevante ou crítico) — N+1, query em loop, ausência de paginação/limites, Big O desnecessário, alto consumo de memória.
4. **Bugs / edge cases** (crítico ou relevante) — null, divisão por zero, limites, estados inválidos, concorrência lógica.
5. **SOLID / KISS / DRY** (relevante) — Apenas violações **claras** com impacto; não refatorar por estética.
6. **Design** (relevante) — Acoplamento excessivo, fragilidade, baixa coesão, responsabilidades misturadas.
7. **Race conditions / vazamento** (crítico ou relevante) — Jobs/filas, locks, listeners, recursos (streams, browser, handles) não liberados.
9. **Arquitetura / separação de camadas** (relevante se quebra grave) — Responsabilidades nas camadas corretas; entrada validada na borda; lógica de domínio fora da camada de apresentação; operações assíncronas onde couber.
12. **Estrutura do projeto** (relevante) — Seguir pastas, naming e organização já usados neste repositório.
13. **Erros e logs** (relevante ou crítico) — Não engolir exceções; logar falhas críticas com contexto; mensagens úteis ao operador.
14. **Mudanças de schema** (crítico ou relevante) — FKs, indexes em colunas filtradas/joinadas, `unique` onde necessário, tipos adequados, rollback/reversão possível.
15. **Complexidade cognitiva** (crítico) — Funções curtas e focadas; poucos níveis de indentação; extrair quando misturar responsabilidades.
16. **Duplicação literal** (crítico) — Strings/números mágicos repetidos → constantes, enums ou config.
17. **Type check** (crítico ou relevante) — Usar o sistema de tipos da stack quando disponível; tipos em parâmetros e retorno; evitar tipos amplos ou dinâmicos sem necessidade.

### Apenas sugestão (não bloquear merge)

8. **Clean Code** — Ver regras detalhadas na seção [Clean Code](#clean-code-critério-8) abaixo. Severidade 🟢 por padrão; violações graves escalam para os critérios 4, 5, 6, 13, 15 e 16.
10. **Convenções do projeto** — Espelhar os idioms e padrões já usados no repositório (naming, organização de imports, estilo de comentários etc.); não inventar convenção nova sem necessidade.
11. **Design Patterns (GoF)** — Usar padrões do catálogo **Gang of Four** sempre que um problema real se encaixar: criacionais (Factory, Builder, Singleton), estruturais (Adapter, Decorator, Facade, Proxy) e comportamentais (Strategy, Observer, Command, Template Method, etc.). Não forçar pattern onde não há problema correspondente; não reinventar abstração só por estética.
18. **Object Calisthenics** — Ver regras detalhadas na seção [Object Calisthenics](#object-calisthenics-critério-18) abaixo. Severidade 🟢 ou 🟡 conforme cada regra; nunca 🔴.
19. **CQS (Command Query Separation)** — Ver regras detalhadas na seção [CQS](#cqs-critério-19) abaixo. Severidade 🟢 apenas.

## Clean Code (critério 8)

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

## Object Calisthenics (critério 18)

Regras de design orientado a objetos. Aplicar com pragmatismo — DTOs e modelos de persistência na borda podem ser exceção; domínio e serviços devem seguir mais de perto.

1. **Um ponto por linha (Lei de Demeter)** — evitar encadeamentos como `pedido.getCliente().getEndereco().getCidade()`; cada objeto fala só com vizinhos diretos. Delegar: `pedido.cidadeDeEntrega()`. *(🟡 relevante — acoplamento e fragilidade; ver critério 6)*
2. **Não abrevie** — nomes completos e claros (`quantidade`, não `qtd`; `gerenciador`, não `gerenc`). Facilita leitura e busca. *(🟢 sugestão — reforça Clean Code critério 8.3)*
3. **Não use a palavra-chave `else`** — preferir early return, guard clauses ou polimorfismo; reduz caminhos condicionais complexos. *(🟢 sugestão — escala para critério 15 se gerar aninhamento grave)*
4. **Máximo de 2 variáveis de instância por classe** — força coesão; se precisar de mais, a classe provavelmente faz coisa demais e deve ser quebrada. *(🟡 relevante — ver critérios 5 e 6)*
5. **Sem getters/setters/properties públicos** — expor comportamento, não estado (`conta.sacar(valor)` em vez de `conta.setSaldo(conta.getSaldo() - valor)`). Tell, Don't Ask. *(🟡 relevante — encapsulamento no domínio)*

## CQS (critério 19)

Command Query Separation — cada método é **ou** Command **ou** Query, nunca os dois. *(🟢 sugestão — reforça Clean Code 8.12 e Object Calisthenics 18.5)*

- **Query** — retorna dados, **não altera estado** (sem efeitos colaterais). Pode ser chamada quantas vezes quiser sem mudar o resultado.
- **Command** — **altera estado** (efeito colateral), **não retorna dado** (idealmente `void` / sem valor de retorno útil).

Violação típica: método que persiste, loga ou muta cache e ainda devolve um valor — separar em query + command.

## Ao escrever código

- Preferir o **menor diff correto**; não refatorar código intocado.
- Se detectar problema crítico fora do escopo pedido, **avisar** sem expandir o escopo sem permissão.
- Em mudanças de schema: avaliar indexes, FKs e constraints em colunas de busca e status.
- Validar entrada na borda do sistema; não confiar em dados externos sem checagem.
- Em consultas a dados: evitar N+1, carregar relações necessárias, paginar ou limitar volumes grandes.
- Quando o código novo tiver variação de comportamento, criação complexa, extensão sem herança ou acoplamento a implementações concretas, avaliar qual padrão GoF resolve o caso antes de improvisar.
- Em classes de domínio: preferir comportamento a getters/setters; evitar train wrecks (um ponto por linha); quebrar classes com mais de 2 campos de instância quando a coesão estiver baixa.
- Separar leitura de escrita: queries sem efeito colateral; commands sem retorno de dado (CQS).
- Convenções específicas de linguagem, framework ou ferramentas: seguir o que já está estabelecido no projeto — não inventar padrões novos sem necessidade.

## Checklist rápido antes de entregar

- [ ] Regra de negócio correta nos fluxos alterados
- [ ] Entrada validada; sem exposição indevida de dados
- [ ] Sem N+1 nem query em loop
- [ ] Edge cases tratados ou conscientemente aceitos
- [ ] Funções com uma responsabilidade; ≤ 3 parâmetros (ou DTO)
- [ ] Nomes autoexplicativos; sem código morto/comentado
- [ ] Aninhamento ≤ 2–3 níveis (early return quando possível)
- [ ] Sem efeitos colaterais escondidos em getters/leitores
- [ ] Literais duplicados extraídos
- [ ] Design Patterns (GoF) avaliados onde há problema real; sem pattern forçado nem abstração por estética
- [ ] Tipos corretos conforme o sistema de tipos disponível
- [ ] Erros logados/tratados onde falha é operacionalmente relevante
- [ ] Mudanças de schema com indexes/constraints adequados (se aplicável)
- [ ] Aderência à estrutura de pastas e idioms já usados no repositório

## Relação com outras skills

| Skill | Uso |
|-------|-----|
| `code-quality` (esta) | Critérios e severidades para escrever ou revisar código |
| `branch-code-review` | Aplica estes critérios no diff da branch atual e gera relatório |
