# Regras de negócio (explicadas sem jargão técnico)

Esta página explica como o SafeTrv funciona do ponto de vista de quem usa o produto no dia a dia — sem termos de programação. Se você é gestor, motorista, ou está avaliando o produto comercialmente, esta é a página certa. Para detalhes técnicos de como isso é implementado, ver [Sistema](../system/overview.md).

## Quem usa o sistema

Cada empresa cliente tem sua própria área isolada no sistema — chamamos isso de **Conta**. Ninguém de uma empresa vê dados de outra. Dentro de uma Conta, existem três tipos de usuário:

### Administrador

É quem tem acesso total dentro da empresa. Cria e remove Gerentes e Viajantes, vê todas as viagens, todos os veículos e todos os usuários da empresa, e configura os dados gerais da conta. Normalmente é alguém da diretoria ou da área de operações/segurança.

### Gerente de viagem

É quem planeja as viagens no dia a dia: escolhe origem, destino, qual motorista vai, qual veículo, e acompanha o andamento em tempo real. Pode cadastrar novos Viajantes (motoristas), mas não pode criar outros Gerentes nem Administradores — só quem já tem acesso maior pode conceder mais acesso, nunca o contrário.

### Viajante

É o motorista ou responsável pelo trajeto. Só vê as viagens que foram atribuídas a ele. Usa o aplicativo do celular como um GPS durante a viagem, e recebe alertas sonoros e visuais se algo arriscado aparecer no caminho.

**Uma pessoa pode ter mais de um papel.** Por exemplo, um Administrador também pode planejar viagens como se fosse Gerente — não precisa de um segundo usuário para isso. A única regra fixa é: você só consegue dar a outra pessoa um nível de acesso igual ou menor que o seu. Um Gerente nunca pode "promover" alguém a Gerente ou Administrador — só um Administrador pode fazer isso.

## Como uma viagem é planejada

1. O Gerente cadastra a viagem informando: de onde para onde (podendo incluir paradas no meio do caminho), qual motorista vai, qual veículo, que tipo de carga está sendo transportada, e o horário previsto de saída.
2. O sistema calcula automaticamente o caminho a percorrer e divide esse caminho em pedaços de aproximadamente 12 minutos de trajeto cada um — chamamos cada pedaço de **segmento**.
3. Para cada segmento, o sistema calcula o horário em que o veículo deve passar por ali e consulta: qual é a previsão de chuva para aquele horário e local, e o quão perto esse trecho passa de um rio que pode transbordar.
4. Com essas informações, cada segmento recebe uma **nota de risco de 0 a 100**, e a viagem inteira recebe uma classificação geral baseada no **pior segmento** — ou seja, se um único trecho no meio de uma viagem de 8 horas for muito arriscado, a viagem inteira é sinalizada como arriscada, mesmo que o resto do caminho esteja tranquilo. Isso é proposital: não queremos que um problema seja "diluído" numa média e passe despercebido.
5. A viagem é então atribuída ao motorista, que a recebe no aplicativo do celular.
6. Durante o trajeto, o aplicativo do motorista envia a localização em tempo real, e o Gerente acompanha isso num painel no computador.
7. Se algo perigoso for identificado no caminho (chuva forte, proximidade de área de risco), tanto o motorista quanto o Gerente recebem um alerta.

Uma viagem pode ser cadastrada com até **7 dias de antecedência**. Quanto mais perto da data de saída, mais confiável fica a previsão do tempo usada — então o sistema foi pensado para recalcular o risco conforme a data se aproxima (essa parte de recálculo automático ainda está no roadmap, não implementada).

## Como interpretar o nível de risco

| Nível | O que significa na prática |
|---|---|
| 🟢 **Baixo** | Sem sinais relevantes de risco climático no trajeto. |
| 🟡 **Médio** | Alguma chance de chuva ou proximidade de rio, mas ainda dentro do esperado — vale acompanhar, não é motivo para adiar a viagem. |
| 🟠 **Alto** | Combinação de fatores (chuva forte, proximidade de rio, carga sensível, horário noturno) que justifica atenção redobrada — o sistema já sinaliza que vale considerar um caminho alternativo. |
| 🔴 **Crítico** | Situação de risco elevado, incluindo os casos em que há um **aviso oficial do governo** (INMET) de "Grande Perigo" cobrindo aquele trecho no horário da viagem — recomenda-se fortemente rever a rota ou o horário antes de sair. |

