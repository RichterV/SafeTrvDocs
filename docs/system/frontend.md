# Frontend (Ionic + Angular)

App único para web e mobile em `frontend/` no repositório **SafeTrv**: Ionic 9 + Angular 22 (componentes standalone, signals), Leaflet + tiles OpenStreetMap para o mapa, Capacitor configurado (`br.com.safetrv.app`) para gerar o app nativo depois. Criado em 2026-09-25; usabilidade e visual reformados em 2026-10-10, com o layout dividido por papel.

## Quem usa e em qual aparelho

O layout se divide por papel. Cada visão funciona no outro tamanho de tela, mas é otimizada para o seu aparelho principal.

- **Viajante → celular.** Usa a tela dirigindo ou parado na estrada. Mapa grande, alerta de risco impossível de ignorar, botões grandes (alvo de toque de 56 px; as ações da viagem, 64 px), pausar em 2 toques, texto legível sob sol forte.
- **Gerente/Admin → computador.** Acompanha várias viagens ao mesmo tempo. Informação densa, lista e mapa lado a lado, triagem por risco.

## Telas

| Rota | Quem acessa | O que faz |
|---|---|---|
| `/login`, `/signup` | não logado | Entrar; autocadastro de empresa (cria a Conta e o usuário vira Administrador). No desktop, painel da marca à esquerda e formulário à direita. |
| `/` | logado | Redireciona por papel: Admin/Gerente → `/central`; Viajante → `/viagem`. |
| `/central` | Admin, Gerente | **Central de Risco** (tela inicial do staff). Ver abaixo. |
| `/viagem` | todos | "Viagem" do Viajante: abre direto a viagem em andamento ou pausada; senão, a próxima planejada (inclusive com até 12 h de atraso na saída); sem nenhuma, um estado vazio. |
| `/trips` | todos | Staff: tabela com filtro de status (ativas e planejadas, concluídas, canceladas, todas), busca por cidade, viajante ou placa, e ordenação por pior risco ou por saída. Viajante: cartões grandes, com a viagem atual destacada no topo. |
| `/trips/new` | Admin, Gerente | Planejar viagem. Ordem: trajeto (origem, paradas, destino), quando (atalhos "Daqui a 1 h", "Amanhã 05:00", "Amanhã 08:00" + seletor de data/hora em 24 h, até 7 dias), quem e com o quê (viajante, veículo, carga). Mapa fixo à direita no desktop. O botão nunca fica desabilitado: ao tocar, mostra o que falta. |
| `/trips/:id` | todos (no escopo) | Tela da viagem. Ver abaixo. |
| `/team` | Admin, Gerente | Lista primeiro, com busca e filtro por papel; usuário inativo marcado. "Novo usuário" abre um painel ao lado (desktop) ou no lugar da lista (celular), só com os papéis que o criador pode atribuir. |
| `/vehicles` | Admin, Gerente | Mesmo padrão: lista com busca; "Novo veículo" com atalhos para os tipos comuns. |
| `/conta` | todos | "Conta" (Viajante) / "Mais" (staff no celular): quem está logado, atalhos (Equipe, Veículos, Minha viagem) e Sair. |

### Tela da viagem (`/trips/:id`)

- **Celular:** responde a três perguntas, nesta ordem: tem perigo à frente? (banner do alerta pendente); onde estou e quando chego ao próximo risco? (mapa + "próximo risco"); o que eu faço? (um botão grande).
  - **Banner de alerta** fixo no topo, na cor e com a forma do nível, até o "Ciente" (botão de 56 px). Nunca some sozinho. Com mais de um pendente, mostra "mais N alertas depois deste".
  - **Faixa ao vivo:** "Enviando sua posição" + velocidade; aviso amarelo se o envio estiver desligado ou sem sinal de GPS; faixa escura "Pausada — motivo desde HH:MM".
  - **Próximo risco:** selo + horário grande + "em X min (trecho N)", e a faixa de risco do trajeto (um bloco por trecho, o que já passou esmaecido). Tocar num bloco mostra chuva, rio e horário daquele trecho e leva o mapa até ele.
  - **Barra de ações** no rodapé: Iniciar / Pausar / Retomar. "Concluir viagem", "Ver detalhes" e "Parar/voltar a enviar minha posição" ficam no "⋯". Concluir e cancelar sempre pedem confirmação.
  - **Pausar em 2 toques:** "Pausar" abre uma folha com os 6 motivos em botões de 76 px; "Outro" pede um texto curto.
  - **Detalhes** (resumo, rota alternativa, alertas com a situação de cada um, trechos de risco agrupados, pausas) numa folha que sobe.
