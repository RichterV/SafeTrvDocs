# Frontend (Ionic + Angular)

App único para web e mobile em `frontend/` no repositório **SafeTrv**: Ionic 9 + Angular 22 (componentes standalone, signals), Leaflet + tiles OpenStreetMap para o mapa, Capacitor configurado (`br.com.safetrv.app`) para gerar o app nativo depois. Criado em 2026-09-25 — esta é a primeira fatia: cobre tudo o que o backend já oferece.

## Telas

| Rota | Quem acessa | O que faz |
|---|---|---|
| `/login`, `/signup` | não logado | Entrar; autocadastro de empresa (cria a Conta e o usuário vira Administrador) |
| `/trips` | todos | Lista de viagens (staff vê a conta toda, com o nome do viajante; Viajante vê só as suas). Filtro "Planejadas e em andamento" / "Todas" |
| `/trips/new` | Admin, Gerente | Planejar viagem: viajante, veículo, tipo de carga, saída (até 7 dias), origem, paradas e destino com prévia no mapa |
| `/trips/:id` | todos (no escopo) | Mapa com os trechos coloridos pelo nível de risco, rota alternativa tracejada, botões de status conforme o papel, rastreamento ao vivo, resumo, alertas, trechos com risco médio ou maior |
| `/team` | Admin, Gerente | Listar e cadastrar usuários — só com papéis que o criador pode atribuir (mesma regra do backend) |
| `/vehicles` | Admin, Gerente | Listar e cadastrar veículos |

Menu lateral (fixo em tela larga, gaveta no celular) com Viagens, Equipe, Veículos e Sair — Equipe e Veículos só para staff.

## Tema — fonte única de cores e fontes

**Toda** cor, fonte e medida visual vive em `src/theme/variables.scss`: paleta Ionic (verde), superfícies, tipografia (Inter), cores dos níveis de risco (`--risk-baixo|medio|alto|critico`) e cores do mapa (`--map-*`). Componentes só referenciam `var(--...)` — nunca valores literais. O mapa (Leaflet desenha em SVG e precisa de valores concretos) lê essas variáveis em tempo de execução via `src/app/core/theme.ts`. Para trocar a identidade visual inteira, basta editar esse arquivo.

Modo escuro está desativado: as paletas escuras prontas do Ionic sobrescreveriam as cores do tema. Quando for implementar, a variante escura deve ser definida no mesmo `variables.scss`.

## Estrutura

```
frontend/src/
  theme/variables.scss        # tema central (ver acima)
  global.scss                 # estilos globais — só usam variáveis do tema
  environments/               # apiUrl / wsUrl (dev: localhost:8000)
  app/
    core/                     # modelos (espelho dos schemas do backend), AuthService,
                              # interceptor, guards, ApiService, LiveTrackingService,
                              # regras de transição de status, rótulos
    shared/                   # trip-map (Leaflet), place-search, risk-badge
    pages/                    # login, signup, trips, trip-new, trip-detail, team, vehicles
```

## Autenticação

- Tokens (access + refresh) no `localStorage`. O papel do usuário vem de `GET /users/me`, carregado pelo guard ao abrir o app.
- Interceptor HTTP anexa `Authorization: Bearer`. Num `401`, troca o refresh token por um novo access token **uma vez** e repete a requisição; requisições simultâneas compartilham a mesma troca. Se o refresh falhar, desloga.
- Guards: `authGuard` (logado), `guestGuard` (login/signup só deslogado), `staffGuard` (Admin/Gerente).

## Rastreamento ao vivo

`LiveTrackingService` abre o WebSocket da viagem ao entrar na tela de detalhe e fecha ao sair. Antes de cada conexão (inclusive reconexões), pede um ticket de uso único em `POST /trips/{id}/ws-ticket` e conecta com `?ticket=` — o access token nunca vai na URL. O pedido do ticket passa pelo interceptor HTTP, então um access token expirado é renovado automaticamente.

