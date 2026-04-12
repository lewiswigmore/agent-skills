# Webview Implementation

Create rich HTML-based UI panels in VS Code.

## Basic Webview Panel

```typescript
import * as vscode from "vscode";

export function createWebviewPanel(context: vscode.ExtensionContext) {
  const panel = vscode.window.createWebviewPanel(
    "myWebview", // Identifier
    "My Webview", // Title
    vscode.ViewColumn.One, // Editor column
    {
      enableScripts: true, // Enable JavaScript
      retainContextWhenHidden: true, // Keep state when hidden
      localResourceRoots: [
        // Allowed local resources
        vscode.Uri.joinPath(context.extensionUri, "media"),
      ],
    },
  );

  panel.webview.html = getWebviewContent(panel.webview, context.extensionUri);

  return panel;
}
```

## HTML Content

```typescript
function getWebviewContent(
  webview: vscode.Webview,
  extensionUri: vscode.Uri,
): string {
  // Get URI for local resources
  const styleUri = webview.asWebviewUri(
    vscode.Uri.joinPath(extensionUri, "media", "style.css"),
  );
  const scriptUri = webview.asWebviewUri(
    vscode.Uri.joinPath(extensionUri, "media", "main.js"),
  );

  // CSP nonce for security
  const nonce = getNonce();

  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'none'; style-src ${webview.cspSource}; script-src 'nonce-${nonce}';">
  <link href="${styleUri}" rel="stylesheet">
</head>
<body>
  <h1>Hello Webview!</h1>
  <button id="btn">Click Me</button>
  <script nonce="${nonce}" src="${scriptUri}"></script>
</body>
</html>`;
}

