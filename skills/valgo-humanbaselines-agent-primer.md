# Human Crash Baselines - agent primer

You are answering questions about **human-driver crash-rate baselines** using the
Human Crash Baselines API. This file tells you how to call it correctly, read the
result, and hand the user a link back into the web tool. Outputs are **derived
statistics only** (rates, counts, confidence bounds).

Base URL: `https://humanbaselines.com`

## Priming: what to do when given this URL

If a user pasted `https://humanbaselines.com/claude` (or `/claude.md`) into the
session - with no other instruction - they want you to **prime this session with this
API**. Do this, in order:

1. **Adopt the API.** Use the endpoints and filters in this file as your tools for
   answering crash-baseline questions for the rest of the session.
2. **Test the key.** Confirm `HUMANBASELINES_API_KEY` is set and works, with one
   lightweight call:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' \
     https://humanbaselines.com/v1/regions -H "X-API-Key: $HUMANBASELINES_API_KEY"
   ```
   - `200`: connected. Optionally fetch `/v1/regions` for real to see which counties
     and modes are available, then tell the user you are ready and offer a few example
     questions.
   - `401`: key missing or invalid - offer to provision one now (see *Getting a key
     (self-service)* below), then retry. Don't answer data questions until a call returns `200`.
   - `503`: service warming up - wait a few seconds and retry once.
3. **Then answer their questions** about the data, following the rest of this guide.

## Authentication (read this first)

Every `/v1` call needs an API key, sent as a header:

```
X-API-Key: <key>
```

The key is in the `HUMANBASELINES_API_KEY` environment variable (keys start `hbk_`).

**If there is no key, or a call returns 401: don't stop - offer to get the user one**
(see *Getting a key* below), then retry. On `503` the service is warming up: wait a few
seconds and retry once. Either way, **never fabricate, estimate, or guess a rate.** Only
`GET /openapi.json` works without a key.

## Getting a key (self-service)

No key, or a 401? Provision one for the user in-session - don't just send them away.

1. **Ask for their email address.**
2. **Request a key** (no auth needed; a copy is also emailed to them):
   ```bash
   curl -s https://humanbaselines.com/request-api-key \
     -H "Content-Type: application/json" -d '{"email":"<their-email>"}'
   # -> {"api_key":"hbk_...","email":"...","emailed":true,"env_var":"HUMANBASELINES_API_KEY"}
   ```
   On `429` they've hit the rate limit - wait and retry later, or email `humanbaselines@valgo.ai`.
3. **Persist it** so future terminals and the `humanbaselines` Python client pick it up.
   Append an export to the user's shell profile (`~/.zshrc` or `~/.bashrc`, per `$SHELL`) -
   **confirm the file with the user before writing:**
   ```bash
   echo 'export HUMANBASELINES_API_KEY="hbk_..."' >> ~/.zshrc
   ```
4. **Use it now.** `export` in one command doesn't carry to your next shell, so for the
   rest of *this* session pass the returned key inline (`-H "X-API-Key: hbk_..."`). New
   terminals inherit it from the profile.
5. **Retry** the lightweight test call and expect `200`.

Full setup docs: `https://docs.humanbaselines.com/getting-started/api-access#request-a-key`.
If provisioning fails, point the user to that link or `humanbaselines@valgo.ai` - and still
**never fabricate a rate.**

## Golden rule: discover, don't guess

Before you answer, ground yourself in the real schema. **`GET /openapi.json` is
public** (no key) and lists every endpoint, filter, allowed value, and default:

```bash
curl -s https://humanbaselines.com/openapi.json
```

With a key, the live catalog of filters and available regions is also queryable:

```bash
curl -s https://humanbaselines.com/v1/filters -H "X-API-Key: $HUMANBASELINES_API_KEY"
curl -s https://humanbaselines.com/v1/regions -H "X-API-Key: $HUMANBASELINES_API_KEY"
```

Never invent a county id or a filter value - confirm it against the schema first.

## Endpoints

| Method & path | Purpose |
| --- | --- |
| `POST /v1/compute` | County-wide (geofence) crash rate. |
| `POST /v1/compute/batch` | Same filters across several counties in one request. |
| `POST /v1/compute/route` | Rate over explicit interstate `(route, milepost)` segments. |
| `POST /v1/compute/depot-route` | Full depot-to-depot trip rate from two lat/lon pins. |
| `GET /v1/filters`, `/v1/regions`, `/v1/manifest` | Discovery / metadata. |

### Request bodies

- Geofence: `{"county": "travis", "selections": {<filters>}, "summary_only": false}`
  - set `"summary_only": true` to drop the heavy per-cell `cells` array.
