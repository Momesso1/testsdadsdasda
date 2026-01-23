Object Mapping (Reconstrução 3D)
================================

O nó ``ObjectMapping`` é responsável pela reconstrução densa de objetos específicos de interesse.
Diferente de um mapeamento SLAM (que mapeia todo o ambiente), este nó foca em **acumular nuvens de pontos temporalmente** de um único objeto alvo, criando um modelo 3D denso e completo necessário para algoritmos de preensão complexos (como a inferência de *GraspNet*).

Este nó opera em estreita colaboração com o nó de geração de poses de escaneamento (*GenerateScanPoses*), atuando como o "acumulador" passivo enquanto o braço robótico move a câmera ao redor do objeto.

Demonstração Visual
-------------------

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Demonstração do processo de escaneamento e reconstrução.</em></p>
       
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/vision/demo_scan_and_map.webm" type="video/webm">
           Seu navegador não suporta a tag de vídeo.
       </video>

       <p style="font-size: 0.9em; color: #555; margin-top: 10px;">
           <strong>Legenda:</strong> O <strong>objeto verde</strong> sendo formado no centro é o resultado do mapeamento feito por este nó (ObjectMapping). 
           As <strong>setas verdes</strong> ao redor indicam as poses de câmera calculadas pelo nó 
           <a href="../GenerateScanPoses/generate_scan_poses_v1.0.html">GenerateScanPoses</a>.
       </p>
   </div>

Controle de Estabilidade (Stop-and-Scan)
----------------------------------------

Uma das maiores fontes de erro em reconstrução 3D é o *Motion Blur* (borrão de movimento). Se o robô adicionar pontos ao mapa enquanto a câmera está vibrando ou se movendo rapidamente, o modelo 3D resultante ficará distorcido e inutilizável.

Para resolver isso, o nó implementa um filtro de estabilidade rigoroso:

1.  **Monitoramento de Juntas:** O nó escuta o tópico de estados das juntas (`/isaac_joint_states`).
2.  **Verificação de Velocidade:** Calcula se a velocidade de qualquer junta excede um limite configurável (`velocity_threshold`).
3.  **Tempo de Assentamento:** Mesmo após o robô parar, o sistema aguarda um período adicional (`settlement_duration`) para garantir que vibrações mecânicas residuais cessem.

Apenas quando a flag interna ``is_robot_stopped_`` é verdadeira, o sistema permite a entrada de novos dados na nuvem de pontos.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/ObjectMapping/object_mapping_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: StabilityCheck
   :end-before: // DOC-END: StabilityCheck

API Pública e Controle de Alvo
------------------------------

Para evitar o consumo excessivo de memória mapeando objetos irrelevantes, este nó permanece em estado "ocioso" até receber uma ordem explícita.

A função ``ObjectToMap`` é a interface pública exposta para o **Server Node**. Ela define um filtro de lista branca (*whitelist*).

* **Funcionamento:** Quando a Behavior Tree decide que precisa pegar a "redbox_01", o Server Node chama esta função passando o ID "redbox_01".
* **Efeito:** A partir desse momento, o nó de mapeamento começa a filtrar a nuvem semântica recebida. Ele descarta dados de mesas, chão ou outros objetos, e acumula apenas os pontos pertencentes ao ID autorizado.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/ObjectMapping/object_mapping_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: PublicAPI
   :end-before: // DOC-END: PublicAPI

Processo de Acumulação
----------------------

O coração do nó é o callback de processamento da nuvem semântica. O fluxo de dados é:

1.  **Verificação de Estabilidade:** Se o robô estiver se movendo, a nuvem é descartada imediatamente. Isso acontece porque pode haver problemas na sincronização da transformação e do tópico que publica a nuvem de pontos.
2.  **Filtragem de ID:** Verifica se o rótulo da nuvem recebida corresponde ao alvo definido via ``ObjectToMap``.
3.  **Fusão:** Se os critérios forem atendidos, os novos pontos são adicionados (*append*) à nuvem acumulada existente.

Isso permite que o robô olhe para o objeto de cima, de lado e de frente, e o nó funda todas essas vistas em um único sistema de coordenadas coerente.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/ObjectMapping/object_mapping_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Accumulation
   :end-before: // DOC-END: Accumulation

Visualização e Saída
--------------------

Para facilitar o debug e a visualização no RViz, o nó publica a nuvem acumulada em um tópico específico (`/mapped_object`).

Para distinguir visualmente diferentes objetos (caso a lógica de mapeamento mude), o nó aplica uma coloração baseada em **Hash do ID**. Isso garante que o mesmo objeto sempre tenha a mesma cor, sem a necessidade de transmitir informações de textura pesadas.

.. literalinclude:: ../../../../../docs_central/legacy_src/vision/ObjectMapping/object_mapping_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Publishing
   :end-before: // DOC-END: Publishing