Shared Obstacle Graph
=====================

O nó ``SharedObstacleGraph`` é uma outra maneira de acessar o grafo de obstáculos. Essa ideia foi implementada na versão 1.1 de :doc:`Server Node v1.1<../../task_planning/ServerNode/server_node_v1.1>`.
O nó :doc:`Obstacle Graph v1.0 <../ObstacleGraph/obstacle_graph_with_occupancy_grid_v1.0>` ainda publica o grafo de obstáculos, isso acontece porque 
determinados nós podem não rodar no mesmo computador de :doc:`Server Node <../../task_planning/ServerNode/index>`. Entretanto, 
para diminuir a latência e evitar criar várias cópias do mesmo mapa este nó mantém o mapa na memória RAM e compartilha apenas o ponteiro de leitura (`std::shared_ptr`) com outros nós no mesmo processo.

.. note::
   **Padrão "Writer-Reader":**
   Este nó é o único **Escritor** (Writer) do mapa. O ``IsPathClear`` e ``IKValidator`` são apenas **Leitores** (Readers). Isso simplifica drasticamente o controle de concorrência.

---

1. Estratégia de Concorrência (Double Buffering)
------------------------------------------------

Para garantir que o planejamento de caminho nunca bloqueie o recebimento de novos sensores (e vice-versa), o nó implementa uma estratégia de **Troca Atômica**.

1.  **Back Buffer (Construção):** Quando uma nova nuvem de pontos chega, o nó cria um *novo* `unordered_set` na memória local. Ele processa e preenche este novo mapa sem travar o mutex.
2.  **Atomic Swap (Publicação):** Apenas quando o novo mapa está 100% pronto, o nó trava o mutex, troca o ponteiro `current_map_` para apontar para o novo mapa e destrava. Isso leva nanosegundos.
3.  **Leitura Segura:** Os leitores que pegaram o ponteiro *antigo* continuam lendo o mapa antigo até terminarem. O mapa antigo só é deletado da memória quando o último leitor o solta (`shared_ptr` ref count chega a 0).

.. literalinclude:: ../../../../../docs_central/legacy_src/navigation/SharedObstacleGraph/shared_obstacle_graph_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: point_cloud_callback
   :end-before: // DOC-END: point_cloud_callback

---

2. Acesso aos Dados
-------------------

Esta é a interface pública utilizada pelos nós de lógica (`ServerNode`, `Reachability`).
O método retorna um ponteiro constante (`const`), garantindo que ninguém altere o mapa externamente.

.. literalinclude:: ../../../../../docs_central/legacy_src/navigation/SharedObstacleGraph/shared_obstacle_graph_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: get_current_map
   :end-before: // DOC-END: get_current_map

---

3. Discretização e Hashing
--------------------------

O mapa não armazena a nuvem de pontos bruta. Ele discretiza o espaço baseado na resolução configurada (ex: 5cm). Isso reduz o tamanho do mapa e permite verificações de colisão em tempo constante O(1).

* **Round to Multiple:** Arredonda coordenadas float (ex: 1.034 -> 1.05) para garantir que pontos próximos caiam na mesma célula do grid.

.. literalinclude:: ../../../../../docs_central/legacy_src/navigation/SharedObstacleGraph/shared_obstacle_graph_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: round_to_multiple
   :end-before: // DOC-END: round_to_multiple

---

4. Inicialização e Parâmetros
-----------------------------

O construtor inicializa um mapa vazio para evitar *Segmentation Faults* caso algum nó tente ler o mapa antes da primeira leitura do sensor.
Também configura o QoS como `Best Effort`, ideal para dados de sensores de alta frequência onde a perda de um pacote antigo não é crítica.

.. literalinclude:: ../../../../../docs_central/legacy_src/navigation/SharedObstacleGraph/shared_obstacle_graph_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Constructor
   :end-before: // DOC-END: Constructor