function getNonce(): string {
  let text = "";
  const chars =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
  for (let i = 0; i < 32; i++) {
    text += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return text;
}
```

### Embed initial data safely (no document.write)

```html
<script id="initial-data" type="application/json">
  ${serializeForWebview(initialData)}
</script>
<script nonce="${nonce}">
  (function () {
    var vscode = acquireVsCodeApi();
    var initialData = {};
    try {
      var el = document.getElementById("initial-data");
      if (el && el.textContent) initialData = JSON.parse(el.textContent) || {};
    } catch (e) {
      initialData = {};
    }
    // ... use initialData ...
  })();
</script>
```

- Avoid Base64 + `document.write`; inject JSON as text and parse.
- Escape `<`, U+2028/2029 before embedding to keep the script tag valid.
- Keep the CSP nonce on the executable script only.

## Message Passing

### Extension → Webview

```typescript
// In extension
panel.webview.postMessage({ command: "update", data: { count: 42 } });
```

```javascript
// In webview (media/main.js)
window.addEventListener("message", (event) => {
  const message = event.data;
  if (message.command === "update") {
    console.log("Count:", message.data.count);
  }
});
```

### Webview → Extension

```javascript
// In webview
const vscode = acquireVsCodeApi();

document.getElementById("btn").addEventListener("click", () => {
  vscode.postMessage({ command: "buttonClicked", text: "Hello!" });
});
```

```typescript
// In extension
panel.webview.onDidReceiveMessage(
  (message) => {
    switch (message.command) {
      case "buttonClicked":
        vscode.window.showInformationMessage(message.text);
        return;
    }
  },
  undefined,
  context.subscriptions,
);
```

## State Persistence

```javascript
// In webview - save state
const vscode = acquireVsCodeApi();
vscode.setState({ count: 5 });

// Restore state
const state = vscode.getState();
if (state) {
  console.log("Restored count:", state.count);
}
```

## VS Code Theme Integration

Use CSS variables for consistent theming:

```css
/* media/style.css */
body {
  font-family: var(--vscode-font-family);
  font-size: var(--vscode-font-size);
  color: var(--vscode-foreground);
  background-color: var(--vscode-editor-background);
}

button {
  background-color: var(--vscode-button-background);
  color: var(--vscode-button-foreground);
  border: none;
  padding: 8px 16px;
  cursor: pointer;
}

button:hover {
  background-color: var(--vscode-button-hoverBackground);
}
```

## Sidebar Webview (WebviewViewProvider)

For webviews in the sidebar instead of editor panels:

```typescript
class MyWebviewProvider implements vscode.WebviewViewProvider {
  resolveWebviewView(webviewView: vscode.WebviewView) {
    webviewView.webview.options = { enableScripts: true };
    webviewView.webview.html = getWebviewContent();
  }
}

// Register in extension.ts
vscode.window.registerWebviewViewProvider(
  "myExtSidebarView",
  new MyWebviewProvider(),
);
```

```json
"contributes": {
  "views": {
    "explorer": [{
      "type": "webview",
      "id": "myExtSidebarView",
      "name": "My Webview"
    }]
  }
}
```

## Fallback Patterns

### Promise-based Callback Fallback

When using Promise-based callbacks (e.g., `resolveCreate`), always provide a fallback mechanism:

```typescript
// ❌ Bad: Single callback dependency
case "createTask": {
  if (!resolveCreate) {
    return; // Silent failure if callback not set
  }
  resolveCreate(data);
  break;
}

// ✅ Good: Fallback to alternative handler
case "createTask": {
  const result = buildResult(data);
  if (resolveCreate) {
    resolveCreate(result);
    resolveCreate = undefined;
  } else if (onAction) {
    // Fallback to action handler
    onAction({ action: "create", data: result });
  }
  break;
}
```

### VS Code Internal API Fallback

When using internal/unstable APIs (`vscode.lm`, `vscode.chat`), always implement fallback:

```typescript
// ✅ Good: API availability check + fallback
static async getAvailableModels(): Promise<Model[]> {
  const models: Model[] = [{ id: "", name: "Default" }];

  try {
    if (typeof vscode.lm !== "undefined" && "selectChatModels" in vscode.lm) {
      const available = await (vscode.lm as any).selectChatModels({});

      // Null check for API result
      if (available && Array.isArray(available)) {
        for (const model of available) {
          models.push({
            id: model.id || model.family,
            name: model.name || model.family || model.id,
          });
        }
      }
    }
  } catch (error) {
    console.log("API not available, using fallback", error);
  }

  // Return fallback if API returned nothing useful
  if (models.length <= 1) {
    return getFallbackModels();
  }

  return models;
}
```

### Path Consistency

When handling both local and global paths, use consistent format:

```typescript
// ❌ Bad: Mixed path formats
templates.push({
  source: "local",
  path: relativePath, // Relative
});
templates.push({
  source: "global",
  path: file.fsPath, // Absolute - inconsistent!
});

// ✅ Good: Consistent relative paths
templates.push({
  source: "local",
  path: path.relative(workspaceRoot, file.fsPath).replace(/\\/g, "/"),
});
templates.push({
  source: "global",
  path: path.relative(globalRoot, file.fsPath).replace(/\\/g, "/"),
});
```

## Reliable Webview Communication Pattern

### Recommended Pattern (Simple & Reliable)

Wrap the entire webview script in an IIFE and send webviewReady at the end:

` ypescript
function getWebviewContent(): string {
return <!DOCTYPE html>

<html>
<head>
  <meta charset="UTF-8">
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'none'; style-src ${webview.cspSource} 'unsafe-inline'; script-src 'nonce-${nonce}';">
</head>
<body>
  <div id="app"></div>
  
  <script nonce="${nonce}">
    (function() {
      const vscode = acquireVsCodeApi();
      
      // Initialize UI
      // ... DOM setup ...
      
      // Handle messages from extension
      window.addEventListener('message', event => {
        const message = event.data;
        switch (message.type) {
          case 'updateData':
            renderData(message.data);
            break;
        }
      });
      
      // Initial render
      renderUI();
      
      // Notify extension that webview is ready (LAST!)
      vscode.postMessage({ type: 'webviewReady' });
    })();
  </script>
</body>
</html>;
}
`

### Extension Side Handler

`	ypescript
panel.webview.onDidReceiveMessage(message => {
  switch (message.type) {
    case 'webviewReady':
      console.log('[Extension] Webview reported ready');
      webviewReady = true;
      // Send initial data AFTER webview is ready
      panel.webview.postMessage({
        type: 'updateAgents',
        agents: cachedAgents,
      });
      panel.webview.postMessage({
        type: 'updateModels', 
        models: cachedModels,
      });
      break;
  }
});
`

### ❌ Anti-Pattern: Complex Handshakes

Avoid adding complexity like ping/ACK/retry/fallback mechanisms:

` ypescript
// ❌ Bad: Overly complex handshake
let webviewReadyAcked = false;
let webviewReadyRetryTimer = null;
let webviewReadyAttempts = 0;

function startWebviewReadyHandshake() {
sendWebviewReady();
webviewReadyRetryTimer = setInterval(() => {
if (webviewReadyAcked || webviewReadyAttempts >= 12) {
clearInterval(webviewReadyRetryTimer);
return;
}
sendWebviewReady(); // Retry
}, 500);
}

// Host pings webview, webview responds, host ACKs...
// This adds complexity and often fails in unexpected ways
`

**Why it fails:**

- More moving parts = more failure modes
- Race conditions between ping/ACK/retry timers
- Fallback mechanisms mask the real problem

**Simple pattern is best:** One webviewReady message at script end, host waits for it.

### Debugging Tips

1. **Host logs vs Webview logs are separate**
   - Extension host: console.log() appears in Debug Console
   - Webview: console.log() appears in Webview Developer Tools
   - Use Developer: Open Webview Developer Tools command

2. **Verify script execution via message**
   `javascript
// First line after acquireVsCodeApi
vscode.postMessage({ type: 'scriptStarted' });
`
   If host receives this, script is running. If not, check CSP/nonce.

3. **Check CSP errors in Webview DevTools**
   - Open Webview Developer Tools
   - Look for CSP violation errors in Console

## Webview JavaScript Anti-Patterns

Webviews run in modern Chromium, so ES6+ features (arrow functions, `const`/`let`, default parameters) are fully supported. The following patterns address real webview-specific pitfalls.

### 1. Always Check for null

`getElementById` can return null. Always perform a null check:

```javascript
// ❌ Bad: No null check
document.getElementById("my-input").value = "xxx";

// ✅ Good: With null check
var element = document.getElementById("my-input");
if (element) element.value = "xxx";
```

### 2. Prefer Event Delegation

Direct event registration on dynamically rendered NodeLists can fail. Use event delegation instead:

```javascript
// ❌ Bad: Direct event registration on NodeList
document.querySelectorAll(".btn").forEach(function (btn) {
  btn.addEventListener("click", handleClick);
});

// ✅ Good: Event delegation
document.addEventListener("click", function (e) {
  var target = e.target;
  if (target && target.classList && target.classList.contains("btn")) {
    e.preventDefault();
    handleClick(target);
  }
});
```

### 3. Embed Initial Data

Don't hardcode "Loading..." — embed initial data directly when available:

```typescript
// ❌ Bad: Hardcoded loading state
return `<select id="agent-select">
  <option value="">Loading...</option>
</select>`;

// ✅ Good: Embed initial data if available
const options =
  agents.length > 0
    ? agents.map((a) => `<option value="${a.id}">${a.name}</option>`).join("")
    : '<option value="">Loading...</option>';
return `<select id="agent-select">${options}</select>`;
```

### 4. Async Fallbacks

Always provide try/catch and fallback data for API calls:

```typescript
// ❌ Bad: No fallback
async function getModels(): Promise<Model[]> {
  return await vscode.lm.selectChatModels({});
}

// ✅ Good: With fallback
async function getModels(): Promise<Model[]> {
  try {
    const models = await vscode.lm.selectChatModels({});
    if (models && models.length > 0) {
      return models;
    }
  } catch {
    // API may not be available
  }
  return getFallbackModels();
}
```

### 5. Bundle Actions with data-action + Delegation

```javascript
// ✅ Good: render attributes, delegate once
function renderTasks(tasks) {
  return tasks
    .map(function (task) {
      var id = escapeAttr(task.id || "");
      return '<button data-action="run" data-id="' + id + '">Run</button>';
    })
    .join("");
}

document.addEventListener("click", function (e) {
  var target = e.target;
  var host =
    target && typeof target.closest === "function"
      ? target.closest("[data-action]")
      : null;
  if (!host) return;
  var action = host.getAttribute("data-action");
  var id = host.getAttribute("data-id");
  if (!action || !id) return;
  if (action === "run") window.runTask(id);
  if (action === "edit") window.editTask(id);
  // ... other actions ...
});
```

- ❌ Avoid inline `onclick="..."` (prone to broken quotes and SyntaxErrors after minification).
- ❌ Avoid TypeScript cast strings (`as HTMLElement` can leak into HTML and cause SyntaxErrors).
- ✅ Always escape attributes and handle events via delegation.

### 6. Post-Build HTML Sanity Check

- Output a `debug-webview.html` at build time and open it in a browser/VS Code to check for SyntaxErrors.
- Inspect the Webview Developer Tools Console for CSP violations, broken quotes, or `document.write` errors.
- Manually test key interactions (tab switching, dropdowns, etc.) once and verify no errors appear in the log.

## Double-Escaping Regex Literals

When writing regular expressions inside template literals, backslashes get stripped:

```typescript
// ❌ Bad: Backslash gets stripped in template literal
const html = `<script>var everyN = /^\*\/(\d+)$/.exec(minute);</script>`;
// Result in browser: /^*/(\d+)$/ → SyntaxError: Nothing to repeat

// ✅ Good: Double-escape backslashes
const html = `<script>var everyN = /^\\*\\/(\\d+)$/.exec(minute);</script>`;
// Result in browser: /^\*\/(\d+)$/ → Works correctly
```

**Affected patterns:**
- `\d` → `\\d`
- `\s` → `\\s`
- `\*` → `\\*`
- `\/` → `\\/`

**How to debug:**
1. Open Webview Developer Tools
2. Look for `Invalid regular expression: /^*/: Nothing to repeat` in Console
3. Check the regex in build output (`out/extension.js`)

## Reacting to Configuration Changes

Pattern for immediately re-rendering the webview when settings (e.g. language) change:

```typescript
// extension.ts
const configWatcher = vscode.workspace.onDidChangeConfiguration((e) => {
  if (e.affectsConfiguration("myExtension.language")) {
    // Re-render webview with new language
    MyWebview.refreshLanguage(getCurrentData());
  }
  
  if (
    e.affectsConfiguration("myExtension.globalPromptsPath") ||
    e.affectsConfiguration("myExtension.globalAgentsPath")
  ) {
    // Clear cache and re-fetch
    void refreshCachedData(true);
  }
});

context.subscriptions.push(configWatcher);
```

```typescript
// webview.ts
static refreshLanguage(data: any[]): void {
  if (this.panel) {
    // Dispose and recreate the panel (reflects language change)
    this.panel.dispose();
    this.panel = undefined;
    void this.show(this.extensionUri, data, this.onAction);
  }
}
```

## Moving Inline JS to an External File (Recommended)

Large inline `<script>` blocks inside TypeScript template literals are hard to edit,
cause merge conflicts, and slow down the webview parse step. Prefer an external file.

### Anti-pattern (inline)

```typescript
// BAD – hundreds of lines of JS buried in a TS template literal
return `<html>...
  <script nonce="${nonce}">
    // 1000 lines of JS here
  </script>
</html>`;
```

### Preferred pattern (external file)

```
my-extension/
├── media/
│   └── webview.js     ← all webview logic lives here
└── src/
    └── myWebview.ts   ← only HTML skeleton + initial-data injection
```

```typescript
// myWebview.ts – only the skeleton remains in TypeScript
const scriptUri = webview.asWebviewUri(
  vscode.Uri.joinPath(extensionUri, "media", "webview.js"),
);

return `<!DOCTYPE html>
<html>
<head>
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'none';
                 style-src ${webview.cspSource} 'unsafe-inline';
                 script-src 'nonce-${nonce}';
                 img-src ${webview.cspSource};
                 font-src ${webview.cspSource};">
</head>
<body>
  <script nonce="${nonce}" id="initial-data"
          type="application/json">${serializeForWebview(data)}</script>
  <script nonce="${nonce}" src="${scriptUri}"></script>
</body>
</html>`;
```

```typescript
// Tighten localResourceRoots to only what is needed
this.panel = vscode.window.createWebviewPanel(
  "myWebview", "My View", vscode.ViewColumn.One,
  {
    enableScripts: true,
    retainContextWhenHidden: true,
    localResourceRoots: [
      vscode.Uri.joinPath(extensionUri, "media"),  // JS / CSS
      vscode.Uri.joinPath(extensionUri, "images"), // icons
      // ❌ Don't pass extensionUri directly – too broad
    ],
  },
);
```

### Serialising initial data safely

```typescript
function serializeForWebview(value: unknown): string {
  const json = JSON.stringify(value ?? null) ?? "null";
  return json
    .replace(/</g, "\\u003c")
    .replace(/\u2028/g, "\\u2028")
    .replace(/\u2029/g, "\\u2029");
}
```

```javascript
// media/webview.js – read the injected data
(function () {
  var vscode = acquireVsCodeApi();
  var initialData = {};
  try {
    var el = document.getElementById("initial-data");
    if (el) initialData = JSON.parse(el.textContent || "{}");
  } catch (e) { /* ignore */ }
  // … use initialData …
})();
```

## Prompting Reload After Extension Update

Because `activationEvents: ["onStartupFinished"]` fires only once per VS Code
startup, users who update the extension **without restarting VS Code** will keep
running stale code. Show a "Reload Now" notification when the version changes.

```typescript
// extension.ts
const LAST_VERSION_KEY = "lastKnownVersion";

export function activate(context: vscode.ExtensionContext): void {
  const currentVersion =
    (context.extension.packageJSON as { version?: string }).version ?? "0.0.0";
  const lastVersion = context.globalState.get<string>(LAST_VERSION_KEY);

  if (lastVersion && lastVersion !== currentVersion) {
    void vscode.window
      .showInformationMessage(
        `Extension updated to v${currentVersion}. Reload to activate.`,
        "Reload Now",
      )
      .then((choice) => {
        if (choice === "Reload Now") {
          void vscode.commands.executeCommand("workbench.action.reloadWindow");
        }
      });
  }
  void context.globalState.update(LAST_VERSION_KEY, currentVersion);
}
```

> **Why not `vscode.env.reload()`?** It reloads immediately without user consent.
> The pattern above lets users finish their current work first.
