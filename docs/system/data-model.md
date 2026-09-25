# Modelo de dados

Entidades implementadas em `backend/app/models/`. Visão de produto e o "porquê" de cada campo em [Regras de negócio](../business/rules.md); aqui o foco é a estrutura real das tabelas.

## Diagrama de relacionamento

```
Account ──< User >── UserRole (papéis: admin | gerente | viajante, N:N via tabela)
   │
   ├──< Vehicle
   │
   └──< Trip ──< RouteSegment ──< SegmentRiskAssessment
          │            
          ├── TripRiskSummary (1:1)
          ├── AlternativeRoute (1:1, opcional)
          ├──< Alert
          └──< LivePosition >── User (traveler_id)

RiverGeometry, State, Municipality — tabelas de referência geográfica,
sem chave estrangeira para as demais (consultadas por posição, não por join)
```

## Contas e usuários

### `accounts`

| Coluna | Tipo | Observação |
|---|---|---|
| `id` | UUID (PK) | |
| `name` | string(200) | |
| `created_at`, `updated_at` | timestamptz | |

### `users`

| Coluna | Tipo | Observação |
|---|---|---|
| `id` | UUID (PK) | |
| `account_id` | UUID (FK → accounts) | todo usuário pertence a exatamente uma conta |
| `name`, `email`, `phone` | string | `email` é único **globalmente** (não só por conta) |
| `password_hash` | string | bcrypt |
| `is_active` | bool | default `true` — usuários inativos não conseguem logar nem autenticar no WebSocket |

### `user_roles`

Tabela N:N entre `users` e os papéis (`role` é um enum: `admin`, `gerente`, `viajante`) — um usuário pode acumular mais de um papel na mesma conta (constraint de unicidade em `(user_id, role)` evita duplicar o mesmo papel duas vezes).

## Veículos

### `vehicles`

| Coluna | Tipo |
|---|---|
| `id` | UUID (PK) |
| `account_id` | UUID (FK → accounts) |
| `plate` | string(10) |
| `type` | string(50) — livre, sem enum (ex: "caminhao", "van") |

## Viagens

### `trips`

