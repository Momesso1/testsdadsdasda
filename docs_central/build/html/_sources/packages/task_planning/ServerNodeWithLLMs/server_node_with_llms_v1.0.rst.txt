Server Node With LLMs (Task Planner)
====================================

O **Server Node** atua como o cérebro central do robô autônomo. Ele é responsável por orquestrar todas as operações complexas — navegação, percepção, manipulação e gerenciamento de estoque — através de uma **Behavior Tree (BT)**.

Diferente de uma máquina de estados finitos rígida, este nó implementa um planejador reativo híbrido: ele pode carregar comportamentos pré-definidos em arquivos XML ou receber novas árvores de comportamento geradas dinamicamente por Inteligência Artificial em tempo de execução.

Arquitetura de Componentes
--------------------------

Para garantir latência mínima e alta eficiência, o Server Node não se comunica com seus subsistemas críticos via tópicos ROS tradicionais. Em vez disso, ele utiliza um padrão de **Injeção de Dependência** e **Composição**.

O nó instancia os seguintes componentes na mesma memória (heap) e compartilha o acesso via ponteiros inteligentes (*Shared Pointers*):

* **Manipulação:** Validadores de Cinemática Inversa (IK) e Análise de Alcançabilidade.
* **Navegação:** Grafo de Obstáculos Dinâmico.
* **Estoque:** Gerenciador de Banco de Dados e Algoritmos de Organização (*Bin Packing*).
* **Percepção:** Ponte para Inferência Neural (GraspNet) e Mapeamento de Objetos.

Essa arquitetura permite que a Behavior Tree tome decisões complexas (ex: "Posso alcançar aquele objeto sem colidir?") em tempo real, acessando os dados brutos dos outros sistemas instantaneamente.

---

Geração Dinâmica via LLMs (Phi 3.5)
-----------------------------------

Uma das inovações deste nó é a capacidade de executar missões baseadas em comandos de linguagem natural, processados localmente ou via API. Nesta implementação, o sistema utiliza o modelo de linguagem **Phi 3.5** para estruturar o planejamento.

O pipeline funciona em três estágios:

1.  **Interpretação (Phi 3.5 -> JSON):**
    O modelo **Phi 3.5** recebe um comando do usuário. Devido à especificidade do ambiente, o usuário deve fornecer os **IDs explícitos** dos objetos e locais de armazenamento.
    
    * *Exemplo de Comando:* "Pegue as **redbox_01** até **redbox_05** e guarde em **storage_02**."
    
    A LLM processa a lógica de repetição (do 01 ao 05) e gera um plano de tarefa estruturado em formato **JSON**, contendo a sequência de ações para cada objeto individualmente.

2.  **Conversão (JSON -> BT XML):**
    Um nó intermediário recebe esse JSON. Ele valida a estrutura e traduz as instruções para a sintaxe XML padrão da *BehaviorTree.CPP*, garantindo que as ações geradas (como ``PickObject`` com ``id="redbox_03"``) correspondam às ações C++ registradas no sistema.

3.  **Injeção via Tópico:**
    A árvore XML resultante é publicada em um tópico ROS (ex: ``/behavior_tree_updates``). O **Server Node** possui um *subscriber* escutando este tópico. Assim que uma nova árvore válida é recebida, o nó interrompe a missão atual e carrega a nova lógica instantaneamente.

---

Gerenciamento da Behavior Tree
------------------------------

A função central do nó é garantir a execução segura dessas árvores, sejam elas estáticas ou dinâmicas. O ciclo de vida do gerenciamento ocorre da seguinte forma:

1.  **Registro de Nós (Factory):** Ao iniciar, o sistema registra na fábrica da BT todas as Ações e Condições customizadas desenvolvidas em C++. Isso cria o vocabulário que tanto os arquivos XML locais quanto as árvores geradas pela LLM devem utilizar.
2.  **Carregamento Inicial (Híbrido):**
    * *Modo Estático:* O nó lê o parâmetro ``bt_xml_path`` para carregar uma missão padrão do disco.
    * *Modo Dinâmico:* O nó aguarda no tópico de atualização. Ao receber o XML vindo do pipeline da LLM, ele faz o *Hot-Swap* (troca a quente) da lógica de controle.
