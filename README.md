# PublicDataCleanup

PublicDataCleanup is a static, privacy-first web app for collecting field notes, formatting alert text, and exporting summaries.

## Tech Stack

- Vanilla HTML/CSS/JavaScript (`index.html`, `styles.css`, `app.js`)
- Leaflet (`1.9.4`) for map rendering
- Luxon (`3.4.4`) for date/time handling
- Driver.js (`1.3.6`) for guided tutorials
- js-yaml (`4.1.0`) for human-readable tutorial configuration files

## Local Development

1. Install dependencies:

```bash
npm install
```

2. Start local server:

```bash
npm run dev
```

3. Open `http://127.0.0.1:8080`.

## App Structure

- `index.html`: app layout, modals, script/style includes
- `styles.css`: core styling, responsive rules, modal styles, tutorial styles
- `app.js`: app state, rendering, event handling, import/export, map interactions, tutorial orchestration
- `config.yml`: deployment-level defaults and internal app configuration
- `config.generated.js`: pre-baked browser config loaded at runtime (`window.__PDC_CONFIG__`)
- `config.example.yml`: editable template for new deployments
- `config.schema.json`: schema for validating `config.yml`
- `tours/`: tutorial manifest + YAML tour definitions
- `sessions/`: optional JSON session examples used by tutorials

## Deployment Config

This app now reads deployment config from a baked browser object (`window.__PDC_CONFIG__` in `config.generated.js`).
`app.js` includes fallback defaults for safety, except map/location features, which are disabled when geocoding API config is missing.

### Configure

1. Edit `config.yml` (or start from `config.example.yml`).
2. Bake it into `config.generated.js` in your hosting pipeline.
3. Serve `config.generated.js` before `app.js` (already wired in `index.html`).

### Validation Guidance

- Validate `config.yml` against `config.schema.json` in your CI/CD pipeline before baking.
- If you use `yq` + `ajv`, a typical flow is:
  1. Convert YAML to JSON.
  2. Validate JSON against `config.schema.json`.
  3. Fail deployment on schema errors.

### Secrets Guidance

- Do not put private API keys in frontend config. This app is static and client-visible.
- Use a backend/proxy for secret-bearing integrations, then expose only safe public endpoints in `config.yml` (for example `apis.geocoding.baseUrl`).

## Tutorial System

Tutorials are loaded from YAML files in `tours/` and listed through `tours/index.json`.

### Why a Manifest File Exists

The app is static and browser file-system listing is not reliable across hosts. To make tutorial discovery predictable, the app reads `tours/index.json`.

### `tours/index.json` Format

Use either strings or objects:

```json
[
  "1_welcome-basics.yaml",
  { "file": "2_dispatch-workflow.yaml", "label": "Dispatch Workflow Tour" }
]
```

Rules:

- `file` must be a relative `.yaml` path.
- If `file` does not start with `tours/`, the app prepends it.
- `label` is optional; fallback is filename-based.
- Filename order convention: prefix tour files with `N_` (example: `1_intro.yaml`, `2_dispatch.yaml`).
- Tour picker order uses that numeric prefix first, then label.

### Tour YAML Schema

Each tour YAML file supports:

- `example_file`: `null` (default) or a relative `.json` path
- `welcome`: text shown in tutorial welcome modal
- `completion`: text shown in tutorial completion modal
- `stops`: ordered list of class-based tutorial targets

Example:

```yaml
example_file: "sessions/tutorial-dispatch-baseline.json"
welcome: "This walkthrough introduces Dispatch workflow controls."
completion: "Tour complete. Try creating a card and exporting a summary."
stops:
  - class: "dispatch-actions"
    heading: "Session Controls"
    detail: "Import or clear sessions here."
    try_this: "Click Import to open the file picker, then cancel."
  - class: "shift-controls"
    heading: "Shift Setup"
    detail: "Set area, time, and view mode."
```

### Stop Targeting Rules

- `class` uses a class name from the DOM (first class token is used).
- The app converts `class: "shift-controls"` to selector `.shift-controls`.
- Invalid class names are rejected.
- Stops with missing elements are skipped at runtime.
- If target is inside an inactive tab panel, the app auto-switches to that tab before highlighting.
- Optional `try_this` adds a non-blocking in-popover prompt (`Try this now: ...`).
- If `try_this` is present, the stop is treated as a keep-open exploration step (user-guided, not gated).

### Example Session Files

If `example_file` is set:

- The file is fetched and parsed as JSON.
- The user is asked for overwrite confirmation.
- On confirmation, local state is replaced with the example session.

Path safety checks:

- Must be relative (no absolute paths)
- Must not contain `..`
- Must end in `.json`

### UI Flow

1. Home button: **How to Use This App**
2. Picker modal: **Pick a Tutorial.**
3. Welcome modal: **[Filename] Tour** + YAML welcome text
4. Guided steps: heading + detail + Next/Finish
5. Completion modal: YAML completion text + **Tour Complete** button

## Adding a New Tour

1. Create `tours/N_your-tour.yaml` (for example `3_summary-practice.yaml`).
2. Add it to `tours/index.json`.
3. Optional: create `sessions/your-example.json` and reference it in `example_file`.
4. Reload app and open **How to Use This App**.

## Security Notes

- Tutorial and session file paths are validated before fetch.
- YAML text is escaped before rendering in tour popovers (no raw HTML from tour files).
- Session import requires explicit user confirmation before overwriting data.
- CSP is defined in `index.html` and should be updated deliberately when adding external assets.

## Versioning Notes

If you update Driver.js/js-yaml versions, update both:

- script/style URLs in `index.html`
- documentation in this `README.md`