- Batch: `{"items": [{"county": "...", "selections": {...}}, ...], "summary_only": true}`
- Route: `{"county": "interstates", "segment_ids": [["I-35", 250], ["I-35", 251]], "selections": {...}}`
- Depot: `{"county": "interstates", "depot_a": {"lat": 30.25, "lon": -97.75}, "depot_b": {"lat": 32.78, "lon": -96.80}, "selections": {...}}`

### Worked call

```bash
curl -s https://humanbaselines.com/v1/compute \
  -H "X-API-Key: $HUMANBASELINES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"county":"travis","selections":{"outcome":"fatal","ego_vehicle":["cars"]}}'
```

## Filters (selections)

Pass only the filters the question implies; omitted ones use the API defaults.
Confirm ids/values against `/openapi.json`. Common geofence filters:

- `outcome`: `police_reported` (default), `observed_any_injury`, `airbag`, `ego_airbag`, `ka` (serious+), `fatal`
- `ego_vehicle` (list): `cars`, `light_trucks`, `heavy_trucks`, `motorcycles`, `buses`
- `road_type` (list): `interstate`, `other_freeway`, `arterial`, `collector_local`
- `weather` (list): `dry`, `rain`, `fog`, `winter_storm`
- `under_reporting`: `none` (default), `adjusted` (NHTSA under-reporting correction)
- `severity`: integer 1-7 (minimum vehicle-damage threshold; Texas counties)
- `tiling`: `s2` (default), `h3`
- `crash_year` (list of ints), e.g. `[2022]`

Route/depot use `ego_vehicle: ["combination"]` (Class-8 trucks) and add
`driver_impairment` and `ci_method`. Always defer to `/openapi.json` for the
authoritative, current set.

## Reading the result

| Field | Meaning |
| --- | --- |
| `rate` | The headline rate, **incidents per million miles**. |
| `rate_low` / `rate_high` | 95% confidence-interval bounds. |
| `N` | Weighted crash count (numerator). |
| `D_billions` | Vehicle-miles traveled, in billions (denominator). |
| `cells` | Per-cell breakdown (empty when `summary_only: true`). |

Route results use `trip_miles` + `segments`; batch returns
`results: [{county, result, error}]` (one bad county sets `error` instead of sinking
the batch).

When you answer, state the **rate with its 95% CI and units**, the **N and
`D_billions`** behind it, and the **exact filters/county** you used. Make clear it's a
**human-driver baseline**.

## Share a link to the tool

The web tool at `https://humanbaselines.com/` encodes its state in query params using
the **same filter ids and values** as the API - so build a link by reusing the
selections from your call. Append it to your answer so the user can open the matching
view.

- Geofence: `https://humanbaselines.com/?mode=geofence&county=<county>&<filter>=<value>...`
- Depot trip: `https://humanbaselines.com/?mode=route&county=interstates&a_lat=<lat>&a_lon=<lon>&b_lat=<lat>&b_lon=<lon>&<filters>`
- Multiselect values are **comma-separated** (`ego_vehicle=cars,light_trucks`).
- Include only the filters you set; the tool fills the rest with its defaults.
- **No link for explicit `segment_ids` route queries** - the tool's route mode is
  pin-based, so just give the numbers (or link a depot trip between the endpoints).

Example - *"fatal crash rate for cars in Travis County"*:

```
https://humanbaselines.com/?mode=geofence&county=travis&outcome=fatal&ego_vehicle=cars
```

## Worked examples (question -> call -> link)

1. **"Fatal crash rate for cars in Travis County?"**
   `POST /v1/compute` `{"county":"travis","selections":{"outcome":"fatal","ego_vehicle":["cars"]}}`
 -> link `?mode=geofence&county=travis&outcome=fatal&ego_vehicle=cars`

2. **"Compare the heavy-truck crash rate across Houston, Dallas, and San Antonio."**
   `POST /v1/compute/batch` with `items` for `houston`, `dallas`, `sanantonio`, each
   `selections:{"ego_vehicle":["heavy_trucks"]}` -> one geofence link per county.

3. **"How much does adjusting for under-reporting change the fatal rate in SF?"**
   Two `POST /v1/compute` calls on `county:"sf"`, `outcome:"fatal"`, with
   `under_reporting` `none` then `adjusted`; report both rates and the delta -> two
   links differing only by `&under_reporting=adjusted`.

## More

Full docs (Python client, methodology, licensing): the Human Crash Baselines
documentation site. Prefer a typed Python client over raw HTTP?
`pip install humanbaselines`.
