Brain Node (LLM Adapter)
========================

O **Brain Node** é o componente responsável pela interface entre a linguagem natural e o sistema de controle robótico. Ele atua como um tradutor semântico, convertendo comandos humanos complexos em estruturas de árvore de comportamento (Behavior Trees - BT) que o robô pode executar.

Este nó utiliza uma *Large Language Model* (LLM) — especificamente via cliente Ollama — para interpretar a intenção do usuário, extrair parâmetros e estruturar loops lógicos.



Fluxo de Dados
--------------

1.  **Entrada:** O nó subscreve ao tópico ``/human_command`` (tipo `std_msgs/String`).
2.  **Processamento:**
    * Envia o texto para a LLM via ``OllamaClient``.
    * Recebe um plano estruturado em JSON.
    * Converte o JSON para XML compatível com a biblioteca *BehaviorTree.CPP v4*.
3.  **Saída:** Publica o XML resultante no tópico ``/behavior_tree_xml``, que é consumido pelo *Server Node*.

Protocolo JSON da LLM
---------------------

Para garantir a estabilidade do sistema, o *Brain Node* espera que a LLM retorne um JSON com uma estrutura específica. O nó realiza o *parsing* robusto deste formato.

Estrutura de Comandos Simples
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Para ações diretas, o JSON deve conter uma lista de comandos com `skill` (habilidade) e `params`.

.. code-block:: json

   {
     "commands": [
       {
         "skill": "pick",
         "params": { "id": "red_bottle" }
       },
       {
         "skill": "place",
         "params": { "id": "storage_bin_A" }
       }
     ]
   }

Estrutura de Loops e Repetições
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

O nó possui uma lógica avançada para lidar com comandos como *"Pegue as caixas 1 a 5"*. A LLM deve gerar uma estrutura de `loop` que o nó C++ expande automaticamente.

.. code-block:: json

   {
     "commands": [
       {
         "loop": {
           "item": "box_01:box_05",    // Range expandido pelo C++
           "dest": "shelf_A"           // Destino fixo ou lista
         },
         "do": [
           {
             "skill": "pick",
             "params": { "id": "{item}" }
           },
           {
             "skill": "place",
             "params": { "id": "$dest" } // Variável especial
           }
         ]
       }
     ]
   }

Geração de XML (Parsing)
------------------------

O método ``build_bt_xml`` orquestra a conversão. Ele cria uma árvore principal (`MainPlan`) e insere uma sequência de *SubTrees*.

.. literalinclude:: ../../../../../docs_central/legacy_src/llms/BridgeToPhi3_5/bridge_to_phi_3_5_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: build_bt_xml
   :end-before: // DOC-END: build_bt_xml

Expansão de Loops (ForEach)
^^^^^^^^^^^^^^^^^^^^^^^^^^^

A função ``build_loop`` é crítica para a eficiência do sistema. Em vez de criar 50 nós na árvore para pegar 50 objetos, ela cria um único nó customizado ``<ForEach>``.

O código C++ detecta padrões de *range* (ex: `01:05`) e gera uma lista delimitada por `|` no XML. Isso mantém a árvore leve e legível.

.. literalinclude:: ../../../../../docs_central/legacy_src/llms/BridgeToPhi3_5/bridge_to_phi_3_5_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: build_loop
   :end-before: // DOC-END: build_loop

Skills Suportadas
-----------------

O nó atualmente suporta a tradução das seguintes habilidades básicas:

* **Pick:** Requer um `id` de objeto.
* **Place:** Requer um `id` de armazenamento ou coordenadas `(x, y, z)`.
* **GoToLocation:** Requer coordenadas ou um `id` de destino (navegação sem manipulação).

A função ``build_subtree`` encapsula a lógica de validação de parâmetros para cada uma dessas skills antes de gerar o XML.

.. literalinclude:: ../../../../../docs_central/legacy_src/llms/BridgeToPhi3_5/bridge_to_phi_3_5_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: build_subtree
   :end-before: // DOC-END: build_subtree