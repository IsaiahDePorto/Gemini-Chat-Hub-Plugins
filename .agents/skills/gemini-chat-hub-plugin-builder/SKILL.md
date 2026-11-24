---
name: gemini-chat-hub-plugin-builder
description: Comprehensive operational guide for designing, engineering, testing, and deploying custom plugins for Gemini Chat Hub. Covers OpenAI Function Calling schemas, Node.js server VM sandboxing (zero-CORS), persistent local data storage (storage/localStorage), code execution workspace bridging (workspace/runtimeWorkspace into Python IPython kernel, Bash, and Bun), Secret Vault security, HTTP Actions (JMESPath/Handlebars post-processing, binary media interception, workspace auto-save), mid-turn multimodal returns (images to Gemini + audio/video UI players), output destinations, and runtime limitations.
license: MIT
compatibility: Node.js VM Server Sandbox, Gemini Interactions API (v1beta), Gemini Chat Hub
metadata:
  version: "2.2.0"
  author: "Gemini Chat Hub Engineering"
---

# Gemini Chat Hub Custom Plugin Development Guide

This operational manual outlines the technical requirements, manifest schema, runtime architecture, supported features, and hard limitations for engineering Custom Plugins within **Gemini Chat Hub**.

---

## 1. Core Architectural Overview

Gemini Chat Hub plugins extend Google Gemini models using **OpenAI Function Calling API Specifications** through the native **Gemini Interactions API (`v1beta/interactions`)** (and are also exposed in **Live Voice Mode**).

When a plugin is enabled, Gemini Chat Hub normalizes its manifest via `normalizePluginManifest` and registers its tool declarations with the model. During multi-turn interactions, the model autonomously invokes plugin tools, receives structured execution output, and synthesizes responses in real time.

Plugins in Gemini Chat Hub can also return **mid-turn multimodal media**:
- **Images** are delivered natively to Gemini inside the `function_result` subcontent array (`{ type: "image", data | uri, mime_type }`) so Gemini can visually inspect them mid-turn, **and** are rendered in the chat UI.
- **Audio and Video** are rendered as interactive media players in the chat UI (`clientMedia.audio` / `clientMedia.video`), while Gemini receives a descriptive text placeholder (see Section 6 for API constraints).

```
┌────────────────────────────────────────────────────────┐
│                   Gemini Chat Hub                      │
│             (Next.js Server Runtime)                   │
├────────────────────────────────────────────────────────┤
│  Interactions API Multi-Turn Function Calling Loop     │
│  (+ Live Voice Mode tool registry)                     │
│                         │                              │
│         ┌───────────────┴───────────────┐              │
│         ▼                               ▼              │
│ ┌───────────────────────┐   ┌────────────────────────┐ │
│ │  JavaScript Sandbox   │   │      HTTP Action       │ │
│ │   (Node.js VM Core)   │   │ (Templated Fetcher +   │ │
│ │   90s VM Timeout      │   │  JMESPath/Handlebars)  │ │
│ └───────────┬───────────┘   └───────────┬────────────┘ │
│             │                           │              │
│    ┌────────┼─────────────────┐         │              │
│    ▼        ▼                 ▼         ▼              │
│ [Storage] [Vault]        [Workspace Bridge]            │
│ (Postgres (Secrets       (/workspace/chats/<id>/       │
│  + Disk)   Redaction)     in, out, tmp)                │
│                               │                        │
│                               ▼                        │
│                  ┌────────────────────────┐            │
│                  │ Stateful Code Runtimes │            │
│                  │ • IPython Kernel       │            │
│                  │ • Persistent Bash      │            │
│                  │ • Polyglot Bun (TS/JS) │            │
│                  └────────────┬───────────┘            │
│                               ▼                        │
│             ┌────────────────────────────────────┐     │
│             │   Multimodal Normalization Pipe    │     │
│             │   • Images -> Gemini + UI Gallery  │     │
│             │   • Audio  -> UI Player (+ text)   │     │
│             │   • Video  -> UI Player (+ text)   │     │
│             └─────────────────┬──────────────────┘     │
│                               ▼                        │
│             Gemini function_result subcontent parts    │
│             (ImageContent & TextContent)               │
└────────────────────────────────────────────────────────┘
```

