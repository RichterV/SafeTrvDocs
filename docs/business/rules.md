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
| 🔴 **Crítico** | Situação de risco elevado, incluindo os casos em que há um **aviso oficial do governo** (INMET) de tempestade severa cobrindo aquele trecho no horário da viagem — recomenda-se fortemente rever a rota ou o horário antes de sair. |

**Um detalhe importante:** se o governo (INMET) já emitiu um aviso oficial de tempestade severa cobrindo a área e o horário por onde o veículo vai passar, o sistema **nunca** vai mostrar aquele trecho como "Baixo" ou "Médio" — mesmo que os outros cálculos apontassem um risco menor, um aviso oficial ativo sempre eleva a classificação para pelo menos "Alto". A lógica é: se o governo já emitiu um alerta oficial, o sistema não vai contradizer isso.

**Outro detalhe:** viagens transportando **carga perigosa** (combustíveis, químicos) recebem uma nota de risco mais alta nos trechos onde já existe risco de alagamento — o mesmo nível de chuva/proximidade de rio pesa mais quando a carga em si já é um risco ambiental caso algo dê errado. Viagens à noite também recebem um pequeno acréscimo no risco, porque chuva combinada com pouca visibilidade noturna aumenta a chance de acidente.

## Sugestão automática de rota alternativa

Quando a viagem calculada dá risco Alto ou Crítico, o sistema busca automaticamente, no mesmo instante em que a viagem é criada, se existe um caminho alternativo entre a origem e o destino com risco menor. Se encontrar um caminho melhor, ele fica disponível junto com a viagem (distância, tempo estimado e nível de risco de cada opção) e um alerta específico é gerado avisando que há uma alternativa mais segura.

**Duas limitações importantes de hoje:**
- Só funciona para viagens **sem paradas no meio do caminho** (só origem → destino direto).
- Só funciona para trajetos de **até ~100 km** — é um limite técnico do provedor de rotas usado pelo sistema. Como viagens de transporte de carga costumam ser mais longas que isso (ver perfis de cliente), na prática essa sugestão ainda não aparece na maioria das viagens de longa distância — é uma limitação conhecida, já mapeada para ser resolvida numa próxima etapa.
- Quando um aviso oficial do governo cobre uma área muito grande, é comum que **nenhuma alternativa seja encontrada** — se toda a região ao redor está sob o mesmo aviso, qualquer caminho alternativo teria o mesmo risco mínimo, então o sistema corretamente não sugere trocar de rota "só para trocar".

## O que o sistema ainda não faz (mas está no plano)

- **Marcar quando uma viagem realmente começou** — hoje o sistema registra a localização do motorista assim que o aplicativo conecta, mas não existe ainda um botão de "iniciar viagem" que mude o status oficial dela.
- **Confirmar que o motorista viu um alerta** — o sistema já guarda essa informação na estrutura de dados, mas ainda não existe a tela/ação para o motorista confirmar "vi o alerta". Isso é importante para a empresa comprovar, se precisar, que o motorista foi avisado do risco antes de seguir viagem.
- **Considerar neblina, deslizamento, vento forte e outros riscos climáticos** além de chuva e rio — hoje o sistema olha só para esses dois fatores; a expansão para outros tipos de risco está priorizada no roadmap (ver visão de produto).

## Modelo comercial

Por enquanto, todas as empresas cadastradas têm acesso completo ao sistema, sem diferenciação por plano pago — o cadastro de conta é feito pela própria empresa (sem necessidade de um vendedor configurar nada manualmente). Estrutura de planos pagos e limites de uso está fora do escopo da primeira versão do produto, mas o sistema já foi construído pensando em introduzir isso depois sem precisar refazer o modelo de dados.
