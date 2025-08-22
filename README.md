# 2i2c Hub Configurator

A self-contained, client-only web app to configure JupyterHub offerings across 2i2c’s **Essential**, **Advanced**, and **Premier** tiers. The entire application (HTML, CSS, JS) lives in a single **`index.html`** and dynamically loads a tier/options **catalog** at runtime.

---

## Highlights

* **Single-file deploy:** drop `index.html` on any static host (or open it locally).
* **External catalog:** loads options from a `catalog.json` that you manage.
* **Per-tier state:** choices persist independently for Essential/Advanced/Premier as you switch.
* **Sharable URLs:** the full configuration is encoded into the page URL (`?state=…`).
* **Machine-readable output:** `?json=true` returns **only the selected tier’s** choices as JSON, plus catalog metadata.
* **No build step:** vanilla JavaScript + JSON5; no server required.

---

## Quick start

1. **Save the app**
   Store the provided `index.html` where you can open it (locally or on a static host).

2. **Open it**

   * Local: double-click `index.html`.
   * Hosted: place `index.html` on any static site (S3, GitHub Pages, Netlify, etc.).

3. **Configure**

   * Pick a **Tier** using the buttons at the top.
   * Adjust options in the **two-column** layout (left: description; right: control):

     * `single` → fixed, read-only value
     * `flag` → informational “Included / Not included” badge
     * `select` → dropdown with allowed choices
   * Switch tiers freely—each tier’s selections remain intact.

4. **Share**
   The app continuously encodes state into the URL. Copy the **Sharable link** shown in the UI; opening it restores the exact configuration.

---

## URL parameters

* `state=<base64url>`
  The configuration (current tier + all tiers’ selections) encoded as base64url JSON. The app updates this automatically as you edit.

* `catalog=<url>`
  Override the default catalog location. Use a URL-encoded absolute URL.
  Example:

  ```
  index.html?catalog=https%3A%2F%2Fexample.org%2Fmy-catalog.json
  ```

* `json=true`
  Emit **only the active tier’s** selections as JSON, plus `tier` and catalog metadata.
  Examples:

  * Defaults with the default catalog:
    `index.html?json=true`
  * With a saved state:
    `index.html?state=...&json=true`
  * With a custom catalog:
    `index.html?catalog=...&state=...&json=true`

> **Note on `json=true`**
> Implemented client-side by generating a JSON **Blob** and navigating to it, so browsers see proper `Content-Type: application/json`. Tools that don’t execute JavaScript (e.g., calling the HTML directly via `curl`) won’t get the JSON without a JS runtime. For server-level JSON responses, place the app behind a minimal endpoint that returns the same payload when `json=true` is present.

---

## Catalog format (required)

The app accepts **only** the following shape:

```json5
{
  "name": "2i2c Hub Catalog",
  "version": "2025.08.22",
  "items": [
    {
      "id": "support",
      "group": "Support",
      "label": "Support level",
      "help": "Access to 2i2c engineers",
      "perTier": {
        "Essential": { "kind": "single", "text": "Community support" },
        "Advanced":  { "kind": "select", "options": ["Business hours", "24/7"], "default": "Business hours" },
        "Premier":   { "kind": "select", "options": ["Business hours", "24/7"], "default": "24/7" }
      }
    }
    // …more items…
  ]
}
```

### Item schema

Each item must have:

* `id` (string, unique)
* `group` (string; used as a visual section)
* `label` (string; UI title)
* `help` (string; optional helper text)
* `perTier` (object keyed by `"Essential" | "Advanced" | "Premier"`), each value one of:

#### `single`

```json
{ "kind": "single", "text": "Fixed value for this tier" }
```

#### `flag`

```json
{ "kind": "flag", "available": true }   // or false
```

#### `select`

```json
{
  "kind": "select",
  "options": ["optA", "optB", "optC"],
  "default": "optB"   // optional; defaults to first option
}
```

> The catalog is parsed with **JSON5**, so comments and trailing commas are allowed.

---

## State model

* The app holds a `selections` map **per tier**:

  ```json
  {
    "Essential": { "<item.id>": "<value>|true|false|null", ... },
    "Advanced":  { ... },
    "Premier":   { ... }
  }
  ```
* Switching tiers doesn’t erase choices.
* On first load (no `state`), each tier is seeded from catalog defaults.
* The URL updates on every change via `history.replaceState`.

---

## JSON output (`?json=true`)

When present, the app returns a JSON payload with **only the active tier**:

```json
{
  "catalog": { "name": "2i2c Hub Catalog", "version": "2025.08.22" },
  "tier": "Advanced",
  "selections": {
    "support": "Business hours",
    "someFlag": true,
    "someSingle": "Fixed text"
  }
}
```

---

## Theming & accessibility

* Styles draw from 2i2c’s visual language (blue/purple hues), with **light/dark** via `prefers-color-scheme`.
* Two-column layout (description → control) for scanability.
* Native controls for accessibility and keyboard use:

  * Tier tabs are buttons with ARIA roles.
  * Dropdowns are standard `<select>` elements.

---

## Hosting notes

* Any static host works.
* If loading the catalog from another origin, ensure it serves `Access-Control-Allow-Origin: *` (GitHub Raw does).
* For offline scenarios, host the catalog alongside `index.html` and reference it with `?catalog=./catalog.json`.

---

## Development

* No build step; edit `index.html` directly.
* Extension points:

  * **UI:** tweak CSS variables at the top for brand alignment.
  * **Kinds:** if you add new `kind` types in the catalog, extend the renderer in the script (`renderGrid`).
  * **Export:** to add a “Download JSON” button (instead of `?json=true`), reuse the same payload and trigger a blob download.

---

## Troubleshooting

* **No JSON in `curl`:** `curl` won’t execute JS. Use a headless browser (Playwright/Puppeteer) or add a small server endpoint for `?json=true`.
* **Catalog won’t load:** check the `?catalog=` URL (must be URL-encoded) and CORS headers. Open the browser console for errors.
* **Choices reset on reload:** use the **Sharable link**; it embeds the state via `?state=`.

---

## License & attribution

Use and adapt freely. Keep your catalog values, descriptions, and defaults aligned with your organization’s current product definitions.