### Key Differences from Browser-Based Plugins (e.g., TypingMind)
| Capability | Legacy Browser Plugins (TypingMind) | Gemini Chat Hub Plugins |
| :--- | :--- | :--- |
| **Execution Environment** | Client-side browser `<iframe>` (`allow-scripts`) | **Server-side Node.js VM (`node:vm`)** |
| **Network & CORS** | Restricted by browser CORS; requires proxy workers | **Direct server fetch (Zero CORS restrictions)** |
| **Execution Timeout** | Typically 10–15 seconds | **90s (JS Sandbox) / 30s (HTTP Action) / 120s (Parent Guard)** |
| **Data Persistence** | Volatile or browser `localStorage` | **Dual-persistence: PostgreSQL DB + disk mirror** |
| **Code Execution Bridge** | None | **Direct bridge to Python, Bash, and Bun runtimes** |
| **Output Artifacts** | Manual copy/paste | **Files in `out/` auto-index as UI download artifacts** |
| **Secret Management** | Exposed plain-text settings | **Integrated Secret Vault with active output scrubbing** |
| **Mid-Turn Multimodal** | Text only; cannot feed media back to model | **Native Images mid-turn to Gemini; Audio/Video UI players** |
| **Model Integration** | Chat completions API | **Gemini Interactions API (`v1beta/interactions`) + Live Mode** |

---

## 2. Manifest Format & Compatibility Aliases

Gemini Chat Hub accepts both native **Gemini Chat Hub** manifests and legacy **TypingMind** manifests. All manifests pass through `normalizePluginManifest()`, which supports both `camelCase` and `snake_case` aliases:

| Canonical / Preferred Field | Accepted Alias(es) | Type | Description |
| :--- | :--- | :--- | :--- |
| `id` | `uuid`, derived from `name` | String | Unique plugin identifier (defaults to sanitized `name`). |
| `name` | `title` | String | **Required** (either `name` or `title`). User-facing plugin name. |
| `description` | `overviewMarkdown` | String | Plugin description. Note: `normalizePluginManifest` merges `description` and `overviewMarkdown` into the same string. |
| `emoji` | — | String | Optional emoji icon. |
| `iconURL` | `icon_url` | String | Optional URL to an icon image or SVG (rendered in the Plugins modal). |
| `version` | — | String | Semantic version string (defaults to `"1.0.0"`). |
| `author` | — | String | Author name. |
| `contextPrompt` | `context_prompt` | String | System instruction injected whenever the plugin is enabled. |
| `userSettings` | `user_settings` | Array | Configuration field descriptors for the plugin UI. |
| `pluginFunctions` | `functions` | Array | Tool function definitions (see Section 8). |
| `localStorage` | `local_storage` | Object | Declarative metadata `{ enabled, initialData }` (see Section 4). |
| `workspaceAccess` | `workspace_access` | Object | Declarative metadata `{ enabled, allowedDirectories }` (see Section 5). |
| `vaultSecrets` | `vault_secrets` | Array[Str] | Declarative metadata listing expected Vault secret names. |

> [!NOTE]
> Fields outside this schema (such as `authenticationType`) are ignored and stripped during `normalizePluginManifest()`.

---

## 3. Interface Configuration: User Settings & Vault Integration

User Settings define configuration inputs required for plugin execution (such as API keys, custom endpoints, or user preferences) rendered in the **Plugins Modal**.

### JSON Field Descriptor Schema
Each setting object supports the following properties:

| Property | Type | Description | Required |
| :--- | :--- | :--- | :--- |
| `name` | String | Key identifier used to retrieve the variable programmatically (camelCase or snake_case). | Yes |
| `label` | String | User-facing input label displayed in the UI (defaults to `name`). | No |
| `type` | String | Input widget type: `text`, `password`, `email`, `number`, or `enum`. (Defaults to `text`). | No |
| `required` | Boolean | Declares if the field is marked required in metadata (not strictly blocked by UI save). | No |
| `description` | String | Explanatory subtext or format constraints. | No |
| `placeholder` | String | Placeholder helper text inside empty input fields. | No |
| `values` | Array[Str] | Valid options array. **Required** when `type` is `"enum"`. (Do **not** use `enum` as the property name—the parser and UI read `values`). | Conditional |
| `defaultValue` | String | Fallback value if the field is left empty. | No |
| `vaultSecret` | `vault_secret` | Optional default Secret Vault key to suggest for autofill (e.g. `"GITHUB_PAT"`). | No |

### Automatic API Key Fallbacks & Vault Linking
1. **Tier-Aware Gemini Key Fallbacks:** Gemini Chat Hub automatically populates `userSettings.google_api_key` and `userSettings.gemini_api_key` from the active server key tier (`GEMINI_FREE_API_KEY`, `GEMINI_PAID_API_KEY`, or `GEMINI_API_KEY`) if the user leaves those fields empty.
2. **Vault Secret Dynamic Linking:** Users can autofill from the Vault in the UI or reference secrets as `{{VAULT:SECRET_NAME}}` or `{{vault.SECRET_NAME}}` inside any setting value. The server resolves these tokens securely at execution time before passing `userSettings` to your function.

### Config Blueprint: User Settings Array
```json
[
  {
    "name": "exa_api_key",
    "label": "Exa Search API Key",
    "type": "password",
    "required": true,
    "vaultSecret": "EXA_API_KEY",
    "description": "Enter your Exa API key or link it from the Secret Vault."
  },
  {
    "name": "result_limit",
    "label": "Result Limit",
    "type": "number",
    "defaultValue": "10",
    "description": "Maximum number of search results to retrieve."
  },
  {
    "name": "search_category",
    "label": "Search Category",
    "type": "enum",
    "values": ["general", "research paper", "news", "github"],
    "defaultValue": "general"
  }
]
```

