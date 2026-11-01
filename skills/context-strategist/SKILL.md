---
name: context-strategist
description: Gestão de contexto e tokens para sessões longas e projetos grandes (monorepos, refatorações, migrações, pipelines de várias fases). Trata a janela de contexto como orçamento, guarda o estado em disco (.context/), troca leitura de arquivo inteiro por busca direcionada, delega exploração pesada a subagentes com saída comprimida e compacta só em pontos lógicos, com handoff para retomar depois. Use SEMPRE em tarefas com mais de 1 fase ou mais de 10 arquivos, quando o usuário falar em "projeto grande", "economizar tokens", "contexto cheio", "compactar", "/compact", "sessão longa", "retomar de onde parou", "handoff", ou invocar /context-strategist. Substitui e amplia a strategic-compact.
---

# Context Strategist

A janela de contexto é **orçamento**, não armazenamento. O que precisa durar fica **em disco**. O contexto guarda só o que a etapa atual exige.

Três leis:
1. **Buscar antes de ler.** Nunca abra um arquivo inteiro para achar uma coisa.
2. **O disco é a memória.** Decisão que não está em `.context/` vale como perdida.
3. **Compactar em fronteira, nunca no meio.** Antes de compactar, escrever o handoff.

---

## 1. Início de sessão (máx. 3 chamadas)

1. Se existir `.context/STATE.md` na raiz do projeto, leia **só ele**. Não releia o histórico nem o código "para lembrar".
2. Se não existir e a tarefa for grande (mais de 1 fase ou mais de 10 arquivos), crie `.context/` com os modelos de `references/templates.md` (leia esse arquivo só neste momento).
3. Se houver `graphify-out/`, `ARCHITECTURE.md`, `CLAUDE.md`/`AGENTS.md` ou `.context/MAP.md`, use-os como mapa antes de explorar.

Escolha o **modo** pelo tamanho:

| Escopo | Modo |
|---|---|
| 1–3 arquivos, 1 fase | **Leve**: sem `.context/`, só a disciplina de leitura (§3) |
| 4–30 arquivos ou 2+ fases | **Padrão**: `STATE.md` e `MAP.md` |
| Monorepo, migração, vários dias ou vários agentes | **Pesado**: `.context/` completo, subagentes, um módulo por sessão |

---

## 2. Orçamento por zonas (sem hook automático, você mesmo monitora)

Estimativa: **cerca de 4 caracteres por token**. Referências: 100 linhas de código ≈ 1–1,5k tokens; arquivo de 2.000 linhas ≈ 25k; um `search_game_tree` ou listagem sem limite pode passar de 10k.

Contabilize de cabeça o peso das saídas grandes e aplique a zona correspondente:

| Zona | Uso estimado da janela | Ação |
|---|---|---|
| 🟢 Verde | até 40% | Trabalhe normal, mantendo a disciplina de leitura. |
| 🟡 Amarela | 40–60% | Pare de ler arquivos inteiros. Exploração nova vai para subagente. Atualize `STATE.md`. |
| 🟠 Laranja | 60–75% | Termine a unidade atual, escreva o handoff (§6) e proponha compactar na próxima fronteira. |
| 🔴 Vermelha | acima de 75%, ou respostas lentas, repetidas ou incoerentes | Pare. Escreva o handoff agora e compacte ou abra sessão nova. Não comece nada. |

Gatilhos que **forçam reavaliar a zona** mesmo sem contar: mais de 3 leituras de arquivo grande seguidas, qualquer saída de ferramenta truncada, mais de 40 chamadas desde a última compactação, troca de tarefa ou módulo.

---

## 3. Disciplina de leitura (onde se ganha 80% da economia)

- **grep/glob → read com offset/limit.** Leia a janela de ±30 linhas em volta do que achou, não o arquivo.
- **Nunca releia** o que já está no contexto e não mudou. Se precisar depois de compactar, busque o trecho exato.
- **Limite toda saída:** `head_limit`, `max_depth`, `limit`, `query` no `http_get`, `Select-Object -First N`, `--stat` antes do diff completo, `git log --oneline -20`.
- **Logs e testes:** filtre (`Select-String "FAIL|Error"`, `-Tail 50`). Nunca jogue um log inteiro no contexto.
- **Arquivos gerados** (lockfiles, dist/, minificados, .map, JSON gigante, binários): não leia. Use metadados, contagem ou `jq` com caminho específico.
- **Imagens e screenshots** custam caro. Capture só quando a verificação visual for necessária, e uma por vez.
- **Ao editar:** use edição pontual (`old_string`/`new_string`). Não reescreva o arquivo inteiro e não ecoe o arquivo de volta para "confirmar".

---

## 4. Delegação: deixe o volume fora do seu contexto

Use subagente ou teammate quando a tarefa **lê muito e produz pouco**: mapear um módulo, achar todos os usos de uma API, auditar N arquivos, ler documentação externa, revisar um diff grande.

Peça sempre **retorno comprimido e estruturado**:

```
Retorne no máximo 25 linhas:
- ARQUIVOS: caminho:linha — papel (1 linha cada)
- FATOS: o que descobriu, sem narrar o processo
- RISCOS/DÚVIDAS: só o que muda a decisão
Não cole código, salvo trechos de 10 linhas ou menos que sejam indispensáveis.
```

