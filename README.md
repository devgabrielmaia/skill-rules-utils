# skill-utils

Coleção de **skills** e **rules** para agentes de IA (Cursor e compatíveis). O conteúdo fica em `agent-resources/` para você decidir onde e como conectá-lo à ferramenta que estiver usando.

## Estrutura

```
agent-resources/
├── skills/          # Skills do agente (cada pasta contém um SKILL.md)
│   ├── branch-code-review/
│   ├── code-quality/
│   ├── pr-description/
│   ├── project-conventions/
│   └── project-docs/
└── rules/           # Regras persistentes (.mdc)
    ├── project-conventions.mdc
    └── no-auto-tests.mdc
```

## Como usar

A pasta `agent-resources` **não precisa** ficar com esse nome nem nesse caminho. O que importa é que o conteúdo chegue nos diretórios que a sua IA espera.

### Opção 1: Link simbólico (recomendado)

Crie links simbólicos apontando para este repositório. Assim, atualizações aqui refletem automaticamente no agente.

**Skills e rules no projeto** (compartilhadas com quem clona o repo):

### Opção 2: Renomear ou copiar a pasta

Se preferir não usar symlinks, renomeie ou copie o conteúdo para o local esperado pela ferramenta.

## Adaptação
Talvez precise adaptar as rules para a ferramenta que estiver usando.
As Skills são padrão pra todos (eu acho.)