- **Desktop:** mapa à esquerda e painel de 420 px à direita, com tudo acima visível sem abrir folhas. Staff vê Concluir e Cancelar como botões.
- **Trechos de risco agrupados:** trechos seguidos com o mesmo nível (Médio ou maior) viram um item só ("Trechos 9 a 12 · 19:33–20:09"), em vez de uma lista de trechos quase iguais.

### Central de Risco (`/central`)

Responde "onde eu preciso olhar agora". Dados de `GET /trips/overview` e `GET /alerts?pending=true` ([Referência da API](api-reference.md)), atualizados a cada 30 s com a tela aberta.

- **Números do topo:** em andamento, pausadas, saem hoje, alertas sem ciente. Período: "Agora e próximas 24 h" ou "Próximos 7 dias".
- **Lista de triagem** em dois grupos ("Na estrada", "Saem…"), na ordem do servidor (pior risco primeiro). Cada linha: tarja na cor do risco, trajeto, viajante, placa, velocidade ou "parado (motivo) desde HH:MM", "Risco alto às HH:MM (em X min)", chegada prevista ou saída, alertas sem ciente.
- **Mapa da frota** com a última posição de cada viagem na estrada (quadrado escuro = pausada). Clicar numa linha (ou num veículo) destaca o veículo e desenha a rota daquela viagem; clicar de novo, ou em "Abrir a viagem", abre o detalhe.
- **Alertas sem ciente** de toda a conta; os só para o gerente (pausa longa) têm o botão "Ciente" ali mesmo.
- Desktop largo (≥ 1280 px): 3 colunas (lista 380 px | mapa | alertas 340 px). Tablet/desktop estreito: lista e mapa; os alertas viram uma aba da lista. Celular: abas Viagens · Mapa · Alertas e botão flutuante "Nova viagem".

## Navegação e grid

| | Viajante | Gerente / Admin |
|---|---|---|
| Celular (< 768 px) | Barra inferior: Viagem · Minhas viagens · Conta | Barra inferior: Central · Viagens · Equipe · Mais |
| Tablet (768–1023 px) | Barra inferior | Barra lateral só com ícones (76 px) |
| Desktop (≥ 1024 px) | Barra lateral (216 px) | Barra lateral (216 px) com "Nova viagem" em destaque |

Não há menu "☰". A navegação fica em `shared/app-nav.component.ts` (`variant="rail"` ou `"bar"`); a casca (`app.component`) usa o `ion-split-pane` com o breakpoint do papel e mostra a barra inferior quando a lateral está escondida. Os breakpoints viram signals em `shared/layout.service.ts` (`tablet` 768, `desktop` 1024, `wide` 1280) para o que o CSS sozinho não resolve. Gerente que também é viajante ganha "Minha viagem" na barra lateral.

## Identidade visual e tema

**Sinalização de estrada.** Azul-petróleo sóbrio para a marca; o risco é a única coisa colorida na tela. O verde da primeira versão saiu porque se confundia com "risco baixo" e com "em andamento".

**Toda** cor, fonte e medida visual vive em `src/theme/variables.scss`. Componentes só referenciam `var(--...)`, nunca valores literais. O mapa (Leaflet precisa de valores concretos) lê as cores **e as espessuras de linha** dessas variáveis em tempo de execução via `src/app/core/theme.ts` (`themeColor`, `riskColor`, `themeNumber`).

- **Tipografia:** Barlow Semi Condensed (títulos, números, botões; desenhada a partir de placas de rodovia, cabe nomes longos de cidade numa linha de celular) e Atkinson Hyperlegible (texto e formulários; criada para baixa visão). Botões e abas sem caixa alta.
- **Cores da interface:** marca `#12384f` (12,3:1 com branco), ação/ao vivo `#1d6fb8` (5,2:1), texto `#0f1b26`, texto secundário `#5a6573` (5,4:1 sobre o fundo), fundo `#f3f5f7`, erro `#b3261e`.
- **Níveis de risco — cor + forma + texto, sempre juntos:**

