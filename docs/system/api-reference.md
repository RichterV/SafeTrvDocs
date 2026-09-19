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
- Falha ao buscar rota alternativa (provedor indisponível, ou limite de 100km do ORS) **não** derruba a criação da viagem — degrada graciosamente, sem `alternative_route` na resposta.

Retorna `201` com o objeto `Trip` completo: dados da viagem, lista de `segments` (cada um com seus `risk_assessments`), `risk_summary` (score máximo, médio, nível), `alternative_route` (ver abaixo, `null` se nenhuma alternativa foi sugerida) e `alerts` gerados.

**`alternative_route`** (presente só quando: risco Alto/Crítico, sem paradas intermediárias, e uma alternativa com risco menor foi encontrada dentro do limite de 100km do provedor — ver [Serviços internos](services.md#sugestao-automatica-de-rota-alternativa)):

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

### `GET /trips/{trip_id}`

Retorna a viagem completa (mesmo formato do `POST`). `404` se não existir na conta do usuário, **ou** se existir mas o usuário for um Viajante que não é o dono dela — a resposta é o mesmo 404 nos dois casos, para não vazar a existência de viagens de outros usuários.

## WebSocket (`/trips/{trip_id}/ws`)

Documentado separadamente em [WebSocket — rastreamento ao vivo](websocket.md), pois o protocolo (mensagens, autenticação por query param, broadcast) é bem diferente do resto da API REST.

## Códigos de erro — convenções gerais

| Código | Quando |
|---|---|
| `401` | Token ausente, inválido, expirado, ou tipo errado (ex: usar um refresh token como access token) |
| `403` | Usuário autenticado, mas papel não permite a ação (ex: Viajante tentando criar veículo) |
| `404` | Recurso não existe **ou** existe mas fora do escopo do usuário (conta diferente, ou viagem de outro Viajante) — nunca se distingue as duas situações na resposta |
| `409` | Conflito de unicidade (e-mail já cadastrado) |
| `422` | Corpo da requisição inválido (validação Pydantic) ou regra de negócio violada (papel incompatível, data fora da janela permitida) |
| `502` | Dependência externa crítica falhou (OpenRouteService) |
