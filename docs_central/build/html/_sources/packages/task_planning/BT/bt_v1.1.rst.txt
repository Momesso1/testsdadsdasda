Referência dos Nós da Behavior Tree (v1.1)
==========================================

Esta seção documenta a biblioteca de nós (Nodes) desenvolvidos para a composição das árvores de comportamento do sistema. Esta referência foca na lógica de operação, entradas e saídas de dados na *Blackboard*.



1. Demonstração da BT na simulação

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização do comportamento da BT.</em></p>
       
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/task_planning/BT e simulação.webm" type="video/webm">
           Seu navegador não suporta a tag de vídeo.
       </video>

   </div>


Índice Rápido
-------------

.. list-table::
   :widths: 30 15 55
   :header-rows: 1

   * - Nome do Nó
     - Tipo
     - Descrição Resumida
   * - **ParallelAny**
     - Control
     - Executa filhos em paralelo com lógica OR (Short-circuit).
   * - **IsReachable**
     - Condition
     - Verifica alcance cinemático e calcula pose de navegação.
   * - **IsGripperHoldingObject**
     - Condition
     - Verifica sensor de contato da garra.
   * - **IsPathClear**
     - Condition
     - Verifica colisão no caminho atual a cada tick.
   * - **DetectObject**
     - Action
     - Processa visão computacional e seleciona alvos.
   * - **ClearTarget**
     - Action
     - Reseta o alvo atual para permitir nova busca.
   * - **ComputePath**
     - Action
     - Solicita planejamento global de rota (A*).
   * - **FollowPath**
     - Action
     - Executa a trajetória (Controlador) em malha fechada.
   * - **PickObject**
     - Action
     - Executa sequência de manipulação para pegar.
   * - **PlaceObject**
     - Action
     - Executa sequência de manipulação para largar.
   * - **GetStorageInfo**
     - Action
     - Consulta banco de dados de armazenamento.
   * - **ComputePoseToOrganize**
     - Action
     - Calcula pose de *Place* usando Bin Packing.
   * - **ComputePoseToStore**
     - Action
     - Calcula pose de *Place* simples (pilha).
   * - **IncrementOrganizedStorageIndexes**
     - Action
     - Confirma ocupação de espaço (Commit).
   * - **DecrementStorageCount**
     - Action
     - Libera espaço em caso de falha (Rollback).

---

Controle Customizado
--------------------

ParallelAny
^^^^^^^^^^^
* **Tipo:** Control Node
* **Natureza:** Síncrono



Diferente do nó paralelo padrão (que aguarda um número N de sucessos ou falhas), este nó implementa uma lógica de "curto-circuito" (similar a um **OR** lógico em paralelo).

* **Comportamento:**
    1. Executa **todos** os nós filhos simultaneamente a cada ciclo de atualização (*tick*).
    2. Retorna **SUCCESS** imediatamente se *qualquer* filho retornar Sucesso.
    3. Retorna **FAILURE** imediatamente se *qualquer* filho retornar Falha.
    4. Retorna **RUNNING** apenas se todos os filhos estiverem rodando sem conclusão.

* **Aplicação Principal:**
    Implementação de **condições de guarda** em paralelo. Por exemplo, executar "Seguir Caminho" simultaneamente com "Monitorar Sensor de Queda". Se o sensor indicar queda (Falha), a navegação é abortada imediatamente.

---

Condições (Conditions)
----------------------

IsReachable
^^^^^^^^^^^
* **Tipo:** Condition Node
* **Natureza:** Síncrono (Cálculo Geométrico / Cinemática)

Verifica se a pose de manipulação (para pegar ou largar um objeto) é alcançável pelo robô a partir de sua posição atual. Se não for, calcula uma posição de base válida para navegação.

* **Entradas (Blackboard):**
    * ``target``: A pose 3D do objeto ou local de deposição desejado.
    * ``object_id``: Identificador do objeto (usado para contexto de colisão).
    * ``robot_base_z``: Altura da base do robô em relação ao solo.
    * ``max_reach_3d``: Raio máximo absoluto de alcance do braço (esfera de alcance).
* **Saídas (Blackboard):**
    * ``adjustment_pose``: Chave onde a nova pose de navegação será escrita caso o alvo não esteja alcançável.
* **Lógica:**
    1. **Pré-filtro:** Verifica se a distância euclidiana até o alvo está dentro do raio máximo.
    2. **Validação IK:** Se estiver no raio, tenta resolver a Cinemática Inversa para garantir que o braço alcança a pose sem colisão.
    3. **Resultado:**
        * **SUCCESS:** O alvo é alcançável da posição atual (a árvore pode prosseguir para a Manipulação).
        * **FAILURE:** O alvo não é alcançável. O nó calcula uma nova posição de base (projetada no plano XY ao redor do alvo) e disponibiliza essa pose para que a árvore acione a navegação.

IsGripperHoldingObject
^^^^^^^^^^^^^^^^^^^^^^
* **Tipo:** Condition Node
* **Natureza:** Síncrono (Leitura de Sensor)

Verifica o estado físico do efetuador final (garra) para garantir que o objeto está seguro.

* **Lógica:**
    Consulta o monitoramento de hardware da garra.
    * **SUCCESS:** Se o sensor de força/pressão indicar contato firme.
    * **FAILURE:** Se a garra estiver vazia ou aberta.
    * **Efeito Colateral:** Frequentemente usado como gatilho para paradas de emergência se o objeto cair durante o transporte.

IsPathClear
^^^^^^^^^^^
* **Tipo:** Condition Node
* **Natureza:** Síncrono (Verificação Contínua)

