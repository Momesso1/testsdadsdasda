Ollama Client (LLM Interface)
=============================

O ``OllamaClient`` é a classe de baixo nível responsável por gerenciar a comunicação HTTP com o servidor de inferência local (Ollama).

Sua principal responsabilidade é abstrair a complexidade do protocolo de rede (via **libcurl**) e garantir que o modelo de linguagem (Phi 3.5) receba um contexto estrito para gerar saídas determinísticas em formato JSON.



Prompt inicial
-------------------------------

Para garantir que a LLM gere um JSON válido e compatível com o parser do sistema, um prompt inicial é passado para a LLM.
O prompt não apenas pede o JSON, mas fornece exemplos explícitos de como traduzir frases para a estrutura de comandos e loops esperada.

Isso "treina" o modelo em tempo de execução para entender a sintaxe proprietária de loops (ex: ``box_01:box_03``) e variáveis (ex: ``$item``).

.. literalinclude:: ../../../../../docs_central/legacy_src/llms/OllamaClient/ollama_client_v1.0.hpp
   :language: cpp
   :start-after: // DOC-START: Prompt_Engineering
   :end-before: // DOC-END: Prompt_Engineering

Configuração da Inferência
--------------------------

O método ``infer`` constrói a requisição POST para o endpoint ``/api/generate``.
Para aplicações robóticas, a criatividade da IA deve ser suprimida em favor da precisão. Por isso, os seguintes parâmetros são forçados:

* **Model:** ``phi35_leve`` (Versão quantizada do Phi 3.5 para velocidade).
* **Format:** ``json`` (Força a saída a ser um objeto JSON válido).
* **Temperature:** ``0.0`` (Remove aleatoriedade na amostragem).
* **Top_P:** ``0.1`` (Considera apenas os tokens mais prováveis).
* **Stream:** ``false`` (Aguarda a resposta completa antes de processar).

.. literalinclude:: ../../../../../docs_central/legacy_src/llms/OllamaClient/ollama_client_v1.0.hpp
   :language: cpp
   :start-after: // DOC-START: Infer_Method
   :end-before: // DOC-END: Infer_Method

Parsing de Resposta
-------------------

O Ollama retorna um JSON "envelope" contendo metadados (tempo de inferência, contagem de tokens) e o texto gerado.
O cliente realiza um parsing em duas etapas:

1.  Decodifica a resposta HTTP para extrair o campo ``response`` (que é uma string contendo o JSON gerado pela LLM).
2.  Decodifica essa string interna para retornar um objeto ``nlohmann::json`` estruturado para o *Brain Node*.

Ciclo de Vida (cURL)
--------------------

A classe gerencia o ciclo de vida da biblioteca ``libcurl``, inicializando o ambiente global no construtor e liberando recursos no destrutor, garantindo que não haja vazamentos de memória nas requisições de rede.

.. literalinclude:: ../../../../../docs_central/legacy_src/llms/OllamaClient/ollama_client_v1.0.hpp
   :language: cpp
   :start-after: // DOC-START: OllamaClient_Class
   :end-before: // DOC-END: OllamaClient_Class