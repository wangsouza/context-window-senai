# Janela de Contexto e Eficiência de Custos — Aula 2

Materiais práticos da Aula 2 da disciplina **Janela de Contexto e Eficiência de Custos** (SENAI), cujo tema é **Limitações do Contexto Longo, Confiabilidade e Avaliação Experimental**.

## Sumário

- [Apresentação](#apresentação)
- [Conteúdo do repositório](#conteúdo-do-repositório)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
- [Configuração das APIs](#configuração-das-apis)
- [Como executar os experimentos](#como-executar-os-experimentos)
- [Custos, limites e cuidados](#custos-limites-e-cuidados)
- [Solução de problemas](#solução-de-problemas)
- [Referências](#referências)

## Apresentação

Os notebooks deste repositório permitem executar experimentos reais com LLMs e observar como **tamanho, posição, relevância e conflitos entre informações** afetam as respostas. Todos os cenários e documentos são fictícios e foram criados para fins didáticos.

Ao final das atividades, espera-se que você consiga:

- distinguir a capacidade nominal da janela de contexto da qualidade observada no uso do contexto;
- controlar a posição de uma evidência mantendo os demais fatores constantes;
- comparar contexto completo e contexto selecionado quanto a tokens, latência e qualidade da resposta;
- avaliar o efeito de documentos irrelevantes e de fontes revogadas;
- formular conclusões proporcionais às observações, reconhecendo as limitações de cada experimento.

## Conteúdo do repositório

| Arquivo | Descrição |
|---|---|
| `Laboratorio_Lost_in_the_Middle_Aula_02_Alunos_Gemini_2.ipynb` | Demonstração *Lost in the Middle* (Gemini ou OpenAI). |
| `Laboratorios_Aula_02_Contexto_Ruido_Conflitos_Alunos.ipynb` | Laboratórios 1 e 2: contexto, ruído e conflitos (somente Gemini). |
| `env.example` | Modelo do arquivo de variáveis de ambiente `.env`. |
| `requirements.txt` | Dependências Python utilizadas pelos notebooks. |
| `resultados_lab_aula02/` | Pasta de checkpoints e CSV dos Laboratórios 1 e 2. É criada pelo notebook e está no `.gitignore`. |

### Demonstração — Lost in the Middle

**Arquivo:** `Laboratorio_Lost_in_the_Middle_Aula_02_Alunos_Gemini_2.ipynb`

**Pergunta de investigação:** quando uma informação está dentro da janela de contexto, o modelo a utiliza com a mesma confiabilidade independentemente da posição em que ela aparece?

#### Conceitos

| Conceito | Significado |
|---|---|
| Contexto nominal | Capacidade máxima de tokens admitida pelo modelo ou serviço. |
| Uso efetivo do contexto | Qualidade observada ao recuperar e usar informações em uma tarefa específica; não é um limite universal do modelo. |
| *Lost in the Middle* | Padrão descrito na literatura em que a informação posicionada no meio do contexto tende a ser menos utilizada do que a posicionada nas extremidades. |
| *Primacy* e *recency* | Maior influência de informações próximas ao início e ao final da sequência, respectivamente. |
| *Needle in a Haystack* | Família de testes em que uma informação específica (a "agulha") é inserida em um contexto longo de textos irrelevantes (o "palheiro"). O notebook segue essa lógica: um documento relevante entre dezenas de distratores. O notebook não usa esse nome explicitamente. |

#### Como o experimento é organizado

- **Cenário:** *runbooks* técnicos fictícios de serviços, ambientes e versões. A tarefa é encontrar a configuração correta e citar o ID da fonte.
- **Casos (`CASOS`):** três perguntas, cada uma com resposta de referência.

  | Caso | Serviço | Campo | Valor esperado | Fonte |
  |---|---|---|---|---|
  | Timeout em produção | Checkout Orion | `read_timeout` | 17 segundos | OPS-227 |
  | Retenção em produção | Ledger Aurora | `retention_days` | 43 dias | OPS-314 |
  | Lote em produção | Notify Pulsar | `max_batch_size` | 275 eventos | OPS-502 |

- **Documentos (`criar_documentos`):** um documento relevante e `N_DOCUMENTOS - 1` distratores. Os distratores são deliberadamente parecidos (mesmo serviço em homologação, mesmo serviço em versão anterior, ou outro serviço com a mesma versão) e nunca repetem a combinação exata de serviço, ambiente e versão do alvo.
- **Posições (`POSICOES`):** `Início`, `Meio` e `Final`. A função `ordenar_documentos` troca o documento relevante de lugar com o documento da posição-alvo, sem acrescentar ou remover fontes. Com 60 documentos, as posições são 1, 31 e 60.
- **Controles internos:** asserções verificam que o conjunto de IDs é idêntico nas três condições, que existe um único documento relevante e que os prompts têm o mesmo número de caracteres.
- **Isolamento:** cada posição é uma requisição independente, sem histórico compartilhado.

#### Avaliação

A função `avaliar_resposta` extrai o JSON retornado pelo modelo e calcula:

| Métrica | Significado |
|---|---|
| `acerto_exato` | Valor e unidade iguais à referência (comparação normalizada, ignorando acentos, caixa e pontuação final). |
| `fonte_correta` | O ID citado é igual ao da fonte de referência. |
| `acerto_e_fonte` | Ambas as condições acima. É a métrica usada no gráfico por posição. |
| `tokens_entrada` / `tokens_saida` | Contagens reportadas pela API. |
| `segundos` | Latência total da requisição, incluindo rede e serviço. |
| `status` | `completed` ou `incomplete`. Respostas incompletas (por exemplo, por limite de saída) ficam fora do cálculo de acerto. |

Há ainda dois **controles opcionais**: *Oracle* (apenas o documento correto) e *Sem evidência* (somente distratores; a resposta esperada é `NAO_ENCONTRADO`).

### Laboratórios 1 e 2 — Contexto, Ruído e Conflitos

**Arquivo:** `Laboratorios_Aula_02_Contexto_Ruido_Conflitos_Alunos.ipynb`

**Pergunta fixa (`PERGUNTA`):** "Comprei um curso há 9 dias e tive problemas de acesso. Ainda posso solicitar reembolso?"

O cenário usa sete documentos fictícios (`DOCUMENTOS`) de uma plataforma de cursos. Os principais são **D1** (regra geral de 7 dias) e **D4** (exceção vigente de 15 dias para falha comprovada de acesso, mediante evidências). **D6** é uma cópia de D1 e **D7** é uma política revogada que fixava 7 dias. D2, D3 e D5 são irrelevantes para a pergunta.

São quatro cenários (`CENARIOS`), cada um correspondente a uma chamada ao LLM:

| Código | Experimento | Documentos enviados |
|---|---|---|
| `L1_T1` | Laboratório 1, Teste 1: contexto completo | D1, D2, D3, D4, D5, D6 |
| `BASE` | Laboratório 1, Teste 2 e Laboratório 2-A: selecionado (baseline) | D1, D4 |
| `RUIDO` | Laboratório 2-B: com ruído | D1, D4, D2, D3, D5 |
| `CONFLITO` | Laboratório 2-C: com versão revogada | D1, D4, D7 |

- **Laboratório 1** testa a hipótese de que remover documentos irrelevantes e duplicados reduz o consumo sem prejudicar a resposta.
- **Laboratório 2** testa se documentos irrelevantes ou uma política antiga interferem no uso das evidências, ou seja, se o modelo distingue **informação encontrada** de **política vigente**.

#### Métricas

| Métrica | Origem |
|---|---|
| Tokens de entrada e de saída | Reportados pela API (`tokens_entrada`, `tokens_saida`). |
| Latência (s) | Medida por `time.perf_counter()` em cada chamada (`latencia_s`). Uma única chamada não comprova qual estratégia é mais rápida. |
| Fontes citadas | Extraídas automaticamente da resposta por expressão regular (`D1` a `D7`). |
| Economia de entrada | Calculada entre `L1_T1` e `BASE`, com uma execução por cenário. |
| Correção (0-2) e Fundamentação (0-2) | **Preenchidas manualmente** no CSV. O notebook não corrige as respostas. |

Critério das notas: **2** aplica corretamente a política e cita as fontes vigentes; **1** é parcial ou pouco rastreável; **0** contradiz a política vigente, inventa regra ou cita documento inadequado como vigente. A resposta de referência está no final do notebook.

## Pré-requisitos

- **Python 3.10 ou superior** (os notebooks foram salvos com o kernel Python 3.12.3).
- **VS Code** com as extensões Python e Jupyter, **ou** JupyterLab.
- **Git**.
- **Conta e chave de API** de ao menos um provedor:
  - Gemini (Google AI Studio): necessário para os dois notebooks.
  - OpenAI: opcional, apenas para a demonstração *Lost in the Middle*.

## Instalação

Os comandos abaixo são para Linux/Ubuntu.

**1. Instale os pacotes do sistema (se necessário).**

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-pip
```

**2. Clone o repositório.**

```bash
git clone https://github.com/wangsouza/context-window-senai.git
```

**3. Acesse o diretório do projeto.**

```bash
cd context-window-senai
```

**4. Crie o ambiente virtual.**

```bash
python3 -m venv .venv
```

**5. Ative o ambiente virtual.**

```bash
source .venv/bin/activate
```

**6. Instale as dependências.**

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

| Biblioteca | Uso |
|---|---|
| `openai` | Cliente das APIs. Também é usado para o Gemini, por meio do endpoint compatível com OpenAI. Não é necessário um pacote separado do Google. |
| `python-dotenv` | Leitura do arquivo `.env`. |
| `numpy`, `pandas` | Manipulação e exibição das tabelas de resultados (`numpy` é usado apenas na demonstração). |
| `matplotlib` | Gráficos. |
| `ipykernel` | Kernel Jupyter para o ambiente virtual (também fornece `IPython.display`). |

O ambiente em que os notebooks foram desenvolvidos usava `openai` 3.40.0, `pandas` 3.1.0, `numpy` 2.5.4, `matplotlib` 3.11.2, `python-dotenv` 1.2.4 e `ipykernel` 7.4.0. O `requirements.txt` define apenas versões mínimas.

**7. Configure o arquivo `.env`** conforme a seção [Configuração das APIs](#configuração-das-apis).

**8. Abra os notebooks e selecione o kernel do ambiente virtual.**

*VS Code:*

```bash
code .
```

Abra o notebook, clique em **Select Kernel** (canto superior direito), escolha **Python Environments** e selecione o interpretador `.venv`.

*JupyterLab (alternativa):*

```bash
python -m pip install jupyterlab
python -m ipykernel install --user --name context-window-senai --display-name "Python (context-window-senai)"
jupyter lab
```

Ao abrir o notebook, selecione o kernel **Python (context-window-senai)**.

> Inicie o VS Code ou o JupyterLab a partir da raiz do projeto. Os notebooks procuram o `.env` e gravam os resultados no diretório de trabalho (`Path.cwd()`), que normalmente é a pasta do notebook.

## Configuração das APIs

Crie o `.env` a partir do modelo e preencha somente as chaves do provedor que pretende usar:

```bash
cp env.example .env
```

Edite o `.env` com um editor de texto (por exemplo, `nano .env`).

### Variáveis utilizadas

| Variável | Valor padrão no `env.example` | Usada em | Descrição |
|---|---|---|---|
| `LLM_PROVIDER` | `gemini` | Somente *Lost in the Middle* | Provedor: `gemini` ou `openai`. Qualquer outro valor interrompe a execução com `ValueError`. |
| `GEMINI_API_KEY` | vazio | Ambos | Chave do Google AI Studio. |
| `GEMINI_MODEL` | `gemini-3.5-flash-lite` | Ambos | Modelo Gemini. Confirme a disponibilidade na sua conta. |
| `OPENAI_API_KEY` | vazio | Somente *Lost in the Middle* (com `LLM_PROVIDER=openai`) | Chave da OpenAI. |
| `OPENAI_MODEL` | `gpt-5-mini` | Somente *Lost in the Middle* (com `LLM_PROVIDER=openai`) | Modelo OpenAI. |

Exemplo para usar Gemini (não inclua chaves reais em nenhum arquivo versionado):

```env
LLM_PROVIDER=gemini
GEMINI_API_KEY=sua_chave_aqui
GEMINI_MODEL=gemini-3.5-flash-lite
```

### Suporte a provedores por notebook

| Notebook | Gemini | OpenAI | Como seleciona o provedor |
|---|:---:|:---:|---|
| Lost in the Middle | Sim | Sim | Variável `LLM_PROVIDER` do `.env`. |
| Laboratórios 1 e 2 | Sim | Não | Fixo em Gemini; ignora `LLM_PROVIDER`. |

Detalhes de implementação:

- **Gemini:** usa o cliente `OpenAI` com `base_url="https://generativelanguage.googleapis.com/v1beta/openai/"` e `client.chat.completions.create`.
- **OpenAI (somente demonstração):** usa a Responses API (`client.responses.create`); para modelos que começam com `gpt-5`, define `reasoning={'effort': 'low'}`.
- Usar OpenAI nos Laboratórios 1 e 2 exigiria alterar o código do notebook.

### Onde obter as chaves

| Provedor | Link |
|---|---|
| Google AI Studio (Gemini) | <https://aistudio.google.com/apikey> |
| OpenAI Platform | <https://platform.openai.com/api-keys> |

Documentação do endpoint de compatibilidade do Gemini com a biblioteca OpenAI: <https://ai.google.dev/gemini-api/docs/openai>.

> **Nunca** coloque a chave no código, no README ou em commits. O `.env` já consta no `.gitignore`.

## Como executar os experimentos

### Atenção ao valor inicial de `EXECUTAR_API`

Nos dois notebooks, a variável `EXECUTAR_API` está salva como `True`. Isso significa que, com uma chave configurada, **as chamadas reais são feitas assim que as células correspondentes são executadas**. Para apenas inspecionar o material sem consumir a API, altere para `EXECUTAR_API = False` antes de executar as demais células.

### Demonstração — Lost in the Middle

Arquivo: `Laboratorio_Lost_in_the_Middle_Aula_02_Alunos_Gemini_2.ipynb`

**Parâmetros da célula de configuração**

| Variável | Valor salvo | Efeito |
|---|---|---|
| `EXECUTAR_API` | `True` | Habilita as chamadas reais. Com `False`, nada é simulado. |
| `N_DOCUMENTOS` | `60` | Documentos por contexto (24 para a demonstração inicial; 60 ou 90 para o desafio). |
| `PARAGRAFOS_POR_DOCUMENTO` | `7` | Aumenta o tamanho de cada documento e do contexto. |
| `SEMENTE` | `42` | Semente da geração dos distratores. |
| `MAX_OUTPUT_TOKENS` | `1200` | Limite de saída; pode incluir tokens de raciocínio. |
| `REPETICOES` | `3` | Repetições por caso e posição. |
| `QUANTOS_CASOS` | `3` | Quantidade de perguntas de `CASOS` usadas. |
| `EXECUTAR_CONTROLES` | `True` | Habilita os controles opcionais (célula do passo 8). |

**Número de chamadas:** `QUANTOS_CASOS * REPETICOES * 3`. Com os valores salvos são **27 chamadas** de contexto longo, mais **2** dos controles. Para um teste inicial de 3 chamadas, use `QUANTOS_CASOS = 1` e `REPETICOES = 1`.

**Ordem recomendada**

1. Execute a célula de configuração e confira o provedor, o modelo e o número de chamadas impresso (`Chamadas piloto`).
2. Execute a célula de `CASOS` e `criar_documentos` e a de `ordenar_documentos`/`montar_entrada`. Elas geram `docs_base`, `arranjos`, `prompts` e a tabela `controle_df` com a posição do alvo em cada condição. As asserções devem passar sem erro.
3. Execute o diagrama de posições e a célula de auditoria (IDs do primeiro, do último e do alvo em cada condição).
4. Execute a célula que define `CHAVE_API`, `TEM_CHAVE`, `SDK_DISPONIVEL` e `criar_cliente`, e confira a mensagem de chave e SDK.
5. Execute a célula de `avaliar_resposta`. Os autotestes são locais e não usam a API.
6. Execute a célula que define `executar_chamada` e `montar_plano` e confira o tamanho do plano.
7. Execute a célula de execução principal, que é **a única que faz as chamadas dos testes principais**. Há uma pausa fixa de 25 segundos após cada chamada, portanto 27 chamadas levam bem mais de 11 minutos.
8. Execute as células de análise: gráfico de acerto por posição e tabela de tokens e latência.
9. Opcional: execute os controles (`EXECUTAR_CONTROLES`), que fazem 2 chamadas adicionais. A célula de planejamento por tamanho de contexto (10, 30 e 60 documentos) apenas estima o volume de texto e não chama a API.

**Resultados produzidos** (apenas em memória e na tela, sem gravação em disco):

- `resultados_df`: uma linha por chamada, com resposta bruta, métricas de acerto, tokens, latência e status.
- Gráfico de `acerto_e_fonte` por posição, com `n` válido em cada barra.
- Tabela de tokens, latência e status.
- `controles`: resultado do Oracle e da condição sem evidência.

**Como interpretar**

- Compare `acerto_e_fonte` entre `Início`, `Meio` e `Final` **para o mesmo caso e o mesmo tamanho de contexto**.
- Um acerto com fonte incorreta não é bem fundamentado.
- Se todas as posições acertarem, registre o resultado normalmente. O padrão em U não é esperado nem exigido.
- O gráfico por posição só inclui chamadas com `status == 'completed'`. Poucas execuções (menos de 9) geram resultado descritivo, não uma taxa de acerto robusta.
- Na condição **Sem evidência**, o avaliador automático compara a resposta com o valor do caso, não com `NAO_ENCONTRADO`. Leia `resposta_bruta` manualmente: responder `NAO_ENCONTRADO` é o comportamento correto, mesmo que as colunas de acerto mostrem `False`.

**Como evitar execuções repetidas desnecessárias**

Este notebook **não possui checkpoint**. Reexecutar a célula de execução principal repete todas as chamadas e perde `resultados_df` ao reiniciar o kernel. Para evitar custo desnecessário:

- teste primeiro com `QUANTOS_CASOS = 1` e `REPETICOES = 1`;
- não reexecute a célula de execução principal sem necessidade;
- salve ou copie a tabela exibida antes de reiniciar o kernel.

### Laboratórios 1 e 2 — Contexto, Ruído e Conflitos

Arquivo: `Laboratorios_Aula_02_Contexto_Ruido_Conflitos_Alunos.ipynb`

**Parâmetros da célula de configuração**

| Variável | Valor salvo | Efeito |
|---|---|---|
| `MODELO` | `GEMINI_MODEL` do `.env` (padrão `gemini-3.5-flash-lite`) | Modelo usado nas chamadas. |
| `EXECUTAR_API` | `True` | Habilita as chamadas reais. |
| `PAUSA_SEGUNDOS` | `12` | Intervalo entre requisições novas dentro de um grupo. |

**Ordem recomendada**

1. Execute a célula `%pip -q install openai python-dotenv pandas matplotlib`. Ela é dispensável se você já instalou o `requirements.txt` no ambiente selecionado.
2. Execute a célula de configuração (`MODELO`, `EXECUTAR_API`, `PAUSA_SEGUNDOS`) e confira a linha impressa com modelo, presença da chave e estado da API.
3. Execute a célula que define `DOCUMENTOS`, `INSTRUCAO`, `CENARIOS` e as funções `contexto`, `prompt`, `resultado_arquivo`, `executar` e `executar_grupo`. Ela também exibe os sete documentos em blocos expansíveis.
4. **Laboratório 1:** execute `RESULTADO_LAB1 = executar_grupo(["L1_T1", "BASE"])` (2 chamadas).
5. **Laboratório 2:** execute `RESULTADO_LAB2 = executar_grupo(["BASE", "RUIDO", "CONFLITO"])`. `BASE` é reaproveitado do checkpoint do Laboratório 1, portanto apenas 2 chamadas novas são feitas.
6. Execute a célula de comparação. Ela monta a tabela, grava o CSV e gera o gráfico de tokens de entrada.
7. Abra o CSV, preencha **Correção (0-2)**, **Fundamentação (0-2)** e **Observações** e responda às quatro perguntas de entrega do notebook.

No total, são **4 chamadas** (`L1_T1`, `BASE`, `RUIDO`, `CONFLITO`).

**Resultados produzidos** em `resultados_lab_aula02/`:

| Arquivo | Conteúdo |
|---|---|
| `<codigo>_<hash>.json` | Um checkpoint por cenário: resposta, fontes, `citacoes`, `tokens_entrada`, `tokens_saida`, `latencia_s`, `modelo`, `executado_em`. |
| `comparacao_laboratorios_aula02.csv` | Tabela comparativa com colunas em branco para sua avaliação manual. |

**Como interpretar**

- Compare `Tokens entrada` entre `Completo` e `Selecionado` para estimar a economia de contexto (o notebook imprime o percentual com base em uma execução por cenário).
- Verifique se todas as respostas reconhecem a exceção de 15 dias de **D4**, mencionam a necessidade de evidências e distinguem *poder solicitar* de *aprovação automática*.
- Em `CONFLITO`, verifique se **D7** (revogado) foi tratado como histórico e não como regra vigente.
- Não conclua que uma estratégia é mais rápida a partir de uma única medição de latência.
- Se todos os cenários estiverem corretos, o resultado continua válido: analise a economia de contexto.

**Como evitar execuções repetidas desnecessárias**

- O notebook usa **checkpoints**: antes de qualquer chamada, `executar` procura o arquivo `resultado_arquivo(codigo)` e, se existir, reutiliza a resposta com `origem = "checkpoint"`, mesmo com `EXECUTAR_API = False`.
- O nome do arquivo inclui um hash de modelo, instrução, pergunta e documentos. Alterar qualquer um desses itens (inclusive `GEMINI_MODEL`) gera um novo checkpoint e uma nova chamada.
- Para refazer um cenário com os mesmos parâmetros, é preciso remover manualmente o JSON correspondente em `resultados_lab_aula02/`. Isso consome nova cota.
- Ao reutilizar um checkpoint, `executar` regrava o JSON com o campo `citacoes` recalculado a partir da resposta.

## Custos, limites e cuidados

- **Cada chamada consome tokens** e pode gerar custo ou contar contra os limites de uso (cota gratuita ou paga) do provedor. O notebook não calcula valores em dinheiro.
- **Contextos longos consomem muitos tokens.** Na demonstração, cada chamada envia dezenas de documentos; a configuração salva (60 documentos, 7 parágrafos cada, 27 chamadas) é bem mais pesada do que os Laboratórios 1 e 2. Comece com 3 chamadas.
- **Limites de taxa:** erros como `429 RESOURCE_EXHAUSTED` indicam cota esgotada. Consulte as cotas no AI Studio e aguarde ou reduza o experimento. As pausas entre chamadas (25 s na demonstração, `PAUSA_SEGUNDOS = 12` nos laboratórios) ajudam a respeitá-los.
- **Verifique modelos e preços** na documentação do provedor antes de aumentar a escala. Respostas podem variar entre execuções mesmo sem alterar parâmetros.
- **Checkpoints:** somente os Laboratórios 1 e 2 os possuem (veja a seção anterior). A demonstração *Lost in the Middle* não grava resultados em disco.
- **Segredos:** o arquivo `.env` contém sua chave e **não deve ser enviado ao GitHub**. Ele já está listado no `.gitignore`; confirme com `git status` antes de cada commit. Se uma chave for exposta, revogue-a no painel do provedor e gere outra.
- A pasta `resultados_lab_aula02/` também está no `.gitignore`.

## Solução de problemas

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `ModuleNotFoundError` | Kernel diferente do ambiente `.venv` | Selecione o kernel correto e reinstale com `python -m pip install -r requirements.txt`. |
| `Chave presente: False` ou "Configure GEMINI_API_KEY no .env" | `.env` ausente, fora da pasta de execução ou sem a chave | Crie o `.env` na pasta de trabalho do notebook e reinicie o kernel. |
| `ValueError: LLM_PROVIDER deve ser gemini ou openai` | Valor inválido no `.env` | Use `gemini` ou `openai`. |
| `429 RESOURCE_EXHAUSTED` | Cota esgotada | Aguarde, reduza `QUANTOS_CASOS` e `REPETICOES` ou verifique o plano. |
| Erro de modelo não encontrado | `GEMINI_MODEL` ou `OPENAI_MODEL` indisponível na conta | Ajuste o modelo no `.env`. |
| Status `incomplete` | Resposta cortada por `MAX_OUTPUT_TOKENS` | Aumente o limite ou analise a resposta bruta; não conte como erro de recuperação. |

## Referências

- Liu, N. F. et al. (2024). **Lost in the Middle: How Language Models Use Long Contexts.** *Transactions of the Association for Computational Linguistics*, 12, 157-173. <https://aclanthology.org/2024.tacl-1.9/> (versão no arXiv: <https://arxiv.org/abs/2307.03172>). Código dos autores: <https://github.com/nelson-liu/lost-in-the-middle>.
- Hsieh, C.-P. et al. (2024). **RULER: What's the Real Context Size of Your Long-Context Language Models?** <https://arxiv.org/abs/2404.06654>. Código: <https://github.com/NVIDIA/RULER>.
- Google. **Compatibilidade do Gemini API com a biblioteca OpenAI.** <https://ai.google.dev/gemini-api/docs/openai>.
- OpenAI. **Responses API.** <https://platform.openai.com/docs/api-reference/responses>.

> Os notebooks adaptam a **metodologia** dos artigos com dados sintéticos. Eles não reproduzem os resultados quantitativos publicados.