**Um detalhe importante:** se o governo (INMET) emitiu um aviso oficial cobrindo a área e o horário por onde o veículo vai passar, o trecho recebe um **nível mínimo** de risco, conforme a gravidade do aviso:

| Aviso do INMET | Cor do aviso | Nível mínimo do trecho |
|---|---|---|
| Perigo Potencial | amarelo | 🟡 Médio |
| Perigo | laranja | 🟠 Alto |
| Grande Perigo | vermelho | 🔴 Crítico |

Só contam os avisos de **chuva ou tempestade** (ex: "Tempestade", "Chuvas Intensas", "Acumulado de Chuva"). Avisos de onda de calor, baixa umidade, geada ou vento ainda não entram no índice, que hoje olha só para chuva e rio.

Se os outros cálculos já apontarem um risco maior, vale o maior — o aviso só sobe o nível, nunca baixa. A lógica é: se o governo já emitiu um alerta oficial, o sistema não vai contradizer isso. Avisos amarelos são muito comuns e costumam cobrir estados inteiros, por isso levam só a "Médio": não geram sozinhos um alerta de risco nem uma busca de rota alternativa.

**Outro detalhe:** viagens transportando **carga perigosa** (combustíveis, químicos) recebem uma nota de risco mais alta nos trechos onde já existe risco de alagamento — o mesmo nível de chuva/proximidade de rio pesa mais quando a carga em si já é um risco ambiental caso algo dê errado. Viagens à noite também recebem um pequeno acréscimo no risco, porque chuva combinada com pouca visibilidade noturna aumenta a chance de acidente.

## O que aparece no mapa da viagem

O mapa mostra o caminho colorido pela cor do risco de cada trecho e, quando existe, a rota alternativa sugerida (tracejada). O risco nunca depende só da cor: cada nível tem também uma forma de placa (círculo para Baixo, triângulo para Médio, losango para Alto, octógono para Crítico), a linha fica mais grossa quanto maior o risco e os trechos Alto/Crítico ganham um marcador com a forma.

**Travessia de rio.** Em cada ponto onde o caminho **cruza um rio** (em geral uma ponte), o mapa mostra um **ícone de rio** sobre a rota. Assim o Gerente e o motorista enxergam de relance onde estão os pontos que podem alagar, sem precisar abrir trecho por trecho.

- Contam os mesmos rios usados no cálculo do risco: os de porte relevante. Córregos e riachos pequenos ficam de fora, senão o mapa ficaria coberto de ícones.
- Ao tocar no ícone aparecem o nome do rio (quando a base oficial tem o nome), o horário previsto de passagem e o nível de risco daquele trecho nesse horário.
- O ícone aparece mesmo quando o risco do trecho está Baixo. Ele indica que ali tem um rio, não que há perigo. O perigo continua sendo mostrado pela cor do trecho e pelos alertas.
- Vale para a rota planejada e para a rota alternativa, e aparece tanto no painel do Gerente quanto no app do motorista.
- Um caminho que corre **ao lado** de um rio sem atravessá-lo não ganha ícone. Essa proximidade já entra na nota de risco do trecho.

*Ainda não implementado.*

## Sugestão automática de rota alternativa

Quando a viagem calculada dá risco Alto ou Crítico, o sistema busca automaticamente, no mesmo instante em que a viagem é criada, se existe um caminho alternativo entre a origem e o destino com risco menor. Se encontrar um caminho melhor, ele fica disponível junto com a viagem (distância, tempo estimado e nível de risco de cada opção) e um alerta específico é gerado avisando que há uma alternativa mais segura.

Funciona para viagens de **qualquer distância** e **com ou sem paradas** no meio do caminho. Em viagens longas, o sistema não troca a viagem inteira por outra: ele procura um **desvio local** só em volta de cada trecho perigoso e mantém o resto do caminho original. A rota sugerida continua passando por todas as paradas combinadas.

