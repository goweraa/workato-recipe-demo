# Build steps — Daily FX-rate pipeline

Built in the Workato recipe editor using three standard connectors (Scheduler, HTTP,
Google Sheets) and one public API that needs no key.

## 0. Prep (once)

- In Google Sheets, create a spreadsheet `fx_rates` with:
  - a `rates` tab, header row: `date | usd_eur | usd_gbp | usd_pln`
  - an `errors` tab, header row: `date | reason | raw_response`
- In Workato → **Connections**, add:
  - an **HTTP** connection: Authentication `None`, Base URL `https://api.frankfurter.app`
  - a **Google Sheets** connection (OAuth — sign in with the Google account that owns the sheet)

## 1. Trigger — Scheduler

- Trigger: **Scheduler by Workato → New scheduled event**
- Time unit `Days`, trigger every `1`, at a set time (e.g. 09:00).

## 2. Action — HTTP GET the FX rates

- **HTTP → Send request**
- Method `GET`, Request URL `/latest?from=USD&to=EUR,GBP,PLN` (relative to the Base URL)
- Response content type `JSON`. Under **Response schema → Use JSON**, paste a sample so
  Workato generates the output fields:
  ```json
  {"amount":1,"base":"USD","date":"2026-08-30","rates":{"EUR":0.86,"GBP":0.74,"PLN":3.65}}
  ```
- Test the step — expect a `200` with the day's rates.

## 3. Data-quality check — IF condition

- Add an **IF condition** with three conditions joined by **AND**:
  - `Rates EUR` **is present**
  - `Rates GBP` **is present**
  - `Rates PLN` **is present**

### 3a. Matching branch — write the clean row
- **Google Sheets → Add row** to the `rates` tab:
  - `date` ← `Date` · `usd_eur` ← `Rates EUR` · `usd_gbp` ← `Rates GBP` · `usd_pln` ← `Rates PLN`

### 3b. ELSE branch — route the bad data
- After the IF, add an **ELSE** step, and inside it a **Google Sheets → Add row** to the
  `errors` tab:
  - `date` ← `Date` · `reason` ← text `Missing or invalid rate` · `raw_response` ← `Body`

## 4. Test both paths

- Full URL (`/latest?from=USD&to=EUR,GBP,PLN`) → row lands in `rates` only.
- Drop currencies (`/latest?from=USD&to=EUR`) → GBP/PLN missing → row lands in `errors` only.
- Restore the full URL.

## Bugs I hit (and fixed)

1. **`bad URI` on the HTTP step.** A stray leading space had crept into the connection's
   Base URL (`" https://…"`), making it an invalid URI. Retyping the Base URL cleanly
   fixed it.
2. **Recipe errored on missing fields.** With conditions written as `rate > 0`, a response
   that omitted GBP/PLN caused a *"missing field value"* error instead of routing to ELSE.
   Fixed by switching the conditions to **`is present`**, which handles absent fields
   gracefully.
3. **Errors tab always populated.** The errors row was first added *after* the IF (so it
   ran every time). Moving it **inside the ELSE branch** made it conditional. Also had to
   switch the condition joiner from **OR to AND** so all three rates are required.

## Notes

- Only free/standard connectors and a public, no-auth API — nothing proprietary, no
  secrets in the exported recipe.
- Possible next step: add a Workato **AI ("Ask AI")** step to summarise the day's rate
  movement — showcasing the "Applied Gen AI in Workato" skill directly.
