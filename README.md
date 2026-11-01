# Context Strategist

> Uma skill para agentes de IA (Claude Code, Codex, DeepSeek Harness e outros compatíveis com o formato `SKILL.md`) que trata a janela de contexto como **orçamento**: gasta menos tokens, guarda o estado em disco e compacta só em pontos lógicos, para que projetos grandes e sessões longas não percam o fio.

🇺🇸 [English summary below](#english-summary)

---

## Por que usar

Em projetos grandes o agente costuma:
- ler arquivos inteiros para achar uma linha;
- encher o contexto com logs, árvores de diretório e JSON gigante;
- compactar no meio de uma tarefa (ou ser compactado automaticamente) e perder decisões, caminhos e preferências;
- repetir, depois da compactação, abordagens que já tinham falhado.

A **Context Strategist** corrige isso com três leis:

1. **Buscar antes de ler.** grep/glob e depois a leitura de um trecho, nunca o arquivo inteiro.
2. **O disco é a memória.** Estado, decisões, becos sem saída e preferências ficam em `.context/`.
3. **Compactar em fronteira, nunca no meio.** E sempre com handoff escrito antes.

## O que a skill faz

| Recurso | Descrição |
|---|---|
| **3 modos** | Leve (1–3 arquivos), Padrão (4–30 arquivos ou várias fases), Pesado (monorepo, migração, vários dias ou vários agentes). Não cria burocracia em tarefa pequena. |
| **Orçamento por zonas** | 🟢 até 40%, 🟡 40–60%, 🟠 60–75%, 🔴 acima de 75% da janela, cada zona com uma ação definida. Funciona sem hook: o agente estima (cerca de 4 caracteres por token). |
| **Disciplina de leitura** | offset/limit, `head_limit`, logs filtrados, nunca reler, não abrir arquivos gerados, edição pontual em vez de reescrever o arquivo. |
| **Delegação comprimida** | Exploração pesada vai para subagentes, que devolvem no máximo 25 linhas estruturadas (arquivos, fatos, riscos). |
| **Estado em disco** | `.context/STATE.md` (snapshot de até 60 linhas), `MAP.md`, `DECISIONS.md`, `DEAD-ENDS.md`, `PREFS.md`. |
| **Protocolo de compactação** | Tabela de quando compactar, passo a passo do handoff e o comando `/compact` pronto, com a reidratação mínima depois. |
| **Táticas para projetos grandes** | Escopo delimitado, um módulo por sessão, mapa gerado uma vez, testes focados, equipes de agentes com escopo de escrita próprio. |
| **Antipadrões e checklist** | O que fazer o agente parar, e uma checagem de 4 itens a cada fronteira. |

A própria skill é econômica: os modelos de arquivo ficam em `references/templates.md` e só são lidos quando a pasta `.context/` precisa ser criada.

## Estrutura do repositório

```
skills/
└── context-strategist/
    ├── SKILL.md                 # regras da skill (carregado pelo agente)
    └── references/
        └── templates.md         # modelos de .context/ (lido sob demanda)
```

---

## Instalação

### Opção 1: `npx skills` (qualquer agente compatível)

```bash
npx skills add lmennela/context-strategist
```

### Opção 2: Claude Code (manual)

**Pessoal** (vale para todos os projetos):

```bash
# macOS / Linux
git clone https://github.com/lmennela/context-strategist.git /tmp/cs
mkdir -p ~/.claude/skills
cp -r /tmp/cs/skills/context-strategist ~/.claude/skills/
```

```powershell
# Windows (PowerShell)
git clone https://github.com/lmennela/context-strategist.git $env:TEMP\cs
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item "$env:TEMP\cs\skills\context-strategist" "$HOME\.claude\skills\" -Recurse
```

**Só para um projeto** (vai junto no git para o time):

```bash
mkdir -p .claude/skills
cp -r /tmp/cs/skills/context-strategist .claude/skills/
```

Depois abra uma **sessão nova** do Claude Code (as skills são carregadas ao iniciar).

### Opção 3: Codex CLI e outros agentes

Copie `skills/context-strategist/` para a pasta de skills do seu agente, por exemplo `~/.agents/skills/` ou `~/.codex/skills/`, conforme a documentação dele. O formato é o padrão `SKILL.md` com frontmatter `name`/`description`.

### Manter atualizado

Clone o repositório num lugar fixo e use um link no lugar da cópia:

```bash
# macOS / Linux
ln -s ~/src/context-strategist/skills/context-strategist ~/.claude/skills/context-strategist
```

```powershell
# Windows
cmd /c mklink /J "$HOME\.claude\skills\context-strategist" "C:\src\context-strategist\skills\context-strategist"
```

Para atualizar, rode `git pull` no clone.

---

## Como usar

### Ativação

- **Explícita:** `/context-strategist`
- **Automática:** o agente aciona a skill quando você fala em "projeto grande", "economizar tokens", "contexto cheio", "compactar", "sessão longa", "retomar de onde parou" ou "handoff", ou quando a tarefa tem mais de 1 fase ou mais de 10 arquivos.

### Exemplos de prompt

Começar um projeto grande:

```text
/context-strategist — modo pesado. Vamos migrar o backend de Express para Fastify
(~40 rotas em src/routes). Crie o .context/, gere o MAP.md com um subagente e
proponha o plano por módulo.
```

Ao terminar uma fase:

```text
/context-strategist — terminamos a fase de exploração. Atualize o .context/ e me dê o /compact pronto.
```

Retomar em sessão nova (ou depois do `/compact`):

```text
Leia .context/STATE.md (só ele) e execute o "Próximo passo".
```

### O ciclo na prática

```
início → lê .context/STATE.md (ou cria .context/)
   ↓
trabalha por unidades → atualiza STATE.md ao fim de cada uma
   ↓
zona 🟡/🟠 ou fronteira de fase → handoff (STATE, DECISIONS, DEAD-ENDS, PREFS)
   ↓
/compact Fase: <próxima>. Retomar lendo .context/STATE.md; próximo passo: <...>
   ↓
reidrata: STATE.md e depois só os arquivos do próximo passo
```

### A pasta `.context/`

| Arquivo | Para quê | Regra |
|---|---|---|
| `STATE.md` | objetivo, escopo, fase, feito, próximo passo executável | snapshot de até 60 linhas, **reescrito** |
| `MAP.md` | mapa do código: caminhos, papéis, comandos | até 150 linhas, gerado uma vez |
| `DECISIONS.md` | decisão, porquê e alternativa rejeitada | só anexa |
| `DEAD-ENDS.md` | o que falhou e o sinal que provou | evita repetir erros depois de compactar |
| `PREFS.md` | preferências ditas na conversa | anote na hora, é o que mais se perde |

Recomendação: adicione `.context/` ao `.gitignore`, a menos que o time queira versionar o estado.

---

## Perguntas frequentes

**Preciso de hook ou script?**
Não. O agente estima o uso de contexto pelas regras da skill. Se seu ambiente já tem um hook de aviso de compactação, os dois convivem bem.

**E se meu agente não tiver `/compact`?**
A skill orienta abrir uma sessão nova com `Leia .context/STATE.md e continue.` O resultado é o mesmo.

**Funciona em tarefas pequenas?**
Sim, no modo Leve: só a disciplina de leitura, sem criar `.context/`.

**Qual a relação com a `strategic-compact`?**
É uma evolução. A `strategic-compact` diz *quando* compactar. A Context Strategist cobre também *como gastar menos*, *onde guardar o estado* e *como retomar*.

---

## English summary

**Context Strategist** is an agent skill (standard `SKILL.md` format; works with Claude Code, Codex, and compatible agents) for **token-efficient work on large projects and long sessions**. The skill text is written in Portuguese, and agents follow it regardless of the conversation language.

- **Search before reading:** grep/glob, then read a slice. Cap every output, filter logs, never re-read.
- **Disk is memory:** `.context/` holds `STATE.md` (60-line snapshot), `MAP.md`, `DECISIONS.md`, `DEAD-ENDS.md`, `PREFS.md`.
- **Budget zones:** 🟢 <40%, 🟡 40–60%, 🟠 60–75%, 🔴 >75% of the window, each with a defined action. No hook required.
- **Compressed delegation:** subagents explore and return 25 lines or fewer.
- **Compaction protocol:** compact only at phase boundaries, after writing a handoff. A ready-made `/compact` command is provided, with minimal rehydration afterwards.

**Install (Claude Code):**

```bash
npx skills add lmennela/context-strategist
# or
git clone https://github.com/lmennela/context-strategist.git /tmp/cs && cp -r /tmp/cs/skills/context-strategist ~/.claude/skills/
```

Start a new session and invoke `/context-strategist`.

---

## Licença

[MIT](LICENSE). Use, modifique e compartilhe.

Contribuições são bem-vindas: abra uma issue ou um PR.
