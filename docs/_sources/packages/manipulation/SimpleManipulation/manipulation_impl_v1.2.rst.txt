manipulation_impl.cpp (Atual v1.2)
==================================

Esta página documenta a evolução da lógica de manipulação (Versão 1.2).
Esta versão expande significativamente a robustez da v1.1, introduzindo duas grandes novidades: a capacidade de **Seguimento de Caminho (Path Following)** e a integração nativa com **GraspNet** para preensão baseada em redes neurais.

O sistema agora suporta dois modos de operação distintos dentro da mesma Action ``PickObject``:
1. **Modo Pick/Place:** Tentativa de alcançar uma pose alvo, pegar o objeto e retentar em caso de falha (herança da v1.1).
2. **Modo Follow Path:** Execução de uma trajetória complexa definida por múltiplos *waypoints*, com validação contínua de IK e colisões.



[Image of logic flow chart]


---

1. Novidade Principal: Seguimento de Caminho (Follow Path)
----------------------------------------------------------

Diferente do movimento ponto-a-ponto tradicional, esta função recebe um vetor de poses e garante uma execução suave e contínua. É ideal para tarefas onde o *End-Effector* precisa percorrer uma geometria específica (ex: inspeção ou aplicação de material).

A função ``follow_path_with_consistent_ik`` realiza:
1. **Validação de IK em Lote:** Verifica se *todos* os pontos do caminho possuem solução de cinemática inversa válida.
2. **Consistência de Juntas:** Garante que a solução de IK do ponto `N` é próxima da solução do ponto `N-1`, evitando movimentos bruscos (flip de cotovelo).
3. **Geração de Trajetória Temporal:** Utiliza o algoritmo ``TimeOptimalTrajectoryGeneration`` (TOTG) para parametrizar a velocidade e aceleração entre os pontos.



.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: follow_path
   :end-before: // DOC-END: follow_path

---

2. Novidade Principal: Integração com GraspNet
----------------------------------------------

A função de cálculo de pose global foi aprimorada para suportar inferências vindas de redes neurais (GraspNet).
Quando o parâmetro ``use_graspnet`` está ativo, o nó aplica transformações de rotação específicas (rotação de $\pi$ em Roll) para alinhar o gripper corretamente com as predileções da rede neural, que muitas vezes utiliza referenciais diferentes do padrão do MoveIt.

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: calculate_global_pose
   :end-before: // DOC-END: calculate_global_pose
   :emphasize-lines: 20-35

---

3. Lógica de Execução da Action (Atualizada)
--------------------------------------------

O método ``execute`` agora atua como um despachante (dispatcher). Ele verifica a flag ``goal->follow_path`` e decide qual estratégia de controle utilizar. Além disso, mantém a verificação rigorosa de cancelamento (preemption) para garantir segurança.

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: execute
   :end-before: // DOC-END: execute

---

4. Componentes Base (Herança v1.1)
----------------------------------

Os componentes abaixo permanecem fundamentais para a operação do nó, garantindo a comunicação com o hardware e a configuração inicial.

Construtor e Configuração
^^^^^^^^^^^^^^^^^^^^^^^^^

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: SimpleManipulation
   :end-before: // DOC-END: SimpleManipulation

Inicialização Tardia e Segurança
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: initMoveGroup
   :end-before: // DOC-END: initMoveGroup

Controle da Garra
^^^^^^^^^^^^^^^^^

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: close_gripper
   :end-before: // DOC-END: close_gripper

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: open_gripper
   :end-before: // DOC-END: open_gripper

Planejamento de Movimento (Híbrido)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Combinação de movimentos cartesianos lineares e planejamento livre (RRT).

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: attempt_cartesian_move
   :end-before: // DOC-END: attempt_cartesian_move

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: positions_for_arm
   :end-before: // DOC-END: positions_for_arm

Gerenciamento de Colisões e Serviços
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: set_collision_allowance
   :end-before: // DOC-END: set_collision_allowance

.. literalinclude:: ../../../../../docs_central/legacy_src/manipulation/Manipulation/manipulation_impl_v1.2.cpp
   :language: cpp
   :start-after: // DOC-START: send_request
   :end-before: // DOC-END: send_request