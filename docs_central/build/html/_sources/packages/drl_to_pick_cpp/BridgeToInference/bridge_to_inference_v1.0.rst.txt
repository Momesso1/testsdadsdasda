Bridge To Inference (v1.0 - Atual)
==================================

O nó ``BridgeToInference`` atua como uma ponte crítica de alto desempenho entre o ecossistema ROS 2 (C++) e um servidor de inferência externa (geralmente Python/PyTorch).

Diferente de uma simples troca de mensagens ROS, este nó implementa um cliente TCP customizado utilizando **MessagePack** para serialização binária eficiente de nuvens de pontos. Isso permite latência mínima ao transferir grandes volumes de dados de sensores para redes neurais de geração de grasp (como GraspNet ou Contact-GraspNet).

Arquitetura de Componente (Composable Node)
-------------------------------------------

Este nó é desenvolvido como um **Composable Node** do ROS 2.
Isso significa que ele pode ser instanciado dentro de um *Container* existente (como o do MoveIt ou do driver da câmera) evitando overhead de comunicação entre processos se necessário, embora sua função primária seja I/O de rede.



A arquitetura permite que outros componentes C++ (como nós de Behavior Tree) acessem os dados de grasp diretamente via ponteiros de memória (métodos públicos), sem a necessidade de serialização/deserialização de mensagens de tópicos ROS.

Parâmetros de Configuração
--------------------------

Os seguintes parâmetros podem ser definidos no arquivo de lançamento ou via linha de comando:

* **server_host** (string, default: "localhost"): Endereço IP do servidor de inferência.
* **server_port** (int, default: 5000): Porta TCP do servidor.
* **score_threshold** (double, default: 0.15): Valor mínimo de confiança para aceitar um grasp.
* **max_grasps** (int, default: 50): Número máximo de grasps a serem retornados e armazenados.
* **target_frame** (string, default: "world"): Frame de referência para publicação (se ativada).

.. literalinclude:: ../../../../../docs_central/legacy_src/drl_to_pick_cpp/BridgeToInference/bridge_to_inference_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Constructor
   :end-before: // DOC-END: Constructor

API Pública (Thread-Safe)
-------------------------

A principal vantagem deste nó ser um componente é a capacidade de expor métodos públicos para outros nós no mesmo processo.

.. note::
   O método ``get_latest_grasps`` é protegido por um ``std::mutex``, garantindo que a leitura dos grasps seja segura mesmo enquanto uma nova nuvem de pontos está sendo processada e atualizada na *thread* de callback.

Isso é ideal para **Behavior Trees** que precisam consultar "Quais são os grasps atuais?" instantaneamente em um nó de *Condition* ou *Action*, sem latência de *callbacks* de tópicos.

.. literalinclude:: ../../../../../docs_central/legacy_src/drl_to_pick_cpp/BridgeToInference/bridge_to_inference_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: PublicAPI
   :end-before: // DOC-END: PublicAPI

Fluxo de Dados e Processamento
------------------------------

O fluxo de execução é disparado pela recepção de uma ``sensor_msgs::msg::PointCloud2`` no tópico ``/mapped_object``.

1.  **Conversão:** A mensagem ROS é convertida para PCL (Point Cloud Library).
2.  **Verificação:** Se a nuvem estiver vazia, o cache de grasps é limpo por segurança.
3.  **Inferência:** A nuvem é enviada ao servidor, que retorna matrizes de transformação.
4.  **Atualização:** O vetor interno ``latest_grasps_`` é atualizado atomicamente.



.. literalinclude:: ../../../../../docs_central/legacy_src/drl_to_pick_cpp/BridgeToInference/bridge_to_inference_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Callback
   :end-before: // DOC-END: Callback

Protocolo de Comunicação (TCP/MsgPack)
--------------------------------------

A comunicação com o servidor de inferência segue um protocolo binário rigoroso para garantir velocidade.

**Envio (Request):**
O nó envia um mapa MsgPack contendo:
* ``num_points``: Inteiro.
* ``points``: Array linear de floats [x, y, z, x, y, z, ...].

**Recebimento (Response):**
O servidor deve responder com um mapa MsgPack contendo:
* ``scores``: Lista de floats (confiabilidade de cada grasp).
* ``grasps``: Lista linear de matrizes 4x4 achatadas (16 floats por grasp).
* ``centroid``: Vetor [x, y, z] do centro do objeto (usado para translação relativa).



O código abaixo gerencia a abertura do socket, serialização, timeouts e reconstrução das Poses ROS a partir das matrizes recebidas:

.. literalinclude:: ../../../../../docs_central/legacy_src/drl_to_pick_cpp/BridgeToInference/bridge_to_inference_v1.0.cpp
   :language: cpp
   :start-after: // DOC-START: Networking
   :end-before: // DOC-END: Networking