projected_reachability_analysis.cpp (Atual v1.0)
================================================

O nó ``ProjectedReachabilityAnalysis`` é responsável por resolver o problema de **Alcançabilidade Inversa Geométrica**.

Dada a posição de um objeto alvo no espaço 3D, este componente calcula **onde a base do robô pode estar posicionada** para que o braço consiga alcançar o objeto. Ele não valida cinemática inversa complexa (juntas), mas sim a viabilidade geométrica baseada no comprimento máximo do braço.

.. note::
   **Conceito de "Donut":**
   O resultado prático deste algoritmo é uma zona em formato de anel (ou disco perfurado).
   * **Raio Externo:** Limitado pelo alcance máximo do braço ajustado pela altura.
   * **Raio Interno:** Limitado pela ``security_distance`` (o robô não pode estar em cima do objeto).

---

1. Matemática da Projeção 3D -> 2D
----------------------------------

Para determinar a viabilidade da manipulação antes de mover a base, o sistema calcula a **Região Viável da Base** (Kinematic Footprint) no plano :math:`z`. Nesse caso o robô estava em cima de 
uma base móvel em :math:`z = 0.11`.

O algoritmo projeta a esfera de alcance máximo do manipulador (Workspace Sphere) no chão, considerando o offset vertical da montagem do braço.

**Lógica Geométrica:**
A função deriva o raio 2D (:math:`r_{2d}`) utilizando o Teorema de Pitágoras, onde a hipotenusa é o alcance máximo do braço e o cateto vertical é a diferença de altura entre o objeto e o ombro do robô.

.. math::

    r_{2d} = \sqrt{R_{max}^2 - (z_{obj} - z_{base})^2}

Onde:

* :math:`R_{max}`: Alcance máximo do manipulador menos um offset para evitar singularidade (Definido como 0.9m).
* :math:`z_{base}`: Altura da primeira junta do manipulador (Definido como 0.11m).

**Visualização de Debug (MarkerArray):**
Para facilitar o desenvolvimento e a depuração visual no RViz, esta função publicava um ``MarkerArray`` contendo:

1.  **Disco Translúcido:** Representa a zona de estacionamento válida no chão.
2.  **Cubo Vermelho:** A posição exata do objeto alvo percebido.
3.  **Triângulo Vetorial:** Linhas coloridas desenhando os catetos e a hipotenusa para validação visual da matemática.

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização do raio de alcançabilidade em 2D.</em></p>
       <img src="../../../_static/manipulation/raio_de_alcancabilidade_2d.png" alt="Visualização do raio de alcançabilidade em 2D." style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização lateral do Triângulo de Cálculo (Catetos e Hipotenusa).</em></p>
       <img src="../../../_static/manipulation/vista_lateral_do_triangulo.png" alt="Visualização lateral do Triângulo de Cálculo." style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização tanto do raio de alcançabilidade quanto do triângulo retângulo.</em></p>
       <img src="../../../_static/manipulation/raio_de_alcancabilidade_e_triangulo_retangulo.png" alt="Screenshot do RViz." style="width: 100%; height: auto;">
   </div>


---

2. Parâmetros e Inicialização
-----------------------------

O nó configura a resolução da malha de busca e a distância mínima de segurança. Também prepara os publicadores para depuração visual (Marcadores e Nuvem de Pontos).

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/ProjectedReachabilityAnalysis/projected_reachability_analysis_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Constructor
   :end-before: // DOC-END: Constructor

---

3. Algoritmo de Expansão (BFS)
------------------------------

Para gerar os pontos discretos candidatos, o algoritmo utiliza uma **Busca em Largura (BFS - Breadth-First Search)** expandindo a partir da projeção do objeto no chão.

**O Processo:**

1.  **Cálculo do Raio:** Determina o $R_{2d}$ disponível.

2.  **Expansão:** Começa no centro (x, y do objeto) e expande para os 8 vizinhos.

3.  **Filtro de Alcance Máximo:** Se a distância do ponto atual até o centro for maior que $R_{2d}$, o ponto é descartado e a expansão para naquela direção.

4.  **Filtro de Segurança (Raio Interno):** Se a distância for menor que ``security_distance``, o ponto é visitado para continuar a expansão, mas **não** é adicionado à lista de candidatos válidos (`valid_candidates`).

Isso garante que o robô não tente planejar uma rota para colidir com o objeto (ficar "dentro" dele).

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da BFS dentro do raio de alcançabilidade.</em></p>
       <img src="../../../_static/manipulation/bfs_de_alcancabilidade.png" alt="Visualização da BFS dentro do raio de alcançabilidade" style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da BFS com o modelo do robô ao lado do objeto.</em></p>
       <img src="../../../_static/manipulation/bfs_de_alcancabilidade_com_modelo_do_robo.png" alt="Visualização lateral do Triângulo de Cálculo." style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da simulação real.</em></p>
       <img src="../../../_static/manipulation/simulacao_real_em_que_bfs_foi_aplicada.png" alt="Screenshot do RViz." style="width: 100%; height: auto;">
   </div>

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/ProjectedReachabilityAnalysis/projected_reachability_analysis_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: get_reachable_points
   :end-before: // DOC-END: get_reachable_points

---

