# Runbook — Ingesta de esports (Leaguepedia)

Esta es la única forma soportada de poblar la base de datos con equipos, jugadores y partidos reales. Sin esto, la pestaña **Esports** muestra el empty state "Sin datos ingeridos".

Origen: Leaguepedia (Cargo API, CC-BY-SA, sin auth). Función Edge: `supabase/functions/ingest-leaguepedia/`.

## Requisitos

- Supabase CLI autenticado (`supabase login`)
- Variables ya seteadas en el proyecto: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`
- Función desplegada **con el código más reciente del repo**:

  ```bash
  supabase functions deploy ingest-leaguepedia
  ```

  Importante: si recibes `400 Unknown scope: all` desde el dashboard o el CLI, casi seguro la versión desplegada es anterior al commit que añadió ese case. Vuelve a desplegar y reintenta. Los scopes válidos en el código actual son: `teams`, `players`, `tournaments`, `matches`, `stats`, `team-stats` y `all`.

## Comando recomendado para arrancar (LCK 2026)

Pobla teams + players + tournaments de la liga coreana del año en curso. Es la liga que GRA-71 acordó como mínimo viable.

```bash
supabase functions invoke ingest-leaguepedia \
  --body '{"scope":"all","region":"LCK","season":2026}'
```

Salida esperada:

```json
{
  "ok": true,
  "scope": "all",
  "result": {
    "teams":       { "upserted": 10 },
    "players":     { "upserted": ~50 },
    "tournaments": { "upserted": 2 },
    "note": "Match and stats ingestion runs separately per tournament/match to stay within Edge Function timeout."
  }
}
```

## Pasos siguientes (opcionales)

Una vez tengas las regiones base, puedes seguir profundizando:

```bash
# Partidos programados/jugados de un torneo concreto (external_id = OverviewPage en Leaguepedia)
supabase functions invoke ingest-leaguepedia \
  --body '{"scope":"matches","tournament":"LCK/2026 Season/Spring Season"}'

# Stats de jugador por partida (requiere que el match ya esté ingerido)
supabase functions invoke ingest-leaguepedia \
  --body '{"scope":"stats","match":"LCK 2026 Spring Week 1"}'

# Stats por equipo (objetivos, bans) por partida
supabase functions invoke ingest-leaguepedia \
  --body '{"scope":"team-stats","match":"LCK 2026 Spring Week 1"}'
```

## Más ligas

Mismo comando, distinta `region`. La función traduce el código corto a los valores reales que usa Leaguepedia (LCK → "Korea" para `Teams.Region`, etc.) — ver tabla `LP_COUNTRY` en `supabase/functions/ingest-leaguepedia/index.ts`. Códigos soportados: `LCK`, `LPL`, `LEC`, `LCS`, `LLA`, `CBLOL`, `PCS`, `VCS`, `LJL`, `TCL`.

```bash
supabase functions invoke ingest-leaguepedia \
  --body '{"scope":"all","region":"LEC","season":2026}'
```

Si una región devuelve `upserted: 0` en `teams` o `players` aunque la liga esté activa, confirma el valor que Leaguepedia usa para esa región con una query rápida (sustituye `T1` por un equipo conocido de la liga):

```bash
curl 'https://lol.fandom.com/api.php?action=cargoquery&format=json&tables=Teams&fields=Teams.Name,Teams.Region&where=Teams.Short%3D%22T1%22&limit=1'
```

Y ajusta `LP_COUNTRY` si el valor que ves no coincide con el del mapeo.

## Datadragon (catálogo de campeones + parches)

Separado, ya cubierto por la función `ingest-datadragon`. La pestaña **Campeones** y **Parches** dependen de esta:

```bash
supabase functions invoke ingest-datadragon \
  --body '{"scope":"snapshot"}'

# y luego, para generar patch_changes contra el parche anterior:
supabase functions invoke ingest-datadragon \
  --body '{"scope":"diff"}'