| Coluna | Tipo | Observação |
|---|---|---|
| `account_id`, `created_by_id`, `traveler_id`, `vehicle_id` | UUID (FK) | `created_by_id` e `traveler_id` referenciam `users` com foreign keys distintas |
| `cargo_type` | enum: `normal`, `perigosa`, `refrigerada` | default `normal` |
| `status` | enum: `planejada`, `em_andamento`, `concluida`, `cancelada` | default `planejada`; transicionado via `PATCH /trips/{id}/status` (regras em [Referência da API](api-reference.md#patch-tripstrip_idstatus)) |
| `origin_label`/`lat`/`lng`, `destination_label`/`lat`/`lng` | string + float | |
| `waypoints` | JSONB | lista de paradas opcionais, formato livre (`[{label, lat, lng}, ...]`) |
| `scheduled_departure_at` | timestamptz | validado na criação: entre agora e `max_trip_planning_days` dias à frente |
| `distance_km`, `duration_estimated_minutes` | float/int, nullable | vêm do resultado do OpenRouteService |
| `started_at` | timestamptz, nullable | preenchido quando a viagem passa para `em_andamento` |
| `ended_at` | timestamptz, nullable | preenchido quando a viagem passa para `concluida` ou `cancelada` |

### `route_segments`

Um trecho da rota (`sequence` define a ordem). `start_lat/lng`, `end_lat/lng` interpolados pela segmentação (ver [Serviços internos](services.md#segmentacao-segmentationpy)); `estimated_arrival_at` é o horário estimado de chegada ao **fim** do trecho.

### `segment_risk_assessments`

Uma avaliação de risco por segmento — hoje sempre 1:1 com `route_segments` (calculada uma única vez, na criação da viagem), mas o modelo permite histórico (múltiplas avaliações por segmento ao longo do tempo, para quando o índice for recalculado conforme a viagem se aproxima — CLAUDE.md seção 4). Guarda os insumos brutos (`rain_probability`, `rain_intensity_mm`, `river_distance_m`, `official_alert_active`) além do `score` e `risk_level` finais — importante para auditoria/explicabilidade do índice.

### `trip_risk_summaries`

1:1 com `trips` (`trip_id` é `unique`). Agrega `score_max` (usado como o nível de risco geral da viagem — CLAUDE.md seção 5: "pior segmento") e `score_avg` como contexto adicional.

### `alternative_routes`

1:1 opcional com `trips` (`trip_id` é `unique`) — só existe quando o sistema encontrou uma rota alternativa com risco estritamente menor que a rota principal (ver [Serviços internos](services.md#sugestao-automatica-de-rota-alternativa)). Diferente de `route_segments`/`segment_risk_assessments`, **não** tem uma tabela relacional própria para os segmentos — é um resumo agregado, deliberadamente mais simples que a rota principal:

| Coluna | Tipo | Observação |
|---|---|---|
| `distance_km`, `duration_estimated_minutes` | float/int | |
| `geometry` | JSONB | lista de `[lat, lng]` — toda a geometria da rota, para desenhar no mapa do frontend |
| `segments_summary` | JSONB | lista de objetos `{sequence, start_lat, start_lng, end_lat, end_lng, distance_km, estimated_arrival_at, score, risk_level}` — um resumo por trecho, análogo a `RouteSegmentRead` mas sem tabela própria |
| `score_max`, `score_avg`, `risk_level` | float/float/enum | mesma semântica de `trip_risk_summaries` |
| `calculated_at` | timestamptz | |

### `alerts`

| Coluna | Tipo |
|---|---|
| `type` | enum: `chuva`, `rio`, `rota_alternativa`, `oficial` |
| `severity` | enum `RiskLevel` (`baixo`..`critico`) |
| `message` | texto livre |
| `acknowledged`, `acknowledged_at` | suporte a "registro de reconhecimento de alerta" (Horizonte 1 do backlog de produto) — campos já existem no modelo, mas **nenhum endpoint ainda os atualiza** |

Gerados automaticamente por `trip_planner.py`: `oficial`/`chuva` quando um segmento atinge Alto/Crítico, e `rota_alternativa` quando uma rota com risco menor é encontrada (ver `alternative_routes` acima). `rio` existe no enum mas ainda não é usado.

## Rastreamento ao vivo

### `live_positions`

Uma linha por posição recebida via WebSocket (não é upsert — o histórico completo fica registrado). `traveler_id` é redundante com `trips.traveler_id` mas guardado aqui também, por simplicidade de consulta e para robustez caso um dia a viagem troque de motorista (Horizonte 3 do backlog).

## Dados de referência geográfica (PostGIS)

### `river_geometries`

Malha hidrográfica estática (ANA/SNIRH). `geom` é `MULTILINESTRING` em SRID 4326 (WGS84). Populada via `scripts/import_geodata.py` — hoje 15.305 trechos (filtrados por ordem de Strahler ≥ 4, ver [Ambiente local](../setup/local-dev.md)). Consultada por proximidade (`ST_Distance`), nunca por join com outras tabelas.

### `states` / `municipalities`

Malha de estados (27) e municípios (5.570) do IBGE, geometria `MULTIPOLYGON` SRID 4326. **Importados mas sem nenhuma feature consumindo ainda** — candidatos a uso futuro: reverse-geocoding, cruzamento com os `geocodes` de avisos do INMET, camada de referência no mapa do frontend.

## Convenções gerais

- Toda tabela de negócio usa UUID como chave primária (`UUIDPrimaryKeyMixin`), gerado em Python (`uuid.uuid4`), não pelo banco.
- Enums são armazenados como `Enum(..., native_enum=False)` — ou seja, como `VARCHAR` no Postgres, não como um tipo `ENUM` nativo. Trade-off deliberado: adicionar um novo valor de enum não exige uma migração de schema `ALTER TYPE`, só o código Python muda.
- `TimestampMixin` (`created_at`/`updated_at`, via `server_default=func.now()`) está em `Account`, `User`, `Vehicle`, `Trip` — mas **não** em `RouteSegment`, `SegmentRiskAssessment`, `TripRiskSummary`, `LivePosition`, `Alert`, `RiverGeometry`, `State`, `Municipality` (essas têm seus próprios campos de data quando fazem sentido, ex: `recorded_at`, `calculated_at`, `imported_at`, `created_at` só em `Alert`).
