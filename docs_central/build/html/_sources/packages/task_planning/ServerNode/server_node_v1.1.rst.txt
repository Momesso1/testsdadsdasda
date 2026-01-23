Server Node (Planejador de Tarefas v1.1)
========================================

O **Server Node v1.1** representa uma evolução significativa na inteligência do robô. Atuando como o cérebro central do sistema, este componente orquestra a navegação, a manipulação e a tomada de decisão através de **Behavior Trees (Árvores de Comportamento)**.

Diferente da versão anterior, que operava de forma puramente reativa, esta versão integra uma arquitetura deliberativa com verificação contínua, utilizando acesso direto à memória compartilhada para maximizar a performance.

.. image:: https://placeholder.com/wp-content/uploads/2018/10/placeholder.com-logo1.png
   :alt: Diagrama da Arquitetura Híbrida Reativa-Deliberativa
   :align: center

.. note::
   **Arquitetura de Container:** O nó atua como um container de alto desempenho que hospeda diversos sub-componentes (Validadores Cinemáticos, Grafos de Obstáculos, Analisadores de Alcance). Estes componentes são instanciados via composição, eliminando a latência de rede típica da comunicação entre processos.

---

1. Novidades da Versão 1.1
--------------------------

A transição para a versão 1.1 foca na segurança e na previsibilidade das ações do robô.

* **Validação Cinemática Prévia:** Antes de iniciar qualquer movimento da base, o robô verifica geometricamente se a posição de destino permitirá que o braço alcance o objeto desejado.
* **Monitoramento Dinâmico de Caminho:** Enquanto o robô está ocupado calculando ou executando tarefas, o sistema verifica continuamente se o caminho planejado foi obstruído por novos obstáculos móveis.
* **Acesso O(1) ao Mapa:** O grafo de obstáculos não depende mais de callbacks de tópicos lentos; ele é acessado instantaneamente via ponteiros de memória compartilhada.
* **Análise de Alcançabilidade Projetada:** O sistema projeta volumes de alcance no ambiente para decidir a melhor pose de aproximação.

---

2. Estrutura e Inicialização
----------------------------

A arquitetura do nó segue o padrão de **Injeção de Dependência**. Durante a inicialização, o planejador recebe referências para todos os subsistemas críticos (Visão, Navegação, Manipulação).

**Modelo de Componentes:**
Em vez de gerenciar múltiplos executáveis espalhados, o sistema centraliza a lógica. Isso permite que a Árvore de Comportamento acesse dados de percepção e mapas de custo diretamente da memória RAM, garantindo decisões em tempo real (< 1ms).

---

3. Lógica da Behavior Tree
--------------------------

A inteligência do robô é descrita por nós lógicos. Abaixo, detalhamos o comportamento esperado de cada tipo de nó disponível no sistema.

Condições de Guarda (Guards)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Estas condições protegem a execução, garantindo que ações só ocorram (ou continuem ocorrendo) se o ambiente estiver seguro.

**Verificação de Caminho Livre (Path Clear)**
Esta condição é vital para a segurança em ambientes dinâmicos.

* **Lógica:** O nó compara a trajetória futura planejada contra o mapa de obstáculos atualizado em tempo real. Se qualquer ponto da trajetória coincidir com um novo obstáculo, a condição falha, gatilhando uma interrupção imediata do movimento do robô.

**Verificação de Alcançabilidade (Reachable)**
Assegura a viabilidade da manipulação antes do deslocamento.

* **Lógica:**
    1. O sistema gera um anel de posições candidatas ao redor do objeto alvo.
    2. Executa testes de Cinemática Inversa (IK) nessas posições considerando os obstáculos 3D.
    3. Se a posição atual do robô for válida, permite a operação. Caso contrário, calcula e sugere uma nova posição de ajuste para a base móvel.

**Verificação de Posse (Gripper Holding)**
Monitora o estado físico da garra.