```

## Cuándo correr

- **Una vez** para arrancar (LCK 2026).
- **Diario** durante temporadas activas para refrescar matches y stats.
- Cada **nuevo parche** (típicamente cada 2 semanas) para `ingest-datadragon`.

El proyecto aún no tiene cron de Supabase configurado — por ahora se corre a mano. La función `ingest-leaguepedia` añade un delay de 1 s entre páginas para respetar a Leaguepedia (wiki voluntaria); si aun así pegas su rate limit recibirás un error tipo `Leaguepedia API error [ratelimited]: ...`. Espera 1–2 minutos y reintenta el mismo scope.

## Fallback: ingesta desde tu IP local (cuando Supabase está IP-baneada)

Las Edge Functions de Supabase comparten egress IP entre todos los tenants. Si otros han abusado de la API de Fandom, esa IP queda en su blocklist y nuestra función recibe `ratelimited` aunque solo hagamos 3 requests. Cuando eso pasa, **no es problema del código**: es la IP. Solución: corre la ingesta desde tu máquina, que casi siempre tiene IP limpia.

Hay un script ya escrito para esto:

```bash
# 1. Exporta las credenciales del proyecto (las mismas que usa la Edge Function):
export SUPABASE_URL='https://<ref>.supabase.co'
export SUPABASE_SERVICE_ROLE_KEY='<service_role_key>'

# 2a. Atajo retrocompat — teams + players + tournaments para una región/año:
node scripts/ingest-leaguepedia-local.mjs LCK 2026

# 2b. Pipeline completo de una región: teams + players + tournaments + matches
#     + stats (con MVPs) + team-stats (objetivos + bans). Más lento (varios
#     minutos por región dependiendo de cuántos torneos haya) pero deja la
#     región lista de punta a punta:
node scripts/ingest-leaguepedia-local.mjs --scope full --region LCK --season 2026

