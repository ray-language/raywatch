# raywatch

Monitor de servicios con **dashboard en vivo**, escrito en [raylang](https://github.com/roberto-ayala/raylang): checks programados (HTTP, TCP, TLS, Redis, DNS) — una fibra por check —, historia en SQLite, panel web con **SSE** (la tabla se actualiza sola), uptime %, y alertas por **webhook** cuando un servicio cambia de estado (no en cada fallo).

```text
$ raywatch --config raywatch.toml
raywatch: 5 check(s), dashboard on http://127.0.0.1:8090/

$ curl -s localhost:8090/api/status
[{"detail":"HTTP 200","latency_ms":42,"name":"web","ok":true,"since_ms":…,"uptime_pct":100}]

$ curl -N localhost:8090/events        # SSE: un evento JSON por muestra
data: {"detail":"connected","latency_ms":1,"name":"ssh","ok":true,…}
```

## Config (tablas con nombre, como raygate)

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
type = "dns"                   # ¡ojo!: UDP bloqueante, ver hallazgos
target = "example.com@8.8.8.8:53"
```

## Cómo está hecho

- **Una fibra por check** durmiendo su intervalo (el experimento "¿N fibras
  con sleep escala?" del catálogo — sí, sin drama).
- **Un actor recorder** dueño de todo: inserta en SQLite, mantiene el estado
  vigente + uptime, hace fan-out a los suscriptores SSE (cada `/events` es un
  `stream_response` con su canal), y dispara el webhook en fibra aparte al
  CAMBIAR el estado.
- Dashboard: una página HTML con `EventSource` — cero build, cero deps.
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
| Tests (E2E completo con webhook sink) | ✅ 1 |
| Días-para-expirar del certificado TLS | ❌ bloqueado (ver hallazgos) |
| Gráficas de latencia histórica, agrupación, silencios | 📋 v2 |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Anotados en `raylang/IDEAS.md` §70:

1. **No se puede leer el certificado del peer** — la predicción del catálogo:
   `tls_connect` devuelve solo el handle. El handshake de rustls ya valida
   cadena y fechas (por eso el check `tls` sirve), pero "expira en N días" —
   EL check de TLS que todo el mundo quiere — es inexpresable. Superficie
   candidata: `net.tls_peer_cert(h) -> {not_after, san, issuer}`.
2. **`net/dns` hereda el UDP bloqueante** (§64): una consulta congela TODAS
   las fibras del monitor mientras espera, y sin timeout de `recv_from` un
   paquete perdido cuelga el proceso entero. Para un monitor es la diferencia
   entre "check lento" y "monitor muerto".
3. `tcp_connect` no tiene timeout de conexión (el del SO, ~75 s): un host que
   descarta SYNs bloquea esa fibra mucho más que su `timeout_ms`.
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
