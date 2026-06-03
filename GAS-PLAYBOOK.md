# Google Apps Script Playbook

> A governing document + index of Google's official **`apps-script-samples`** library, written
> for the **Cockpit / CoPilot** project and reusable across any future Google Apps Script (GAS)
> work. It maps the sample library to the parts of Cockpit (Gmail/Calendar sidebars, web-app SPA,
> Sheet-as-database, and N8N-driven background flows) and records the current source-of-truth and
> hard platform limits that shape GAS architecture.
>
> _Derived from a review of the `apps-script-samples` repo (a fork of
> [`googleworkspace/apps-script-samples`](https://github.com/googleworkspace/apps-script-samples))
> plus current Google documentation. Last reviewed: 2026-06._

---

## 1. How to use this document

This playbook serves two purposes:

1. **An index/map** of the sample library — so you (or an AI agent like CoPilot) can jump straight
   to a proven, Google-maintained reference for any task instead of inventing patterns.
2. **A rulebook** for GAS — the runtime quirks, hard limits, and conventions that any Cockpit code
   must respect.

**The two-source rule (important):**

| Source | Treat as | Use for |
| --- | --- | --- |
| [`developers.google.com/apps-script`](https://developers.google.com/apps-script) + [`/workspace/add-ons`](https://developers.google.com/workspace/add-ons) | **Source of truth** | API behavior, scopes, quotas, deprecations, release notes |
| [`googleworkspace/apps-script-samples`](https://github.com/googleworkspace/apps-script-samples) | **Pattern source** | Copy-ready, lint-clean reference implementations |
| This repo / fork | A snapshot of the above | Local browsing; may lag upstream |

> When the samples and the docs disagree, **the docs win** — samples can lag. Always confirm a
> behavior against the docs/release notes before relying on it.

---

## 2. Source of truth & currency

- **Canonical samples** live at `github.com/googleworkspace/apps-script-samples` (Apache-2.0,
  Google-maintained, community PRs). This repository is a **fork** of it. It is *not* the only
  source of truth, but it is the canonical *sample code*.
- **Canonical docs** are the developer site + the per-product **release notes**
  ([Apps Script release notes](https://developers.google.com/apps-script/release-notes),
  [Workspace add-ons release notes](https://developers.google.com/workspace/add-ons/release-notes)).
- **Currency:** the upstream repo is actively maintained (recent commits as of late 2025 are
  Biome/type-checking cleanups and Chat-webhook work), so its patterns reflect current GAS. Re-verify
  any specific API or quota against the release-notes pages, which change more often than samples.

**How to re-check currency quickly:** read the two release-notes pages above; check the repo's recent
commit history; confirm your manifest uses `"runtimeVersion": "V8"`.

---

## 3. The runtime — what GAS is and isn't

GAS runs on the **V8 runtime** (modern ECMAScript), **not** Node.js or a browser. The Rhino (ES5)
runtime **stopped executing on 2026-01-31** — all code must be V8 (`"runtimeVersion": "V8"` in the
manifest). The repo's own [`GEMINI.md`](GEMINI.md) is the authoritative in-tree guide; the essentials:

**Use modern JS:** `const`/`let` (never `var`), arrow functions, destructuring, template literals,
default params, `for...of` (preferred over `forEach` for large arrays + `break`/`continue`).

**These standard APIs do NOT exist in GAS — use the replacement:**

| Missing | Use instead |
| --- | --- |
| `setTimeout` / `setInterval` (macrotasks) | `Utilities.sleep(ms)` (synchronous only); installable time-driven triggers for "later" |
| `fetch` / `FormData` / `Blob` / `URL` | `UrlFetchApp.fetch(url, params)`; `Utilities` + `*App` services |
| `atob` / `btoa` | `Utilities.base64Decode()` / `Utilities.base64Encode()` |
| `crypto` / `SubtleCrypto` | `Utilities.computeDigest()`, `computeHmacSha256Signature()`, `computeRsaSha256Signature()` |
| `window`, `navigator`, `process` | none (server context) |
| ES6 `import` / `export` modules | Apps Script **Libraries**, or bundle into one file |
| Private (`#field`) / static (`static x = 0`) class fields | closures / `WeakMap`; assign statics after class def |

**Concurrency model:** the runtime is **fundamentally synchronous**. `async`/`await`/`Promise`
microtasks work, but there is **no event loop** for macrotasks. **All I/O is blocking** — to
parallelize network calls use **`UrlFetchApp.fetchAll(requests)`**, not `Promise.all` over individual
fetches. (WebAssembly is the one non-blocking exception.)

**File execution order:** all `.gs` files share one global scope; avoid top-level side effects and
ensure definitions exist before cross-file use.

---

## 4. Platform limits & quotas that drive design

These are hard ceilings — design around them, don't fight them.
(Authoritative: [Quotas for Google Services](https://developers.google.com/apps-script/guides/services/quotas).)

| Limit | Value (Workspace business / consumer) | Architectural consequence |
| --- | --- | --- |
| Script execution time | **6 min / invocation** | Chunk long jobs; use continuation tokens + a time-driven trigger to resume |
| Trigger total runtime | **90 min / day** | Keep background enrichment lean; batch work |
| `UrlFetchApp` calls | **100k / 20k per day**, **50 MB / call** | Fine for N8N webhooks; batch with `fetchAll` |
| Call-based service quotas | per-service caps on `SpreadsheetApp`, `GmailApp`, etc. | **Batch** — one `getValues`/`setValues`, not per-cell calls |
| Simultaneous executions | limited per user | Use **`LockService`** to serialize concurrent writes (e.g. N8N + GAS hitting the same sheet) |
| `PropertiesService` | small per-value/total size | Config & secrets only — not bulk data |
| `CacheService` | ~6 hr TTL, small values | Short-lived caching of lookups/API responses |

---

## 5. Index of the sample library

Scannable map of every top-level directory. "Reach for it when…" tells you when to open it.

| Directory | What's in it | Reach for it when… |
| --- | --- | --- |
| [`templates/`](templates/) | Starter skeletons: `web-app/`, `standalone/`, `sheets-addon/`, `docs-addon/`, `forms-addon/`, `custom-functions/` | Bootstrapping any new project type |
| [`ui/`](ui/) | Dialogs, sidebars, web apps, **`communication/`** (`google.script.run`), HTML templating, user identity | Building any HtmlService UI or client↔server bridge |
| [`gmail/`](gmail/) | `add-ons/` (CardService + contextual triggers), `sendingEmails/`, `mailmerge/`, `markup/`, `inlineimage/` | Gmail add-on UI or email automation |
| [`calendar/`](calendar/) | Calendar API quickstart; see also `solutions/automations/vacation-calendar` | Reading/writing events |
| [`sheets/`](sheets/) | `api/` advanced snippets, `customFunctions/`, form-response handling, Maps integration | Anything Sheets, incl. Sheet-as-DB |
| [`docs/`](docs/) `[slides/]`(slides/) `[forms/]`(forms/) | Editor add-ons, dialog/sidebar UI, translate/progress, Slides/Forms automation | Editor add-ons for those hosts |
| [`drive/`](drive/) | File/folder mgmt, activity tracking (`activity-v2/`) | Drive operations |
| [`tasks/`](tasks/) | Tasks API + **`simpleTasks/` web app** (`doGet`) | A small CRUD web app over an API |
| [`people/`](people/) `[classroom/]`(classroom/) | Contacts / Classroom quickstarts | Those APIs |
| [`adminSDK/`](adminSDK/) | Directory, Reports, Reseller | Workspace admin automation |
| [`advanced/`](advanced/) | 30+ single-file examples for advanced services (BigQuery, Analytics, YouTube, Sheets, Docs, Chat…) | Calling any Google REST API via an advanced service |
| [`service/`](service/) | **`propertyService.gs`** (Script/User/Document properties), **`jdbc.gs`** (SQL) | Config/secret storage or external SQL |
| [`triggers/`](triggers/) | **`form/`**: installable trigger + `ScriptApp.getAuthorizationInfo()` auth handling | Trigger setup + auth-status patterns |
| [`forms-api/`](forms-api/) | REST Forms API snippets + **`demos/` web app**; `ScriptApp.getOAuthToken()` + `UrlFetchApp` | Calling a Google REST API with the script's own token |
| [`apps-script/`](apps-script/) | Functions invoked externally via the Apps Script API | Letting an external system call a GAS function |
| [`picker/`](picker/) | Google Picker dialog; passes `getOAuthToken()` to client to avoid double-auth | File-picker UX inside an add-on |
| [`chat/`](chat/) | Chat API quickstart + advanced service + app auth | Google Chat apps/bots |
| [`data-studio/`](data-studio/) | Looker Studio connector (`build.gs`, `auth.gs`, semantics, errors) | Custom Looker Studio connectors |
| [`ai/`](ai/) | Gemini/Vertex integrations: `email-classifier/`, `gmail-sentiment-analysis/`, custom-function AI, link-preview | Any AI feature (CoPilot!) |
| [`solutions/`](solutions/) | ~30 end-to-end builds: **automations**, **add-ons**, custom functions, assistants, **webhooks** | A complete, documented reference close to your use case |
| [`mashups/`](mashups/) | Multi-service combinations | Cross-service glue ideas |
| [`utils/`](utils/) | Logging/timing helpers | Lightweight perf logging |
| [`wasm/`](wasm/) | Rust/Python → WebAssembly in GAS | Heavy compute (niche) |

---

## 6. Cockpit component playbooks

The tailored core. Each maps a Cockpit part to the runtime model and the best reference samples.

### 6a. Gmail & Calendar sidebars — Workspace Add-ons (CardService)

Cockpit's Gmail/Calendar sidebars are **Google Workspace Add-ons**, which render UI as
**CardService cards** (not HTML). Key facts:

- **Editor add-ons ≠ Workspace add-ons** and can't be upgraded between the two; CardService is **not**
  available in editor (Docs/Sheets-bound) add-ons. Keep the model straight per host.
- `KeyValue` is **deprecated → use `DecoratedText`**.
- Calendar add-ons can now attach files to events via the CardService **`Attachment`** class.
- Minimize `oauthScopes` (drives the consent screen + OAuth verification).

**Canonical manifest (`appsscript.json`) — Workspace add-on:**

```json
{
  "timeZone": "America/New_York",
  "runtimeVersion": "V8",
  "exceptionLogging": "STACKDRIVER",
  "oauthScopes": [
    "https://www.googleapis.com/auth/gmail.addons.execute",
    "https://www.googleapis.com/auth/gmail.addons.current.message.metadata",
    "https://www.googleapis.com/auth/calendar.addons.execute",
    "https://www.googleapis.com/auth/script.external_request"
  ],
  "addOns": {
    "common": {
      "name": "Cockpit",
      "logoUrl": "https://.../logo-64.png",
      "layoutProperties": { "primaryColor": "#1a73e8" }
    },
    "gmail":    { "homepageTrigger": { "runFunction": "onHomepage", "enabled": true },
                  "contextualTriggers": [{ "unconditional": {}, "onTriggerFunction": "onGmailMessage" }] },
    "calendar": { "homepageTrigger": { "runFunction": "onHomepage", "enabled": true },
                  "eventOpenTrigger": { "runFunction": "onCalendarEvent" } }
  }
}
```

**Best references:** [`gmail/add-ons/`](gmail/add-ons/) (contextual cards + message metadata),
[`ai/email-classifier/Cards.gs`](ai/email-classifier/Cards.gs) (card builders, button actions with
`LoadIndicator.SPINNER`, **card navigation** via `CardService.newNavigation().updateCard(...)`),
[`ai/gmail-sentiment-analysis/`](ai/gmail-sentiment-analysis/) (full add-on + AI).

### 6b. Web-app multi-tab SPA — HtmlService

Cockpit's SPA is served by **`doGet(e)`** returning an `HtmlService` page. GAS runs HTML in an
**IFRAME sandbox**; client code talks to the server **only** through `google.script.run`.

**Canonical SPA shell:**

```javascript
/** Server entry — serve the SPA; read tab/route from URL params if present. */
function doGet(e) {
  const t = HtmlService.createTemplateFromFile('Index');
  t.initialTab = (e && e.parameter.tab) || 'home';
  return t.evaluate()
    .setTitle('Cockpit')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL); // only if embedding elsewhere
}

/** Server-side include: lets Index.html pull in JavaScript.html / Stylesheet.html. */
function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}
```

```html
<!-- Index.html -->
<!DOCTYPE html><html><head>
  <base target="_top">                 <!-- required so links/forms escape the iframe -->
  <?!= include('Stylesheet'); ?>
</head><body>
  <!-- tab nav + <div id="view"> rendered client-side -->
  <?!= include('JavaScript'); ?>
</body></html>
```

```html
<!-- JavaScript.html: client↔server bridge -->
<script>
  function loadTab(name) {
    google.script.run
      .withSuccessHandler(html => document.getElementById('view').innerHTML = html)
      .withFailureHandler(err => showError(err.message))
      .renderTab(name);                // a server .gs function returning HTML/JSON
  }
  loadTab('<?= initialTab ?>');
</script>
```

**Best references:** [`templates/web-app/`](templates/web-app/) (`Code.gs` + `Index/JavaScript/Stylesheet.html`),
[`ui/webapp/`](ui/webapp/), [`ui/communication/`](ui/communication/) (success/failure/userObject
patterns), [`ui/sidebar/`](ui/sidebar/), [`tasks/simpleTasks/`](tasks/simpleTasks/) and
[`forms-api/demos/`](forms-api/demos/) (real `doGet` apps).

### 6c. Google Sheet as a database

Treat the Sheet as a row store. The cardinal rule is **batch** — one read, work in memory, one write
(call-based quotas punish per-cell access).

**Canonical batch read/write with header-keyed rows + a lock:**

```javascript
/** Read a sheet as an array of objects keyed by header row. */
function readRows(sheetName) {
  const sh = SpreadsheetApp.getActive().getSheetByName(sheetName);
  const [headers, ...rows] = sh.getDataRange().getValues();   // ONE call
  return rows.map(r => Object.fromEntries(headers.map((h, i) => [h, r[i]])));
}

/** Append a row safely under contention (N8N + GAS writing concurrently). */
function appendRow(sheetName, obj) {
  const lock = LockService.getScriptLock();
  lock.waitLock(30000);                                       // serialize writers
  try {
    const sh = SpreadsheetApp.getActive().getSheetByName(sheetName);
    const headers = sh.getRange(1, 1, 1, sh.getLastColumn()).getValues()[0];
    sh.appendRow(headers.map(h => obj[h] ?? ''));             // ONE write
  } finally {
    lock.releaseLock();
  }
}
```

For multi-range / formatting / pivot operations use the **Sheets advanced service**
(`Sheets.Spreadsheets.Values.batchGet/batchUpdate`). Cache hot lookups in `CacheService`; keep
config in `PropertiesService`.

**Best references:** [`sheets/api/spreadsheet_snippets.gs`](sheets/api/spreadsheet_snippets.gs),
[`advanced/sheets.gs`](advanced/sheets.gs), [`solutions/automations/import-csv-sheets/`](solutions/automations/import-csv-sheets/),
[`service/propertyService.gs`](service/propertyService.gs).

### 6d. Background flows & N8N integration

Two directions plus triggers.

**Inbound (N8N → GAS): a `doPost(e)` webhook receiver.** Deploy as a Web App (*Execute as: me*,
*Who has access: Anyone*) and **verify a shared secret** — a public web-app URL is unauthenticated.

```javascript
function doPost(e) {
  const body = JSON.parse(e.postData.contents);
  if (body.secret !== PropertiesService.getScriptProperties().getProperty('WEBHOOK_SECRET')) {
    return ContentService.createTextOutput(JSON.stringify({ ok: false }))
      .setMimeType(ContentService.MimeType.JSON);             // do not leak why
  }
  const lock = LockService.getScriptLock();
  lock.waitLock(30000);
  try {
    handleInbound(body);                                      // enrich + write to the sheet
  } finally { lock.releaseLock(); }
  return ContentService.createTextOutput(JSON.stringify({ ok: true }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**Outbound (GAS → N8N):** `UrlFetchApp.fetch(n8nUrl, { method:'post', contentType:'application/json',
payload: JSON.stringify(data), muteHttpExceptions:true })`; batch with `fetchAll` for fan-out.

**Triggers (the "new row → enrich" flow):** prefer an **installable** trigger — time-driven (poll
for unprocessed rows) or `onChange`/`onEdit`. Make enrichment **idempotent** (mark a `processed`
column) and wrap writes in `LockService`. Manage triggers programmatically:

```javascript
ScriptApp.newTrigger('enrichNewRows').timeBased().everyMinutes(5).create();
// audit/cleanup: ScriptApp.getProjectTriggers() → ScriptApp.deleteTrigger(t)
```

Check auth status for unattended triggers via `ScriptApp.getAuthorizationInfo(ScriptApp.AuthMode.FULL)`
(pattern in [`triggers/form/`](triggers/form/)).

**Best references:** [`solutions/webhook-chat-app/`](solutions/webhook-chat-app/) (UrlFetchApp webhook
posting), [`triggers/form/`](triggers/form/) (installable trigger + auth handling),
[`solutions/automations/vacation-calendar/`](solutions/automations/vacation-calendar/) and
[`import-csv-sheets/`](solutions/automations/import-csv-sheets/) (time-driven + `PropertiesService`
last-run state).

### 6e. External APIs & secrets

- **Google APIs:** use `ScriptApp.getOAuthToken()` + `UrlFetchApp` with `Authorization: Bearer …`
  (no extra creds) — see [`forms-api/snippets/retrieve_all_responses.gs`](forms-api/snippets/retrieve_all_responses.gs).
- **Third-party / service-account:** use the **OAuth2 library** (store the key in `PropertiesService`).
- **Never inline secrets** — always `PropertiesService.getScriptProperties()`.
- Whitelist outbound hosts with `urlFetchWhitelist` in the manifest where you can.

Reference: [`solutions/add-on/book-smartchip/`](solutions/add-on/book-smartchip/) (external Books API).

### 6f. AI / CoPilot integration

Call Gemini/Vertex over `UrlFetchApp`; authenticate a service account via the OAuth2 library
(`cloud-platform` scope), key in Script Properties. **Constrain the model to JSON** and parse it —
don't consume free text in automation.

**Best references:** [`ai/gmail-sentiment-analysis/Vertex.gs`](ai/gmail-sentiment-analysis/Vertex.gs)
(service-account auth + `:generateContent` request shape),
[`ai/email-classifier/ClassifyEmail.gs`](ai/email-classifier/ClassifyEmail.gs) (structured-output
prompting), and `ai/custom-func-*` for AI-backed custom Sheet functions.

> If CoPilot uses the **Claude API** instead of Gemini, the structure is identical — Script-Properties
> secret + `UrlFetchApp` POST; only the endpoint and request body change.

---

## 7. Coding conventions (adopt for Cockpit)

From [`GEMINI.md`](GEMINI.md) — make these the house style:

- **V8 modern JS** everywhere (`const`/`let`, arrow fns, destructuring, template literals, `for...of`).
- **Type every function with JSDoc** (`@param`, `@return`, `@typedef` for objects, `@type` casts).
  Do **not** reference the `GoogleAppsScript` namespace in JSDoc — define local typedefs + link to docs.
- Use **optional chaining** (`?.`) on API responses that use field masks.
- **Guard advanced services**: `if (!Sheets) { /* must be enabled */ return; }` (also narrows types).
- **Don't** wrap code in `try/catch` just to `console.log(err)` — let the runtime report it (except
  where you genuinely recover, e.g. lock release / webhook responses).
- Avoid top-level side effects; mind cross-file execution order.

---

## 8. Dev & deploy tooling (copy the upstream setup)

The repo ships a strong, portable toolchain worth replicating in Cockpit:

- **`clasp`** for local dev / push (`.clasp.json` per project). The
  [`.github/scripts/clasp_push.sh`](.github/scripts/clasp_push.sh) helper pushes **only changed**
  project dirs.
- **Biome** lint/format ([`biome.json`](biome.json)): `pnpm lint`, `pnpm format`.
- **Type-check `.gs` without renaming**: [`.github/scripts/check-gs.ts`](.github/scripts/check-gs.ts)
  copies `.gs`→`.ts`, runs `tsc` with `@types/google-apps-script`. `pnpm check [path]`. **Highest-ROI
  thing to adopt** — real type safety on otherwise-untyped add-on code.
- **CI publish**: [`.github/workflows/publish.yaml`](.github/workflows/publish.yaml) writes
  `~/.clasprc.json` from a `CLASP_CREDENTIALS` secret and runs the push script on `main`.

---

## 9. Decision cheat-sheets

**UI: CardService vs HtmlService**

| Use **CardService** | Use **HtmlService** |
| --- | --- |
| Gmail/Calendar/Drive **Workspace add-on** sidebars | Standalone **web app** (Cockpit SPA), editor (Sheets/Docs) sidebars & dialogs |
| Native look, minimal code, cross-host | Full custom HTML/CSS/JS, rich SPA UX |
| No custom HTML needed | Need a real DOM / client-side framework |

**Built-in service vs advanced service**

| Built-in (`SpreadsheetApp`, `GmailApp`…) | Advanced service (`Sheets`, `Gmail`…) |
| --- | --- |
| Simpler, ergonomic, enough for most | Full REST surface, batch ops, fields masks |
| Default choice | Reach for batch/perf or features the built-in lacks (must enable in manifest) |

**Trigger selection**

| Need | Trigger |
| --- | --- |
| React to a sheet edit | simple `onEdit` (no auth) / installable `onEdit` (needs auth, more power) |
| React to structural change / external row append | installable `onChange` |
| Scheduled work / polling | time-driven (`everyMinutes/Hours`, `atHour`) |
| Form submission | `onFormSubmit` |
| Inbound HTTP from N8N | `doPost(e)` web app (not a trigger) |

**Storage choice**

| Data | Use |
| --- | --- |
| Small config / secrets | `PropertiesService` (Script/User/Document) |
| Short-lived cache (≤6 hr) | `CacheService` |
| Structured records / app data | the **Sheet** (batched) |
| Relational / large | external SQL via `jdbc.gs` or an external API |

---

## 10. Notes for CoPilot (AI agent usage)

When generating or reviewing Cockpit GAS code, an AI agent should:

1. Treat the **sample paths in §5–§6 as the canonical pattern source**, and
   `developers.google.com/apps-script` as the **behavioral source of truth** (samples can lag).
2. Enforce the **runtime constraints in §3** (no `setTimeout`/`fetch`/`crypto` globals; `fetchAll`
   for parallel I/O; V8 manifest) and the **limits in §4** (batch Sheet I/O; `LockService` on shared
   writes; chunk >6-min jobs).
3. Apply the **§7 conventions** (JSDoc types, no decorative `try/catch`, optional chaining).
4. Keep the Gmail/Calendar sidebars (CardService) and the web-app SPA (HtmlService) on their correct
   models — they are **not** interchangeable (§6a vs §6b).