# 2c. Multi-región en una sola corrida:
node scripts/ingest-leaguepedia-local.mjs --scope full --regions LCK,LPL,LEC,LCS --season 2026
node scripts/ingest-leaguepedia-local.mjs --scope full --regions all-majors      --season 2026
node scripts/ingest-leaguepedia-local.mjs --scope full --regions all             --season 2026
```

`all` = `LCK,LPL,LEC,LCS,LLA,CBLOL,LJL,TCL,PCS,VCS`. `all-majors` = `LCK,LPL,LEC,LCS`.

Scopes soportados (mismos que la Edge Function, más `full`):

| scope         | qué hace                                                                                              |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| `teams`       | upsert equipos por región                                                                             |
| `players`     | upsert jugadores por región (resuelve `team_id`)                                                      |
| `tournaments` | upsert torneos por región/año                                                                         |
| `matches`     | calendario de partidos — itera todos los torneos de la región/año, o uno solo si pasas `--tournament` |
| `stats`       | game stats de jugador + MVPs — itera todos los matches de los torneos seleccionados                   |
| `team-stats`  | objetivos por equipo (drakes, barons, torres) + bans — itera todos los matches                        |
| `all`         | teams + players + tournaments (equivalente al `scope=all` del Edge)                                   |
| `full`        | encadena todo el pipeline — teams → players → tournaments → matches → stats → team-stats              |

Flags adicionales:

- `--region LCK` — una sola región.
- `--regions LCK,LPL,...` — lista separada por comas, o las palabras clave `all` / `all-majors`.
- `--season 2026` — año. Default 2026.
- `--tournament "LCK/2026 Season"` — para `matches`/`stats`/`team-stats`, restringe a un torneo (acepta prefijo).

Idempotencia: todos los upserts usan `onConflict`, así que reanudar tras un rate-limit no duplica filas — vuelves a correr el mismo comando y continúa desde donde quedó.

**Importante:** la `SUPABASE_SERVICE_ROLE_KEY` da acceso completo a la base. No la commitees, no la pongas en `.env` versionado, no la pegues en chats — exportala en la shell justo antes de correr el script y desexporta después.

## Fallback total: Oracle's Elixir (cuando ambas IPs están baneadas)

Cuando la IP residencial **también** está en la blocklist de Fandom (raro pero pasa, p. ej. tras varios reintentos rápidos), el script local de Leaguepedia tampoco funciona. Ahí va este fallback: Oracle's Elixir publica un CSV oficial con TODA partida de pro play desde 2014. No depende de Leaguepedia ni de la Riot API — es un dump anual hosteado en Google Drive (CC-BY, citar la fuente).

**Cobertura**: equipos, jugadores, calendario, KDA por jugador, drakes/barons/torres por equipo, bans, oro, daño, CS. Suficiente para llenar las pantallas de **región / torneo / equipo / jugador** con datos reales.

**No cobertura**: stats agregadas de soloQ (eso sigue dependiendo de la Riot prod key).

### Pasos

1. **Descarga el CSV (1 clic)**

   - Abre [oracleselixir.com/tools/downloads](https://oracleselixir.com/tools/downloads).
   - Click en "Match Data Download" → abre un folder de Google Drive.
   - Descarga el archivo del año actual (p. ej. `2026_LoL_esports_match_data_from_OraclesElixir.csv`).
   - Guárdalo como `data/oracleselixir-2026.csv` en el repo. `data/` está gitignored.

2. **Verifica el parseo en seco** (no escribe nada):

   ```bash
   node scripts/ingest-oraclesElixir.mjs --csv data/oracleselixir-2026.csv --dry-run
   ```

   Salida esperada: conteos de teams / players / tournaments / matches / game_stats. Si el CSV está mal o le falta una columna esperada, aquí se ve antes de tocar la DB.

3. **Ingiere a Supabase**:

   ```bash
   export SUPABASE_URL='https://<ref>.supabase.co'
   export SUPABASE_SERVICE_ROLE_KEY='<service_role_key>'

   # Todas las ligas mapeadas (LCK, LPL, LEC, LCS, LLA, CBLOL, LJL, TCL, PCS, VCS):
   node scripts/ingest-oraclesElixir.mjs --csv data/oracleselixir-2026.csv

   # Filtrar por subset:
   node scripts/ingest-oraclesElixir.mjs --csv data/oracleselixir-2026.csv --regions LCK,LPL,LEC,LCS
   node scripts/ingest-oraclesElixir.mjs --csv data/oracleselixir-2026.csv --regions all-majors
   ```

   El año se infiere del nombre del archivo (`2026_...csv`). Para forzarlo: `--year 2026`.

### Detalles del mapeo

- `tournaments.external_id` se construye como `OE:<region>:<year>:<split>[:Playoffs]`. No choca con los external_ids de Leaguepedia (que son OverviewPage strings).
- `matches.external_id` = `gameid` de OE (string opaco tipo `ESPORTSTMNT01_2690210`). Tampoco choca.
- `players.team_id` se deja `null` en este import porque un jugador puede cambiar de equipo durante el año en el dump; el `team_id` por partida sí está en `game_stats`, que es donde el frontend lo necesita.
- `matches.best_of` se setea siempre a `1`: OE no expone la serie, sólo el game individual. No rompe nada — la UI muestra resultados por game.

### Idempotencia y mezclado con Leaguepedia

Todos los upserts usan `onConflict`. Puedes re-correr el script con un CSV actualizado y solo se actualizan filas existentes / se insertan las nuevas. Si en el futuro la Edge Function de Leaguepedia se desbanea y la corres, los nuevos `external_id` no chocan con los de OE — las dos fuentes coexisten en las mismas tablas sin duplicar.

## Por qué no lo corre el agente

Las funciones Edge necesitan `SUPABASE_SERVICE_ROLE_KEY`, que es una credencial del proyecto. Los agentes no tienen acceso a secretos del workspace — esta acción la hace el owner / Erwin desde su entorno autenticado.