**Limitações de hoje:**
- Se o trecho perigoso fica **colado na origem, no destino ou numa parada**, não tem como desviar dele — o veículo precisa passar por ali de qualquer jeito.
- Numa viagem com muitos trechos perigosos, o sistema tenta desviar dos **4 piores**; os demais continuam como na rota original.
- Hoje o sistema escolhe a alternativa com **menor risco**, sem limite de quanto tempo a mais ela leva. **Decidido em 2026-10-10 (a implementar):** só sugerir alternativa que custe no máximo +30% do tempo da viagem e no máximo +60 minutos; acima disso, nenhuma alternativa é sugerida.
- Quando um aviso oficial do governo cobre uma área muito grande, é comum que **nenhuma alternativa seja encontrada** — se toda a região ao redor está sob o mesmo aviso, qualquer caminho alternativo teria o mesmo risco mínimo, então o sistema corretamente não sugere trocar de rota "só para trocar".

## Início, fim e cancelamento de uma viagem

Toda viagem nasce como **planejada**. A partir daí:

- **Só o motorista (Viajante) inicia a viagem** — é ele quem sabe quando de fato saiu. A partir desse momento a viagem fica **em andamento** e o sistema registra o horário real de início. Ao iniciar, o risco do caminho é recalculado com o horário real de saída.
- **Um motorista tem no máximo uma viagem em andamento (ou pausada) por vez.**
- **O motorista pode pausar a viagem** (descanso/pernoite, refeição, abastecimento, carga/descarga, aguardando o clima melhorar ou outro motivo) e retomá-la depois. Durante a pausa o trajeto não é registrado, só o lugar onde ela começou. Se a pausa passar de 14 horas, o gerente é avisado uma vez. Detalhes em [App do Viajante](traveler-app.md#4-pausa-dormir-comer-abastecer).
- **Durante a viagem, o risco do caminho à frente é reavaliado** a cada 15 minutos e sempre que o governo publica um aviso novo que alcança o caminho. Se um trecho à frente passar a Alto ou Crítico, o motorista é avisado.
- **A localização em tempo real só é registrada com a viagem em andamento.** Antes de iniciar ou depois de encerrar, o aplicativo pode continuar aberto, mas a posição do motorista não é gravada — assim o histórico de trajeto reflete só o período real da viagem.
- **Concluir** a viagem pode ser feito pelo motorista ou pelo Gerente/Administrador (o Gerente pode encerrar caso o motorista esqueça).
- **Cancelar** é decisão da gestão: só Gerente ou Administrador cancelam, tanto antes de sair quanto no meio do caminho. O motorista não cancela sozinho uma viagem que foi planejada para ele.
- Uma viagem concluída ou cancelada **não pode ser reaberta** — se precisar refazer o trajeto, cadastra-se uma nova viagem. A exceção é o próprio celular ter concluído sozinho: aí o motorista tem 30 minutos para desfazer ("Ainda não cheguei").

Quem estiver acompanhando a viagem (motorista e Gerente) vê a mudança de situação na hora.

## O app do motorista durante a viagem

As regras do aplicativo do Viajante durante a viagem estão em [App do Viajante](traveler-app.md). Elas cobrem:

- início e conclusão automáticos;
- pausa para descanso ou refeição;
- reavaliação do clima a caminho;
- alertas por voz;
- o que fazer quando não há desvio;
- funcionamento sem internet.

A parte do sistema (servidor e tela web) já funciona: pausa, reavaliação do clima a caminho, alertas com "ciente" e aviso de pausa longa. Falta o app nativo do celular (segundo plano, voz com a tela bloqueada, início/conclusão automáticos, offline) — ver o que já está pronto em [App do Viajante](traveler-app.md#o-que-ja-esta-implementado).

## O que o sistema ainda não faz (mas está no plano)

- **Considerar neblina, deslizamento, vento forte e outros riscos climáticos** além de chuva e rio — hoje o sistema olha só para esses dois fatores; a expansão para outros tipos de risco está priorizada no roadmap (ver visão de produto).

## Modelo comercial

Por enquanto, todas as empresas cadastradas têm acesso completo ao sistema, sem diferenciação por plano pago — o cadastro de conta é feito pela própria empresa (sem necessidade de um vendedor configurar nada manualmente). Estrutura de planos pagos e limites de uso está fora do escopo da primeira versão do produto, mas o sistema já foi construído pensando em introduzir isso depois sem precisar refazer o modelo de dados.
