---
name: vscode-extension-guide-en
description: "Build, test, package, and publish Visual Studio Code extensions. Use for extension APIs, webviews, TreeViews, Marketplace distribution, Copilot customizations, or extension troubleshooting."
license: CC BY-NC-SA 4.0
metadata:
  author: lewiswigmore
  original-author: yamapan (https://github.com/aktsmm)
---

# VS Code extension guide

Use the references below for extension APIs, testing, publishing, and troubleshooting.

## Quick start

```bash
# Scaffold without a global install
npx --package yo --package generator-code -- yo code

# Or minimal manual setup
mkdir my-extension && cd my-extension
npm init -y && npm install -D typescript @types/vscode
```

## Project structure

```
my-extension/
├── package.json          # Extension manifest
├── src/extension.ts      # Entry point
├── out/                  # Compiled JS (gitignore)
├── images/icon.png       # At least 128x128 PNG for Marketplace
└── .vscodeignore         # Exclude files from VSIX
```

## Building and packaging

```bash
npm run compile      # Build once
npm run watch        # Watch mode (F5 to launch debug)
npx @vscode/vsce package   # Creates .vsix
```

## Done criteria

- [ ] Extension activates without errors
- [ ] All commands registered and working
- [ ] Inspect VSIX contents with `npx @vscode/vsce ls` and confirm its size is appropriate
- [ ] README.md includes Marketplace/GitHub links

## Quick troubleshooting

| Symptom               | Fix |
| --------------------- | --- |
| Extension not loading | Check activation events and `engines.vscode`; contributed commands/views auto-activate since VS Code 1.74 |
| Command not found     | Match command ID in `package.json` and `registerCommand` |
| Shortcut not working  | Check the `when` clause and keybinding conflicts |

## References

| Topic               | Reference                                                            |
| ------------------- | -------------------------------------------------------------------- |
| AI Customization    | [references/ai-customization.md](references/ai-customization.md)     |
| Code Review Prompts | [references/code-review-prompts.md](references/code-review-prompts.md) |
| TreeView            | [references/treeview.md](references/treeview.md)                     |
| Webview             | [references/webview.md](references/webview.md)                       |
| Testing             | [references/testing.md](references/testing.md)                       |
| Publishing          | [references/publishing.md](references/publishing.md)                 |
| Troubleshooting     | [references/troubleshooting.md](references/troubleshooting.md)       |
| Extension management | [Install, disable, update, and manage extensions](https://code.visualstudio.com/docs/configure/extensions/extension-marketplace) |

## Best practices

### Naming consistency

Unify package name, setting keys, command IDs, and view IDs before publishing:

| Item | Example |
|------|---------|
| Package name | `copilot-scheduler` |
| Setting key | `copilotScheduler.enabled` |
| Command ID | `copilotScheduler.createTask` |
| View ID | `copilotSchedulerTasks` |

### Centralised notification handling

```typescript
type NotificationMode = "toast" | "silentStatus";

function getNotificationMode(): NotificationMode {
  const config = vscode.workspace.getConfiguration("myExtension");
  return config.get<NotificationMode>("notificationMode", "toast");
}

function notifyInfo(message: string, timeoutMs = 4000): void {
  const mode = getNotificationMode();
  switch (mode) {
    case "silentStatus":
      vscode.window.setStatusBarMessage(message, timeoutMs);
      break;
    default:
      void vscode.window.showInformationMessage(message);
  }
}

function notifyError(message: string, timeoutMs = 6000): void {
  const mode = getNotificationMode();
  if (mode === "silentStatus") {
    vscode.window.setStatusBarMessage(`⚠ ${message}`, timeoutMs);
    console.error(message);
    return;
  }
  void vscode.window.showErrorMessage(message);
}
```

Expose a `notificationMode` setting so users can control how notifications are delivered.
