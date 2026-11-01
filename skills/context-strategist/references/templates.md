# Modelos de `.context/`

Leia este arquivo só ao criar `.context/`. Copie, preencha e apague o que não usar.

## STATE.md (snapshot: reescreva, não anexe; até 60 linhas)

```markdown
# STATE: <projeto>, atualizado <AAAA-MM-DD HH:MM>

## Objetivo
<1–2 linhas: o que é "pronto">

## Escopo
- Dentro: <dirs/módulos>
- Fora (não explorar): <dirs>

## Fase atual
<exploração | plano | implementação | testes | entrega>. Unidade: <nome>

## Feito
- [x] <unidade>: <resultado em 1 linha> (commit <hash> se houver)

## Próximo passo (executável)
1. <ação concreta>, em `<caminho>`, verificar com `<comando>`

## Arquivos do próximo passo
- `<caminho>`: <por quê>

## Bloqueios / perguntas abertas
- <item>

## Verificação
- build: `<comando>`
- teste focado: `<comando>`
```

## MAP.md (até 150 linhas)

```markdown
# MAP: <projeto>

## Entradas
- `<caminho>`: <papel>

## Módulos
| Caminho | Papel | Notas |
|---|---|---|

## Comandos
- instalar / build / testar / rodar: `<...>`

## Convenções
- <padrão de nomes, estilo, onde ficam os testes>
```

## DECISIONS.md (só anexa)

```markdown
- <AAAA-MM-DD> <decisão>. Por quê: <motivo>. Rejeitado: <alternativa>.
```

## DEAD-ENDS.md (só anexa)

```markdown
- <AAAA-MM-DD> Tentei <abordagem>. Falhou porque <causa>. Sinal: <erro/medida>. Não repetir.
```

## PREFS.md

```markdown
- <preferência do usuário, como foi dita>, (<data>)
```

## Handoff / comando de compactação

```
/compact Fase: <próxima>. Retomar lendo .context/STATE.md; próximo passo: <1 linha>.
```

Sessão nova (harness sem /compact):

```
Leia .context/STATE.md (só ele) e execute o "Próximo passo". Use MAP.md/DECISIONS.md/DEAD-ENDS.md só se surgir dúvida.
```

## Pedido a subagente (retorno comprimido)

```
Tarefa: <o que mapear/achar/auditar>. Escopo: <dirs>. Não edite nada.
Retorne no máximo 25 linhas:
- ARQUIVOS: caminho:linha — papel
- FATOS: descobertas, sem narrar o processo
- RISCOS/DÚVIDAS: só o que muda a decisão
Sem colar código, salvo trechos de 10 linhas ou menos.
```