* **Lógica:** Utiliza sensores de força ou feedback dos motores da garra para confirmar se o objeto está firmemente seguro durante o transporte.

Ações de Navegação
^^^^^^^^^^^^^^^^^^

**Cálculo de Rota (Compute Path)**
Solicita ao planejador global um caminho otimizado entre a posição atual e o destino. Esta operação é assíncrona para não bloquear o ciclo de decisão da árvore.

**Seguir Caminho (Follow Path)**
Envia a trajetória calculada para o controlador de base. Este nó monitora o progresso do robô e reporta sucesso quando o destino é atingido dentro da tolerância estabelecida.

Ações de Percepção
^^^^^^^^^^^^^^^^^^

**Detecção de Objeto (Detect Object)**
Funciona como um filtro inteligente de percepção.

* **Lógica:** O nó analisa o fluxo de dados da visão computacional. Uma vez que um objeto de interesse é identificado e validado, ele é "travado" como o alvo atual, ignorando ruídos subsequentes até que a tarefa seja concluída.

**Limpar Alvo (Clear Target)**
Reseta a memória de curto prazo do robô, permitindo que o sistema de visão busque novos objetos para a próxima tarefa.

Ações de Manipulação
^^^^^^^^^^^^^^^^^^^^

**Pegar e Largar (Pick & Place)**
Interfaces de alto nível para o sistema de controle do braço. Estes nós encapsulam toda a complexidade de planejamento de movimento do braço, gerenciamento de colisão e atuação da garra.

**Gestão de Estoque**
O sistema consulta um banco de dados interno para encontrar espaços vazios em caixas ou prateleiras, garantindo que o robô nunca tente colocar dois objetos no mesmo lugar.

---

4. Fluxo de Dados de Percepção
------------------------------

O sistema opera de forma reativa às mudanças do ambiente através de atualizações contínuas de estado.



* **Odometria:** A posição global do robô é atualizada em alta frequência, servindo de base para todos os cálculos de distância e alcance.
* **Visão Computacional:** As detecções são filtradas por classe e confiança. Quando um objeto novo e válido aparece, o sistema pode interromper comportamentos de "busca" para iniciar comportamentos de "aproximação".

---

5. Evolução Arquitetural (v1.0 vs v1.1)
---------------------------------------

A refatoração para a versão 1.1 trouxe mudanças profundas na forma como as tarefas assíncronas são gerenciadas, priorizando a segurança e a integridade dos dados (Thread-Safety).

Planejamento de Caminho: De Feedback para Atômico
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* **Versão Anterior:** O cliente de ação dependia de callbacks de feedback contínuos para decidir se devia recalcular a rota, criando um acoplamento complexo e propenso a erros.
* **Versão Atual:** O planejamento agora é uma operação atômica (calcula e entrega). A responsabilidade de verificar se o caminho continua válido foi delegada para a Árvore de Comportamento (via nó de *Guarda*), desacoplando o planejamento da execução e aumentando a robustez.

Controle de Base: Gestão de Estado Proativa
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

O sistema agora realiza uma limpeza proativa da memória de caminho ao finalizar ou cancelar movimentos. Isso impede que o robô tente executar fragmentos de trajetórias antigas ou inválidas em caso de reinício rápido de tarefas.

Robustez na Manipulação
^^^^^^^^^^^^^^^^^^^^^^^

A comunicação com o controlador do braço recebeu mecanismos aprimorados de *timeout* e tratamento de erros. Se o servidor de manipulação demorar a responder, o nó central toma a decisão de abortar a tarefa de forma graciosa, em vez de travar a execução da árvore.

Mecanismo de Cancelamento Seguro
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Foi introduzida uma camada de segurança que permite à Árvore de Comportamento solicitar a **Parada Imediata** de qualquer subsistema (base ou braço). Isso é fundamental quando uma *Condição de Guarda* falha (ex: uma pessoa entra na frente do robô), garantindo que o hardware pare instantaneamente antes de colidir.