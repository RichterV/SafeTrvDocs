# Referência da API

Todas as rotas REST ficam sob o prefixo `/api/v1`. A API também expõe documentação interativa automática (Swagger UI) em `/docs` e `/redoc` quando o servidor está rodando — esta página é o resumo curado, com o porquê de cada regra.

Autenticação: header `Authorization: Bearer <access_token>` em toda rota autenticada (todas, exceto `/auth/signup` e `/auth/login`).

## Auth (`/auth`)

### `POST /auth/signup`

Autocadastro: cria uma nova **Conta** e o usuário vira **Administrador** dela (CLAUDE.md seção 2 — cadastro de conta é self-service).

| Campo | Tipo | Regra |
|---|---|---|
| `account_name` | string | 2–200 caracteres |
| `name` | string | 2–200 caracteres |
| `email` | string | e-mail válido; deve ser único no sistema |
| `password` | string | 8–128 caracteres |

Retorna `201` com `{access_token, refresh_token, token_type}`. `409 Conflict` se o e-mail já existe.

### `POST /auth/login`

`{email, password}` → `200` com `{access_token, refresh_token, token_type}`. `401` se e-mail/senha inválidos ou usuário inativo.

### `POST /auth/refresh`

`{refresh_token}` → `200` com `{access_token, token_type}`. `401` se o refresh token for inválido, do tipo errado, ou o usuário não existir/estiver inativo.

## Usuários (`/users`)

Regra de hierarquia (CLAUDE.md seção 3): um usuário só cria outro com papel igual ou inferior ao seu. Admin cria Gerente/Viajante; Gerente cria só Viajante; Viajante não cria ninguém.

### `POST /users` — Admin, Gerente

`{name, email, password, phone?, roles: ["admin"|"gerente"|"viajante", ...]}` → `201` com o usuário criado.

- `403` se algum papel pedido não estiver entre os que o criador pode atribuir (a resposta lista quais papéis são permitidos).
- `409` se o e-mail já existe.
- O usuário criado sempre pertence à mesma `account_id` de quem está criando — não é possível criar um usuário em outra conta.

### `GET /users/me` — qualquer papel

Retorna o usuário autenticado (mesmo formato do item de `GET /users`, com `roles`). Usado pelo frontend para saber os papéis de quem está logado — um Viajante não pode chamar `GET /users`.

### `GET /users` — Admin, Gerente

Lista todos os usuários da própria conta.

## Veículos (`/vehicles`)

### `POST /vehicles` — Admin, Gerente

`{plate, type}` → `201`. Escopado à conta do usuário autenticado.

### `GET /vehicles`

Lista os veículos da própria conta. Qualquer papel autenticado pode listar.

## Viagens (`/trips`)

### `POST /trips` — Admin, Gerente

