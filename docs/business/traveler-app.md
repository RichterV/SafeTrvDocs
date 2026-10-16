# App do Viajante — regras de negócio

> **Status: parcialmente implementada** — a parte do servidor e da tela web desde 2026-09-26; o app nativo, ainda não. Ver [O que já está implementado](#o-que-ja-esta-implementado). Definida em 2026-09-25 a partir de decisões do responsável pelo produto. Descreve **o que** o app do motorista deve fazer, não **como** construir. Os números (distâncias, tempos) são valores iniciais, ajustáveis depois de testes com motoristas — ver [Parâmetros](#parametros-ajustaveis).

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
- Não envia o rastro contínuo de posições enquanto a pausa durar, para poupar bateria e preservar a privacidade no pernoite. Só a posição do início da pausa é enviada (confirmado em 2026-09-26).
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

**RN-27 — Pausa longa.** A pausa não tem limite de duração e o sistema nunca muda o status da viagem sozinho por causa dela. Mas, se a viagem ficar pausada por mais de **14 horas** (as 11 h de descanso entre jornadas, com folga), o gerente recebe **um** aviso: "viagem pausada há mais de 14 h — motivo X desde HH:MM". Serve para pegar uma pausa esquecida. O aviso sai uma vez por pausa; o motorista não é avisado.

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
- surge um **aviso oficial do INMET** novo cobrindo o caminho, de qualquer severidade. A mensagem diz qual é (Perigo Potencial, Perigo ou Grande Perigo), e o nível de risco do trecho segue a graduação das [regras gerais](rules.md).

**RN-33 — Sem repetição à toa.** O mesmo trecho com o mesmo nível de risco não gera novo alerta a cada reavaliação. Só gera se o risco **subir** ou se for o **lembrete de proximidade** (RN-43). Quando o risco de um trecho **diminui**, o app atualiza o mapa e a lista, sem notificação sonora.

## 6. O alerta: com e sem caminho alternativo

**RN-40 — Existe caminho alternativo mais seguro.** O alerta diz qual é o risco, onde está e quando o motorista chega lá, e mostra a alternativa: quantos km e minutos a mais, e o risco dela. O motorista escolhe:

- **"Usar alternativa":** o app reabre o Waze/Google Maps com o novo caminho (RN-02);
- **"Seguir na rota atual":** o motorista segue pelo caminho atual; o app o trata como "sem desvio" (RN-41) e repete o aviso perto do trecho (RN-43).

A decisão é **do motorista**, porque é ele quem vê a estrada. O sistema não troca a rota sozinho nem espera aprovação. A escolha fica registrada e **o gerente é avisado** do que foi decidido.

**Limite de custo do desvio** (decidido em 2026-10-10, vale também para a rota alternativa do planejamento): o sistema só sugere um caminho alternativo se ele custar **no máximo +30% do tempo da viagem e no máximo +60 minutos**. Um desvio mais caro que isso não é oferecido: o alerta segue como "sem desvio" (RN-41), com a parada segura.

**RN-41 — Não existe desvio.** Quando não há caminho alternativo com risco menor, o alerta:

- avisa para **dirigir com cautela** no trecho, com a orientação adequada ao tipo de risco: por exemplo, reduzir a velocidade e aumentar a distância na chuva forte, ou atenção a pontos de alagamento perto de rio;
- **sugere uma parada segura antes do trecho de risco**: o último **posto de combustível ou área urbana** antes do trecho, buscado no OpenStreetMap ao longo da rota (decidido em 2026-10-10; não depende de cadastro). Quando a previsão permitir, também diz **quando o tempo deve melhorar** (ex: "chuva forte até ~16h");
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
| RN-27 | Pausa a partir da qual o gerente é avisado | 14 h |
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

## O que já está implementado

Em 2026-09-26, a parte do **servidor** e da **tela web** (detalhes técnicos em [Referência da API](../system/api-reference.md#patch-tripstrip_idstatus) e [Serviços internos](../system/services.md#acompanhamento-durante-a-viagem-trip_monitorpy)):

| Regra | Situação |
|---|---|
| RN-10 início manual | ✅ |
| RN-11 início automático | ⏳ servidor pronto (aceita "foi automático"); a detecção de movimento é do app nativo |
| RN-13 desfazer o início automático (5 min) | ✅ servidor; o botão fica para o app nativo |
| RN-14 uma viagem de cada vez | ✅ |
| RN-15 recalcular o risco ao iniciar | ✅ |
| RN-20, RN-21, RN-22, RN-26 pausa com motivo, só pelo motorista, só a posição do início | ✅ |
| RN-23 retomar | ✅ manual; o automático é do app nativo |
| RN-24 recalcular ao retomar depois de 30 min | ✅ |
| RN-25 Lei do Motorista (5h30 / 30 min) | ❌ ainda não |
| RN-27 aviso de pausa longa (14 h) | ✅ |
| RN-30 reavaliação a cada 15 min e com aviso novo do INMET | ✅ (o acompanhamento roda a cada 5 min; cada viagem é reavaliada a cada 15 min, ou na hora se surgir um aviso novo no caminho) |
| RN-31 fora da rota planejada | ❌ ainda não |
| RN-32, RN-33 quando avisar, sem repetição | ✅ (trechos seguidos viram um alerta só) |
| RN-40 caminho alternativo durante a viagem | ❌ ainda não — hoje o alerta é sempre o de "sem desvio" |
| RN-41 sem desvio: cautela | ✅ o texto orienta como dirigir; falta a parada segura sugerida e "quando o tempo melhora" |
| RN-42 parar é decisão do motorista | ✅ registro da decisão; o app pausa com "aguardando o clima melhorar" |
| RN-43 lembrete perto do trecho | ❌ app nativo (depende do GPS em segundo plano) |
| RN-44 lido em voz alta e "Ciente" | ✅ registro das etapas; na web o alerta é lido em voz alta com o app aberto |
| RN-45 o gerente vê tudo | ✅ ao vivo |
| RN-50 a RN-53 offline | ❌ app nativo |
| RN-60, RN-61 conclusão automática e desfazer (30 min) | ⏳ servidor pronto; a detecção é do app nativo |

Decisões tomadas na implementação, confirmadas pelo responsável pelo produto em 2026-10-10:

- **Trechos seguidos que sobem de risco viram um alerta só**, em vez de um alerta por trecho.
- **Os horários de passagem são recalculados pelo atraso real**, não pela velocidade instantânea: se o motorista está 30 min atrasado em relação ao plano, todo o resto anda 30 min. A velocidade do momento oscila demais (trânsito, paradas curtas) para prever horas à frente.
- **Só avisos de chuva/tempestade do INMET contam** para o índice e para a RN-32 (ver as [regras gerais](rules.md)). Aviso de **vendaval** fica de fora por enquanto: entra junto com o fator "vento forte" do roadmap, que considera o tipo de veículo.

## Pontos em aberto

Pontos para confirmar antes de implementar:

1. **Pernoite e a regra das 11 h entre jornadas.** A Lei do Motorista também exige 11 h de descanso entre jornadas. A RN-25 cobre só o descanso de 30 min a cada 5h30; incluir as 11 h depende de o sistema conhecer a jornada, e não só a viagem.

Decididos em 2026-09-26:

- **Rastro durante a pausa:** só a posição do início da pausa (RN-22).
- **Pausa esquecida:** aviso ao gerente depois de 14 h (RN-27).
- **Piso do aviso oficial do INMET:** graduado pela severidade do aviso (ver as [regras gerais](rules.md)).

Decididos em 2026-10-10:

- **Limite de custo do desvio:** no máximo +30% do tempo da viagem e +60 min (RN-40).
- **Parada segura:** último posto ou área urbana antes do trecho, pelo OpenStreetMap (RN-41). Uma lista curada de pontos de parada (Horizonte 2 do backlog) pode complementar depois.
- **Vendaval:** fora do índice por enquanto.
- **Horários pelo atraso real** e **trechos seguidos = um alerta**: confirmados.