---

## 4. Persistent Prompting: Plugin Context (`contextPrompt`)

Plugin Context injects instructions, schema definitions, or domain rules directly into Gemini's **system instruction** whenever the plugin is enabled, even before any tool is called.

### Template Interpolation
`contextPrompt` (or `context_prompt`) supports dynamic variable interpolation from `userSettings` using `{settingName}`:
```json
{
  "contextPrompt": "You have access to the internal knowledge base at {api_base_url}. Always query financial records using ISO 8601 UTC dates."
}
```

All active plugin context prompts are compiled into the system instructions under:
`### [Plugin Context: <Plugin Name>]`

> [!WARNING]
> `contextPrompt` only interpolates `{settingName}` keys from `userSettings`. It does **not** resolve `{{VAULT:...}}` tokens, and you should never inject raw secrets into system prompts.

---

## 5. Persistent Local Data Storage (`storage` / `localStorage`)

Gemini Chat Hub provides a built-in, asynchronous persistent key-value storage engine for custom plugins.

### Storage Architecture
- **Database Persistence:** Every plugin has an isolated JSON record in PostgreSQL (`Setting` table under key `plugin_storage:<pluginId>`).
- **Disk Mirroring:** Mirrored to disk at `<workspaceRoot>/plugin_data/<pluginId>.json` (typically `/workspace/plugin_data/<pluginId>.json`).
- **In-Memory Cache:** Fast reads during execution turns with synchronous-feeling caching and DB + disk persistence on writes.

### Manifest Declaration & Behavior
You can declare `localStorage` in the manifest as documentation:
```json
{
  "localStorage": {
    "enabled": true,
    "initialData": {
      "query_history": [],
      "favorites": []
    }
  }
}
```
> [!IMPORTANT]
> `storage` is **always** injected into every JavaScript plugin regardless of `localStorage.enabled`, and `initialData` in the manifest is **not** automatically seeded into storage. Always initialize default values in code when `await storage.get(key)` returns `null`.

### JavaScript API Methods
Available globally as `storage` and `localStorage`, and via `resources.storage`. All methods are **asynchronous** (do not use synchronous browser `localStorage.getItem()`):

| Method | Alias | Signature | Description |
| :--- | :--- | :--- | :--- |
| `get` | `getItem` | `await storage.get<T>(key: string): Promise<T \| null>` | Retrieves a stored item by key. Returns parsed value or `null`. |
| `set` | `setItem` | `await storage.set(key: string, value: any): Promise<void>` | Saves an item (string, number, object, array, boolean). |
| `delete` | `removeItem` | `await storage.delete(key: string): Promise<void>` | Deletes a stored key. |
| `clear` | — | `await storage.clear(): Promise<void>` | Clears all data for this plugin. |
| `list` | `getAll` | `await storage.list(): Promise<Record<string, any>>` | Returns a shallow copy of all key-value entries. |