Verifica continuamente se o caminho planejado permanece livre de obstáculos dinâmicos (ex: pessoas ou outros robôs que entraram na cena após o planejamento inicial).

* **Entradas (Blackboard):**
    * ``path``: O caminho global gerado anteriormente.
* **Lógica:**
    Itera sobre os pontos da trajetória futura e verifica sua validade contra o mapa de custos atualizado.
    * **SUCCESS:** O caminho está livre.
    * **FAILURE:** Um obstáculo foi detectado na rota (sinalizando a necessidade de replanejamento).

---

Percepção e Contexto
--------------------

DetectObject
^^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Síncrono

Gerencia a triagem e seleção de alvos a partir do fluxo de dados do sistema de visão computacional.

* **Saídas (Blackboard):**
    * ``output_pose``: Posição espacial do objeto selecionado.
    * ``output_id``: Identificador único do objeto.
    * ``object_size``: Dimensões físicas (caixa delimitadora).
* **Lógica (Latch):**
    Funciona como um sistema de travamento de alvo:
    1. **Modo Rastreamento:** Se já existe um alvo ativo na memória, retorna seus dados atuais (**SUCCESS**).
    2. **Modo Busca:** Se não há alvo, verifica a fila de novas detecções aplicando filtros de interesse.
    3. Se um novo objeto válido for encontrado, ele é promovido a alvo atual.

ClearTarget
^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Síncrono

Reseta o estado de atenção do robô, limpando a variável interna de alvo atual. É obrigatório chamar este nó após a conclusão bem-sucedida (ou cancelamento) de uma tarefa para permitir que o robô busque novos objetos.

---

Navegação (Navigation Stack)
----------------------------



ComputePath
^^^^^^^^^^^
* **Tipo:** Action Node (Assíncrono)
* **Natureza:** Cliente de Ação

Solicita o cálculo de uma rota livre de obstáculos ao planejador global.

* **Entradas (Blackboard):**
    * ``target``: O destino final desejado.
    * ``planner``: ID do algoritmo de planejamento a ser usado.
* **Saídas (Blackboard):**
    * ``path``: A trajetória calculada, salva na memória para ser consumida posteriormente.
* **Comportamento:**
    * **SUCCESS:** Caminho válido calculado e armazenado.
    * **FAILURE:** Destino inalcançável ou falha no planejador.

FollowPath
^^^^^^^^^^
* **Tipo:** Action Node (Assíncrono)
* **Natureza:** Cliente de Ação

Responsável exclusivamente pela execução da trajetória (controle de base) em malha fechada. Separa a execução do planejamento.

* **Pré-requisito:** Deve receber um caminho válido gerado previamente.
* **Entradas (Blackboard):**
    * ``path``: O caminho a ser seguido.
    * ``controller_id``: ID do controlador de movimento.
* **Comportamento:**
    Envia o caminho para o servidor de controle.
    * **RUNNING:** Enquanto o robô se move ao longo da trajetória.
    * **SUCCESS:** Quando a tolerância do objetivo final é atingida.
    * **FAILURE:** Se o desvio for muito grande, se houver bloqueio ou falha no controlador.

---

Manipulação
-----------

PickObject
^^^^^^^^^^
* **Tipo:** Action Node (Assíncrono)
* **Natureza:** Cliente de Ação

Comanda o braço robótico para realizar a sequência completa de apreensão de um objeto.

* **Entradas (Blackboard):**
    * ``id``: ID do objeto (para evitar auto-colisão no planejamento).
    * ``pose``: A pose alvo do objeto.
* **Comportamento:**
    Executa uma máquina de estados interna: Planejamento de Aproximação -> Fechamento da Garra -> Validação de Sensor -> Movimento de Elevação (Retreat).

PlaceObject
^^^^^^^^^^^
* **Tipo:** Action Node (Assíncrono)
* **Natureza:** Cliente de Ação

Comanda o braço robótico para depositar o objeto em um local específico.

* **Entradas (Blackboard):**
    * ``pose``: A posição final de deposição.
* **Comportamento:**
    Executa: Planejamento de Aproximação -> Abertura da Garra -> Desanexação do Objeto (Física) -> Movimento de Recuo.

---

Gerenciamento de Estoque (Storage)
----------------------------------



GetStorageInfo
^^^^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Consulta Síncrona

Consulta o banco de dados de inventário para determinar o melhor destino (qual caixa ou estante) e quais índices estão disponíveis para armazenamento.

ComputePoseToOrganize
^^^^^^^^^^^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Cálculo Algorítmico

Calcula a pose exata de deposição utilizando algoritmos de empacotamento (*Bin Packing*) para otimizar o espaço dentro da caixa de armazenamento, evitando colisões com itens já guardados.

ComputePoseToStore
^^^^^^^^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Cálculo Simplificado

Realiza um cálculo simplificado para armazenamento, útil para pilhas verticais ou deposição simples sem organização complexa.

IncrementOrganizedStorageIndexes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Transação de Commit

Atualiza o banco de dados marcando a posição calculada como "Ocupada". Deve ser chamado apenas após o sucesso da ação ``PlaceObject`` para garantir a consistência do inventário.

DecrementStorageCount
^^^^^^^^^^^^^^^^^^^^^
* **Tipo:** Action Node (Simples)
* **Natureza:** Transação de Rollback

Libera o espaço reservado no banco de dados. Utilizado em ramos de falha da árvore: se o robô deixar cair o objeto ou falhar ao tentar guardá-lo, este nó garante que o espaço não fique marcado como ocupado erroneamente.