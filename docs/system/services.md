# Serviços internos

Detalhamento de `backend/app/services/`, com os valores e comportamentos **realmente implementados** no código (não a proposta original — ver CLAUDE.md seção 10 para os pontos ainda sujeitos a calibração).

## Roteamento — `routing.py`

Cliente do OpenRouteService: recebe origem, destino e paradas opcionais, retorna a geometria da rota (`RouteResult`, uma lista de pontos `(lat, lng, tempo_decorrido_em_segundos)`), a distância total e a duração estimada. É a única dependência externa cuja falha derruba a criação da viagem (`502`) — sem uma rota não há como segmentar nem avaliar risco.

`get_route_alternatives(origin, destination, target_count=2)` usa o recurso `alternative_routes` da API do OpenRouteService para buscar até `target_count` rotas alternativas entre dois pontos — usado pela sugestão automática de rota alternativa (ver abaixo). **Duas limitações do provedor, não do nosso código:** só aceita exatamente 2 coordenadas (sem paradas intermediárias) e só aceita rotas de **até 100 km** — acima disso a API responde `400` (`"the approximated route distance must not be greater than 100000.0 meters"`), confirmado em teste real. `_find_lower_risk_alternative` (em `trip_planner.py`) nem chega a chamar esse recurso quando origem e destino estão a mais de 95 km em linha reta.

`get_route(coordinates, avoid_polygons=...)` aceita uma lista de áreas (anéis de lat/lng) que a rota deve evitar — usado pelos desvios locais (ver abaixo). Limites da API pública com áreas a evitar, confirmados em teste real: **cada polígono até 200 km²** (`2.0E8` m², código `2003`) e **distância aproximada até 150 km** (código `2004`). A "distância aproximada" do provedor é, na prática, a linha reta entre as coordenadas pedidas, não a distância real da rota. `RouteResult.stop_seconds` guarda em que segundo a rota passa por cada coordenada pedida (origem, paradas, destino), a partir dos `way_points` da resposta.

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

`plan_trip(...)` é o ponto de entrada único chamado por `POST /trips`. Coordena, nesta ordem: rota → segmentação → previsão em lote → avisos INMET → score por segmento (`_score_segments`, compartilhada com a busca de rota alternativa) → persistência de `Trip`/`RouteSegment`/`SegmentRiskAssessment`/`TripRiskSummary`/`Alert` → sugestão de rota alternativa, se aplicável. É o lugar certo para adicionar novos fatores de risco (seção 5-A do CLAUDE.md).

### Sugestão automática de rota alternativa

Implementada em `_find_lower_risk_alternative`, chamada por `plan_trip` sempre que `TripRiskSummary.risk_level` é Alto/Crítico. Junta rotas candidatas de duas estratégias:

1. **Desvio local** (`_local_detour_candidate` + `detour.py`, qualquer distância e com ou sem paradas): em vez de recalcular a rota inteira, recorta janelas curtas da rota principal em volta dos trechos de risco, pede ao provedor um caminho entre as pontas de cada janela evitando pequenas áreas em volta dos pontos de risco, e emenda os desvios na rota original.
2. **`alternative_routes` do ORS**: só quando a viagem não tem paradas e origem/destino estão a até 95 km em linha reta (limite de 100 km do provedor).

Para cada candidata, roda a segmentação e `_score_segments` de novo (mesma função da rota principal, sem duplicar a lógica de risco) e escolhe a de menor `score_max`. Só persiste se ela for **estritamente menor** que o `score_max` da rota principal — se nenhuma for melhor (comum quando um aviso oficial do INMET cobre uma área ampla, já que qualquer rota ali cai no mesmo piso), não sugere nada. Se encontrar, persiste em `AlternativeRoute` (ver [Modelo de dados](data-model.md#alternative_routes)) e gera um `Alert` do tipo `rota_alternativa`.

Qualquer falha nesse processo (`routing.RoutingError`, `weather.WeatherError`) é não-fatal — loga um warning e a viagem é criada normalmente, sem a sugestão. No desvio local, a falha de uma janela só descarta aquela janela; as demais continuam valendo.

### Desvios locais — `detour.py`

Cálculo puro (sem rede nem banco), testado em `tests/test_detour.py`:

- **Janelas** (`plan_detour_windows`): trechos Alto/Crítico consecutivos formam um bloco. Cada bloco ganha uma folga de pelo menos **8 km** (linha reta) antes e depois, andando trecho a trecho, para o desvio ter por onde sair e voltar. Se o bloco passar de **120 km** em linha reta (margem sob o limite de 150 km do provedor), é quebrado em várias janelas. No máximo **4 janelas** por viagem (as de pior score), porque cada janela é uma requisição ao provedor.
- **Áreas evitadas**: um quadrado de 8×8 km (64 km², abaixo do limite de 200 km²) no meio e no fim de cada trecho de risco — os mesmos pontos onde o risco é medido. Quadrados que contêm uma ponta da janela ou uma parada da viagem são descartados, porque o provedor não calcula rota saindo de (ou passando por) uma área evitada. Por isso, risco colado na origem, no destino ou numa parada não tem como ser evitado.
- **Paradas**: paradas intermediárias que caem dentro de uma janela entram como coordenadas intermediárias do desvio, e a rota continua passando por elas.
- **Emenda** (`splice_detours`): substitui o pedaço da rota original entre as pontas de cada janela pela geometria do desvio. Os tempos dali em diante são deslocados pela diferença de duração. A distância total é ajustada (sai a distância do pedaço original, entra a do desvio). As emendas são aplicadas da última janela para a primeira, para os tempos das janelas anteriores continuarem válidos.

Validado ao vivo em 2026-09-25 contra o ORS real, com o risco forçado nos segmentos: São Paulo → Rio de Janeiro (435 km — antes a sugestão nunca disparava nessa distância). Com risco perto de São José dos Campos, o desvio afastou a rota de 0,9 km para 10,2 km do ponto de risco (+15 km, +10 min). Com uma parada em Taubaté e dois blocos de risco, os dois desvios foram emendados e o horário da parada foi deslocado corretamente. A busca completa (`_find_lower_risk_alternative` com Open-Meteo, PostGIS e INMET reais) também escolheu o desvio corretamente.

## Conexões WebSocket — `connection_manager.py`

Ver [WebSocket — rastreamento ao vivo](websocket.md#registro-de-conexoes-limitacao-conhecida).
