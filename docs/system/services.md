# Serviços internos

Detalhamento de `backend/app/services/`, com os valores e comportamentos **realmente implementados** no código (não a proposta original — ver CLAUDE.md seção 10 para os pontos ainda sujeitos a calibração).

## Roteamento — `routing.py`

Cliente do OpenRouteService: recebe origem, destino e paradas opcionais, retorna a geometria da rota (`RouteResult`, uma lista de pontos `(lat, lng, tempo_decorrido_em_segundos)`), a distância total e a duração estimada. É a única dependência externa cuja falha derruba a criação da viagem (`502`) — sem uma rota não há como segmentar nem avaliar risco.

## Segmentação — `segmentation.py`

`build_segments(route, departure_at, segment_seconds)` divide a rota em N trechos de duração aproximadamente igual, onde `N = round(duração_total / segment_seconds)` (hoje `segment_seconds` = 12 min, configurável via `SEGMENT_TARGET_MINUTES`). A posição de início/fim de cada trecho é interpolada linearmente entre os pontos da rota mais próximos daquele instante de tempo — não recalcula a distância real percorrida ponto a ponto dentro do trecho, é uma aproximação proporcional ao tempo.

## Previsão do tempo — `weather.py`

Cliente do Open-Meteo (`GET /v1/forecast`). Busca `precipitation_probability` e `precipitation` horários para **todas as coordenadas dos segmentos em uma única chamada em lote** (o Open-Meteo aceita múltiplas lat/lng por requisição) — evita N chamadas sequenciais para uma viagem com muitos segmentos. Para cada segmento, encontra a hora mais próxima do horário estimado de passagem (`_closest_hour`) e extrai probabilidade (%) e intensidade (mm/h) daquela hora.

## Avisos oficiais — `inmet.py`

Cliente do INMET (`GET /avisos/ativos`). Pontos importantes da implementação:

- **User-Agent de navegador obrigatório** — o WAF do INMET derruba conexões com o User-Agent padrão do httpx (identificado como "de robô"). O serviço envia um header de navegador comum para contornar isso.
- Cada aviso malformado na resposta é **ignorado individualmente** (best-effort) em vez de derrubar a lista inteira — é uma fonte complementar, não deve travar o planejamento se estiver parcialmente instável.
- Datas/horas do INMET vêm em horário de Brasília; como o Brasil não tem mais horário de verão desde 2019 (Decreto 9.918/2019), o serviço usa um offset fixo (UTC-3) em vez de depender de uma base de fusos horários.
- `find_covering_alert(alerts, lat, lng, momento)` testa, para cada aviso ativo, se o horário do segmento está dentro da janela do aviso **e** se o ponto está dentro do polígono de cobertura (`_point_in_polygon`, ray casting — suficiente para polígonos do tamanho de um aviso estadual/regional). Se mais de um aviso cobrir o ponto, retorna o de maior `id_severidade`.
- Se a chamada ao INMET falhar inteira (`InmetError`), `trip_planner.py` captura o erro, loga um warning e segue sem o piso mínimo de risco — não falha a criação da viagem.

## Proximidade a rio — `geo.py`

`nearest_river_distance_m(db, lat, lng)` roda uma consulta PostGIS (`ST_Distance` com `geography`, que já considera a curvatura da Terra) contra `river_geometries` para achar a distância ao trecho de rio mapeado mais próximo. Retorna `None` enquanto a tabela estiver vazia (o score de risco correspondente vira 0 nesse caso) — hoje já populada com 15.305 trechos (ver [Ambiente local](../setup/local-dev.md)).

## Fórmula do índice de risco — `risk.py`

Score de 0 a 100 por segmento, calculado em `calculate_segment_score`:

| Componente | Peso | Constante no código |
|---|---|---|
| Probabilidade de chuva | 25% do score | `RAIN_PROBABILITY_WEIGHT = 25.0` — proporcional direto à % de chance (0–100) |
| Intensidade de chuva | 30% do score | `RAIN_INTENSITY_WEIGHT = 30.0` — mapeada em faixas: 0 mm/h → 0; <2.5 → 0.25; <10 → 0.55; <50 → 0.85; ≥50 → 1.0 (multiplicado pelo peso) |
| Proximidade a rio | 30% do score | `RIVER_PROXIMITY_WEIGHT = 30.0` — `proximidade × intensidade_de_chuva`, onde proximidade cai linearmente a zero em `RIVER_INFLUENCE_RADIUS_M = 1000` m (ou seja, um rio a 1 km ou mais não pesa nada nesse score; a 0 m, pesa o máximo, mas só se também estiver chovendo) |

Modificadores aplicados **depois** da soma dos três componentes:

- **Carga perigosa** (`CargoType.PERIGOSA`): multiplica o score por `1.2`, mas só quando há algum risco de rio (`river_score > 0`) — não amplifica o score de segmentos sem exposição a alagamento.
- **Horário noturno**: multiplica por `1.10` quando a hora estimada de passagem está entre 18h e 6h (`NIGHT_START_HOUR = 18`, `NIGHT_END_HOUR = 6`).
- **Aviso oficial ativo do INMET**: aplica um **piso mínimo** de `OFFICIAL_ALERT_FLOOR = 50.0` (nunca abaixo de "Alto") se o segmento estiver coberto por um aviso — isso acontece por último, depois dos multiplicadores, e nunca reduz um score que já seria maior.

O score final é limitado a 100 e arredondado a 1 casa decimal. Faixas de classificação (`risk_level_from_score`): Baixo (<25), Médio (25–49), Alto (50–74), Crítico (≥75) — idênticas à proposta original (CLAUDE.md seção 5).

> Estes pesos e constantes ainda não foram calibrados com dados reais de sinistro/impacto — são a primeira implementação da proposta da seção 5 do CLAUDE.md, ajustáveis conforme validação com clientes-piloto (seção 10).

## Orquestração — `trip_planner.py`

`plan_trip(...)` é o ponto de entrada único chamado por `POST /trips`. Coordena, nesta ordem: rota → segmentação → previsão em lote → avisos INMET → distância a rio por segmento → score por segmento → persistência de `Trip`/`RouteSegment`/`SegmentRiskAssessment`/`TripRiskSummary`/`Alert`. É o lugar certo para adicionar novos fatores de risco (seção 5-A do CLAUDE.md) ou a sugestão automática de rota alternativa (próximo item do backlog).

## Conexões WebSocket — `connection_manager.py`

Ver [WebSocket — rastreamento ao vivo](websocket.md#registro-de-conexoes-limitacao-conhecida).
