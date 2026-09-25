# Visão geral da arquitetura

O SafeTrv é hoje um monorepo com um backend FastAPI e um frontend Ionic + Angular (web + mobile, ver [Frontend](frontend.md)). Esta seção documenta o sistema como ele **realmente está implementado** (não a proposta original) — para a visão de produto/negócio e o backlog priorizado, ver [Negócio](../business/overview.md) e [Regras de negócio](../business/rules.md).

## Stack

| Camada | Tecnologia |
|---|---|
| API | FastAPI (Python 3.10+), assíncrona |
| ORM / migrações | SQLAlchemy 2.0 (modo async) + Alembic |
| Banco de dados | PostgreSQL + PostGIS (via GeoAlchemy2), para consultas espaciais |
| Autenticação | JWT (access + refresh), `pyjwt` + `bcrypt` |
| Tempo real | WebSocket nativo do FastAPI/Starlette |
| Roteamento | OpenRouteService (API pública, chave em `backend/.env`) |
| Previsão do tempo | Open-Meteo (previsão horária por coordenada) |
| Avisos oficiais | INMET (`apiprevmet3.inmet.gov.br/avisos/ativos`) |
| Dados de referência geográfica | IBGE (estados/municípios) + ANA/SNIRH (rios), importados para PostGIS |
| Gerenciador de dependências | [uv](https://docs.astral.sh/uv/) |
| Testes | pytest + pytest-asyncio |
| Lint | ruff |

## Estrutura de pastas (`backend/app`)

```
app/
  core/      # configuração (Settings via pydantic-settings), segurança (hash de senha, JWT)
  db/        # engine assíncrono, sessão (AsyncSessionLocal), Base declarativa
  models/    # entidades SQLAlchemy (Account, User, Trip, RouteSegment, LivePosition, ...)
  schemas/   # modelos Pydantic de request/response, separados dos models de banco
  api/
    deps.py           # dependências FastAPI: get_current_user, require_roles, resolve_user_from_access_token
    v1/
      auth.py          # signup, login, refresh
      users.py         # CRUD de usuários com hierarquia de papéis
      vehicles.py      # CRUD de veículos
      trips.py         # planejamento e consulta de viagens
      live_tracking.py # WebSocket de rastreamento ao vivo
      router.py        # agrega todos os routers sob /api/v1
  services/
    routing.py            # cliente OpenRouteService — calcula a rota entre origem/destino/paradas
    segmentation.py        # divide a rota em trechos de duração ~fixa (build_segments)
    weather.py              # cliente Open-Meteo — previsão horária em lote por coordenada
    inmet.py                # cliente INMET — avisos ativos + verificação de cobertura geo/horária
    geo.py                  # haversine e distância ao rio mais próximo (PostGIS)
    risk.py                 # fórmula do score de risco por segmento
    trip_planner.py         # orquestra todo o fluxo de POST /trips (rota → segmentos → clima → risco → persistência)
    connection_manager.py   # registro em memória das conexões WebSocket ativas por viagem
scripts/
  import_geodata.py   # baixa/importa estados, municípios e rios para o PostGIS
```

Ver o detalhamento de cada serviço em [Serviços internos](services.md), os endpoints em [Referência da API](api-reference.md) e o protocolo de tempo real em [WebSocket — rastreamento ao vivo](websocket.md).

## Fluxo de uma requisição `POST /api/v1/trips`

1. `trips.py` valida que o viajante indicado existe, tem o papel `viajante` e pertence à mesma conta; valida o veículo; valida a janela de `scheduled_departure_at` (até `max_trip_planning_days`, hoje 7 dias — `app/core/config.py`).
2. Delega para `trip_planner.plan_trip`, que:
   - chama `routing.py` (OpenRouteService) para calcular a rota geométrica entre origem, paradas e destino;
   - chama `segmentation.build_segments` para dividir a rota em trechos de ~`segment_target_minutes` (hoje 12 min);
   - chama `weather.py` para buscar a previsão horária do Open-Meteo **em lote** para todos os segmentos de uma vez (evita N chamadas sequenciais);
   - chama `inmet.py` para obter avisos ativos e verificar, por segmento, se o ponto/horário está coberto por algum aviso;
   - chama `geo.nearest_river_distance_m` (consulta PostGIS) para a distância ao rio mapeado mais próximo de cada segmento;
   - chama `risk.calculate_segment_score` para cada segmento (ver fórmula em [Serviços internos](services.md#formula-do-indice-de-risco-riskpy));
   - persiste `Trip` + `RouteSegment` + `SegmentRiskAssessment` + `TripRiskSummary`, e cria um `Alert` (`oficial` ou `chuva`) quando o risco do segmento é Alto ou Crítico.
3. Se o INMET estiver indisponível, o fluxo **degrada graciosamente**: loga um warning e segue sem o piso mínimo de risco, em vez de falhar a criação da viagem inteira. Se o OpenRouteService falhar, a criação da viagem falha com `502 Bad Gateway` (não há como segmentar/avaliar risco sem uma rota).

## Autenticação

- JWT com dois tipos de token: `access` (curta duração, `access_token_expire_minutes`) e `refresh` (`refresh_token_expire_days`).
- `POST /auth/login` e `/auth/signup` retornam os dois tokens; `POST /auth/refresh` troca um refresh token válido por um novo access token.
- Para requisições REST, o access token vai no header `Authorization: Bearer <token>`.
- Para o WebSocket, o handshake do navegador não permite headers, então o cliente troca o access token (via HTTP) por um **ticket de uso único** de 30 s e conecta com `?ticket=...` — o access token nunca vai na URL. Ver [WebSocket](websocket.md#autenticacao-ticket-de-uso-unico-nao-o-access-token).

## Ambiente de execução

Ver [Ambiente local](../setup/local-dev.md) para como rodar tudo. Em resumo: `docker compose up -d` sobe Postgres+PostGIS na raiz do repo; `uv sync` + `uv run alembic upgrade head` prepara o backend; `scripts/import_geodata.py` popula os dados de referência geográfica (obrigatório para o risco de proximidade a rio funcionar — sem isso a tabela fica vazia e esse fator sempre resulta em zero).