3.  **Blackboard:** É criada uma memória compartilhada (*Blackboard*) onde os nós da árvore podem trocar dados (ex: ID do objeto detectado, Posição de destino).
4.  **Execução (Ticking):** Uma *thread* dedicada roda a 50Hz, enviando pulsos (*ticks*) para a raiz da árvore. O sistema monitora mudanças de estado e reage a interrupções.

Visualização em Tempo Real
^^^^^^^^^^^^^^^^^^^^^^^^^^

O nó integra nativamente um publicador para o **Groot2**. Isso permite monitorar a execução da árvore visualmente via rede na porta 1666. É possível ver em tempo real a árvore que foi gerada pelo Phi 3.5 sendo executada, facilitando o *debug* das decisões da IA.

---

Integração com Banco de Dados e Estoque
---------------------------------------

Uma das características mais avançadas deste planejador é sua capacidade de interagir com o sistema de armazenamento (`StorageNode` e `OrganizeNode`) para decidir onde guardar os objetos recolhidos.

O fluxo de decisão na árvore segue o padrão **Consulta -> Cálculo -> Persistência**:

1.  **Consulta de Vaga (GetStorageInfo):**
    A árvore solicita uma vaga para um determinado ID (ex: "redbox_01"). O sistema consulta o banco de dados interno para encontrar uma caixa ou prateleira (ex: "storage_02") com espaço disponível e compatível com as dimensões do objeto.
    
    * *Saída:* Retorna a pose da caixa, suas dimensões limites e os índices de ocupação atuais.

2.  **Cálculo de Organização (ComputePoseToOrganize):**
    Com as informações da caixa e do objeto em mãos, a árvore aciona o algoritmo de **Bin Packing**. Este algoritmo calcula matematicamente a posição exata (x, y, z) onde o objeto deve ser colocado dentro da caixa para maximizar o espaço e evitar colisões com itens já armazenados.
    
    * *Saída:* Uma pose exata de "Place" e os novos índices de ocupação da matriz de armazenamento.

3.  **Persistência (IncrementOrganizedStorageIndexes):**
    Se — e somente se — o robô conseguir colocar o objeto com sucesso, a árvore chama esta ação para "commitar" a mudança no banco de dados, marcando aquele espaço como ocupado permanentemente.

4.  **Rollback (DecrementStorageCount):**
    Caso a operação de manipulação falhe após a reserva do espaço, a árvore possui mecanismos de falha que chamam esta ação para liberar a vaga no banco de dados, mantendo a consistência do estoque.

---

Integração com GraspNet
-----------------------

Na versão atual, o Server Node suporta um pipeline avançado de preensão utilizando Redes Neurais. Dentro da ação de ``PickObject``, existe uma máquina de estados interna que gerencia a comunicação com o nó de inferência:

1.  **Solicitação de Scan:** O robô se posiciona e solicita uma nuvem de pontos.
2.  **Inferência Remota:** A nuvem é enviada via TCP/IP (Bridge) para um servidor Python que roda a GraspNet.
3.  **Recebimento de Poses:** O nó recebe dezenas de candidatos de preensão.
4.  **Tentativa e Erro:** O nó itera sobre as poses recebidas, tentando planejar movimentos válidos com o MoveIt. Se falhar, ele pode solicitar um novo scan ou tentar outra pose, respeitando um limite máximo de tentativas configurável.

---

Segurança e Navegação Reativa
-----------------------------

O planejador garante a segurança do robô através de condições de guarda ("Guards") que rodam a cada ciclo:

* **IsPathClear:** Verifica continuamente se o caminho planejado para a base móvel foi obstruído por um obstáculo dinâmico (ex: uma pessoa cruzando a frente do robô). Se detectar colisão iminente, aborta a navegação imediatamente.
* **IsReachable:** Antes de tentar mover o braço, verifica geometricamente se o alvo está dentro do volume de trabalho do robô e se existe uma solução de cinemática inversa válida, evitando movimentos inúteis que resultariam em falha.

---

Parâmetros Principais
---------------------

* ``bt_xml_path``: Caminho absoluto para o arquivo .xml da Behavior Tree (usado como *fallback* ou missão inicial).
* ``yaml_file``: Caminho para o arquivo de configuração de objetos permitidos e metadados.
* ``use_graspnet``: Habilita/Desabilita o uso de inferência neural para preensão.
* ``max_graspnet_attempts``: Número máximo de tentativas de re-scan e re-planejamento caso a preensão falhe.