Cria uma viagem e dispara todo o pipeline de cálculo de risco (ver [Visão geral da arquitetura](overview.md#fluxo-de-uma-requisicao-post-apiv1trips)).

```json
{
  "traveler_id": "uuid",
  "vehicle_id": "uuid",
  "cargo_type": "normal | perigosa | refrigerada",
  "origin": { "label": "string", "lat": -23.55, "lng": -46.63 },
  "destination": { "label": "string", "lat": -22.9, "lng": -47.06 },
  "waypoints": [{ "label": "string", "lat": 0, "lng": 0 }],
  "scheduled_departure_at": "2026-09-20T10:00:00Z"
}
```

Validações antes de calcular a rota:

- `traveler_id` precisa existir na mesma conta e ter o papel `viajante` (`404`/`422`).
- `vehicle_id` precisa existir na mesma conta (`404`).
- `scheduled_departure_at` precisa estar entre agora e `max_trip_planning_days` dias à frente (hoje 7 — CLAUDE.md seção 4) — `422` fora dessa janela.

Erros do pipeline:

- `502 Bad Gateway` se o cálculo de rota (OpenRouteService) falhar — sem rota não há como segmentar nem avaliar risco.
- Falha do INMET **não** derruba a criação da viagem — degrada graciosamente (loga warning, segue sem o piso mínimo de risco oficial).
- Falha ao buscar rota alternativa (provedor indisponível, limites do ORS) **não** derruba a criação da viagem — degrada graciosamente, sem `alternative_route` na resposta.

Retorna `201` com o objeto `Trip` completo: dados da viagem, lista de `segments` (cada um com seus `risk_assessments`), `risk_summary` (score máximo, médio, nível), `alternative_route` (ver abaixo, `null` se nenhuma alternativa foi sugerida) e `alerts` gerados.

**`alternative_route`** (presente só quando: risco Alto/Crítico e uma alternativa com risco estritamente menor foi encontrada — por desvio local em volta dos trechos de risco, em viagens de qualquer distância e com ou sem paradas, ou pelas rotas alternativas do provedor em viagens curtas sem paradas — ver [Serviços internos](services.md#sugestao-automatica-de-rota-alternativa)):

```json
{
  "distance_km": 52.3,
  "duration_estimated_minutes": 48,
  "geometry": [[-24.9558, -53.4552]],
  "segments_summary": [
    {"sequence": 1, "start_lat": 0, "start_lng": 0, "end_lat": 0, "end_lng": 0,
     "distance_km": 6.2, "estimated_arrival_at": "2026-09-19T03:12:00+00:00",
     "score": 32.0, "risk_level": "medio"}
  ],
  "score_max": 32.0,
  "score_avg": 28.5,
  "risk_level": "medio",
  "calculated_at": "2026-09-19T01:26:46Z"
}
```

### `GET /trips`

Lista viagens da própria conta, ordenadas por `scheduled_departure_at` decrescente. Cada item inclui um resumo (sem a lista completa de segmentos — ver `GET /trips/{id}` para o detalhe).

- Admin/Gerente: veem todas as viagens da conta; podem filtrar por `?traveler_id=<uuid>`.
- Viajante: vê **somente** as próprias viagens (o filtro `traveler_id` é ignorado nesse caso — o escopo já é implícito).

### `GET /trips/overview` — Admin, Gerente

Dados da **Central de Risco** (tela inicial do gerente no frontend), numa chamada só. Devolve as viagens da conta que estão **em andamento**, **pausadas** ou **planejadas com saída nas próximas `hours_ahead` horas** (padrão 24, máximo 168 = 7 dias). Planejadas com a saída já passada há até 12 h (o motorista ainda não iniciou) continuam na lista.

Já vem **ordenada para triagem**: pior nível de risco primeiro; no empate, as viagens na estrada antes das planejadas; depois, quem chega ao próximo trecho de risco (ou sai) mais cedo.

Cada item tem os campos de `GET /trips`, mais:

| Campo | O que é |
|---|---|
| `traveler_name`, `vehicle_plate` | nome do viajante e placa, para não precisar cruzar com `/users` e `/vehicles` |
| `started_at`, `distance_km`, `duration_estimated_minutes`, `risk_reevaluated_at` | como em `GET /trips/{id}` |
| `estimated_arrival_at` | chegada prevista: passagem pelo último trecho + a duração de um trecho. Acompanha o atraso real, porque a reavaliação durante a viagem desloca os horários de passagem |
| `last_position` | última posição recebida desde o início da viagem (`lat`, `lng`, `speed_kmh`, `heading_degrees`, `recorded_at`), ou `null` |
| `open_pause` | pausa em curso (mesmo formato de `pauses[]`), ou `null` |
| `pending_alerts` | quantos alertas da viagem ainda não têm "Ciente" (inclui os só para o gerente) |
| `next_risk` | primeiro trecho **Alto/Crítico** que o motorista ainda vai percorrer (`segment_sequence`, `risk_level`, `score`, `estimated_arrival_at`), ou `null`. Na viagem em andamento/pausada, ignora os trechos que já terminaram; na planejada, vale o trajeto todo |

Erros: `403` para quem não é Admin/Gerente; `422` para `hours_ahead` fora de 1–168. Cálculos em `app/services/fleet_overview.py`.

### `GET /trips/{trip_id}`

Retorna a viagem completa (mesmo formato do `POST`). `404` se não existir na conta do usuário, **ou** se existir mas o usuário for um Viajante que não é o dono dela — a resposta é o mesmo 404 nos dois casos, para não vazar a existência de viagens de outros usuários.

### `GET /trips/{trip_id}` — campos do app do Viajante

Além do planejamento, a resposta traz:

- `pauses` — histórico de pausas (`reason`, `note`, `lat`/`lng` do início, `started_at`, `ended_at` — nulo na pausa em curso, `long_pause_notified_at`);
- `started_automatically`, `completed_automatically` — se o início/conclusão foi feito sozinho pelo celular (só esses podem ser desfeitos);
- `risk_reevaluated_at` — última reavaliação do risco durante a viagem;
- em cada trecho, `risk_assessments` vem **da mais recente para a mais antiga**: `risk_assessments[0]` é a avaliação vigente. Durante a viagem o risco é recalculado e cada recálculo acrescenta uma avaliação (o histórico fica);
- em cada alerta, o ciclo de vida: `audience` (`todos` | `staff`), `segment_sequence`, `read_aloud_at`, `acknowledged`/`acknowledged_at`, `decision` (`usar_alternativa` | `seguir_rota` | `parar`) e `decided_at`.

Alertas com `audience: "staff"` (hoje, só o de pausa longa) **não aparecem** para o Viajante.

### `PATCH /trips/{trip_id}/status`

Muda o status da viagem. Retorna a viagem completa (mesmo formato do `GET /trips/{id}`).

Corpo:

```json
{
  "status": "pausada",
  "automatic": false,
  "pause_reason": "descanso",
  "pause_note": null,
  "lat": -23.2,
  "lng": -46.9
}
```

- `status` — obrigatório: `planejada`, `em_andamento`, `pausada`, `concluida` ou `cancelada`.
- `automatic` — `true` quando o celular fez a mudança sozinho (início ou conclusão automáticos, RN-11/RN-60). Só essas podem ser desfeitas depois.
- `pause_reason` — obrigatório ao pausar: `descanso`, `refeicao`, `abastecimento`, `carga_descarga`, `aguardando_clima` ou `outro`. Com `outro`, `pause_note` (até 120 caracteres) é obrigatório.
- `lat`/`lng` — opcionais, sempre juntos: onde a pausa começou (durante a pausa o rastro de posições para).

Transições permitidas (regras em `app/services/trip_status.py`, efeitos em `app/services/trip_lifecycle.py`):

| De → Para | Quem pode | Observação |
|---|---|---|
| `planejada` → `em_andamento` | somente o Viajante da viagem | preenche `started_at` |
| `em_andamento` → `pausada` | somente o Viajante | abre uma pausa |
| `pausada` → `em_andamento` | somente o Viajante | fecha a pausa; não mexe em `started_at` |
| `em_andamento` ou `pausada` → `concluida` | o Viajante, ou Admin/Gerente | preenche `ended_at` |
| `planejada`, `em_andamento` ou `pausada` → `cancelada` | Admin/Gerente | preenche `ended_at` |
| `em_andamento` → `planejada` | somente o Viajante | desfaz um início **automático**, até 5 min depois (RN-13) |
| `concluida` → `em_andamento` | somente o Viajante | desfaz uma conclusão **automática**, até 30 min depois (RN-61) |

`cancelada` é final; `concluida` também, fora o "desfazer" acima. Um usuário com papéis acumulados pode executar as transições de ambos os papéis.

**Uma viagem ativa por motorista (RN-14):** passar para `em_andamento` (exceto ao retomar uma pausa) é recusado com `409` se o mesmo Viajante já tiver outra viagem `em_andamento` ou `pausada`.

**Recálculo do risco:** ao iniciar (RN-15), e ao retomar depois de mais de 30 min de pausa (RN-24), o risco de todo o restante da viagem é recalculado com os horários reais — ver [Serviços internos](services.md#acompanhamento-durante-a-viagem-trip_monitorpy). Se a previsão do tempo estiver fora, a mudança de status vale mesmo assim, sem o recálculo.

Erros:
- `404` — viagem não existe na conta, ou o usuário é um Viajante que não é o dono dela (mesma regra do `GET`).
- `409` — transição inexistente, "desfazer" fora do prazo ou de uma mudança que não foi automática, ou outra viagem ativa do mesmo motorista.
- `403` — a transição existe, mas o papel do usuário não permite (ex: Gerente pausando pelo motorista, Viajante cancelando).
- `422` — corpo inválido (ex: pausa sem motivo, `outro` sem texto, só `lat` sem `lng`).

A mudança é retransmitida em tempo real para quem estiver conectado no WebSocket da viagem (mensagem `{"type": "status", ...}` — ver [WebSocket](websocket.md#mudanca-de-status)).

### Ciclo de vida do alerta — `POST /trips/{trip_id}/alerts/{alert_id}/...`

Registram o que aconteceu com um alerta (RN-44/RN-45). Todos respondem com o alerta atualizado e o retransmitem pelo WebSocket da viagem (`{"type": "alert", ...}`), para o painel do gerente ver na hora.

| Endpoint | Quem | O que faz |
|---|---|---|
| `.../read-aloud` | o Viajante | marca `read_aloud_at` (o app leu o alerta em voz alta). Idempotente. |
| `.../acknowledge` | o Viajante; nos alertas `audience: "staff"`, Admin/Gerente | "Ciente": `acknowledged = true`, `acknowledged_at`. Idempotente. |
| `.../decision` | o Viajante | corpo `{"decision": "usar_alternativa" \| "seguir_rota" \| "parar"}`; também conta como "ciente". |

Regras da decisão: só para alertas de `chuva`, `rio`, `oficial` e `rota_alternativa` (`422` nos outros); `usar_alternativa` exige que a viagem tenha rota alternativa sugerida (`422`); trocar uma decisão já feita dá `409` (repetir a mesma é aceito). A decisão só registra — para "parar", o app pausa a viagem com `pause_reason: "aguardando_clima"`.

Erros: `404` se o alerta não for da viagem, a viagem não for da conta, ou o alerta for `staff` e o usuário for Viajante; `403` se o papel não puder registrar aquilo (ex: Gerente marcando "lido em voz alta").

### `GET /alerts` — alertas da conta inteira

Alertas de **todas as viagens** que o usuário pode ver, do mais recente para o mais antigo. Usado pela coluna "Alertas sem ciente" da Central de Risco.

- Admin/Gerente: a conta toda, inclusive os alertas só para o gerente (ex: pausa longa).
- Viajante: só os das próprias viagens, sem os alertas só para o gerente.

Parâmetros: `pending=true` (só os sem "Ciente"), `include_finished=true` (inclui viagens concluídas/canceladas — por padrão ficam de fora), `limit` (1–500, padrão 100).

Cada item tem os campos do alerta (ver acima) mais `trip_id`, `trip_status`, `origin_label`, `destination_label`, `traveler_id` e `traveler_name`.

### `POST /trips/{trip_id}/positions` — posições do app nativo

Posições do Viajante **por HTTP**, usadas pelo app Android. Existe porque o Android corta a rede da WebView (inclusive o WebSocket) alguns minutos depois de o app ir para segundo plano; o app manda por HTTP nativo (CapacitorHttp), que continua funcionando com a tela bloqueada. O navegador continua mandando pelo WebSocket.

```json
{ "positions": [ { "lat": -22.97, "lng": -45.44, "speed_kmh": 82, "heading_degrees": 71, "recorded_at": "2026-10-11T00:07:30Z" } ] }
```

- 1 a 100 posições por chamada. `lat`/`lng`/`speed_kmh`/`heading_degrees` com a mesma validação do WebSocket.
- `recorded_at` (opcional) é quando o GPS registrou — posições guardadas sem sinal chegam depois (RN-52). Sem ele, vale a hora do envio. No futuro (mais de 2 min de tolerância) → `422`. De antes do início da viagem → ignorada (não conta em `accepted`).
- Mesmas regras do WebSocket: só o **Viajante da viagem** envia (`403` para staff; `404` fora do escopo) e só com a viagem **em andamento** (`409`).
- Efeitos: grava todas em `live_positions` e retransmite a **mais recente** pelo WebSocket para quem acompanha (mensagem `position`).
- Resposta: `{"accepted": N, "pending_alerts": [...]}` — os alertas da viagem ainda sem "Ciente" do motorista (`audience = todos`). O app notifica os que ainda não conhecia, com som, mesmo com o celular bloqueado.

### `POST /trips/{trip_id}/ws-ticket` — quem pode acompanhar a viagem

Emite um ticket de uso único (30 s, só esta viagem) para abrir o WebSocket: `{"ticket": "...", "expires_in_seconds": 30}`. Staff da conta e o Viajante da viagem; fora disso, `404`. Ver [WebSocket](websocket.md#autenticacao-ticket-de-uso-unico-nao-o-access-token).

## WebSocket (`/trips/{trip_id}/ws?ticket=...`)

Documentado separadamente em [WebSocket — rastreamento ao vivo](websocket.md), pois o protocolo (mensagens, autenticação por query param, broadcast) é bem diferente do resto da API REST.

## Códigos de erro — convenções gerais

| Código | Quando |
|---|---|
| `401` | Token ausente, inválido, expirado, ou tipo errado (ex: usar um refresh token como access token) |
| `403` | Usuário autenticado, mas papel não permite a ação (ex: Viajante tentando criar veículo) |
| `404` | Recurso não existe **ou** existe mas fora do escopo do usuário (conta diferente, ou viagem de outro Viajante) — nunca se distingue as duas situações na resposta |
| `409` | Conflito de unicidade (e-mail já cadastrado), transição de status de viagem inválida, outra viagem ativa do mesmo motorista, ou decisão de alerta já feita |
| `422` | Corpo da requisição inválido (validação Pydantic) ou regra de negócio violada (papel incompatível, data fora da janela permitida) |
| `502` | Dependência externa crítica falhou (OpenRouteService) |