| Nível | Cor | Forma | Texto do selo | No mapa |
|---|---|---|---|---|
| Baixo | `#167a7a` | círculo | branco, 5,1:1 | linha 5 px |
| Médio | `#f2c230` | triângulo | quase-preto, 11:1 | linha 6 px |
| Alto | `#d9661a` | losango (placa de advertência) | quase-preto, 5,4:1 | linha 7 px + marcador ◆ |
| Crítico | `#a3123a` | octógono com barra vazada (placa de proibição) | branco, 7,8:1 | linha 8 px + marcador |

Simulando daltonismo (Machado 2009, severidade total), a menor diferença de cor entre dois níveis fica em ΔE ≥ 19,5 em deuteranopia, protanopia e tritanopia (antes: 13,1 entre Médio e Alto, em deuteranopia). A rota ganha um contorno escuro para se destacar sobre qualquer fundo do OpenStreetMap. As formas ficam em `shared/risk-shape.ts`.

- **Status da viagem:** pílula neutra com ícone (planejada ◷, em andamento ▶ em azul-sinal, pausada ❚❚, concluída ✓, cancelada ✕); não usa as cores de risco.

Modo escuro está desativado: as paletas escuras prontas do Ionic sobrescreveriam as cores do tema. Um "modo noite" para o viajante faz sentido no futuro; a variante escura deve ser definida no mesmo `variables.scss`.

## Componentes base (`src/app/shared/`)

| Componente | O que é |
|---|---|
| `risk-badge` | Selo de risco (cor + forma + texto), tamanhos `sm`/`lg`. Leitor de tela: "Risco alto". |
| `status-pill` | Status da viagem, neutro, com ícone. |
| `risk-strip` | Faixa do trajeto, um bloco por trecho. |
| `alert-banner` | Alerta pendente, fixo até o "Ciente". |
| `state` | Estado vazio ou de erro (ícone, título, explicação, ação). Carregamento usa blocos `.skeleton` no formato do conteúdo. |
| `trip-map` | Mapa Leaflet: rota por nível de risco, alternativa tracejada, pontos, posição ao vivo, frota (Central). |
| `place-search` | Busca de lugar (Nominatim). |
| `app-nav`, `auth-brand` | Navegação por papel; painel da marca no login/cadastro. |

Formulários: rótulo sempre visível acima do campo, campo de 48 px com borda, erro em texto logo abaixo do campo. Estilos comuns em `src/global.scss` (`.form-field`, `.app-input`, `.chips`, `.registry-*`, `.skeleton`).

## Estrutura

```
frontend/src/
  theme/variables.scss        # tema central (ver acima)
  global.scss                 # estilos globais — só usam variáveis do tema
  environments/               # apiUrl / wsUrl (dev: localhost:8000)
  app/
    core/                     # modelos (espelho dos schemas do backend), AuthService,
                              # interceptor, guards, ApiService, LiveTrackingService,
                              # regras de transição de status, rótulos,
                              # format.ts (durações, horários, próximo risco, trechos agrupados)
    shared/                   # componentes base (ver acima), layout.service, icons
    pages/                    # home, central, current-trip, account, login, signup,
                              # trips, trip-new, trip-detail, team, vehicles
```

## Autenticação

- Tokens (access + refresh) no `localStorage`. O papel do usuário vem de `GET /users/me`, carregado pelo guard ao abrir o app.
- Interceptor HTTP anexa `Authorization: Bearer`. Num `401`, troca o refresh token por um novo access token **uma vez** e repete a requisição; requisições simultâneas compartilham a mesma troca. Se o refresh falhar, desloga.
- Guards: `authGuard` (logado), `guestGuard` (login/signup só deslogado), `staffGuard` (Admin/Gerente). Depois do login, a tela inicial é escolhida pelo papel (`/` → `/central` ou `/viagem`).

## Rastreamento ao vivo

