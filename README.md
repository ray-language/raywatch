# raywatch

Monitor de servicios con **dashboard en vivo**, escrito en [raylang](https://github.com/ray-language/raylang): checks programados (HTTP, TCP, TLS, Redis, DNS) — una fibra por check —, historia en SQLite, panel web con **SSE** (la tabla se actualiza sola), uptime %, y alertas por **webhook** cuando un servicio cambia de estado (no en cada fallo).

```text
$ raywatch --config raywatch.toml
raywatch: 5 check(s), dashboard on http://127.0.0.1:8090/

$ curl -s localhost:8090/api/status
[{"detail":"HTTP 200","latency_ms":42,"name":"web","ok":true,"since_ms":…,"uptime_pct":100}]

$ curl -N localhost:8090/events        # SSE: un evento JSON por muestra
data: {"detail":"connected","latency_ms":1,"name":"ssh","ok":true,…}
```

## Config (tablas con nombre o `[[check]]`, como raygate)

Cada check es una tabla con nombre (`[check.web]`) o una entrada de un array de
tablas `[[check]]` con una clave `name` opcional (si falta, `check<N>`); las dos
formas se pueden mezclar y los nombres no pueden repetirse.

```toml
[server]
port = 8090
db = "./raywatch.db"
webhook = "https://hooks.example.com/notify"   # POST JSON al cambiar de estado

[check.web]
type = "http"                  # http | tcp | tls | redis | dns
target = "https://example.com/"
interval_ms = 15000
timeout_ms = 5000
expect_status = 200

[check.tls]
type = "tls"                   # handshake rustls OK = cadena y fechas válidas
target = "example.com:443"

[check.dns]
type = "dns"                   # espera acotada a 5 s desde raylang M121
target = "example.com@8.8.8.8:53"
```

## Cómo está hecho

- **Una fibra por check** durmiendo su intervalo (el experimento "¿N fibras
  con sleep escala?" del catálogo — sí, sin drama).
- **Un actor recorder** dueño de todo: inserta en SQLite, mantiene el estado
  vigente + uptime, hace fan-out a los suscriptores SSE (cada `/events` es un
  `stream_response` con su canal), y dispara el webhook en fibra aparte al
  CAMBIAR el estado.
- Dashboard: una página HTML con `EventSource` — cero build, cero deps. Las
  celdas se rellenan con `textContent`: el detalle viene de los servicios
  sondeados y nunca se interpreta como HTML.
- Un suscriptor SSE que se fue o que no lee no frena al recorder: se le envía
  con `try_send`, y si el canal está cerrado o lleno se le descarta y se
  cierra su stream (el `EventSource` reconecta y recibe la foto actual).
- Verificado E2E: upstream vivo/muerto, snapshot JSON, página, 3 webhooks al
  primer estado, SQLite poblado; y en nativo con SSE en vivo por curl.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Checks http/tcp/tls/redis/dns con intervalo y timeout por check | ✅ |
| Historia en SQLite (`db/sqlite`) + uptime % | ✅ |
| Dashboard SSE en vivo + `/api/status` JSON | ✅ |
| Webhook JSON en cambios de estado | ✅ |
| Binario nativo (SSE y SQLite incluidos) | ✅ |
| Tests (E2E completo con webhook sink, config, recorder) | ✅ 3 |
| Días-para-expirar del certificado TLS | ✅ (raylang M124: `tls_peer_cert`) |
| Gráficas de latencia histórica, agrupación, silencios | 📋 v2 |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Anotados en `raylang/IDEAS.md` §70:

1. **[RESUELTO — raylang M124]** No se puede leer el certificado del peer:
   `net.tls_peer_cert(h)` existe y el check `tls` reporta "cert expires in N
   days (subject)".
2. **[RESUELTO — raylang M121]** `net/dns` hereda el UDP bloqueante: la nota
   resultó parcialmente rancia (la VM cede desde M20.11) y el hueco real —
   el timeout — está cerrado: la espera de respuesta se acota a 5 s.
3. **[RESUELTO — raylang M122]** `tcp_connect` sin timeout: el check `tcp`
   usa `tcp_connect_timeout(host, port, timeout_ms)`.
4. **Positivo**: fibra-por-check + actor + SSE + SQLite compone limpio y
   corre igual en VM y nativo; `stream_response` sirve SSE sin soporte
   dedicado; el webhook en fibra aparte no bloquea al recorder.

## Desarrollo

```sh
ray test
ray run src/main.ray --config raywatch.toml
ray build --native src/main.ray -o raywatch --release
```

Estructura: `src/main.ray` (CLI + scheduler) · `config.ray` (TOML) ·
`checks.ray` (sondas) · `recorder.ray` (actor: SQLite/SSE/webhook) ·
`web.ray` (dashboard).