### Code Example: Caching API Results in Local Storage
```javascript
async function fetch_crypto_prices(params, userSettings, resources) {
  const { coin } = params;
  const cacheKey = `price_${coin.toLowerCase()}`;

  const cached = await storage.get(cacheKey);
  if (cached && (Date.now() - cached.timestamp < 60000)) {
    return JSON.stringify({ ...cached.data, source: "local_cache" });
  }

  const res = await fetch(`https://api.coingecko.com/api/v3/simple/price?ids=${coin}&vs_currencies=usd`);
  const data = await res.json();

  await storage.set(cacheKey, { data, timestamp: Date.now() });
  return JSON.stringify({ ...data, source: "live_network" });
}
```

---

## 6. Code Execution Workspace Bridge (`workspace` / `runtimeWorkspace`)

Gemini Chat Hub includes an integrated, stateful code execution runtime:
- **Resident IPython Kernel (`execute_python`)**: Persists variables, DataFrames, functions, and imports in RAM across turns.
- **Persistent Bash Shell (`execute_bash`)**: Stateful shell for system commands, FFmpeg, Git, and CLI tools.
- **Polyglot Bun Runner (`execute_bun`)**: Fast JavaScript and TypeScript execution.

Plugins share the same per-chat workspace directory as these runtimes, enabling multi-step pipelines where a plugin fetches or stages data and Python/Bash processes it (or vice versa).

### Workspace Directory Structure & Per-Chat Isolation
When `chatId` is present, the workspace is isolated per conversation at `/workspace/chats/<chatId>/` (or `<workspaceRoot>/chats/<chatId>/`):
- `in/` (`/workspace/in/`): Inbound data storage. Files saved here are accessible in Python via `open('in/...')` or `open('/workspace/in/...')` and in Bash.
- `out/` (`/workspace/out/`): Designated output directory. **Any file written here is automatically indexed as an artifact and triggers a downloadable UI card.**
- `tmp/` (`/workspace/tmp/`): Transient scratchpad for intermediate processing.

> [!NOTE]
> All filenames passed to `workspace` methods are sanitized (`path.basename` + non-`[a-zA-Z0-9._-]` replaced with `_`) to prevent directory traversal. Subdirectories inside `in/`, `out/`, or `tmp/` are flattened to the basename.

### Manifest Declaration
```json
{
  "workspaceAccess": {
    "enabled": true,
    "allowedDirectories": ["in", "out", "tmp"]
  }
}
```
*(Note: `workspace` is injected into all JS plugins; `workspaceAccess` is declarative metadata.)*

### JavaScript API Methods
Available globally as `workspace`, `runtimeWorkspace`, and via `resources.workspace`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `writeFile` | `await workspace.writeFile(fileName, content, subDir?): Promise<WorkspaceFileResult>` | Writes `string`, `Buffer`, or `Uint8Array` into `"in"` (default), `"out"`, or `"tmp"`. |
| `readFile` | `await workspace.readFile(fileName, subDir?): Promise<string>` | Reads a workspace file as a UTF-8 string. |
| `readFileBuffer` | `await workspace.readFileBuffer(fileName, subDir?): Promise<Buffer>` | Reads a workspace file as a raw `Buffer`. |
| `saveData` | `await workspace.saveData(name, data, format?): Promise<WorkspaceFileResult>` | Formats array/object as `"json"` (default) or `"csv"` and writes to `in/<name>.[json\|csv]`. |
| `listFiles` | `await workspace.listFiles(subDir?): Promise<string[]>` | Lists file names within `"in"` (default), `"out"`, or `"tmp"`. |
| `getPath` | `workspace.getPath(fileName?, subDir?): string` | Returns the absolute on-disk filesystem path. |

`WorkspaceFileResult` returns: `{ fileName, relativePath, absolutePath, sizeBytes, mimeType, subDir, agentNote }`.

---

## 7. Mid-Turn Multimodal Outputs (Images, Audio, Video, Documents)

Gemini Chat Hub normalizes plugin outputs through `normalizeMultimodalOutput()` before returning them to the model and UI.

### Important API Behavior: What Gemini Sees vs. What the User Sees
Because the Gemini Interactions API `FunctionResultSubcontent` currently supports **`ImageContent` and `TextContent` only**:
1. **Images (`type: "image"`):** Sent to Gemini **mid-turn** as native image parts (`{ type: "image", data, mime_type }` or `{ type: "image", uri, mime_type }`) **AND** rendered as an interactive image gallery in the chat UI.
2. **Audio (`type: "audio"`):** Rendered as an **interactive audio player in the chat UI** (`clientMedia.audio`), while Gemini receives a text placeholder: `[Audio output: <name> (<mimeType>)]`.
3. **Video (`type: "video"`):** Rendered as an **interactive video player in the chat UI** (`clientMedia.video`), while Gemini receives a text placeholder: `[Video output: <name> (<mimeType>)]`.
4. **Documents (`type: "document"`):** Converted to a text placeholder `[Document: <name|mime_type>]` for Gemini (no inline UI media player; save PDFs to `"out"` for a download artifact card).

### Manifest Declaration for Multimodal Tools
```json
{
  "name": "generate_sales_chart",
  "implementationType": "javascript",
  "outputType": "respond_to_ai",
  "returnsMultimodal": true,
  "multimodalTypes": ["image"],
  "permissions": ["multimodal", "workspace"]
}
```
*(Supported `multimodalTypes` values: `"image"`, `"audio"`, `"video"`.)*

### The `multimodal` Sandbox Helper API
Available globally as `multimodal` and via `resources.multimodal`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `multimodal.image` | `multimodal.image(input, mimeType?, options?)` | Accepts base64, `data:image/...` URL, `Buffer`/`Uint8Array`, workspace path (`"out/chart.png"`), or `http(s)://` URL. `options`: `{ resolution?: "low" \| "medium" \| "high" \| "ultra_high", name?, text? }`. |
| `multimodal.audio` | `multimodal.audio(input, mimeType?, options?)` | Accepts base64, `data:audio/...` URL, `Buffer`/`Uint8Array`, workspace path, or `http(s)://` URL. `options`: `{ name?, text? }`. |
| `multimodal.video` | `multimodal.video(input, mimeType?, options?)` | Accepts base64, `data:video/...` URL, `Buffer`/`Uint8Array`, workspace path, or `http(s)://` URL. `options`: `{ name?, processing?, resolution?, text? }`. |
| `multimodal.document` | `multimodal.document(input, mimeType?, options?)` | Accepts base64, `data:...` URL, `Buffer`/`Uint8Array`, or `http(s)://` URL. `options`: `{ name?, text? }`. |
| `multimodal.fromWorkspace` | `multimodal.fromWorkspace(fileName, subDir?, type?, mimeType?)` | **Synchronous.** Builds a multimodal part for `<subDir>/<fileName>` (`subDir` defaults to `"out"`). Pass just the filename (e.g. `multimodal.fromWorkspace("heatmap.png", "out")`), **not** `"out/heatmap.png"`. |
| `multimodal.response` | `multimodal.response(summaryText, ...parts)` | Packages a summary string with one or more multimodal parts (`{ text, parts }`). Also accepts a single object `{ text, images, ... }`. |