- **Reconexão automática** com espera crescente (1 s → 30 s), sem desistir — conectividade instável é esperada na estrada. Falha de rede ou `5xx` ao pedir o ticket também entra nesse ciclo.
- Só desiste se o pedido do ticket voltar `401` (sessão acabou, mesmo após tentar renovar), `403` ou `404` (sem permissão). Nesse caso a tela mostra o motivo, em vez de "Conectando…" para sempre.
- **Viajante**: ao iniciar a viagem, o compartilhamento de localização liga sozinho (há um botão para desligar). Usa `navigator.geolocation.watchPosition` e envia no máximo uma posição a cada 5 s. Uma posição que chega dentro do intervalo não é descartada: sai quando o intervalo vence — senão, com o veículo parado (GPS sem novos eventos), o Gerente ficaria vendo uma posição antiga.
- Erros de GPS transitórios (sem sinal, tempo esgotado — túnel, serra) só mostram um aviso e continuam observando. Só permissão negada desliga o compartilhamento.
- **Staff**: vê a posição no mapa e o horário da última atualização. Mudanças de status feitas por outra pessoa chegam pelo socket e recarregam a tela (com um aviso).

## Busca de lugares

`place-search` usa o **Nominatim** público (geocodificação do OpenStreetMap, restrita ao Brasil, `accept-language=pt-BR`), com debounce de 600 ms. Aceita também `lat, lng` digitado direto. A política de uso do Nominatim público é de ~1 requisição/s e sem uso pesado — suficiente para desenvolvimento, mas **para produção é preciso um geocodificador próprio ou pago**.

## Regras espelhadas do backend

Para decidir o que mostrar, o frontend replica algumas regras — quem valida de fato continua sendo o backend:

- Transições de status (`core/trip-status.ts`, testado em `trip-status.spec.ts`) ⇄ `app/services/trip_status.py`.
- Papéis que cada um pode criar (`pages/team.page.ts`) ⇄ `ROLES_CREATABLE_BY` em `app/api/v1/users.py`.
- Janela de 7 dias de planejamento ⇄ `settings.max_trip_planning_days`.

Se uma dessas regras mudar no backend, atualizar o espelho no frontend.

## Validação (2026-09-25)

Testado de ponta a ponta num Chromium real (Playwright), com backend, Postgres e APIs externas reais:

- cadastro de empresa → cadastro de viajante e veículo → viagem planejada (origem pelo Nominatim, destino por coordenadas) → mapa com a rota;
- viajante (tela de celular, geolocalização simulada) inicia a viagem → o admin vê o status mudar e a posição ao vivo, inclusive após o viajante se mover;
- admin conclui → o viajante vê "Concluída" em tempo real;
- visual do cenário de risco (trechos Alto/Crítico, rota alternativa, alertas), com a resposta da API alterada no teste, porque o clima do dia não tinha risco real.

Sem erros nem avisos no console.

Esse roteiro virou suíte permanente em `frontend/e2e/` (`npm run e2e`, 6 testes: fluxo completo, controle de acesso, visual de risco). Os testes unitários (`npx ng test`, 26) cobrem as partes mais delicadas: interceptor com refresh, reconexão do WebSocket (com um WebSocket falso), throttle de posições (`core/position-throttle.ts`), guards e regras de status. Como rodar: [Ambiente local → Testes e lint](../setup/local-dev.md#testes-e-lint).

## O que falta

- Alerta sonoro/visual em tempo real no app do Viajante (CLAUDE.md seção 6) — hoje os alertas aparecem na tela da viagem, mas não tocam som.
- Cache offline da rota/risco atual (CLAUDE.md seção 6).
- Reconhecimento de alerta ("vi o alerta") — o backend ainda não tem o endpoint.
- Editar/desativar usuários e veículos (o backend também ainda não tem esses endpoints).
- Build nativo Android/iOS via Capacitor (`npx cap add android`) e plugin nativo de geolocalização em segundo plano.
- Geocodificador de produção (ver "Busca de lugares").
