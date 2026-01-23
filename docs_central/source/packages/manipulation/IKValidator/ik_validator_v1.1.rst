IKValidator (v1.1 - Legado)
===============================================

O nó **IKValidator v1.1** evolui seu predecessor ao introduzir **memória e persistência de estado**.
Enquanto a versão 1.0 apenas calculava e retornava um ponto, a versão 1.1 **lembra** qual decisão tomou.
Isso é fundamental para comportamentos reativos (Reactive Behavior Trees), permitindo que o robô verifique continuamente se o plano traçado milissegundos atrás ainda é válido diante de mudanças dinâmicas no ambiente (ex: uma pessoa entrando na frente do robô), sem o custo computacional de recalcular tudo do zero.

.. note::
   **Novidade Principal:** :ref:`revalidacao_reativa_v1_1`.

---

1. Inicialização Assíncrona
---------------------------

O padrão de inicialização tardia (*Delayed Init*) é mantido para evitar o bloqueio do ciclo principal (*spin*) do ROS durante o carregamento pesado dos modelos cinemáticos (URDF/SRDF).
Na versão 1.1, o tempo de espera inicial foi ajustado para garantir maior estabilidade na disponibilidade dos serviços de transformação (TF) antes de liberar o nó para processamento.

---

2. O Cálculo Principal com Persistência
---------------------------------------

A rotina de busca de base é responsável pelo processamento pesado. Ela itera sobre dezenas de candidatos, verifica colisões 2D (navegação), colisões 3D (manipulação), limites de juntas e soluções de cinemática inversa.

A grande mudança na v1.1 está no encerramento desta rotina: **o salvamento do contexto**.
Ao encontrar uma solução válida, o nó não apenas a retorna para a Behavior Tree, mas armazena internamente três informações cruciais:

1.  **Posição Escolhida:** A coordenada exata onde o robô decidiu parar.
2.  **Alvo do Movimento:** Qual era o objeto ou pose que o robô tentava alcançar.
3.  **Contexto de Colisão:** Qual objeto específico o robô recebeu permissão para tocar (ex: o objeto a ser manipulado).

Esse armazenamento de estado prepara o terreno para a função de monitoramento rápido descrita abaixo.

---

.. _revalidacao_reativa_v1_1:

3. Revalidação Reativa (Monitoramento)
--------------------------------------

Esta é a principal inovação da versão 1.1.
Projetada para rodar dentro de nós condicionais da Behavior Tree (como ``<IsStillReachable/>``) em alta frequência (ex: 50Hz), esta função foi otimizada para ser extremamente rápida e leve.

Uso do Estado Interno
^^^^^^^^^^^^^^^^^^^^^
Para funcionar sem exigir que a Behavior Tree reenvie todos os parâmetros a cada ciclo (o que causaria overhead de comunicação), esta rotina depende totalmente do **Estado Interno** persistido na memória do nó. Ela recupera as decisões tomadas anteriormente para reconstruir a cena de simulação exata da decisão original.

Lógica de Execução:
^^^^^^^^^^^^^^^^^^^

1.  **Verificação de Sobrevivência (Fail-Fast):** Se o nó ainda não calculou nada (estado vazio), a validação retorna falso imediatamente.
2.  **Validação 2D Instantânea:** Verifica se a posição de base *salva* foi invadida no mapa dinâmico de custos. Se um obstáculo apareceu onde o robô está (ou planeja estar), a operação é abortada.
3.  **Reconstrução de Cena:** Recria o ambiente de colisão, reaplicando as permissões de toque salvas anteriormente.
4.  **Verificação Cinemática Pontual:** Diferente da busca inicial que testa múltiplos candidatos, esta etapa testa **apenas um**: a configuração salva.
    * Move o "robô virtual" para a posição salva.
    * Tenta resolver a cinemática inversa para o alvo salvo.
    * Verifica se surgiram novas colisões 3D.

Se esta função retornar ``false``, a Behavior Tree saberá instantaneamente que o plano atual se tornou inválido e acionará a parada de emergência ou o replanejamento da navegação.

---

4. Demonstração Visual
----------------------

.. raw:: html

   <div style="text-align: center; margin-bottom: 20px; border: 1px solid #ccc;">
       <video width="100%" height="auto" controls autoplay loop muted>
           <source src="../../../_static/manipulation/ik_validator_v1.1.webm" type="video/webm">
           Seu navegador não suporta vídeos HTML5.
       </video>
       <p><em>Figura 1: Exemplo do comportamento reativo abortando uma tarefa ao detectar um obstáculo dinâmico.</em></p>
   </div>