> [!NOTE]
> - Remote `http(s)://` URLs passed to `multimodal.image()` or `multimodal.audio()` are automatically fetched server-side (15s timeout) and converted to base64. Remote video URLs are **not** auto-fetched—fetch them yourself or save to workspace if needed.
> - `data` and `uri` are enforced mutually exclusive before reaching Gemini; local workspace paths are always read from disk and converted to base64.

### Supported Return Formats
`normalizeMultimodalOutput` accepts any of the following return shapes:

1. **Multimodal Helper Response (Recommended):**
```javascript
return multimodal.response(
  "Rendered Q3 chart.",
  multimodal.image(pngBuffer, "image/png", { resolution: "high" })
);
```

2. **Composite Return Object:**
```javascript
return {
  text: "Generated 1 chart and 1 audio readout.",
  images: ["data:image/png;base64,iVBORw0KGgo..."],
  audio: ["data:audio/mp3;base64,//uQx..."],
  videos: []
};
```

3. **Workspace File Reference:**
```javascript
return {
  filePath: "out/distribution_plot.png",
  text: "Data distribution plotted and ready for visual inspection."
};
```

4. **Gemini Subcontent Parts Array:**
```javascript
return [
  { type: "text", text: "Analysis results below:" },
  { type: "image", data: base64Data, mime_type: "image/png" }
];
```

### Multimodal Code Examples

