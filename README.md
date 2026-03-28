# Automation Diagnostics - Marketing Cloud CloudPage

A real-time diagnostics dashboard for Automation Studio, built as a self-contained Salesforce Marketing Cloud CloudPage. Retrieves live data at page-load via **WSProxy (SOAP)** and renders an interactive dark-themed UI, no external APIs, no data extensions, no page refreshes.

![Status](https://img.shields.io/badge/platform-Salesforce%20Marketing%20Cloud-00A1E0?style=flat-square)
![Layer](https://img.shields.io/badge/layer-CloudPages%20%2F%20SSJS-F59E0B?style=flat-square)
![API](https://img.shields.io/badge/API-WSProxy%20SOAP-22D3A0?style=flat-square)
![Version](https://img.shields.io/badge/version-v1-38BDF8?style=flat-square)

---

## Features

- **Summary stat cards** : Total, Running/Active, Errored, Paused, Inactive — all clickable to filter
- **Filter bar** : one-click status groups (All / Running / Scheduled / Paused / Error / Inactive)
- **Live search** : filters by automation name, description, or customer key as you type
- **Sortable columns** : Name, Status, Type, Last Run, Next Scheduled
- **Expandable detail rows** : per-automation metadata and last 10 run history records inline
- **Error log panel** : appears automatically when errored instances are detected
- **Debug bar** : WSProxy row counts and page counts per object for quick troubleshooting

---

## How It Works

```
Request → SSJS block runs server-side
        → WSProxy queries Automation + AutomationInstance objects
        → JSON serialised into hidden <textarea> via AMPscript
        → Client JS reads textarea, builds all UI dynamically
```

The page is **fully self-contained** — one HTML file, no npm packages, no external dependencies, no configuration.

---

## WSProxy Objects

| Object | Key Field | Purpose |
|---|---|---|
| `Automation` | `ProgramID` | All automations in the BU with status, schedule, and metadata |
| `AutomationInstance` | `TaskProgramID` | Run history records, linked back via `TaskProgramID → Automation.ProgramID` |

---


## Pagination

WSProxy caps each response at ~2,500 rows. The `retrieveAll()` helper pages through all results correctly by passing `ContinueRequest` inside the `props` object on subsequent calls — **not** via a `continueRequest()` method call, which is unreliable.

```js
var props = {};
while (moreData) {
  if (reqID) props.ContinueRequest = reqID;
  var resp = api.retrieve(objectType, cols, filter, opts, props);
  // push resp.Results …
  if (resp.HasMoreRows) { moreData = true; reqID = resp.RequestID; }
}
```

---

## Deployment

1. Copy the full source of `automation-diagnostics.html` into a new **CloudPage** in Marketing Cloud.
2. **Publish** the page.
3. Access it from within an authenticated MC session.

> **Recommended:** place the page inside a locked folder and restrict the URL to admins/ops staff — it exposes automation names, schedules, and raw error messages.

---

## File Structure

```
automation-diagnostics.html
├── <script runat="server">           SSJS block
│   ├── retrieveAll()                 Paginated WSProxy helper
│   ├── Automation retrieve           All BU automations (Status IN [-1…8])
│   └── AutomationInstance retrieve   Run history
├── <style>                           Embedded CSS — dark theme, no external sheets
├── <textarea id="__data">            AMPscript JSON handoff (%%=v(@jsonData)=%%)
└── <script>                          Client-side JS
    ├── SM {}                         Status code → label / CSS class map
    ├── FG {}                         Filter group → status code buckets
    ├── instMap {}                    Instance index keyed by ProgramID
    ├── render()                      Main table renderer
    ├── buildDetail()                 Expandable drawer content per row
    └── renderErrorLog()              Error panel renderer
```

---

## Known Limitations

- Run history depth depends on how long your BU retains `AutomationInstance` records (typically 30–90 days).
- Large BUs with 500+ automations may experience a noticeable load delay as WSProxy pages through all records synchronously.
- Step-level (activity) detail is not available via SOAP WSProxy.

---
