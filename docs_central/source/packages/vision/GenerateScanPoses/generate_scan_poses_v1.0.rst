Generate Scan Poses (Visão Computacional)
=========================================

O nó ``GenerateScanPoses`` implementa um algoritmo de **Planejamento de Next-Best-View (NBV)**.
Seu objetivo é calcular um conjunto mínimo e otimizado de posições para a câmera (acoplada ao braço robótico) de modo a escanear a superfície de um objeto desconhecido para posterior processamento.

Diferente de abordagens que fazem o robô girar cegamente ao redor do objeto, este nó utiliza geometria computacional para garantir que as poses escolhidas realmente "vejam" o objeto, respeitando limitações de FOV e oclusões.

Demonstração Visual
-------------------

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Demonstração do cálculo de poses e cobertura.</em></p>
       
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/vision/demo_scan_and_map.webm" type="video/webm">
           Seu navegador não suporta a tag de vídeo.
       </video>

       <p style="font-size: 0.9em; color: #555; margin-top: 10px;">
           <strong>Legenda:</strong> As <strong>setas verdes</strong> indicam as poses de câmera otimizadas geradas por este nó (GenerateScanPoses). 
           O <strong>objeto verde</strong> sendo formado no centro é o resultado do processamento feito pelo nó 
           <a href="../ObjectMapping/object_mapping_v1.0.html">ObjectMapping</a>.
       </p>
   </div>

API Pública e Integração com Server Node
----------------------------------------

Este nó foi projetado como um componente de alto desempenho para ser consumido diretamente pelo **Server Node**. Ele expõe duas funções públicas críticas que permitem ao planejador de tarefas orquestrar a sequência de escaneamento de forma eficiente.

O fluxo de dados entre o *Server Node* e este componente é o seguinte:

1.  **Etapa de Geração (SortedCandidates):**
    O *Server Node* chama ``getSortedScanPoses`` passando o ID do objeto alvo (ex: "box_01").
    * Esta função gera centenas de poses candidatas ao redor do objeto (um "grid" esférico).
    * Ordena essas poses pela proximidade da posição atual do robô para minimizar o movimento da base.
    * Retorna as poses brutas e a posição atual do robô.

2.  **Etapa de Validação Cinemática (IK Check):**
    O *Server Node* (externamente) pega essas poses candidatas e as submete ao ``IKValidator``. Isso remove poses que são fisicamente inalcançáveis pelo braço robótico devido a limitações de juntas ou colisão.

3.  **Etapa de Otimização (OptimizedScanPoses):**
    O *Server Node* chama ``getOptimizedScanPoses`` passando a lista filtrada de poses válidas (que sobreviveram ao IK).
    * Esta função executa o algoritmo de cobertura gulosa (*Set Cover*).
    * Ela seleciona o **menor número de poses** necessário para ver a maior área possível do objeto.
    * O resultado final é uma lista enxuta (geralmente 3 a 5 poses) pronta para execução.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: SortedScanPoses
   :end-before: // DOC-END: SortedScanPoses

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: OptimizationAPI
   :end-before: // DOC-END: OptimizationAPI

Parâmetros Principais
---------------------

* ``ray_length`` (double, 0.25): Distância ideal da câmera até a superfície do objeto.
* ``grid_resolution`` (double, 0.04): Espaçamento entre os pontos candidatos de escaneamento ao redor do objeto.
* ``voxel_map_resolution`` (double, 0.02): Tamanho do voxel para o mapa de oclusão global.
* ``target_surface_res`` (double, 0.005): Resolução da discretização da superfície do objeto alvo (voxels alvo).
* ``min_coverage_percent`` (double, 0.6): Porcentagem mínima da superfície do objeto que deve ser coberta pelo conjunto de poses retornado.
* ``max_incidence_angle_deg`` (double, 80.0): Ângulo máximo entre a normal da superfície e o vetor de visão da câmera para considerar um ponto como "visível".

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Constructor
   :end-before: // DOC-END: Constructor

Discretização da Superfície (Alvo)
----------------------------------

Para quantificar o quanto do objeto foi "visto", o algoritmo primeiro discretiza a Bounding Box do objeto em uma nuvem densa de pontos superficiais (voxels alvo), cada um contendo sua posição e vetor normal.

O método varre as 5 faces visíveis da caixa (±X, ±Y, +Z), ignorando a base (-Z) que está em contato com a mesa.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: SurfaceDiscretization
   :end-before: // DOC-END: SurfaceDiscretization

Geração de Candidatos (Espaço de Busca)
---------------------------------------

O algoritmo gera um "grid de escaneamento" ao redor do objeto. Imagine uma caixa maior envolvendo o objeto; os pontos desse grid são posições potenciais para a câmera.

Para cada ponto no grid, uma pose é calculada usando ``createPoseLookingAt``, garantindo que o eixo Z da câmera aponte para o centro do objeto e o eixo Y (Up) seja consistente com o mundo.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: CandidateGeneration
   :end-before: // DOC-END: CandidateGeneration

Verificação de Visibilidade e Oclusão
-------------------------------------

Uma pose candidata só é válida se tiver linha de visão desobstruída até o objeto.
O nó utiliza um **Raycasting Voxelizado** no mapa semântico global (construído a partir do ``CombinedSemanticPCL``). Se o raio entre a câmera e o objeto interceptar um voxel com rótulo diferente do alvo (ex: outro obstáculo na frente), a pose é descartada.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: RayCasting
   :end-before: // DOC-END: RayCasting

Além da oclusão física, verifica-se a **Visibilidade Matemática**:
1. O ponto está dentro do *Frustum* (FOV Horizontal e Vertical) da câmera?
2. O ângulo de incidência é aceitável? (Câmeras de profundidade falham em ângulos muito rasos).

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: VisibilityCheck
   :end-before: // DOC-END: VisibilityCheck

Otimização de Cobertura (Greedy Set Cover)
------------------------------------------

Dado um conjunto de poses candidatas válidas, qual o menor subconjunto que vê a maior parte do objeto?
Este é um problema clássico de *Set Cover*. O nó implementa uma heurística gulosa (*Greedy*) em duas passadas:

1.  **Passada 1 (Maximização):** Itera escolhendo a pose que visualiza o maior número de voxels *ainda não vistos*.
2.  **Passada 2 (Refinamento):** Adiciona poses que veem voxels restantes, mesmo que poucos, para fechar buracos na cobertura.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: GreedyOptimization
   :end-before: // DOC-END: GreedyOptimization

Matemática Auxiliar (LookAt)
----------------------------

Função utilitária que calcula a orientação (Quaternion) necessária para que um objeto na posição `origin` olhe para `target`, mantendo o vetor *Up* alinhado com o eixo Z do mundo sempre que possível (evitando *roll* excessivo da câmera).

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/GenerateScanPoses/generate_scan_poses_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: MathHelpers
   :end-before: // DOC-END: MathHelpers