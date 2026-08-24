# raybot

Framework de bots sobre **websocket de larga vida**, escrito en [raylang](https://github.com/roberto-ayala/raylang). Es el estreno de `websocket_client`: un cliente de gateway estilo Discord (HELLO → IDENTIFY → heartbeats por intervalo → dispatches) con **reconexión automática** (backoff exponencial, re-IDENTIFY) y comandos con estado en SQLite que sobrevive a las caídas. Incluye un **gateway falso** (servidor `net/websocket`) para desarrollar y testear el bot entero sin tocar Discord.

```text
$ raybot demo            # gateway falso + bot, todo local
raybot: ready (gen 1)
bot> general	pong
bot> general	ana: 1
bot> general	eva: 2
bot> general	hola bot

$ raybot run --host mi-gateway --port 7470 --token TOKEN
```

Comandos de serie: `!ping` · `!count` (contador por canal en SQLite) ·
`!uptime` · `!echo` · `!help`.

## Arquitectura (la parte interesante)

Un solo canal de eventos multiplexa todo — `select` exige canales del mismo
tipo y no hay `try_recv` (hallazgo de rayrelay §64), así que:

- una fibra lectora empuja `In(gen, frame)`;
- la fibra de heartbeat empuja `Tick(gen)` cada intervalo;
- el bucle principal es el ÚNICO que escribe al socket, y descarta los ticks
  de generaciones muertas (las fibras de heartbeat huérfanas de reconexiones
  pasadas laten inofensivamente).

Reconexión probada de verdad: el test usa un gateway que **corta la conexión
tras cada dispatch** — el bot reconecta (gen 1→2→3), re-identifica con el
token, y el contador de `!count` sigue 1, 2, 3 a través de las caídas
(SQLite).

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Cliente websocket con HELLO/IDENTIFY/heartbeats/dispatches | ✅ |
| Reconexión con backoff + re-IDENTIFY (test con drops forzados) | ✅ |
| Comandos con estado por canal en SQLite | ✅ |
| Gateway falso (servidor `net/websocket`) para dev/test offline | ✅ |
| Binario nativo | ✅ |
| Tests (E2E con reconexión) | ✅ 1 |
| Adaptador Discord real (wss + REST para responder, intents) | 📋 v2 |
| Detección de socket muerto por ACK perdido (zombie connection) | 📋 v2 |

## Hallazgos de dogfood

Anotados en `raylang/IDEAS.md` §72:

1. **`websocket_client` + `net/websocket` (servidor) funcionan a la primera**
   en su estreno conjunto — handshake, framing enmascarado/no, ping/pong
   automático en `read_message`.
2. El patrón generación-en-el-canal es el workaround estándar para "no puedo
   matar una fibra dormida" (sin try_recv/select-timeout, §64): las fibras
   huérfanas laten a un canal que las ignora. Funciona, pero cada reconexión
   deja una fibra durmiente — un `Timer` cancelable o un select con timeout
   lo eliminaría.

## Desarrollo

```sh
ray test
ray run src/main.ray demo
ray build --native src/main.ray -o raybot --release
```

Estructura: `src/main.ray` (CLI) · `proto.ray` (frames JSON) · `client.ray`
(gateway client + reconexión) · `commands.ray` (comandos + SQLite) ·
`gateway.ray` (gateway falso).
