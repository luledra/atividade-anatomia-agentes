# Análise do trace

Agente: `agent.py` instrumentado com Thought / Action / Observation.

Modelo: `qwen/qwen3.8-27b` via Groq.

Tarefa: `encontre e conserte o bug baseado no teste que está falhando em test_inventory.py`

O trace tem duas execuções da mesma tarefa. Na primeira o agente parou sem editar nada. Na segunda ele editou e parou. O trace completo está no arquivo `trace.log`.

---

## Trace comentado

### System prompt (início da conversa)

```
You can perform actions by emitting a single command line in exactly this format, and nothing else on that line:

tool: NAME({"arg": "value"})

Do not use JSON function-calling, a <tool_call> tag, or any other structured tool-call format your training may default to.
```

>> **Tools / ACI.** A interface entre modelo e ferramentas é texto. O agente descreve as três tools no prompt (nome, docstring e assinatura, geradas por `inspect.signature`) e pede uma linha `tool: nome({...})`. Quem decide se houve chamada é `extract_tool_invocations`, que procura linhas começando com `tool:`, separa no primeiro `(` e passa o resto para `json.loads`.

---

### Execução 1

```
-------------------- ITERACAO 1 --------------------
[THOUGHT] (sem pensamento antes da chamada de tool)
[ACTION] list_files({"path": "."})
[OBSERVATION] tool_result({"path": "/home/luis/ufcg/devia/atividade-anatomia-agentes", "files": [...]})
```

>> **Loop.** Iteração completa: o agente chama o modelo (`execute_llm_call`), procura chamada de tool no texto, executa a tool e devolve o resultado para a conversa. O laço continua porque o parser encontrou uma chamada. Se não encontrasse, pararia (`if not tool_invocations: break`).

>> **Contexto.** O resultado volta como uma mensagem `{"role": "user", "content": "tool_result(...)"}` anexada a `conversation` (linhas 211-214 de `agent.py`). Na iteração seguinte a lista inteira é reenviada ao modelo, então a listagem do diretório passa a fazer parte do que ele vê. É por isso que na iteração 2 ele sabe que `test_inventory.py` existe sem ter perguntado.

```
-------------------- ITERACAO 2 --------------------
[ACTION] read_file({"filename": "test_inventory.py"})
[OBSERVATION] tool_result({"content": "...assert apply_discount(200, 10) == 180\n"})

-------------------- ITERACAO 3 --------------------
[ACTION] read_file({"filename": "inventory.py"})
[OBSERVATION] tool_result({"content": "...return price - percent\n"})
```

>> O agente leu o arquivo de teste e o arquivo que contém a função que faz o teste falhar. Agora, o modelo tem o que precisa para comparar o esperado (180) com o que a função faz (190).

```
-------------------- ITERACAO 4 --------------------
[THOUGHT] (resposta final) Found the bug. `apply_discount` should apply a percentage
discount, but instead it subtracts the raw percentage value (e.g., 200 - 10 = 190)
instead of computing 200 - 200*0.10 = 180.

Let me fix it:

[ACTION] nenhuma: o parser nao encontrou 'tool: nome({...})' -> loop encerrado
```

>> **Thought.** O modelo conecta o valor esperado pelo teste à linha errada do código e nomeia a causa.

>> **Falha de parsing.** O modelo escreveu "Let me fix it:" e não emitiu nenhuma linha de edição. O `extract_tool_invocations` não achou nada, o loop tratou como resposta final e parou. Para continuar o processo, rodei o mesmo prompt novamente.

---

### Execução 2 (mesma tarefa, segundo prompt no mesmo processo)

```
-------------------- ITERACAO 1 --------------------
[THOUGHT] Encontrei o bug. A função `apply_discount` está subtraindo o valor bruto da
porcentagem em vez de calcular o desconto percentual correto.
O teste espera que `apply_discount(200, 10)` retorne `180`, mas a implementação atual
retorna `200 - 10 = 190`.
Vou corrigir o código:
[ACTION] edit_file({"path": ".../inventory.py", "old_str": "    return price - percent", "new_str": "    return price - (price * percent / 100)"})
[OBSERVATION] tool_result({"path": ".../inventory.py", "action": "edited"})
```

>> **Tools / ACI.** A linha `tool: edit_file({...})` foi parseada e os argumentos viraram `edit_file_tool(path, old_str, new_str)`. A tool devolve `"action": "old_str not found"` quando a substituição falha, e isso chega ao modelo como texto dentro de `tool_result(...)`, igual ao caso de sucesso. Quem decide se deu certo é o modelo lendo a string.

>> **Contexto.** O modelo editou na primeira iteração desta rodada, sem ler nenhum arquivo. Isso foi possível porque `conversation` é a mesma lista das duas rodadas: o system prompt aparece uma vez só no trace, e as leituras da execução 1 continuam na conversa. O `old_str` (`    return price - percent`) veio do `tool_result` da iteração 3 da execução anterior.

```
-------------------- ITERACAO 2 --------------------
[THOUGHT] (resposta final) Consertado. O bug estava em `inventory.py`: ...
Agora `apply_discount(200, 10)` retorna `180`, como o teste `test_apply_discount` espera.
[ACTION] nenhuma: o parser nao encontrou 'tool: nome({...})' -> loop encerrado
```

>> **Guardrail (ausente).** O agente afirma que o teste passa. Porém, ele nunca rodou o teste: não existe tool para executar comandos, e nada no loop verifica o resultado da edição. A única condição de parada é "o modelo parou de emitir `tool:`". Dessa forma, as duas execuções terminaram do mesmo jeito (por ausência de chamada de tool), apesar de uma ter consertado o bug e a outra não ter tocado em nenhum arquivo.

---

## Ausência de guardrail

As duas execuções mostram as duas falhas que um guardrail pegaria:

- **Execução 1:** o agente parou achando que ia editar. Não editou. Saiu em estado de sucesso aparente com o bug intacto.
- **Execução 2:** o agente editou e declarou que o teste passa, sem ter rodado nada. A frase "Agora `apply_discount(200, 10)` retorna `180`" é uma previsão apresentada como fato. Se o `old_str` não tivesse batido, a tool teria devolvido `"action": "old_str not found"` e o modelo poderia ter declarado sucesso do mesmo jeito.

Um guardrail mínimo seria uma tool `run_tests` e uma condição de parada que só aceita resposta final depois do teste passar. Sem isso, "terminou" quer dizer apenas "o modelo parou de falar no formato certo".
