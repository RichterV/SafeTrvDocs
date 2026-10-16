# WebSocket — rastreamento ao vivo

`WS /api/v1/trips/{trip_id}/ws?ticket=<ticket>` — implementado em `app/api/v1/live_tracking.py`, testado de ponta a ponta com clientes reais em 2026-09-18 e coberto por testes de integração (ver abaixo).

## Autenticação: ticket de uso único (não o access token)

O handshake de WebSocket do navegador (`new WebSocket(url)`) não permite header `Authorization`, então a credencial precisa ir na URL. Até 2026-09-25 ia o próprio **access token** (`?token=`), e isso fazia o token vazar: o log do uvicorn gravava a URL completa a cada conexão, e o mesmo aconteceria em logs de proxy, balanceador etc. Agora:

1. O cliente pede um ticket por HTTP autenticado normal (header `Authorization`):
   `POST /api/v1/trips/{trip_id}/ws-ticket` → `{"ticket": "...", "expires_in_seconds": 30}`.
   Mesmas regras de acesso do WebSocket; fora do escopo → `404`.
2. Conecta com `?ticket=...` em até 30 s.

O ticket (`app/services/ws_tickets.py`) é aleatório (`secrets.token_urlsafe(32)`), vale para **uma única viagem**, expira em **30 s** e é **consumido no primeiro uso**. Uma tentativa com a viagem errada também o invalida. Se aparecer num log, já não serve para nada. `?token=` não é mais aceito.

Como defesa em profundidade, `app/core/log_redaction.py` instala um filtro nos loggers do uvicorn que mascara `token=`, `ticket=`, `access_token=` e `refresh_token=` em qualquer linha de log (`ticket=[redacted]`). Conferido num log real após o e2e: nenhum JWT e nenhum ticket gravados.

**Limitação:** os tickets ficam em memória, por processo — mesma situação do registro de conexões (ver abaixo). Com várias instâncias do backend, o pedido do ticket e a conexão podem cair em processos diferentes. Nesse caso é preciso um armazenamento compartilhado (ex: Redis) ou sessão fixa por cliente no balanceador.

## Quem pode conectar

No handshake, o servidor:

1. Consome o ticket. Se for inválido, expirado, já usado ou de outra viagem, fecha com `WS_1008_POLICY_VIOLATION`.
2. Carrega o usuário dono do ticket e a `Trip`, e **revalida** o acesso: o usuário pode ter sido desativado depois de pedir o ticket. Fecha com `1008` se:
   - o usuário não existir ou estiver inativo;
   - a viagem não existir, ou existir em **outra conta**;
   - o usuário não for nem Staff (Admin/Gerente da mesma conta) nem o Viajante dono da viagem.

## Papéis dentro da conexão

- **Viajante da viagem**: pode enviar posições. É o único papel que efetivamente atualiza o estado.
- **Staff (Admin/Gerente da mesma conta)**: só escuta. Qualquer mensagem que envie é silenciosamente ignorada pelo servidor (`if not is_traveler: continue`) — não gera erro, mas também não tem efeito algum.

## Mensagens

### Cliente → servidor (só o Viajante)

```json
{ "lat": -23.55052, "lng": -46.633308, "speed_kmh": 60, "heading_degrees": 270 }
```

Posições só são aceitas enquanto a viagem está `em_andamento` — com a viagem `planejada`, `pausada`, `concluida` ou `cancelada`, o servidor responde `{"type": "error", "detail": "Posições só são aceitas com a viagem em andamento"}`, não persiste nada e mantém a conexão aberta. O status é conferido no banco a cada mensagem, então a regra vale mesmo que o status mude com o socket já aberto.

Validado por `LivePositionIn` (`app/schemas/live_position.py`): `lat` ∈ [-90, 90], `lng` ∈ [-180, 180], `speed_kmh` ≥ 0 (opcional), `heading_degrees` ∈ [0, 360) (opcional). Mensagem inválida → servidor responde `{"type": "error", "detail": "Mensagem inválida"}` e mantém a conexão aberta (não desconecta por payload malformado).

### Servidor → clientes (broadcast)

A cada posição válida recebida do Viajante, o servidor:

1. Persiste em `LivePosition` (`trip_id`, `traveler_id`, `lat`, `lng`, `speed_kmh`, `heading_degrees`, `recorded_at`).
2. Retransmite para **todos os demais** conectados na mesma viagem (exceto quem enviou):

```json
{
  "type": "position",
  "trip_id": "uuid",
  "traveler_id": "uuid",
  "lat": -23.55052,
  "lng": -46.633308,
  "speed_kmh": 60,
  "heading_degrees": 270,
  "recorded_at": "2026-09-19T00:48:04.830882+00:00"
}
```

### Mudança de status

Quando o status da viagem muda via `PATCH /trips/{id}/status`, o servidor envia a **todos** os conectados na viagem:

```json
{
  "type": "status",
  "trip_id": "uuid",
  "status": "pausada",
  "changed_at": "2026-09-26T16:40:05.578628+00:00",
  "pause": {
    "id": "uuid",
    "reason": "refeicao",
    "note": null,
    "lat": -23.2,
    "lng": -46.9,
    "started_at": "2026-09-26T16:40:05.578628+00:00",
    "ended_at": null,
    "long_pause_notified_at": null
  }
}
```

`pause` é a pausa em curso (mesmo formato de `pauses` em `GET /trips/{id}`), ou `null` fora de uma pausa. Durante a pausa o Viajante não envia posições (RN-22).

### Alertas

Um alerta **novo** (gerado pela reavaliação do risco, ou o aviso de pausa longa) e um alerta **que mudou** (lido em voz alta, ciente, decisão — ver [Referência da API](api-reference.md#ciclo-de-vida-do-alerta-post-tripstrip_idalertsalert_id)) chegam no mesmo formato. O cliente faz *upsert* pelo `id`:

```json
{ "type": "alert", "alert": { "id": "uuid", "type": "chuva", "severity": "alto", "message": "Risco alto à frente, nos trechos 3 a 5, …", "audience": "todos", "...": "mesmos campos de AlertRead" } }
```

Alertas com `audience: "staff"` só vão para os sockets de Admin/Gerente — o `connection_manager` sabe, por conexão, se ela é de staff.

### Reavaliação do risco

Depois de cada reavaliação do risco durante a viagem (ao iniciar, ao retomar depois de uma pausa longa, a cada 15 min, ou quando surge um aviso novo do INMET no caminho):

```json
{
  "type": "risk",
  "trip_id": "uuid",
  "risk_level": "alto",
  "previous_risk_level": "baixo",
  "score_max": 55.0,
  "score_avg": 31.2,
  "calculated_at": "2026-09-26T17:02:05+00:00"
}
```

Os trechos (horários de passagem e avaliações) mudaram; o cliente recarrega a viagem.

### Ao conectar

Qualquer conexão nova (Viajante ou Staff) recebe primeiro o status atual da viagem (`{"type": "status", "trip_id", "status"}`, sem `changed_at`) e, logo em seguida, a última posição conhecida da viagem, se houver alguma registrada — útil para o painel do Gerente já abrir mostrando onde o Viajante está, sem esperar a próxima atualização.

## Registro de conexões — limitação conhecida

`app/services/connection_manager.py` mantém um dicionário em memória (`trip_id → {WebSocket: é staff?}`) **por processo**. Funciona perfeitamente para desenvolvimento e para uma única instância do backend. Para rodar múltiplas instâncias (escala horizontal), seria necessário um pub/sub compartilhado entre processos (ex: Redis Pub/Sub) — hoje não implementado, não é necessário para o volume esperado do MVP.

## O que foi validado (2026-09-18)

Testado com um script Python usando a biblioteca `websockets`, simulando duas conexões simultâneas na mesma viagem:

- Viajante conecta e envia 3 posições em sequência; Staff (Gerente) conectado recebe cada broadcast em tempo real, com os dados corretos.
- Staff tenta enviar uma posição — é ignorado, não gera broadcast nem erro.
- Nova conexão (simulando o Gerente reconectando) recebe a última posição conhecida imediatamente ao abrir o socket.
- Todas as posições enviadas foram conferidas persistidas na tabela `live_positions` via query direta no Postgres.
- Token inválido e usuário de outra conta são rejeitados no handshake.

Validado também em 2026-09-25, junto com o endpoint de status: posição rejeitada com a viagem `planejada`, aceita e retransmitida depois de `em_andamento`, rejeitada de novo após `concluida`; mudanças de status recebidas em tempo real pelo Viajante e pelo Staff.

## Testes automatizados

`backend/tests/integration/test_live_tracking_ws.py` cobre o protocolo contra o Postgres real: pedido de ticket (sem login, outra conta), handshake recusado (ticket inventado, access token no lugar do ticket, ticket de outra viagem, reutilizado, expirado, usuário desativado), status atual ao conectar, posição recusada fora de `em_andamento`, broadcast de status (via `PATCH`) e de posição, mensagem do staff ignorada, validação do payload, última posição na reconexão, persistência em `live_positions`, posição recusada após a conclusão.

## App nativo: posições por HTTP

No app Android, o Viajante **não** manda posições pelo WebSocket: o Android corta a rede da WebView alguns minutos depois de o app ir para segundo plano. O app usa `POST /trips/{id}/positions` ([Referência da API](api-reference.md)) por HTTP nativo, e o servidor retransmite a posição pelo WebSocket para quem acompanha — para o gerente, nada muda. O app continua conectado ao WebSocket para receber status e alertas enquanto está aberto.

## O que falta

- Teste de carga / múltiplas conexões simultâneas além do par viajante+staff (ex: vários Gerentes acompanhando a mesma viagem).