`LiveTrackingService` abre o WebSocket da viagem ao entrar na tela de detalhe e fecha ao sair. Antes de cada conexão (inclusive reconexões), pede um ticket de uso único em `POST /trips/{id}/ws-ticket` e conecta com `?ticket=` — o access token nunca vai na URL. O pedido do ticket passa pelo interceptor HTTP, então um access token expirado é renovado automaticamente.

- **Reconexão automática** com espera crescente (1 s → 30 s), sem desistir — conectividade instável é esperada na estrada. Falha de rede ou `5xx` ao pedir o ticket também entra nesse ciclo.
- Só desiste se o pedido do ticket voltar `401` (sessão acabou, mesmo após tentar renovar), `403` ou `404` (sem permissão). Nesse caso a tela mostra o motivo, em vez de "Conectando…" para sempre.
- **Viajante**: o envio da posição liga sozinho ao iniciar ou retomar a viagem **e ao abrir uma viagem em andamento** (desde 2026-10-10 — antes, recarregar a página no meio da viagem parava o envio sem o gerente perceber). O motorista pode desligar no "⋯"; nesse caso não religa sozinho ao recarregar a viagem, e a tela mostra um aviso amarelo com o botão "Ligar". Usa `navigator.geolocation.watchPosition` e envia no máximo uma posição a cada 5 s. Uma posição que chega dentro do intervalo não é descartada: sai quando o intervalo vence — senão, com o veículo parado (GPS sem novos eventos), o Gerente ficaria vendo uma posição antiga.
- Erros de GPS transitórios (sem sinal, tempo esgotado — túnel, serra) só mostram um aviso e continuam observando. Só permissão negada desliga o compartilhamento.
- **Staff**: vê a posição no mapa, a velocidade e o horário da última atualização. Mudanças de status feitas por outra pessoa chegam pelo socket e recarregam a tela (com um aviso).

## Busca de lugares

`place-search` usa o **Nominatim** público (geocodificação do OpenStreetMap, restrita ao Brasil, `accept-language=pt-BR`), com debounce de 600 ms. Aceita também `lat, lng` digitado direto. A política de uso do Nominatim público é de ~1 requisição/s e sem uso pesado — suficiente para desenvolvimento, mas **para produção é preciso um geocodificador próprio ou pago**.

## Regras espelhadas do backend

Para decidir o que mostrar, o frontend replica algumas regras — quem valida de fato continua sendo o backend:

- Transições de status (`core/trip-status.ts`, testado em `trip-status.spec.ts`) ⇄ `app/services/trip_status.py`.
- Papéis que cada um pode criar (`pages/team.page.ts`) ⇄ `ROLES_CREATABLE_BY` em `app/api/v1/users.py`.
- Janela de 7 dias de planejamento ⇄ `settings.max_trip_planning_days`.
- Próximo trecho de risco e chegada prevista na tela da viagem (`core/format.ts`) ⇄ `app/services/fleet_overview.py` (a Central usa os valores do servidor).

Se uma dessas regras mudar no backend, atualizar o espelho no frontend.

## App do Viajante na tela web

Enquanto o app nativo (Capacitor) não existe, a tela da viagem já cobre a parte do app do Viajante que não depende de segundo plano:

- **Pausar/retomar** (RN-20 a RN-23): "Pausar" abre a folha de motivos; um toque no motivo pausa (em "Outro", um texto curto). Manda a última posição conhecida como local da pausa. Pausar desliga o envio da posição; retomar liga de novo. O gerente vê "Parado — motivo desde HH:MM" ao vivo, e o histórico de pausas.
- **Alertas ao vivo** (mensagem `alert`): para o Viajante, o alerta novo vira o banner fixo no topo e é **lido em voz alta** (Web Speech API, pt-BR), com a leitura registrada (`read-aloud`). O "Ciente" fica pendente até ele tocar no botão. Para o gerente, um aviso rápido. Cada alerta mostra a situação (enviado, lido em voz alta, ciente, decisão).
- **Reavaliação** (mensagem `risk`): recarrega a viagem, com trechos e horários novos.
- Os botões de desfazer início/conclusão automáticos **não** aparecem na web — só fazem sentido no app nativo, que inicia e conclui sozinho.

## App Android (Capacitor)

