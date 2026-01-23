Planejador A* (Versão 1.1)
==========================

O componente de planejamento global na versão 1.1 atua como um planejador de caminho determinístico. Diferente da versão anterior, que operava como um supervisor contínuo de navegação, esta versão funciona como uma **Calculadora de Rota Pura (One-Shot)**.

Ele recebe um mapa de ocupação (nuvem de pontos) e uma posição de destino, calcula a rota ótima e retorna imediatamente, delegando a responsabilidade de monitorar obstáculos dinâmicos para a camada de comportamento superior (Behavior Tree).



---

1. Mudanças Críticas: v1.0 vs v1.1
----------------------------------

A arquitetura do planejador foi simplificada para se adequar a um modelo deliberativo.

Remoção da Distância de Segurança
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
A alteração mais impactante nesta versão refere-se a como o robô aborda o alvo.

* **Versão Anterior:** O planejador recebia um alvo (ex: centro de uma mesa) e truncava o caminho quando a distância atingia um limiar de segurança. O robô parava *antes* de chegar.
* **Versão Atual:** O parâmetro de distância de segurança foi removido. O planejador agora traça uma rota até a coordenada **exata** solicitada (ou o ponto livre mais próximo dela).



.. warning::
   **Mudança de Comportamento:**
   Se for enviada a coordenada do centro de um obstáculo físico para este planejador, ele tentará criar um caminho até lá, limitado apenas pela colisão física descrita no mapa.
   **Motivo:** A responsabilidade de calcular a "Pose de Aproximação" (onde o robô deve parar para interagir com um objeto) foi movida para o componente de análise de alcançabilidade. O planejador agora obedece estritamente ao destino fornecido.

Fim do Loop de Execução e Feedback
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* **Versão Anterior:** O sistema entrava em um loop infinito, monitorando a odometria e recalculando a rota se obstáculos dinâmicos aparecessem.
* **Versão Atual:** A execução é de disparo único (*One-Shot*). O sistema calcula, retorna o resultado e encerra a ação. Não há monitoramento contínuo dentro deste componente. A verificação de novos obstáculos deve ser feita externamente.

---

2. Parâmetros e Inicialização
-----------------------------

Os parâmetros foram ajustados para focar apenas na resolução do grafo e na otimização heurística.

* **Resolução do Caminho:** Define o tamanho da célula do grid (discretização do espaço).
* **Frequência de Verificação de Visada:** Define a agressividade da otimização de "Linha de Visão". A cada determinado número de iterações do algoritmo, tenta-se traçar uma linha reta até o final para encontrar atalhos, acelerando a busca em áreas abertas.

---

3. Núcleo do Algoritmo: A* com Otimização de Raio
-------------------------------------------------

A lógica de busca mantém a robustez contra alvos inválidos, mas introduz uma verificação de linha de visão (*Raycast*) dentro do loop principal de expansão.

Recuperação de Alvo (Busca em Largura - BFS)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Antes de iniciar o planejamento, o sistema verifica se o ponto de partida ou o ponto de destino estão situados dentro de obstáculos (o que tornaria o cálculo impossível). Se estiverem, utiliza-se uma Busca em Largura (BFS) para encontrar o ponto livre mais próximo em um padrão espiral.



**Demonstração da Expansão BFS:**

.. raw:: html

    <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da recuperação de alvo via BFS (vídeo extremamente desacelerado).</em></p>
       
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/navigation/BFS_example_precise.webm" type="video/webm">
           Seu navegador não suporta a tag de vídeo.
       </video>

       <br><br>

       <p><em>Visualização do mapa real. O alvo é o cubo vermelho.</em></p>
       <img src="../../../_static/navigation/mapa_real_bfs.png" alt="Visualização do mapa real." style="width: 100%; height: auto;">
   </div>

Heurística e Otimização
^^^^^^^^^^^^^^^^^^^^^^^
O loop principal do planejador agora contém uma verificação periódica. Se o nó atual tiver uma linha de visão limpa e desobstruída até o alvo, o algoritmo encerra a busca imediatamente e conecta os pontos, economizando processamento computacional.



**Visualização da Expansão do A*:**

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da expansão dos nós</em></p>
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/navigation/A*.webm" type="video/webm">
           Seu navegador não suporta a tag de vídeo.
       </video>
   </div>

---

4. Pós-Processamento: Filtro e Orientação
-----------------------------------------

O método de saída fornece dois tipos de caminhos simultaneamente para diferentes consumidores:

1.  **Caminho Suavizado:** Utilizado para a navegação do controlador. Passa por um algoritmo que remove pontos intermediários desnecessários (pontos colineares ou atalhos seguros), resultando em movimentos mais fluidos.
2.  **Caminho Sem Filtro:** Utilizado pelo nó central para verificação de colisão. Mantém a fidelidade exata do grid para garantir que a checagem de obstáculos seja precisa célula a célula.



**Comparativo: Caminho Bruto vs. Suavizado:**

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Filtro de Suavização (demonstração visual da técnica)</em></p>
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/navigation/Filter_60fps.webm" type="video/webm">
           Seu navegador não suporta a tag de vídeo.
       </video>
   </div>

**Cálculo de Orientação:**
O sistema calcula a orientação de cada pose baseada no vetor de direção para o próximo ponto da trajetória. Isso permite que o robô receba o caminho já orientado para a frente, facilitando o controle.

---

5. Fluxo de Execução
--------------------

A execução da tarefa segue um fluxo linear e sem estado ("stateless"):

1.  Bloqueia o acesso ao mapa e à odometria para obter um *snapshot* consistente do mundo.
2.  Executa o algoritmo de busca.
3.  Aplica os filtros de suavização.
4.  Retorna imediatamente o sucesso ou falha da operação.

.. note::
   Não há laços de repetição monitorando o progresso do robô nesta etapa. O processo termina em milissegundos, correspondendo apenas ao tempo de cálculo da CPU.