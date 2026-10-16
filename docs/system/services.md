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
- **Só avisos de chuva/tempestade entram no índice** (`is_rain_related`: a descrição contém "chuva" ou "tempestade" — ex: "Tempestade", "Chuvas Intensas", "Acumulado de Chuva"). "Onda de Calor", "Baixa Umidade", "Geada", "Vendaval" etc. são descartados em `get_active_alerts`, porque o índice do MVP é chuva + rio. Antes desse filtro (corrigido em 2026-09-26), um aviso "Grande Perigo — Onda de Calor" sobre SP deixava todos os trechos Críticos com 0 mm de chuva.
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
- **Aviso oficial ativo do INMET**: aplica um **piso mínimo** graduado pela severidade do aviso que cobre o segmento (`official_alert_floor(severidade, id_severidade)`, parâmetro `official_alert_floor` de `calculate_segment_score`): Perigo Potencial → `25.0` (Médio), Perigo → `50.0` (Alto), Grande Perigo → `75.0` (Crítico) — tabela `OFFICIAL_ALERT_FLOORS`, pelo rótulo `severidade` do INMET. Se o rótulo não for reconhecido, usa o `id_severidade` (6/7/8, valores observados na API em 2026-09-26) e, por último, `50.0`. O piso é aplicado por último, depois dos multiplicadores, e nunca reduz um score que já seria maior. Quando vários avisos cobrem o mesmo ponto, `inmet.find_covering_alert` escolhe o de maior piso.

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

## Mudança de status — `trip_status.py` e `trip_lifecycle.py`

`trip_status.py` é puro: a tabela de transições por papel (`check_transition`) e os prazos para desfazer o que foi automático (`check_undo_window`: 5 min para o início, 30 min para a conclusão). `trip_lifecycle.change_trip_status` aplica a mudança na viagem: abre/fecha a pausa, marca `started_automatically`/`completed_automatically`, recusa uma segunda viagem ativa do mesmo motorista (RN-14) e diz se o risco do restante deve ser recalculado (ao iniciar; ao retomar depois de mais de 30 min de pausa). Não faz commit — o endpoint faz.

## Acompanhamento durante a viagem — `trip_monitor.py`

Implementa a reavaliação do risco com a viagem em andamento (RN-30 a RN-33) e o aviso de pausa longa (RN-27). Os cálculos puros ficam em `reevaluation.py` (testado sem banco nem rede em `tests/test_reevaluation.py`).

**`reevaluate_trip(db, trip, official_alerts, now, horizon)`:**

1. **Onde o motorista está:** última posição recebida desde `started_at` (sem nenhuma, a origem). `locate_progress` acha o trecho mais próximo e a fração dele (projeção local).
2. **Novos horários de passagem:** `shifted_arrivals` desloca todos os trechos à frente pelo atraso real — o ponto atual deveria ter sido alcançado em *planejado*; tudo o que falta anda `agora − planejado`. Não reescala pela velocidade instantânea, que oscila demais para prever horas à frente. Os `estimated_arrival_at` dos trechos à frente são atualizados.
3. **Pontuação:** os trechos com passagem dentro do horizonte (3 h; `horizon=None` = todo o restante) passam pela mesma `trip_planner.score_segments` do planejamento (Open-Meteo + PostGIS + INMET). Cada um ganha uma nova linha em `segment_risk_assessments`.
4. **Alertas (`plan_alerts`):** trecho que passou a Alto/Crítico, ou subiu de Alto para Crítico, gera alerta; trechos seguidos viram **um** alerta ("nos trechos 3 a 5"), para não disparar uma leitura por trecho. Tipo `rio` quando há chuva a menos de 1 km de rio, senão `chuva`. Aviso do INMET que a viagem ainda não recebeu (pelo `inmet_alert_id`) gera **um** alerta `oficial` por aviso, de qualquer severidade. Mesmo nível, ou risco que desceu, não gera nada (RN-33). O texto é o de "sem desvio" (risco, onde, quando, em horário de Brasília, e como dirigir) — ainda não há busca de desvio com a viagem em andamento.
5. `trip_risk_summaries` é recalculado com a avaliação mais recente de cada trecho, e `trips.risk_reevaluated_at = now`.

Depois do commit, `broadcast_outcome` envia os alertas novos e uma mensagem `risk` pelo WebSocket.

**`run_monitor_cycle(session_factory, now)`** — uma rodada: busca os avisos do INMET uma vez (falhou → segue sem eles); para cada viagem `em_andamento`, reavalia se a última reavaliação tem 15 min ou mais **ou** se há um aviso do INMET ainda não recebido cobrindo um trecho das próximas 3 h (checagem local, sem chamada externa); depois cria o aviso de pausa longa (`pausa_longa`, `audience = staff`, uma vez por pausa) para pausas abertas há mais de 14 h. Falha numa viagem não impede as outras.

**`monitor_forever`** roda o ciclo a cada `TRIP_MONITOR_INTERVAL_SECONDS` (default 300 s — é isso que define quão rápido um aviso novo do INMET chega a quem está na estrada; a reavaliação de cada viagem continua a cada 15 min). Sobe no *lifespan* do app (`app/main.py`) quando `TRIP_MONITOR_ENABLED` (default `true`). Os testes desligam (`tests/conftest.py`) e chamam `run_monitor_cycle` direto. **Limitação:** roda dentro do processo da API; com mais de uma instância, cada uma rodaria o ciclo — precisa de um agendador único (ou lock no banco) antes de escalar.

Validado ao vivo em 2026-09-26 com uvicorn real, Open-Meteo e INMET reais e as 2 viagens em andamento do seed: as duas foram reavaliadas no startup (uma vez só no intervalo), os horários de passagem andaram +6 e +20 min conforme a posição real, e o risco foi recalculado com o clima do dia (SP→Rio: Médio 49 do cenário do seed → Baixo 22,7), sem alerta (risco desceu).

## Central de Risco — `fleet_overview.py`

Cálculos puros (sem banco nem rede) usados por `GET /trips/overview`, testados em `tests/test_fleet_overview.py`:

- `estimated_arrival` — chegada prevista ao destino: passagem pelo último trecho + a duração de um trecho (os trechos têm a mesma duração).
- `next_high_risk` — primeiro trecho Alto/Crítico ainda à frente. Em viagem em andamento ou pausada, descarta os trechos cujo fim (passagem + duração de um trecho) já ficou para trás.
- `triage_key` — ordem da Central: pior risco primeiro; no empate, viagens na estrada antes das planejadas; depois, o horário do próximo trecho de risco (ou da saída).

A rota (`trips.py`) carrega as viagens com trechos, alertas e pausas, e busca a última posição de todas numa consulta só (`DISTINCT ON (trip_id)` em `live_positions`).

## Conexões WebSocket — `connection_manager.py`

Ver [WebSocket — rastreamento ao vivo](websocket.md#registro-de-conexoes-limitacao-conhecida).