APK 1 (2026-10-11): o mesmo app, empacotado com Capacitor 8 (`frontend/android/`), apontando para o servidor (`environment.mobile.ts`, build `npx ng build --configuration mobile`; ver [Publicação no servidor](../setup/server.md)). O que muda em relação ao navegador fica em `src/app/native/`:

- **GPS em segundo plano** (`native/location.ts`, plugin `@capacitor-community/background-geolocation`): ao iniciar/abrir a viagem em andamento, o Android mostra a notificação fixa "Viagem em andamento" e o GPS continua com a tela bloqueada ou com o Waze/Google Maps na frente. Permissão negada abre as configurações do app.
- **Envio por HTTP nativo** (`CapacitorHttp` ligado em `capacitor.config.ts`): o Android corta a rede da WebView em segundo plano, então o app manda as posições por `POST /trips/{id}/positions`, não pelo WebSocket. O que não sai (sem sinal) fica numa fila em memória (`core/position-queue.ts`, até 2.000 posições, lotes de 100) e vai no próximo envio — primeira parte da RN-52.
- **Alertas com o celular bloqueado** (`native/notifications.ts`, `@capacitor/local-notifications`): cada envio de posição devolve os alertas sem "Ciente"; um alerta novo com o app em segundo plano vira **notificação com som e vibração** (canal "Alertas de risco da viagem", importância máxima).
- **Voz** (`native/speech.ts`, `@capacitor-community/text-to-speech`): a WebView do Android não tem Web Speech; o alerta é lido pelo motor de voz do Android, e a leitura é registrada (`read-aloud`).
- Ícone do app: losango de advertência sobre o azul-petróleo da marca; ícone da notificação: losango branco. Cores da marca repetidas em `capacitor.config.ts` e nos recursos do Android (o Android não lê o CSS do tema).

APK 2 (a fazer): botão "Navegar" (Waze/Google Maps), início/retomada/conclusão automáticos com o "desfazer", voz garantida com a tela bloqueada, lembrete perto do trecho.

## Testes

- **Unitários** (`npx ng test`, Vitest, 50): fila de posições do app nativo (`core/position-queue.spec.ts`), interceptor com refresh, reconexão do WebSocket (com um WebSocket falso), throttle de posições, guards, regras de status, formatação e cálculos de tela (`core/format.spec.ts`: duração, horário, próximo risco, trechos agrupados, ordem dos alertas) e a escolha da "viagem atual" do viajante (`pages/current-trip.page.spec.ts`).
- **E2E** (`npm run e2e`, Playwright, 6 testes) num Chromium real, com backend, Postgres e APIs externas reais:
  - fluxo completo com dois navegadores: cadastro → Central vazia → equipe e veículo pelo painel de cadastro (inclusive o erro ao enviar vazio) → viagem planejada → o viajante (celular, GPS simulado) cai direto na viagem e inicia → o admin vê status e posição ao vivo → **o viajante recarrega a página e o envio de posição religa sozinho** → pausa em 2 toques e retomada → admin conclui (com confirmação) → o viajante vê "Concluída";
  - controle de acesso: rotas protegidas, navegação do viajante sem Central/Equipe/Veículos, tela inicial por papel, sessão após recarregar, logout;
  - visual do cenário de risco, com a resposta da API alterada (trecho 1 Alto, trecho 2 Crítico, **os demais forçados para Baixo** — desde 2026-10-10 o teste não depende mais do clima do dia): cores `--risk-*` dos trechos, marcadores com a forma da placa, faixa de risco, banners de alerta, rota alternativa tracejada que some com o botão.
- Conferido por mutação (2026-10-10): desligar a religação automática do envio de posição quebra o passo certo do e2e.

Como rodar: [Ambiente local → Testes e lint](../setup/local-dev.md#testes-e-lint).

## O que falta

- Cache offline da rota/risco atual (CLAUDE.md seção 6).
- Editar/desativar usuários e veículos (o backend também ainda não tem esses endpoints).
- Build nativo Android/iOS via Capacitor (`npx cap add android`) e plugin nativo de geolocalização em segundo plano.
- Botão "Navegar" abrindo Waze/Google Maps (RN-01), junto com o app nativo.
- Atualização ao vivo da Central de Risco por WebSocket da conta (hoje ela recarrega a cada 30 s).
- Geocodificador de produção (ver "Busca de lugares").