Trabalho que dá para paralelizar (N módulos independentes) vai para vários subagentes em paralelo, cada um com escopo de escrita disjunto. Você junta só os resumos.

**Não delegue** a edição crítica que depende de nuances que só você tem no contexto.

---

## 5. Estado em disco: `.context/`

| Arquivo | Conteúdo | Limite |
|---|---|---|
| `STATE.md` | objetivo, fase atual, feito, próximo passo concreto, bloqueios | **até 60 linhas**, reescrito (não acumulado) |
| `MAP.md` | `caminho` → papel em 1 linha, pontos de entrada, comandos de build e teste | até 150 linhas |
| `DECISIONS.md` | data, decisão, porquê e alternativa rejeitada | só anexa; entradas de 1–3 linhas |
| `DEAD-ENDS.md` | o que foi tentado, por que falhou e o sinal que provou | evita repetir erro depois de compactar |
| `PREFS.md` | preferências que o usuário disse só na conversa | é o que mais se perde ao compactar |

Regras:
- Atualize `STATE.md` **ao fim de cada unidade de trabalho**, não só antes de compactar.
- Preferência dita pelo usuário vai para `PREFS.md` **na hora**.
- Commits pequenos e frequentes (quando o projeto usar git) são checkpoints baratos. A mensagem do commit também é memória.
- Sugira `.context/` no `.gitignore`, a menos que o time queira versionar.

---

## 6. Protocolo de compactação

**Quando** (fronteiras):

| Transição | Compactar? |
|---|---|
| Exploração → plano | Sim, com o plano salvo em `STATE.md` |
| Plano → implementação | Sim |
| Uma unidade ou módulo concluído → o próximo | Sim, na zona 🟡 ou acima |
| Depuração resolvida → próxima feature | Sim |
| Abordagem falhou → nova tentativa | Sim, depois de registrar em `DEAD-ENDS.md` |
| No meio de uma edição ou de um conjunto de mudanças acopladas | **Nunca** |

**Como** (nesta ordem):
1. Termine ou reverta a unidade em andamento. Não compacte com o código quebrado sem registrar isso.
2. Reescreva `STATE.md`: feito, próximo passo **executável**, arquivos que o próximo passo vai tocar, comandos de verificação.
3. Anexe as decisões e becos sem saída novos.
4. Entregue ao usuário o comando pronto:
   ```
   /compact Fase: <próxima>. Retomar lendo .context/STATE.md; próximo passo: <1 linha>.
   ```
   Se o harness não tiver `/compact`, oriente abrir uma sessão nova com: `Leia .context/STATE.md e continue.`

**Reidratação** depois de compactar: leia `STATE.md`, depois só os arquivos do próximo passo, por busca. `MAP.md`, `DECISIONS.md` e `DEAD-ENDS.md` só quando surgir dúvida.

---

## 7. Projetos grandes: táticas extras

- **Cercar o escopo:** declare no `STATE.md` quais diretórios estão dentro e quais estão fora. Não explore o que está fora.
- **Um módulo por sessão** em migrações e refatorações amplas. A sessão termina com o módulo verde e o `STATE.md` atualizado.
- **Mapa antes de tudo:** se não houver `MAP.md`, um subagente gera em uma passada (estrutura, entradas, comandos). Custa uma vez e economiza em toda sessão seguinte.
- **Testes focados:** rode só o teste do módulo tocado. A suíte completa fica para o fim da fase, com a saída filtrada.
- **Equipe de agentes:** cada teammate recebe uma fatia e um `write_scope` próprio e escreve o próprio resumo em `.context/agents/<nome>.md`. O lead lê só os resumos.
- **Skills e instruções:** carregue skills só quando forem usadas. Não duplique regras entre `CLAUDE.md`/`AGENTS.md` e skills.

---

## 8. Economia na saída

- Resposta = resultado + o mínimo para o usuário agir. Sem recapitular o que ele já viu nem narrar ferramenta por ferramenta.
- Mostre diffs ou trechos, não arquivos inteiros.
- Tabela em vez de prosa quando houver 3 ou mais itens comparáveis.
- Não repita o plano inteiro a cada passo. Aponte para `STATE.md`.

---

## 9. Antipadrões (pare se perceber)

- Ler arquivo inteiro "para entender o contexto".
- Reler arquivos depois de compactar "por segurança".
- Listar a árvore inteira do projeto ou do jogo sem `max_depth`/`head_limit`.
- Colar log, stack trace ou JSON completo quando 10 linhas bastam.
- Compactar sem handoff, ou com handoff vago ("continuar o trabalho").
- Deixar `STATE.md` crescer como diário. Ele é um **snapshot**: reescreva.
- Usar subagente e pedir "relatório detalhado". Isso traz o volume de volta.
- Repetir uma abordagem que já está em `DEAD-ENDS.md`.

---

## Checklist rápido (a cada fronteira)

```
[ ] Unidade atual concluída ou revertida?
[ ] STATE.md reescrito com o próximo passo executável?
[ ] Decisões, becos e preferências novas anotados?
[ ] Zona atual? 🟡 ou acima → propor /compact com o comando pronto
```
