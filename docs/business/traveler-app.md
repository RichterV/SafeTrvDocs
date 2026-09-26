# App do Viajante — regras de negócio

> **Status: especificação, ainda não implementada.** Definida em 2026-09-25 a partir de decisões do responsável pelo produto. Descreve **o que** o app do motorista deve fazer, não **como** construir. Os números (distâncias, tempos) são valores iniciais, ajustáveis depois de testes com motoristas — ver [Parâmetros](#parametros-ajustaveis).

Esta página cobre o aplicativo que o **Viajante** (motorista) usa durante a viagem, no celular. Para as regras gerais do sistema (papéis, planejamento, índice de risco), ver [Regras de negócio](rules.md).

## Em uma frase

O app acompanha o motorista do início ao fim da viagem pelo GPS do celular. Enquanto a viagem acontece, ele continua vigiando o clima do caminho à frente e avisa por voz e som quando aparece um risco. Se existir um caminho mais seguro, sugere trocar; se não existir, orienta a dirigir com cautela e a considerar uma parada segura.

## Visão geral da viagem

```
               começa a andar perto da origem
Planejada ──────────────────────────────────────▶ Em andamento ◀──────┐
    │          (ou toca em "Iniciar viagem")        │     │            │ volta a andar
    │                                               │     │ "Pausar"   │ (ou "Retomar")
    │ gerente cancela                   gerente     │     ▼            │
    ▼                                   cancela     │   Pausada ───────┘
Cancelada ◀─────────────────────────────────────────┘
                                                    │ chega ao destino e fica parado
                                                    ▼ (ou toca em "Concluir")
                                                Concluída
```

A **pausa** é uma situação nova — hoje o sistema só conhece planejada, em andamento, concluída e cancelada.

---

## 1. Como o motorista se guia

**RN-01 — Navegação pelo Waze ou Google Maps.** O SafeTrv não dá instruções curva a curva. Ele mostra a rota, o risco de cada trecho e os alertas, e oferece um botão **"Navegar"** que abre o Waze ou o Google Maps (o que o motorista preferir) já com o destino. O SafeTrv continua funcionando por trás, rastreando e alertando.

**RN-02 — Paradas e desvios no aplicativo de navegação.** O Google Maps aceita várias paradas no mesmo trajeto; o Waze só aceita um destino por vez. Por isso:

- Com o **Google Maps**, o SafeTrv abre o trajeto com todas as paradas restantes, e também o caminho alternativo, quando o motorista aceitar um.
- Com o **Waze**, o SafeTrv abre um destino por vez: a próxima parada, ou o próximo ponto do desvio. Ao chegar nele, oferece abrir o seguinte.

**RN-03 — O caminho real pode ser diferente do planejado.** O Waze e o Google Maps podem mudar o caminho por causa do trânsito. O SafeTrv sempre avalia o risco do caminho **que o motorista está de fato fazendo** (ver RN-30).

## 2. Funcionamento com o celular bloqueado

**RN-04 — Continua funcionando em segundo plano.** Com a tela bloqueada, ou com o Waze/Google Maps na frente, o app continua registrando a posição e recebendo alertas. Os alertas chegam como **notificação com som**.

**RN-05 — Permissões necessárias.** Para isso o motorista precisa permitir **localização "o tempo todo"** e **notificações**. O app pede as duas antes da primeira viagem e explica por quê.

**RN-06 — Sem as permissões, o app funciona pela metade e avisa.** Se o motorista negar alguma permissão:

- ele é avisado de que perderá alertas com o app fechado;
- o gerente vê a indicação **"rastreamento limitado"** naquela viagem.

A viagem não é bloqueada.

> Isso muda o escopo original do MVP (CLAUDE.md seção 9), que previa alertas só com o app aberto. Notificação em segundo plano passa a fazer parte do app do Viajante.

## 3. Início da viagem

**RN-10 — Início manual.** O motorista pode tocar em **"Iniciar viagem"** a qualquer momento numa viagem planejada atribuída a ele.

**RN-11 — Início automático.** Uma viagem planejada começa sozinha quando o motorista começa a se deslocar, desde que valham **todas** estas condições:

- a viagem é dele e está planejada;
- ele está a até **2 km da origem**;
- o horário está entre **2 h antes e 2 h depois** da partida prevista;
- o movimento é de veículo: mais de **15 km/h** por pelo menos **2 minutos**. Isso evita disparar ao caminhar ou manobrar no pátio.

**RN-12 — Várias viagens no mesmo dia.** Se mais de uma viagem se encaixar na RN-11, inicia a de partida prevista mais próxima do horário atual.

**RN-13 — Aviso e "desfazer".** Quando a viagem começa sozinha, o motorista recebe o aviso **"Viagem para X iniciada"**. Durante **5 minutos** ele pode tocar em **"Não estou viajando"** para voltar a viagem para planejada. Isso cobre, por exemplo, estar de carona em outro carro perto do pátio.

**RN-14 — Uma viagem de cada vez.** O motorista só pode ter **uma viagem em andamento ou pausada** por vez. Enquanto houver uma, nenhuma outra começa, nem manual nem automaticamente.

**RN-15 — Ao iniciar,** o app recalcula o risco da rota com o horário real de saída, que pode ser diferente do previsto. Se o risco mudou de nível, avisa o motorista antes de ele pegar a estrada.

## 4. Pausa (dormir, comer, abastecer)

**RN-20 — Pausar.** Com a viagem em andamento, o motorista pode tocar em **"Pausar viagem"**. A viagem fica **pausada**.

**RN-21 — Motivo obrigatório.** Ao pausar, o motorista escolhe um motivo:

- descanso/pernoite;
- refeição;
- abastecimento;
- carga/descarga;
- outro (com texto curto).

O motivo e a duração ficam no histórico da viagem, e o gerente vê as duas coisas.

**RN-22 — Durante a pausa.**

- O app registra **onde** a pausa começou e mostra ao gerente o veículo como "parado — motivo X desde HH:MM".
- Não envia o rastro contínuo de posições enquanto a pausa durar, para poupar bateria e preservar a privacidade no pernoite.
- Continua observando, só no celular, se o veículo voltou a andar.

**RN-23 — Retomar.** A viagem volta a ficar em andamento de dois jeitos:

- **manual:** o motorista toca em **"Retomar viagem"**;
- **automático:** ele volta a se deslocar como veículo (mesmo critério da RN-11: mais de 15 km/h por 2 min). O app avisa **"Viagem retomada"**.

**RN-24 — Recalcular o risco ao retomar.** Se a pausa durou mais de **30 minutos**, o app recalcula o risco do restante do caminho com os novos horários de passagem. Depois de um pernoite, a previsão do tempo para o resto da viagem pode ser completamente diferente. Se o risco mudou de nível, o motorista é avisado antes de seguir.

**RN-25 — Lei do Motorista (descanso obrigatório).** A Lei 13.103/2015 exige um descanso de 30 minutos a cada 5h30 de direção contínua. O app:

- avisa quando faltarem **30 minutos** para completar 5h30 dirigindo sem pausa;
- avisa de novo ao atingir 5h30;
- conta como descanso qualquer pausa de **30 minutos ou mais**;
- registra no histórico se o descanso foi cumprido ou não. O gerente vê isso, e o dado serve para a auditoria da empresa.

O app **não bloqueia** a viagem; ele lembra e registra.

**RN-26 — Quem pausa.** Só o motorista da viagem pausa e retoma. O gerente vê a pausa, mas não a controla. Cancelar continua sendo só do gerente (regra atual).

## 5. Acompanhamento do clima durante a viagem

**RN-30 — Reavaliação contínua.** Enquanto a viagem estiver em andamento, o sistema reavalia o risco:

- a cada **15 minutos**;
- **imediatamente** quando o INMET publicar um aviso oficial novo que alcance o caminho.

A reavaliação cobre os trechos que o motorista vai percorrer nas **próximas 3 horas**. Ela parte da posição atual e do horário estimado de chegada em cada trecho, recalculado com a velocidade real. Com a viagem pausada, a reavaliação para e volta ao retomar (RN-24).

**RN-31 — Saiu da rota planejada.** Se o motorista ficar a mais de **1 km** da rota planejada por mais de **5 minutos**, por exemplo porque o Waze desviou por causa do trânsito, o sistema:

- recalcula o caminho da posição atual até o destino, respeitando as paradas que faltam;
- passa a avaliar o risco desse novo caminho.

O gerente é avisado **apenas se o novo caminho tiver risco maior** que o planejado.

**RN-32 — Quando avisar.** O motorista recebe um alerta quando:

- um trecho à frente passa a ter risco **Alto ou Crítico**, e antes não tinha;
- um trecho que já era Alto sobe para **Crítico**;
- surge um **aviso oficial do INMET** novo cobrindo o caminho.

**RN-33 — Sem repetição à toa.** O mesmo trecho com o mesmo nível de risco não gera novo alerta a cada reavaliação. Só gera se o risco **subir** ou se for o **lembrete de proximidade** (RN-43). Quando o risco de um trecho **diminui**, o app atualiza o mapa e a lista, sem notificação sonora.

## 6. O alerta: com e sem caminho alternativo

**RN-40 — Existe caminho alternativo mais seguro.** O alerta diz qual é o risco, onde está e quando o motorista chega lá, e mostra a alternativa: quantos km e minutos a mais, e o risco dela. O motorista escolhe:

- **"Usar alternativa":** o app reabre o Waze/Google Maps com o novo caminho (RN-02);
- **"Seguir na rota atual":** o motorista segue pelo caminho atual; o app o trata como "sem desvio" (RN-41) e repete o aviso perto do trecho (RN-43).

A decisão é **do motorista**, porque é ele quem vê a estrada. O sistema não troca a rota sozinho nem espera aprovação. A escolha fica registrada e **o gerente é avisado** do que foi decidido.

**RN-41 — Não existe desvio.** Quando não há caminho alternativo com risco menor, o alerta:

- avisa para **dirigir com cautela** no trecho, com a orientação adequada ao tipo de risco: por exemplo, reduzir a velocidade e aumentar a distância na chuva forte, ou atenção a pontos de alagamento perto de rio;
- **sugere uma parada segura antes do trecho de risco**, no último ponto de parada razoável antes dele (posto ou cidade). Quando a previsão permitir, também diz **quando o tempo deve melhorar** (ex: "chuva forte até ~16h");
- **avisa o gerente** de que o motorista vai passar por um trecho de risco sem alternativa;
- exige **"Ciente"** (RN-44).

**RN-42 — Parar é decisão do motorista.** Se ele aceitar a parada sugerida, o app pausa a viagem com o motivo "aguardando clima melhorar". Ao retomar, vale a RN-24.

**RN-43 — Lembrete perto do trecho.** Além do aviso na hora em que o risco é detectado, o motorista recebe um **lembrete quando estiver chegando** ao trecho de risco: cerca de **5 km** ou **5 minutos** antes, o que vier primeiro. O lembrete vale para os dois casos: sem desvio, ou quando o motorista preferiu seguir na rota atual.

**RN-44 — "Ciente" e segurança ao volante.** Mexer no celular dirigindo é perigoso e proibido pelo Código de Trânsito. Por isso:

- com o veículo **em movimento**, o alerta é **lido em voz alta**, com som e notificação, e não exige nenhum toque;
- o **"Ciente"** pode ser dado por um botão grande na tela ou fica **pendente até o veículo parar** (velocidade quase zero por 1 minuto, ou uma pausa);
- quando o veículo para, o app mostra os alertas pendentes para o motorista confirmar;
- o sistema registra quando o alerta foi enviado, quando foi lido em voz alta e quando o motorista deu "Ciente". Isso é a trilha de auditoria da empresa.

**RN-45 — O gerente vê tudo.** Todo alerta enviado ao motorista também aparece no painel do gerente, com a situação: enviado, lido em voz alta, ciente, desvio aceito ou recusado, parada aceita.

## 7. Sem sinal de internet

**RN-50 — O app leva a rota consigo.** Ao iniciar a viagem, e a cada reavaliação, o app guarda no celular a rota, os trechos de risco conhecidos e a última avaliação.

**RN-51 — Alertas mesmo sem internet.** Sem conexão, o app continua usando o GPS, que não depende de internet. Ao se aproximar de um trecho de risco já conhecido, ele avisa normalmente (RN-43).

**RN-52 — Nada se perde.** As posições registradas sem conexão ficam guardadas e são enviadas quando o sinal voltar. O gerente vê o trajeto completo depois, e o app mostra um aviso discreto: "sem conexão — alertas com base na avaliação das HH:MM".

**RN-53 — Limite do offline.** Sem internet o sistema **não consegue descobrir riscos novos**: a reavaliação e os avisos novos do INMET dependem da conexão. Assim que o sinal volta, o app faz uma reavaliação imediata.

## 8. Chegada e conclusão

**RN-60 — Conclusão automática.** Quando o motorista chega a até **500 m do destino** e fica **parado por 10 minutos**, a viagem é concluída sozinha e ele recebe o aviso "Viagem concluída".

**RN-61 — Desfazer.** Por **30 minutos** depois da conclusão automática, o motorista pode tocar em **"Ainda não cheguei"** para voltar a viagem para em andamento. Isso cobre, por exemplo, quando ele parou perto do destino mas a entrega é em outro portão.

**RN-62 — Conclusão manual continua valendo.** O motorista, ou o gerente, pode concluir manualmente a qualquer momento, como já funciona hoje.

**RN-63 — Paradas intermediárias** não concluem a viagem: ao chegar numa parada, o app marca a parada como feita e oferece navegar até a próxima (RN-02).

---

## Parâmetros ajustáveis

Valores iniciais. Devem ser calibrados com testes reais e, no futuro, podem virar configuração por empresa.

| Regra | Parâmetro | Valor inicial |
|---|---|---|
| RN-11 | Distância máxima da origem para o início automático | 2 km |
| RN-11 | Janela de horário em torno da partida prevista | ± 2 h |
| RN-11, RN-23 | O que conta como "está dirigindo" | > 15 km/h por 2 min |
| RN-13 | Prazo para desfazer o início automático | 5 min |
| RN-24 | Pausa a partir da qual o risco é recalculado ao retomar | 30 min |
| RN-25 | Direção contínua máxima / descanso mínimo (Lei 13.103/2015) | 5h30 / 30 min |
| RN-25 | Antecedência do lembrete de descanso | 30 min |
| RN-30 | Intervalo da reavaliação do clima | 15 min |
| RN-30 | Horizonte da reavaliação | próximas 3 h |
| RN-31 | Distância / tempo para considerar "fora da rota" | 1 km / 5 min |
| RN-43 | Antecedência do lembrete perto do trecho de risco | 5 km ou 5 min |
| RN-44 | O que conta como "veículo parado" | velocidade ~0 por 1 min |
| RN-60 | Raio / tempo parado para concluir automaticamente | 500 m / 10 min |
| RN-61 | Prazo para desfazer a conclusão automática | 30 min |

## O que muda no sistema atual

Para quem vai implementar — diferenças em relação ao que existe hoje:

- **Novo status "pausada"** e novas transições: em andamento ⇄ pausada, e pausada → cancelada (pelo gerente). Hoje existem só planejada, em andamento, concluída e cancelada.
- **Reavaliação do risco durante a viagem.** Hoje o risco é calculado **uma única vez**, ao criar a viagem.
- **Alertas com ciclo de vida:** enviado → lido em voz alta → ciente, com a decisão sobre o desvio (aceito ou recusado) e a parada. Hoje o alerta tem só "reconhecido: sim/não".
- **Notificação em segundo plano** (push), GPS em segundo plano e app nativo (Capacitor). Estavam fora do MVP.
- **Início e conclusão automáticos** pelo celular do motorista. São compatíveis com as regras atuais de quem pode mudar o status: é o próprio viajante agindo, pelo app.
- **Registro de direção e descanso** para a Lei do Motorista (hoje no backlog de Horizonte 2).

## Pontos em aberto

Pontos para confirmar antes de implementar:

1. **Rastro durante a pausa (RN-22).** Hoje só a localização do início da pausa é enviada. Se a empresa quiser saber onde o veículo está durante o pernoite (por segurança da carga, por exemplo), isso muda.
2. **"Parada segura" (RN-41)** hoje significa "o último posto ou cidade antes do trecho". Uma lista curada de pontos de parada seguros (Horizonte 2 do backlog) melhoraria muito essa sugestão.
3. **Pernoite e a regra das 11 h entre jornadas.** A Lei do Motorista também exige 11 h de descanso entre jornadas. A RN-25 cobre só o descanso de 30 min a cada 5h30; incluir as 11 h depende de o sistema conhecer a jornada, e não só a viagem.
4. **Viagem pausada por muito tempo.** Não há limite de duração da pausa, nem aviso ao gerente por pausa longa — isso foi uma escolha explícita. Uma pausa esquecida, porém, deixaria a viagem "pausada" indefinidamente.
5. **Piso de risco do aviso oficial do INMET.** Hoje é Alto, e a página de regras gerais fala em Crítico. Isso afeta quais alertas a RN-32 dispara. Decisão pendente, registrada no backlog.