#### Example 1: Dynamic SVG Chart Generator (Mid-Turn Image to Gemini)
```javascript
async function generate_status_chart(params, userSettings, resources) {
  const { labels, values, title = "System Health Metrics" } = params;
  const width = 600, height = 300;
  const maxVal = Math.max(...values, 1);
  const barWidth = Math.floor((width - 80) / values.length) - 10;

  let svgBars = "";
  values.forEach((v, i) => {
    const barHeight = Math.floor((v / maxVal) * (height - 100));
    const x = 50 + i * (barWidth + 10);
    const y = height - 40 - barHeight;
    svgBars += `<rect x="${x}" y="${y}" width="${barWidth}" height="${barHeight}" fill="#3b82f6" rx="4"/>`;
    svgBars += `<text x="${x + barWidth / 2}" y="${y - 8}" text-anchor="middle" font-size="12" fill="#333">${v}</text>`;
    svgBars += `<text x="${x + barWidth / 2}" y="${height - 20}" text-anchor="middle" font-size="12" fill="#666">${labels[i] || i}</text>`;
  });

  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${width}" height="${height}" style="background:#ffffff;font-family:sans-serif;">
    <text x="${width / 2}" y="30" text-anchor="middle" font-size="16" font-weight="bold" fill="#111">${title}</text>
    <line x1="40" y1="${height - 38}" x2="${width - 20}" y2="${height - 38}" stroke="#ccc" stroke-width="1"/>
    ${svgBars}
  </svg>`;

  // Save to workspace/out/ so the user also gets a downloadable artifact
  await workspace.writeFile("status_chart.svg", svg, "out");

  // Return mid-turn multimodal image to Gemini
  const base64Svg = Buffer.from(svg).toString("base64");
  return multimodal.response(
    `Rendered ${title} with ${values.length} data points.`,
    multimodal.image(base64Svg, "image/svg+xml")
  );
}
```

#### Example 2: Reading a Python-Generated Workspace Plot
```javascript
async function inspect_workspace_plot(params, userSettings, resources) {
  const { fileName = "heatmap.png" } = params;
  // Pass fileName and subDir separately to multimodal.fromWorkspace
  const part = multimodal.fromWorkspace(fileName, "out");
  return multimodal.response(`Loaded ${fileName} from workspace/out/.`, part);
}
```

---

## 8. Secret Vault Access (`vault` / `secrets`)

Gemini Chat Hub features a centralized Secret Vault (`secretVault`) that discovers credentials from environment variables (`*_KEY`, `*_SECRET`, `*_TOKEN`, `*_PAT`, `*_PASSWORD`, etc.) and AES-256-GCM encrypted database settings.

### Key Features
1. **Active Redaction Scrubber:** All stdout, stderr, logs, and persisted step outputs are scrubbed—replacing raw secret values (and their URL-encoded forms) with `[REDACTED_SECRET:<NAME>]`.
2. **Programmatic Access:** Plugins can query secrets by name without requiring users to copy/paste keys.

### JavaScript API Methods
Available globally as `vault`, `secrets`, and via `resources.vault`:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `getSecret` | `vault.getSecret(name: string): string \| undefined` | Retrieves the raw secret value (automatically uppercases `name`). |
| `listSecrets` | `vault.listSecrets(): string[]` | Returns a list of all visible secret names (names only, no values). |
| `hasSecret` | `vault.hasSecret(name: string): boolean` | Returns `true` if the secret exists (automatically uppercases `name`). |

---

## 9. Functional Implementation Specs

Every plugin function starts with an OpenAI Function Spec declaring its input parameters (`spec` or `openaiSpec`):

```json
{
  "name": "query_database_records",
  "description": "Searches internal database records using SQL filters and pagination.",
  "parameters": {
    "type": "object",
    "properties": {
      "table_name": {
        "type": "string",
        "description": "The table to query, e.g. 'users', 'invoices', 'metrics'."
      },
      "limit": {
        "type": "number",
        "description": "Maximum number of records to return (1-100)."
      }
    },
    "required": ["table_name"]
  }
}
```

> [!TIP]
> When many tools are active, Gemini Chat Hub may use **Native Tool Retrieval (deferred tool loading)**. Write clear, keyword-rich `name` and `description` fields so the model can discover and select your tool reliably.

---

## 10. Implementation Methods

### Method A: JavaScript Server Sandbox Execution
Runs in an isolated Node.js VM context (`node:vm`) with a **90-second** execution timeout and `AbortSignal` cancellation support.

#### Function Signature & Entrypoints
Define a named function matching your `openaiSpec.name` (recommended), or fallback to `main` or `run`:
```javascript
async function function_name(params, userSettings, resources) {
  // params: Arguments passed by the model (also available globally as `params` / `parameters`)
  // userSettings: Configured plugin settings + Vault/Gemini fallbacks (also `userSettings` / `user_settings`)
  // resources: { userMessage, chatId, keyTier, secretsLoaded, signal, pluginId, storage, workspace, vault, multimodal }
}
```
> [!NOTE]
> Your code **must** define `async function <functionName>(...)` (or `main` / `run`). Bare top-level `return ...` statements outside a named function are not executed.

#### Globals Available in the Sandbox
- **Web & Network**: `fetch` (auto-wired to sandbox `AbortController`), `Headers`, `Request`, `Response`, `URL`, `URLSearchParams`, `AbortController`, `AbortSignal`, `TextEncoder`, `TextDecoder`, `Blob`, `File`, `FormData`, `Buffer`.
- **Timers**: `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval`, `setImmediate`, `clearImmediate` (all tracked and cleaned up on completion/abort).
- **Builtins**: `Array`, `Object`, `String`, `Number`, `Boolean`, `RegExp`, `Map`, `Set`, `WeakMap`, `WeakSet`, `Promise`, `Symbol`, `Error`, `TypeError`, `RangeError`, `SyntaxError`, `ReferenceError`, `Math`, `Date`, `JSON`.
- **Helpers**: `parseInt`, `parseFloat`, `isNaN`, `isFinite`, `encodeURIComponent`, `decodeURIComponent`, `encodeURI`, `decodeURI`, `btoa`, `atob`.
- **Console**: `console.log`, `console.warn`, `console.error` (auto-prefixed with `[Plugin Sandbox <functionName>]` in server logs).
- **Gemini Chat Hub Extensions**: `storage` (`localStorage`), `workspace` (`runtimeWorkspace`), `vault` (`secrets`), `multimodal`, `params` (`parameters`), `userSettings` (`user_settings`), `resources`.

---

### Method B: HTTP Actions (Declarative Integrations)
Executes HTTP calls without custom JavaScript code (`implementationType: "http"`), with a **30-second** timeout and a **25 MB** response body limit.

#### Supported `http_action` Fields
| Field | Type | Description |
| :--- | :--- | :--- |
| `url` | String | **Required.** Target URL (supports template interpolation). |
| `method` | String | `"GET"` (default), `"POST"`, `"PUT"`, `"DELETE"`, or `"PATCH"`. |
| `headers` | Object | Key-value HTTP headers (defaults `User-Agent: GeminiStudioHub/1.0`, and `Content-Type: application/json` when `body` is present). |
| `body` | String \| Object | Request body (sent when `method` is not `GET` or `HEAD`). |
| `has_post_processing` | Boolean | Enables response transformation via JMESPath or Handlebars. |
| `post_processing_type` | String | `"jmespath"` or `"handlebars"`. |
| `post_processing_code` | String | JMESPath query expression or Handlebars template (receives response fields + `__VARIABLES`). |
| `save_to_workspace` | Boolean | Saves response payload to `/workspace/in/`. |
| `workspace_filename` | String | Custom filename in `/workspace/in/` (supports interpolation; also triggers workspace save). |

#### Dynamic Variable & Vault Interpolation
Placeholders are replaced across `url`, `headers`, `body`, and `workspace_filename` from `{ ...userSettings, ...params, CHAT_ID }` (`params` takes precedence over `userSettings` on key collisions):
- `{{VAULT:SECRET_NAME}}` or `{{vault.SECRET_NAME}}`: Injects secret from Secret Vault.
- `{param}` or `{{param}}`: Injects function argument or user setting.
- `{CHAT_ID}`: Injects current conversation ID.

#### Post-Processing & Workspace Auto-Save
- **Post-Processing:** For non-binary responses (`application/json`, `text/*`), `jmespath` extracts fields and `handlebars` formats text using the response object plus `__VARIABLES`.
- **Binary Media Interception:** When an endpoint returns `image/*`, `audio/*`, `video/*`, or `application/pdf`, post-processing is skipped and the HTTP runner automatically converts the binary buffer to base64 for multimodal normalization.
- **Workspace Auto-Save:** When `save_to_workspace: true` or `workspace_filename` is set, the response (JSON, text, or raw binary) is written to `/workspace/in/<filename>` and returned alongside `workspaceFile` and `agentNote`.

```json
{
  "http_action": {
    "url": "https://api.chartgenerator.com/v1/render?symbol={symbol}&format=png",
    "method": "GET",
    "headers": {
      "Authorization": "Bearer {{VAULT:CHART_API_KEY}}"
    },
    "save_to_workspace": true,
    "workspace_filename": "{symbol}_chart.png",
    "has_post_processing": false
  }
}
```

---

## 11. Output Destinations (`outputType` / `output_destination`)

| `outputType` | Normalized `displayMode` | Current Behavior in Gemini Chat Hub |
| :--- | :--- | :--- |
| `"respond_to_ai"` (Default) | `"ai"` | Returns output (text + multimodal images) to Gemini in the function-calling loop; tool payload drawer starts collapsed. |
| `"render_markdown"` | `"markdown"` | Auto-expands the tool payload drawer in the UI and defaults to formatted Markdown rendering. **Note:** Output is still returned to Gemini to complete the turn. |
| `"display_html_to_user"` | `"html"` | Normalized to `displayMode: "html"`. **Limitation:** No interactive HTML/JS iframe widget is currently rendered in the tool trace UI; falls back to standard payload display. |
| `"card"` | `"card"` | Normalized to `displayMode: "card"`. **Limitation:** No dedicated card gallery renderer is currently implemented in the tool trace UI; falls back to standard payload display. |

---

## 12. Platform Limitations & Gotchas

1. **No Node.js Built-ins or `npm` Imports in JS Sandbox:** `require()`, `import`, `process`, `fs`, `path`, `crypto`, and `child_process` are **not** available in the VM. Use `fetch`, `Buffer`, `btoa`/`atob`, `workspace`, `storage`, `vault`, and `multimodal`, or pair your plugin with `execute_python` / `execute_bash` via `/workspace/in/`.
2. **No Browser DOM APIs:** `window`, `document`, and synchronous `localStorage.getItem()` do not exist. Always use `await storage.get()`.
3. **Timeouts & Size Caps:**
   - JS Sandbox: **90 seconds** (`120 seconds` parent run-engine guard).
   - HTTP Actions: **30 seconds** timeout, **25 MB** max response size.
   - Remote Multimodal Fetch: **15 seconds** timeout (images and audio only).
4. **Audio, Video & PDFs Mid-Turn:** Only **images** are sent visually to Gemini mid-turn. Audio and video play in the UI but are seen by Gemini as text placeholders.
5. **Declarative Manifest Flags Are Not Enforced:** `localStorage.enabled`, `localStorage.initialData`, `workspaceAccess.allowedDirectories`, `permissions`, and `vaultSecrets` are metadata only. `initialData` is not auto-seeded, and all 4 runtime helpers (`storage`, `workspace`, `vault`, `multimodal`) are always available.
6. **Live Voice Mode Context:** In Live Voice Mode, `executePluginTool` receives `{ chatId }` only—`resources.userMessage`, `resources.keyTier`, and `resources.signal` are undefined. Code defensively with optional chaining (`resources.signal?.aborted`).
7. **Workspace Filename Flattening:** `workspace.writeFile` / `readFile` sanitize paths to `path.basename()`—nested subdirectories inside `in/`, `out/`, or `tmp/` are not supported.

---

## 13. Canonical Gemini Chat Hub Plugin Manifest

```json
{
  "id": "visual_financial_analyst",
  "uuid": "7a32b904-4c8d-4f12-9856-cb8211993214",
  "name": "Visual Financial Analyst",
  "title": "Visual Financial Analyst",
  "emoji": "📊",
  "description": "Pulls financial metrics, caches ticker data, saves SVG charts to workspace/out, and returns multimodal charts mid-turn directly to Gemini.",
  "version": "2.2.0",
  "author": "Financial Engineering Group",
  "iconURL": "https://api.iconify.design/lucide:candlestick-chart.svg",
  "contextPrompt": "When analyzing tickers with this plugin, always cite price figures with 2 decimal places and explain chart patterns.",
  "localStorage": {
    "enabled": true,
    "initialData": { "tracked_tickers": ["AAPL", "GOOGL", "NVDA"] }
  },
  "workspaceAccess": {
    "enabled": true,
    "allowedDirectories": ["in", "out", "tmp"]
  },
  "vaultSecrets": ["FINANCIAL_API_KEY"],
  "userSettings": [
    {
      "name": "apiKey",
      "label": "Market API Key",
      "type": "password",
      "required": false,
      "vaultSecret": "FINANCIAL_API_KEY",
      "description": "Enter your market data API key or link FINANCIAL_API_KEY from Vault."
    }
  ],
  "pluginFunctions": [
    {
      "id": "fn_render_stock_trend_chart",
      "name": "Render Stock Trend Chart & Return to Gemini",
      "implementationType": "javascript",
      "outputType": "respond_to_ai",
      "returnsMultimodal": true,
      "multimodalTypes": ["image"],
      "permissions": ["local_storage", "workspace", "vault", "multimodal"],
      "openaiSpec": {
        "name": "render_stock_trend_chart",
        "description": "Generates a high-resolution trend chart for a stock symbol, saves SVG to workspace/out, and returns a multimodal image part mid-turn directly to Gemini for visual analysis.",
        "parameters": {
          "type": "object",
          "properties": {
            "symbol": {
              "type": "string",
              "description": "Stock ticker symbol (e.g. 'AAPL', 'NVDA')."
            },
            "points": {
              "type": "array",
              "items": { "type": "number" },
              "description": "Array of recent closing prices to plot."
            }
          },
          "required": ["symbol", "points"]
        }
      },
      "code": "async function render_stock_trend_chart(params, userSettings, resources) {\n  const { symbol, points = [] } = params;\n  const max = Math.max(...points, 1);\n  const min = Math.min(...points, 0);\n  const range = max - min || 1;\n\n  const width = 500;\n  const height = 200;\n  const step = points.length > 1 ? (width - 60) / (points.length - 1) : width;\n\n  const polylinePoints = points.map((pt, i) => {\n    const x = 30 + i * step;\n    const y = height - 30 - ((pt - min) / range) * (height - 60);\n    return `${x},${y}`;\n  }).join(' ');\n\n  const svg = `<svg xmlns=\"http://www.w3.org/2000/svg\" width=\"${width}\" height=\"${height}\" style=\"background:#1e1e2e;color:#cdd6f4;font-family:sans-serif;\">\n    <text x=\"20\" y=\"25\" fill=\"#cdd6f4\" font-weight=\"bold\">${symbol} Price Trend</text>\n    <polyline fill=\"none\" stroke=\"#a6e3a1\" stroke-width=\"3\" points=\"${polylinePoints}\"/>\n  </svg>`;\n\n  await workspace.writeFile(`${symbol.toLowerCase()}_trend.svg`, svg, 'out');\n\n  const base64Svg = Buffer.from(svg).toString('base64');\n  return multimodal.response(\n    `Rendered ${symbol} price trend with ${points.length} points.`,\n    multimodal.image(base64Svg, 'image/svg+xml')\n  );\n}"
    }
  ]
}
```

---

## 14. Verification & Debugging Checklist

Before importing your plugin into Gemini Chat Hub:
1. **Valid Function Entrypoint:** Does your JS code define `async function <openaiSpec.name>(params, userSettings, resources)` (or `main` / `run`)?
2. **Server-Side Sandbox Only:** No `window`, `document`, `require()`, `import`, `process`, or `fs`. Use `await storage.get()` (not synchronous `localStorage`).
3. **Enum Settings:** Use `"values": [...]` (not `"enum"`) for `type: "enum"` user settings.
4. **Multimodal Accuracy:** Use `multimodal.fromWorkspace(fileName, "out")` (two arguments, synchronous) and remember only images are visually perceived by Gemini mid-turn.
5. **Workspace File Locations:** Write inputs for Python/Bash to `"in"` and user-downloadable deliverables to `"out"` (using flat filenames, no nested folders).
6. **Secret Vault Safety:** Never log or return raw API keys; use `vault.getSecret()` or `{{VAULT:SECRET_NAME}}` in `userSettings` / HTTP Actions.
7. **Defensive `resources` Access:** Guard optional properties (`resources.userMessage?.text`, `resources.signal?.aborted`) so the plugin also works in Live Voice Mode.
8. **Actionable Errors:** Throw descriptive errors (`throw new Error(...)`) so Gemini can self-correct or inform the user.
