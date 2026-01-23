IK Validator (v1.0 - Legado)
============================

O nó **IKValidator** é o componente mais computacionalmente intensivo do sistema de planejamento.
Sua função é validar se uma posição candidata para a base do robô (x, y) permite, geometricamente e cinematicamente, que o *End-Effector* alcance a pose de pega desejada.

Diferente de uma verificação simples de distância, este nó instancia uma simulação completa do robô utilizando a **MoveIt Core API**, considera a orientação da base, os limites das juntas (*joint limits*) e as auto-colisões.

---

1. Inicialização Assíncrona e Monitoramento de Cena
---------------------------------------------------

O MoveIt 2 exige o carregamento do URDF e SRDF, o que pode demorar alguns segundos. Para não bloquear o funcionamento principal do sistema, utiliza-se um padrão de **Inicialização Atrasada**.

O nó mantém um monitoramento da cena de planejamento que preserva uma cópia em tempo real do ambiente do robô (incluindo obstáculos adicionados dinamicamente via sensores).

**Junta Virtual (Virtual Joint):**
O conceito chave aqui é a detecção automática da "Junta Virtual".
No MoveIt, para mover a base de um manipulador móvel durante o planejamento sem mover o robô real, define-se uma junta planar ou flutuante conectando o mundo à base do robô.
O nó busca essa junta automaticamente para injetar as posições candidatas (x, y) durante a validação.

---

2. O Algoritmo de Validação
---------------------------

Esta é a lógica central do componente. Ela recebe uma lista de pontos candidatos e retorna a melhor posição base onde o cálculo de Cinemática Inversa (IK) foi bem-sucedido.

Configuração da Cena (Thread-Safe)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Antes de iterar sobre os candidatos, é preciso "congelar" a cena atual para garantir que obstáculos não se movam durante o cálculo matemático.
Utiliza-se um mecanismo de leitura bloqueada e cria-se uma diferença (*diff*) da cena para modificar o estado localmente sem afetar o sistema global.

**Gerenciamento de Colisões (ACM):**
Para pegar um objeto, o *gripper* precisa tocar nele. Por padrão, o planejador considera isso uma colisão.
A lógica do validador manipula a **Allowed Collision Matrix (ACM)** para permitir explicitamente o contato entre os elos da garra e o objeto alvo durante os testes.

Callback de Validação Customizado
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

O *solver* de IK padrão resolve apenas a matemática das juntas. Ele não sabe necessariamente se o cotovelo do robô colidirá com a parede externa na configuração encontrada.
Por isso, injeta-se uma rotina de validação que é chamada a cada solução matemática encontrada para testar colisões reais na geometria da cena.

Loop de Verificação e Orientação da Base
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Para cada ponto candidato (x, y) na lista:

1.  **Verificação 2D:** Consulta-se o grafo de obstáculos para garantir que a base não esteja dentro de uma parede ou zona proibida.
2.  **Cálculo de Orientação (Yaw):** O robô deve estar virado para o objeto. Calcula-se o ângulo necessário para alinhar a base com o alvo:
    
    .. math::
       \theta_{base} = \text{atan2}(y_{obj} - y_{base}, x_{obj} - x_{base})

3.  **Injeção na Junta Virtual:** Move-se o robô "imaginário" na cena de planejamento para a posição ($x, y, \theta$).
4.  **Resolução de IK:** Aciona-se o *solver* de cinemática inversa. Se o *solver* encontrar uma configuração de juntas válida para o braço alcançar o alvo *a partir dessa base específica*, o ponto é validado e salvo.

---

3. Demonstração Visual
----------------------

Explicação das marcações visuais:

1. **Quadrados pretos:** Representam o grafo de obstáculos do ambiente, indicando áreas onde a base não pode transitar.
2. **Quadrados vermelhos:** Representam a área de alcançabilidade teórica projetada no chão, sem considerar obstáculos físicos para o braço.
3. **Quadrados azuis:** Indicam os pontos validados por este nó, onde o robô consegue efetivamente posicionar a base e estender o braço para interagir com o objeto sem colisões.

Sem parede bloqueando o objeto
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da validação de IK considerando obstáculos.</em></p>
       <img src="../../../_static/manipulation/ik_validator_sem_parede.png" alt="Visualização da validação de IK considerando obstáculos" style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da BFS com o modelo do robô ao lado do objeto.</em></p>
       <img src="../../../_static/manipulation/ik_validator_sem_parede_e_robo.png" alt="Visualização lateral do Triângulo de Cálculo." style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da simulação real.</em></p>
       <img src="../../../_static/manipulation/simulacao_real_sem_parede_ik_validator.png" alt="Screenshot do RViz." style="width: 100%; height: auto;">
   </div>


Com parede bloqueando o objeto
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da validação de IK considerando obstáculos.</em></p>
       <img src="../../../_static/manipulation/ik_validator_com_parede.png" alt="Visualização da validação de IK considerando obstáculos" style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da BFS com o modelo do robô ao lado do objeto.</em></p>
       <img src="../../../_static/manipulation/ik_validator_com_parede_e_robo.png" alt="Visualização lateral do Triângulo de Cálculo." style="width: 100%; height: auto;">
   </div>

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc; padding: 10px;">
       <p><em>Visualização da simulação real.</em></p>
       <img src="../../../_static/manipulation/simulacao_real_com_parede_ik_validator.png" alt="Screenshot do Isaac Sim." style="width: 100%; height: auto;">
   </div>