# Runbook — Alpha MSI 2026 (operación diaria)

Este runbook lo opera el owner durante el MSI 2026. Cada mañana, después de la jornada del día anterior, hay que:

1. Bajar el CSV nuevo de **Oracle's Elixir** (OE).
2. Ingestar las partidas a Supabase.
3. Recalcular los puntos fantasy de cada liga.

Frecuencia: **una vez al día** mientras dure el torneo (2026-06-28 → final del MSI).

Tiempo total estimado: **5–10 min**.

---

## Pre-requisitos (una sola vez)

- Node 20+ y `npm install` corrido en el repo.
- Variables de entorno locales (no en git):
  ```
  SUPABASE_URL=https://<project>.supabase.co
  SUPABASE_SERVICE_ROLE_KEY=<service-role-key>
  ```
  Exportarlas en la sesión donde corras los comandos:
  ```powershell
  $env:SUPABASE_URL = "https://<project>.supabase.co"
  $env:SUPABASE_SERVICE_ROLE_KEY = "<service-role-key>"
  ```
- Carpeta `data/` existe en la raíz del repo (está gitignored).

---

## 1. Bajar el CSV de Oracle's Elixir

OE publica un dump CSV con TODA partida de pro play (incluido MSI 2026). El CSV se hostea en Google Drive y **no** se puede descargar con `curl` (Drive requiere auth interactiva). Bajada manual de 1 click:

1. Abre <https://oracleselixir.com/tools/downloads>.
2. Click en **Match Data Download** — abre un folder de Google Drive.
3. Baja el `.csv` del año en curso (ej. `2026_LoL_esports_match_data_from_OraclesElixir.csv`).
4. Guárdalo en `data/oracleselixir-2026.csv`.

> **Tip**: el CSV típicamente se actualiza el **día siguiente** de cada jornada. Si todavía no incluye los partidos del día anterior, vuelve a chequear más tarde.

---

## 2. Ingestar a Supabase

```bash
node scripts/ingest-oraclesElixir.mjs --csv data/oracleselixir-2026.csv --regions all --year 2026
```

Filtros opcionales (default: todas las ligas mapeadas):

- `--regions LCK,LPL,LEC` — solo esas ligas.
- `--regions all-majors` — solo LCK/LPL/LEC/LCS.
- `--dry-run` — parsea, agrega, imprime totales. **No escribe**. Útil para validar el CSV.

Salida esperada:

```
✓ ingestados N partidos, M jugadores actualizados.
```

Si algo falla:

- Verifica que `$env:SUPABASE_SERVICE_ROLE_KEY` esté seteada.
- Verifica que el CSV existe y tiene contenido.
- Vuelve a correr — el script es idempotente (upsert por external_id).

---

## 3. Recalcular puntos fantasy

Por cada liga fantasy activa, llama a la Edge Function de scoring:

```bash
curl -X POST $SUPABASE_URL/functions/v1/fantasy-scoring/recompute \
  -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"league_id":"<league-uuid>"}'
```

Respuesta esperada:

```json
{
  "ok": true,
  "league_id": "...",
  "starters": 25,
  "fantasy_players": 23,
  "game_scores_written": 115,
  "roster_scores_written": 5
}
```

Para obtener el `league_id` de cada liga:

```bash
curl "$SUPABASE_URL/functions/v1/fantasy-leagues/_mine" \
  -H "Authorization: Bearer <tu-jwt>"
```

o entra a la app: el id está en la URL de cada liga (`/fantasy/<league-uuid>`).

El recompute es **idempotente** — puedes correrlo varias veces el mismo día sin duplicar puntos. Sobrescribe por `(fantasy_player_id, game_stats_id)`.

---

## 4. Verificar en la app

1. Abre la app (Expo Go o web).
2. Login con tu cuenta.
3. Entra al tab **Fantasy** → tu liga.
4. Abre tu plantilla y revisa que los puntos por jornada se hayan actualizado.

> **Nota alpha**: la pantalla muestra el valor de mercado de cada titular y el placeholder de jornada. Cuando tengamos la UI extendida para leer `fantasy_roster_scores`, las cifras reales aparecerán ahí mismo.

---

## Smoke test pre-MSI (correr una sola vez antes del 2026-06-28)

Para garantizar que la cadena completa funciona antes del primer día real del torneo:

1. Baja el CSV de OE actual (puede ser de cualquier torneo reciente, ej. LEC Summer 2026).
2. Corre el `ingest-oraclesElixir.mjs` con `--dry-run` primero — confirma que el CSV se parsea sin errores.
3. Corre la ingesta real con `--regions LEC --year 2026`.
4. Crea una liga de prueba en la app con 2 cuentas, fichen sus plantillas con jugadores LEC.
5. Llama al endpoint de recompute.
6. Verifica en la app que las plantillas muestran sus puntos.

Si todo pasa, el sistema está listo para el primer día del MSI.

---

## Troubleshooting

| Síntoma                                      | Causa probable                                     | Solución                                                                              |
| -------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `Missing SUPABASE_URL ...`                   | env vars no seteadas                               | `$env:VAR = "..."`                                                                    |
| `0 game_scores_written`                      | ningún starter del league tiene games ingeridos    | ¿el CSV cubre la región/jornada correcta?                                             |
| `League not found`                           | UUID equivocado                                    | confirma con `/fantasy-leagues/_mine`                                                 |
| `OE CSV publica tarde`                       | esperado — OE típicamente publica al día siguiente | avisa a los amigos: scoring del día se ve al día siguiente                            |
| Errores de red en `ingest-oraclesElixir.mjs` | rate limit de Supabase                             | esperar 30s, reintentar; el script reintenta automaticamente algunos transient errors |

---

## Contacto

Cualquier bug en el flujo: tagear a Levi en el issue GRA-92 (integration test + bug fixes).

Cualquier duda de scope o decisiones: tagear a Erwin.

Cualquier cambio de scope o aprobaciones de producto: chat con el owner.
