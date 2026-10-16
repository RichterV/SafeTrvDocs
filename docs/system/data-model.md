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
| `status` | enum: `planejada`, `em_andamento`, `pausada`, `concluida`, `cancelada` | default `planejada`; transicionado via `PATCH /trips/{id}/status` (regras em [Referência da API](api-reference.md#patch-tripstrip_idstatus)) |
| `origin_label`/`lat`/`lng`, `destination_label`/`lat`/`lng` | string + float | |
| `waypoints` | JSONB | lista de paradas opcionais, formato livre (`[{label, lat, lng}, ...]`) |
| `scheduled_departure_at` | timestamptz | validado na criação: entre agora e `max_trip_planning_days` dias à frente |
| `distance_km`, `duration_estimated_minutes` | float/int, nullable | vêm do resultado do OpenRouteService |
| `started_at` | timestamptz, nullable | preenchido quando a viagem passa para `em_andamento` |
| `ended_at` | timestamptz, nullable | preenchido quando a viagem passa para `concluida` ou `cancelada` |
| `started_automatically`, `completed_automatically` | bool | início/conclusão feitos sozinhos pelo celular — só esses podem ser desfeitos (RN-13/RN-61) |
| `risk_reevaluated_at` | timestamptz, nullable | última reavaliação do risco durante a viagem; o acompanhamento usa para saber quando reavaliar de novo (15 min) |

### `trip_pauses`

Uma linha por pausa da viagem (RN-20 a RN-27).

| Coluna | Tipo | Observação |
|---|---|---|
| `trip_id` | UUID (FK) | indexado |
| `reason` | enum: `descanso`, `refeicao`, `abastecimento`, `carga_descarga`, `aguardando_clima`, `outro` | |
| `note` | string(120), nullable | obrigatório quando `reason = outro` |
| `lat`, `lng` | float, nullable | onde a pausa começou — o rastro de posições para durante a pausa |
| `started_at`, `ended_at` | timestamptz | `ended_at` nulo = pausa em curso |
| `long_pause_notified_at` | timestamptz, nullable | quando o gerente foi avisado da pausa longa (RN-27) — o aviso sai uma vez por pausa |

### `route_segments`

Um trecho da rota (`sequence` define a ordem). `start_lat/lng`, `end_lat/lng` interpolados pela segmentação (ver [Serviços internos](services.md#segmentacao-segmentationpy)); `estimated_arrival_at` é o horário estimado de chegada ao **fim** do trecho.

### `segment_risk_assessments`

Histórico de avaliações de risco por segmento: a primeira é feita no planejamento, e cada reavaliação durante a viagem acrescenta uma nova (só para os trechos reavaliados). A API as devolve da mais recente para a mais antiga (`order_by calculated_at desc` no relacionamento), então `risk_assessments[0]` é a vigente. Guarda os insumos brutos (`rain_probability`, `rain_intensity_mm`, `river_distance_m`, `official_alert_active`) além do `score` e `risk_level` finais — importante para auditoria/explicabilidade do índice.

### `trip_risk_summaries`

1:1 com `trips` (`trip_id` é `unique`). Agrega `score_max` (usado como o nível de risco geral da viagem — CLAUDE.md seção 5: "pior segmento") e `score_avg` como contexto adicional. Recalculado a cada reavaliação, com a avaliação mais recente de cada trecho.

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
| `type` | enum: `chuva`, `rio`, `rota_alternativa`, `oficial`, `pausa_longa` |
| `severity` | enum `RiskLevel` (`baixo`..`critico`) |
| `message` | texto livre, escrito para ser lido em voz alta |
| `audience` | enum: `todos` (default), `staff` — `staff` não aparece para o Viajante |
| `segment_sequence` | int, nullable — primeiro trecho a que o alerta se refere |
| `inmet_alert_id` | int, nullable — id do aviso do INMET que gerou o alerta; evita repetir o mesmo aviso (RN-33) |
| `read_aloud_at` | timestamptz, nullable — o app leu em voz alta |
| `acknowledged`, `acknowledged_at` | "Ciente" (RN-44) |
| `decision`, `decided_at` | enum nullable: `usar_alternativa`, `seguir_rota`, `parar` (RN-40 a RN-42) |

Enviado = `created_at`. Quem gera:

- `trip_planner.py`, no planejamento: `oficial`/`chuva` quando um segmento atinge Alto/Crítico, e `rota_alternativa` quando uma rota com risco menor é encontrada (ver `alternative_routes` acima);
- `trip_monitor.py`, durante a viagem: `chuva`/`rio` quando trechos sobem para Alto/Crítico, `oficial` para um aviso do INMET novo no caminho, e `pausa_longa` (`audience = staff`) para a pausa de mais de 14 h.

Os enums são gravados pelo **nome** (`TODOS`, `EM_ANDAMENTO`), padrão do SQLAlchemy — relevante para `server_default` em migrações.

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
