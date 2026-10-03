# Análise: anatomia de um agente

## Trace

```
You::encontre e conserte o bug baseado no teste que está falhando em test_inventory.py
[RAW] finish_reason=stop message={'content': 'Vou primeiro listar os arquivos e ler o teste para entender o que está falhando.\n\ntool: list_files({"path": "."})', 'refusal': None, 'role': 'assistant', 'annotations': None, 'audio': None, 'function_call': None, 'tool_calls': None}

===== iteração 1 =====
[THOUGHT] Vou primeiro listar os arquivos e ler o teste para entender o que está falhando.
[PARSE] linhas 'tool:' no texto=1 | reconhecidas pelo parser=1
[ACTION] list_files({"path": "."})
[OBSERVATION] tool_result({"path": "/home/bertoldo/devIA/atividade-anatomia-agentes", "files": [{"filename": "teste_groq.py", "type": "file"}, {"filename": "trace.log", "type": "file"}, {"filename": "README.md", "type": "file"}, {"filename": "inventory.py", "type": "file"}, {"filename": ".gitignore", "type": "file"}, {"filename": ".git", "type": "dir"}, {"filename": "agent.py", "type": "file"}, {"filename": ".pytest_cache", "type": "dir"}, {"filename": "__pycache__", "type": "dir"}, {"filename": ".env", "type": "file"}, {"filename": ".venv", "type": "dir"}, {"filename": "requirements.txt", "type": "file"}, {"filename": "test_inventory.py", "type": "file"}, {"filename": "trace_run1.log", "type": "file"}, {"filename": "trace_run2.log", "type": "file"}]})
```

>> **Thought.** Antes da chamada, o modelo escreve o plano do que vai fazer. Ainda não faz uma tool call.

>> **Tools / ACI.**  O modelo pediu a ação escrevendo uma linha de texto (`tool: list_files({"path": "."})`) e o código achou procurando por `tool:` na resposta. No nativo ela vem num campo da API em JSON, sem precisar procurar em texto. Funcionou porque o modelo escreveu no formato certo.

```
[RAW] finish_reason=stop message={'content': 'tool: read_file({"filename": "test_inventory.py"})', 'refusal': None, 'role': 'assistant', 'annotations': None, 'audio': None, 'function_call': None, 'tool_calls': None}

===== iteração 2 =====
[THOUGHT] (vazio)
[PARSE] linhas 'tool:' no texto=1 | reconhecidas pelo parser=1
[ACTION] read_file({"filename": "test_inventory.py"})
[OBSERVATION] tool_result({"file_path": "/home/bertoldo/devIA/atividade-anatomia-agentes/test_inventory.py", "content": "from inventory import apply_discount\n\n\ndef test_apply_discount():\n    assert apply_discount(200, 10) == 180\n"})
```

>> **Loop.** A iteração 2 faz o ciclo de agente completo. O modelo pede `read_file`, o parser reconhece a chamada, a tool executa e o resultado volta como `[OBSERVATION]`. 

```
[RAW] finish_reason=stop message={'content': 'tool: read_file({"filename": "/home/bertoldo/devIA/atividade-anatomia-agentes/inventory.py"})', 'refusal': None, 'role': 'assistant', 'annotations': None, 'audio': None, 'function_call': None, 'tool_calls': None}

===== iteração 3 =====
[THOUGHT] (vazio)
[PARSE] linhas 'tool:' no texto=1 | reconhecidas pelo parser=1
[ACTION] read_file({"filename": "/home/bertoldo/devIA/atividade-anatomia-agentes/inventory.py"})
[OBSERVATION] tool_result({"file_path": "/home/bertoldo/devIA/atividade-anatomia-agentes/inventory.py", "content": "def apply_discount(price, percent):\n    \"\"\"Aplica um desconto percentual ao preço.\"\"\"\n    return price - percent\n"})
```

>> **Contexto.** O resultado de cada tool entra como uma mensagem `tool_result` e passa a ser vista na proxima chamada. O modelo pede `inventory.py` depois de ler `from inventory import apply_discount` na observação anterior, e usa o caminho que veio no `file_path` dessa observação.

```
[RAW] finish_reason=stop message={'content': 'O bug está em `inventory.py`: a função `apply_discount` subtrai o percent como se fosse um valor absoluto, em vez de calcular o desconto percentual correto.\n\ntool: edit_file({"path": "/home/bertoldo/devIA/atividade-anatomia-agentes/inventory.py", "old_str": "    return price - percent", "new_str": "    return price - (price * percent / 100)"})', 'refusal': None, 'role': 'assistant', 'annotations': None, 'audio': None, 'function_call': None, 'tool_calls': None}

===== iteração 4 =====
[THOUGHT] O bug está em `inventory.py`: a função `apply_discount` subtrai o percent como se fosse um valor absoluto, em vez de calcular o desconto percentual correto.
[PARSE] linhas 'tool:' no texto=1 | reconhecidas pelo parser=1
[ACTION] edit_file({"path": "/home/bertoldo/devIA/atividade-anatomia-agentes/inventory.py", "old_str": "    return price - percent", "new_str": "    return price - (price * percent / 100)"})
[OBSERVATION] tool_result({"path": "/home/bertoldo/devIA/atividade-anatomia-agentes/inventory.py", "action": "edited"})
```

>> **Thought.** Começa com diagnóstico do problema: o modelo conclui que `percent` está sendo tratado como valor absoluto.
>> O `old_str` da edição é exatamente a linha lida na iteração 3, ou seja, veio do resultado da tool.

```
[RAW] finish_reason=stop message={'content': '', 'refusal': None, 'role': 'assistant', 'annotations': None, 'audio': None, 'function_call': None, 'tool_calls': None}

===== iteração 5 =====
[THOUGHT] (vazio)
[PARSE] linhas 'tool:' no texto=0 | reconhecidas pelo parser=0
[WARN] content vazio/None na resposta do modelo (tratado como string vazia)
[NO ACTION] nenhuma chamada de tool reconhecida nesta iteração
[FINAL]
```

Verificação feita por fora do agente, depois da execução:

```
test_inventory.py::test_apply_discount PASSED
1 passed in 0.01s
```

>> **Loop de parada.** O laço parou na iteração 5 porque o parser não achou nenhuma chamada. Ele não parou por ter verificado o resultado, até porque ele não verificou. Não sei por que o modelo devolveu o conteúdo vazio.
>
>> **Guardrail.** O agente editou o arquivo e parou. Não rodou o teste, e nem tem uma tool para fazer isso. Quem rodou o `pytest` fui eu, depois e vi que passou. Ele acertou, mas sem saber que acertou. Parece que para o loop, "editei" e "resolvi" são iguais.
>
>> **Falhas de parsing.** Neste run o parser reconheceu todas as chamadas (1 de 1 nas iterações 1 a 4). A iteração 5 não parece ser falha do parser, mas sim resposta vazia do modelo.
