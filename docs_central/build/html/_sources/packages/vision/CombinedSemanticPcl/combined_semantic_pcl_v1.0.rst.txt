Combined Semantic PCL
=====================

O nó ``CombinedSemanticPcl`` é o núcleo da percepção semântica do robô.
Sua função principal é fundir dados de **profundidade** (nuvem de pontos 3D) com **segmentação semântica** (imagem 2D) para criar uma representação 3D do mundo onde cada ponto possui um rótulo de classe (ex: "garrafa", "caixa", "chão").

Este nó foi desenhado especificamente para trabalhar com o **NVIDIA Isaac Sim**, adaptando-se à forma como o simulador publica os rótulos de classe (JSON Mapping).



Integração com Isaac Sim (Semantic Labels)
------------------------------------------

Uma das características mais críticas deste nó é a lógica de parsing de JSON.
Diferente de sistemas que publicam rótulos fixos, o Isaac Sim publica uma string JSON dinâmica no tópico ``semantic_labels`` que mapeia **IDs numéricos** (presentes na imagem de segmentação) para **Nomes de Classes** legíveis.

O formato típico do JSON do Isaac Sim é:

.. code-block:: json

   {
       "1": {"class": "cube"},
       "2": {"class": "floor"},
       "3": "red_box"
   }

Para garantir robustez contra variações na formatação do JSON (aspas escapadas, objetos aninhados, etc.), o nó utiliza **Expressões Regulares (Regex)** em vez de um parser JSON rígido.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/CombinedSemanticPcl/combined_semantic_pcl_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: LabelsParsing
   :end-before: // DOC-END: LabelsParsing

Sincronização de Sensores
-------------------------

Para que a fusão seja precisa, a imagem de segmentação e a nuvem de pontos de profundidade devem corresponder ao mesmo instante no tempo.
Se usássemos callbacks separados, o robô poderia estar olhando para um lugar na imagem de cor e para outro na profundidade devido ao movimento.

O nó utiliza ``message_filters::Synchronizer`` com uma política de *Approximate Time*. Isso garante que o callback de processamento (`syncedCallback`) só seja disparado quando ambos os dados (imagem e nuvem) estiverem disponíveis e alinhados temporalmente.



Fusão e Projeção 3D
-------------------

O coração do processamento ocorre no `syncedCallback`. O fluxo lógico é:

1.  **Transformação TF2:** Obtém a transformação entre o frame da câmera e o frame alvo (geralmente ``world`` ou ``map``). Isso permite publicar a nuvem semântica em coordenadas globais, facilitando a navegação e o mapeamento.
2.  **Iteração e Downsampling:** Percorre a nuvem de pontos bruta. Para economizar CPU, utiliza-se um passo de *downsample* (ex: processar 1 a cada 4 pontos).
3.  **Correspondência 2D -> 3D:**
    * Cada ponto 3D na nuvem corresponde a um pixel na imagem de segmentação.
    * O nó lê o valor do pixel (ID Inteiro) na imagem de segmentação.
    * Esse ID é associado às coordenadas (X, Y, Z) transformadas para o mundo.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/CombinedSemanticPcl/combined_semantic_pcl_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: SyncedCallback
   :end-before: // DOC-END: SyncedCallback

Saídas do Nó (Publishers)
-------------------------

O nó gera três tipos de saída para diferentes consumidores:

1.  **semantic_pcl (sensor_msgs/PointCloud2):**
    Nuvem técnica contendo os campos `x`, `y`, `z` e `semantic_id` (inteiro). Usada por outros algoritmos que precisam do ID numérico.

2.  **semantic_pcl_colored (sensor_msgs/PointCloud2):**
    Nuvem visual para o RViz. O ID semântico é convertido em uma cor RGB determinística (hash do ID), permitindo que humanos visualizem a segmentação em 3D.

3.  **semantic_pcl_array (Custom Msg):**
    Mensagem customizada que separa a nuvem em clusters por objeto. Útil para o nó de planejamento (Server Node) saber exatamente onde está "a caixa vermelha" sem ter que filtrar uma nuvem gigante manualmente.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/CombinedSemanticPcl/combined_semantic_pcl_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Publishing
   :end-before: // DOC-END: Publishing