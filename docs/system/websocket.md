# WebSocket — rastreamento ao vivo

`WS /api/v1/trips/{trip_id}/ws?token=<access_token>` — implementado em `app/api/v1/live_tracking.py`, testado de ponta a ponta com clientes reais em 2026-09-18 (ver CLAUDE.md seção 0).

## Por que o token vai por query param

O handshake de WebSocket do navegador (`new WebSocket(url)`) não permite definir headers customizados como `Authorization`. Por isso o access token (o mesmo emitido por `/auth/login`) vai como `?token=...` na própria URL de conexão.

## Quem pode conectar

Ao receber a conexão, o servidor:

1. Decodifica o token e carrega o usuário (`resolve_user_from_access_token` — a mesma função usada pela dependência HTTP `get_current_user`, reaproveitada aqui).
2. Carrega a `Trip` pelo `trip_id` da URL.
3. Fecha a conexão com `WS_1008_POLICY_VIOLATION` se:
   - o token for inválido/expirado;
   - a viagem não existir, ou existir em **outra conta**;
   - o usuário não for nem Staff (Admin/Gerente da mesma conta) nem o Viajante dono da viagem.

Validado com um script de teste real: token inválido e usuário de uma conta diferente são ambos rejeitados no handshake (a conexão fecha antes de qualquer mensagem).

## Papéis dentro da conexão

- **Viajante da viagem**: pode enviar posições. É o único papel que efetivamente atualiza o estado.
- **Staff (Admin/Gerente da mesma conta)**: só escuta. Qualquer mensagem que envie é silenciosamente ignorada pelo servidor (`if not is_traveler: continue`) — não gera erro, mas também não tem efeito algum.

## Mensagens

### Cliente → servidor (só o Viajante)

```json
{ "lat": -23.55052, "lng": -46.633308, "speed_kmh": 60, "heading_degrees": 270 }
```

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

### Ao conectar

Qualquer conexão nova (Viajante ou Staff) recebe **imediatamente** a última posição conhecida da viagem, se houver alguma registrada — útil para o painel do Gerente já abrir mostrando onde o Viajante está, sem esperar a próxima atualização.

## Registro de conexões — limitação conhecida

`app/services/connection_manager.py` mantém um dicionário em memória (`trip_id → set[WebSocket]`) **por processo**. Funciona perfeitamente para desenvolvimento e para uma única instância do backend. Para rodar múltiplas instâncias (escala horizontal), seria necessário um pub/sub compartilhado entre processos (ex: Redis Pub/Sub) — hoje não implementado, não é necessário para o volume esperado do MVP.

## O que foi validado (2026-09-18)

Testado com um script Python usando a biblioteca `websockets`, simulando duas conexões simultâneas na mesma viagem:

- Viajante conecta e envia 3 posições em sequência; Staff (Gerente) conectado recebe cada broadcast em tempo real, com os dados corretos.
- Staff tenta enviar uma posição — é ignorado, não gera broadcast nem erro.
- Nova conexão (simulando o Gerente reconectando) recebe a última posição conhecida imediatamente ao abrir o socket.
- Todas as posições enviadas foram conferidas persistidas na tabela `live_positions` via query direta no Postgres.
- Token inválido e usuário de outra conta são rejeitados no handshake.

## O que falta

- Endpoint para transição de status da viagem (`planejada` → `em_andamento` → `concluída`/`cancelada`) — hoje o WebSocket aceita conexões independentemente do status da `Trip`, então não há como o sistema saber se a viagem "está em andamento" de fato.
- Teste de carga / múltiplas conexões simultâneas além do par viajante+staff (ex: vários Gerentes acompanhando a mesma viagem).
