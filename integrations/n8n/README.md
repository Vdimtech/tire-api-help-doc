# Using the Tire Size API with n8n

These are ready-to-import [n8n](https://n8n.io) workflows that call the Tire Size API.
Import a `.json` file, bind your API key once as a credential, and run.

**Base URL:** `https://tire.vdim.app/api/v1`
**Authentication:** every request must send the header `x-api-key: <your-key>`.

| File | What it shows | Endpoint(s) |
|------|---------------|-------------|
| `01-cascading-vehicle-lookup.json` | Drill down Year → Make → Model → Trim → Tire Size | `GET /by_vehicle/model`, `/by_vehicle/trim`, `/by_vehicle/tiresize` |
| `02-tire-dimensions-lookup.json` | Get width / aspect ratio / diameter for one vehicle | `GET /tire_dimensions` |
| `03-reverse-lookup.json` | Given a tire size, find vehicles that use it | `GET /reverse_lookup` |
| `04-usage-check.json` | Check your plan, daily limit, and remaining calls; branch when quota is low | `GET /usage` |

---

## 1. Prerequisites — get an API key

Sign up for a free key at **https://account-tire.vdim.app/signin/signup**, then copy the
key from your dashboard.

> **Demo key caveat:** the public demo key works for testing but is capped at **10 rows per
> response** and **20 requests/hour, 100 requests/day per IP**. Use your own free key for
> complete results.

---

## 2. One-time credential setup (Header Auth)

The workflows authenticate with an n8n **Header Auth** credential so your key is never
stored in the workflow file.

1. In n8n, go to **Credentials → Add credential**.
2. Search for and select **Header Auth**.
3. Set:
   - **Name** (the header name): `x-api-key`
   - **Value**: your API key
4. Give the credential a label such as **Tire API Key** and **Save**.

You only do this once — all three workflows reuse the same credential.

---

## 3. Import a workflow

1. In n8n, open **Workflows → Import from File** (or the **⋯** menu → *Import from File*).
2. Select one of the `.json` files in this folder.
3. Open each **HTTP Request** node and, under **Credential for Header Auth**, choose the
   **Tire API Key** credential you created in step 2. (The files ship with a placeholder
   credential reference, so n8n will ask you to pick one on first run.)
4. Click **Execute Workflow**.

---

## 4. What each workflow does

### 01 – Cascading vehicle lookup
The **Set Inputs** node holds `year`, `make`, `model`, and `trim`. Each HTTP Request adds
one more filter, mirroring how a search UI fills dropdowns top-to-bottom:

- `GET /by_vehicle/model?year=&make=` → available models
- `GET /by_vehicle/trim?year=&make=&model=` → available trims
- `GET /by_vehicle/tiresize?year=&make=&model=&trim=` → tire sizes for that exact vehicle

**Response shape:** these endpoints return `{ "success": true, "data": { "1": {...}, "2": {...} }, "count": N }`.
The results are an **object keyed by row number**, not an array. To turn it into a list of
items in n8n, add an **Item Lists → "Object to items"** (or a Code node with
`Object.values($json.data)`) after the call.

To build true dynamic cascading (each dropdown driven by the previous response), replace the
static values in **Set Inputs** with expressions that read the chosen value from the prior
node, e.g. `={{ $json.data["1"].model }}`.

> **Values are exact, full strings.** Filters match the database value as-is (case-insensitive
> but not partial). A 2023 Camry's trims are `LE AWD`, `LE FWD`, `XLE AWD`, … — there is no
> bare `LE`. Always feed the `tiresize` call a trim that the **trim** call actually returned,
> rather than typing a guess. Querying a combination that doesn't exist returns no rows (see
> the error note below).

### 02 – Tire dimensions lookup
Single call to `GET /tire_dimensions?year=&make=&model=&trim=`. Edit the **Set Vehicle** node
to change the vehicle.

**Response shape:** `{ "success": true, "dimensions": [ { "width": "225", "aspectratio": "60", "diameter": "16", "tiresize": "225/60R16" } ], "metadata": {...} }`.
The **Split Out Dimensions** node expands `dimensions[]` into one item per tire size.
Optionally pass `&tiresize=225/60R16` to filter to a single size.

### 03 – Reverse lookup
Edit **Set Tire Size** (default `225/60R16`) and call `GET /reverse_lookup?tiresize=`. You can
also query by individual components instead: `width=225&aspectratio=60&diameter=16`.

**Response shape:** `{ "success": true, "vehicles": [ { "year": "...", "make": "...", "model": "...", "trim": "...", "tiresize": "..." } ], "count": N }`.
The **Split Out Vehicles** node expands `vehicles[]` into one item per matching vehicle.

### 04 – Usage / rate-limit check
Calls `GET /usage` and reports your plan and remaining quota. The flow is:
**Check Usage** → **Extract Usage Fields** (pulls `plan`, `dailyLimit`, `dailyUsed`,
`dailyRemaining`, `hourlyRemaining`, `usagePercentage`) → **Daily Remaining < 50?** (an IF node).

**Response shape:** `{ "success": true, "data": { "subscriptionType": "free", "dailyLimit": 300, "dailyUsed": 4, "dailyRemaining": 296, "hourlyRemaining": 99, "usagePercentage": 1 } }`.

The IF node has two branches:
- **Low quota** (`dailyRemaining < 50`) → wire this to an alert/stop (Slack, email, or **Stop And Error**).
- **Quota OK** → wire this to the rest of your workflow.

Use it as a **guard before a bulk run**: place this check first, and only proceed on the
"Quota OK" branch. Adjust the threshold (default `50`) in the IF node to suit your job size.
Plan limits: free = 300/day (100/hr), starter = 5,000/day, pro = 50,000/day, business = 80,000/day.

---

## 5. Build your own node against any endpoint

All endpoints follow the same pattern, so you can wire up any of them with one HTTP Request node:

| HTTP Request field | Value |
|--------------------|-------|
| **Method** | `GET` |
| **URL** | `https://tire.vdim.app/api/v1/<endpoint>` |
| **Authentication** | *Generic Credential Type* → *Header Auth* → your **Tire API Key** |
| **Send Query Parameters** | On — add the params the endpoint needs |

Other useful endpoints:

- `GET /search_allyear`, `GET /search_allmake`, `GET /search_allwidth` — full dropdown lists (cacheable ~1h).
- `GET /by_size/aspectratio?width=` and `GET /by_size/diameter?width=&aspectratio=` — size-based drill-down.
- `GET /year_range?make=[&model=]` — min/max model years available.
- `GET /usage` — your current plan, daily limit, and remaining requests.

---

## 6. Rate limits & error handling

Responses include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`
headers. Plan limits range from 300 requests/day (Free) through Starter (5,000),
Pro (50,000), and Business (80,000).

- **Check usage before bulk runs:** call `GET /usage` and read `data.dailyRemaining`.
- **Handle `429 Too Many Requests`:** in the HTTP Request node open **Settings** and enable
  **Retry On Fail** (with a delay), or branch on `{{ $json.statusCode }}` to pause the workflow.
- **`401 Unauthorized`** means the `x-api-key` header is missing or wrong — recheck the
  Header Auth credential name (`x-api-key`) and value.
- **`400 Bad Request`** lists the missing required query parameters in the response body.
- **`404 Not Found`** from a `by_vehicle/*` call means the filter combination matched no rows
  (e.g. a trim that doesn't exist for that model). Double-check each value against what the
  previous step returned — values must be the exact, full strings the API uses (`LE AWD`, not
  `LE`).

Need help integrating? Contact support via https://tire.vdim.app/contact.html.
