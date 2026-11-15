# Troubleshooting

Common issues and solutions for VS Code extension development.

## Extension not loading

| Symptom                     | Cause                      | Solution                                                         |
| --------------------------- | -------------------------- | ---------------------------------------------------------------- |
| Extension never activates   | Missing or mismatched activation trigger | Check the manifest's `contributes` entry, `engines.vscode`, and the [activation event reference](https://code.visualstudio.com/api/references/activation-events); add only the event the extension actually needs |
| "Extension is not active"   | Extension host or activation issue | Check the Extension Host log and use the narrowest applicable activation event |
| Works in dev, not installed | Build output not included  | Check `.vscodeignore`, ensure `out/` is included                 |
| Update not visible          | Auto-update is off/delayed, or extension host needs restart | Check `@updates` in the Extensions view, update manually if needed, and restart extensions when prompted; see [extension management](https://code.visualstudio.com/docs/configure/extensions/extension-marketplace) |

Since VS Code 1.74, activation events for contributed commands and views are generated automatically. Don't add a broad `*` or `onStartupFinished` trigger just to compensate for a command that fails to activate.

In VS Code 1.125, `extensions.autoUpdate` was simplified to `on` or `off`, and disabled extensions no longer update automatically until enabled. Check the current extension management guide before reusing older settings examples.

### Debug activation

```typescript
// Add at top of activate()
console.log("Extension activating...");
vscode.window.showInformationMessage("Extension activated!");
```

Check: **Help** → **Toggle Developer Tools** → **Console**

## Command not found

| Symptom                   | Cause                        | Solution                                             |
| ------------------------- | ---------------------------- | ---------------------------------------------------- |
| "command not found"       | ID mismatch                  | Ensure same ID in package.json and registerCommand() |
| Command not in palette    | Missing contributes.commands | Add command definition to package.json               |
| Command defined but fails | Extension not activated      | Check the extension host; contributed commands auto-activate on VS Code 1.74+ |

### Verify command registration

```typescript
// In activate()
const commands = await vscode.commands.getCommands();
console.log(
  "Registered:",
  commands.filter((c) => c.includes("myExt")),
);
```

## Keyboard shortcuts not working

| Symptom               | Cause                             | Solution                           |
| --------------------- | --------------------------------- | ---------------------------------- |
| Shortcut does nothing | `when` clause doesn't match        | Check context keys and the active editor/focus state |
| Works sometimes       | Context-dependent `when`          | Check active editor, focus state   |
| Conflict with other   | Another extension/VS Code uses it | Use unique key combination         |

### Check for conflicts

1. **Ctrl+K Ctrl+S** → Open Keyboard Shortcuts
2. Search for your key combination
3. Look for conflicts (multiple entries)

### Keybinding context example

```json
{
  "contributes": {
    "keybindings": [{
      "key": "ctrl+alt+m",
      "command": "myExt.open",
      "when": "editorTextFocus"
    }]
  }
}
```

## Packaging issues

| Symptom               | Cause                  | Solution                                    |
| --------------------- | ---------------------- | ------------------------------------------- |
| VSIX too large        | node_modules included  | Add to .vscodeignore                        |
| Files missing in VSIX | Over-aggressive ignore | Use `npx @vscode/vsce ls` to check          |
| Icon not showing      | Wrong path or format   | Use a PNG at least 128x128, and check its path in package.json |

### Inspect VSIX contents

```bash
# List what will be packaged
npx @vscode/vsce ls

# Extract and inspect VSIX
unzip -l my-extension-1.0.0.vsix
```

## Publishing errors

| Symptom             | Cause                  | Solution                                 |
| ------------------- | ---------------------- | ---------------------------------------- |
| PAT invalid         | Legacy PAT invalid or expired | Prefer Trusted Publishing or Entra ID; see [secure publishing](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#secure-automated-publishing-to-visual-studio-marketplace) |
| Publisher not found | ID mismatch            | Verify publisher ID matches exactly      |
| Version exists      | Already published      | Increment version number                 |
| README not showing  | Wrong filename case    | Must be `README.md` not `README.MD`      |

## Runtime errors

| Symptom              | Cause                  | Solution                                            |
| -------------------- | ---------------------- | --------------------------------------------------- |
| "Cannot find module" | Dependency not bundled | Add to dependencies (not devDependencies) or bundle |
| API undefined        | Wrong VS Code version  | Check `engines.vscode` matches API used             |
| Permission denied    | Restricted API         | Check extension permissions/capabilities            |

### Check the VS Code API version

```json
{
  "engines": {
    "vscode": "^1.80.0"
  }
}
```

## Debugging tips

### Enable verbose logging

```typescript
const outputChannel = vscode.window.createOutputChannel("My Extension");
outputChannel.appendLine("Debug message");
outputChannel.show();
```

### Extension host logs

1. **Help** → **Toggle Developer Tools**
2. **Console** tab
3. Filter by your extension name

### Reload without restarting

- **Ctrl+Shift+P** → "Developer: Reload Window"

## Quick fixes

```bash
# Clean rebuild
rm -rf out/ node_modules/
npm install
npm run compile

# Reset installed extension
code --uninstall-extension publisher.extension-id
npx @vscode/vsce package
code --install-extension ./extension-1.0.0.vsix

# Check what's in your VSIX
npx @vscode/vsce ls
```

## Blank webview or SyntaxError

| Symptom | Cause | Solution |
|---------|-------|----------|
| Blank white screen | JavaScript SyntaxError | Check Webview DevTools Console for errors |
| `Invalid regular expression: /^*/` | Backslash stripped from regex | Double-escape in templates (`\\d`, `\\s`) |
| `Unexpected token` | Quotes broken during minification | Switch to `data-action` + event delegation pattern |
| Button not responding | `onclick` lost after innerHTML | Use `document.addEventListener` with delegation |

### Debugging steps

1. Run **Developer: Open Webview Developer Tools**
2. Check the Console tab for errors
3. Search for the relevant line in build output `out/extension.js`
4. Fix regex/quote issues in source and rebuild

## Naming mismatch

| Symptom | Cause | Solution |
|---------|-------|----------|
| Setting has no effect | Setting key doesn't match code | Align package.json and `getConfiguration()` |
| Command not found | Command ID doesn't match package.json | Use the same ID everywhere |

### Naming consistency check

```bash
# Extract command/setting keys from package.json
grep -E '"myExt\.' package.json

# Search for usage in source code
grep -r "myExt\." src/
```

Align names before publishing. Changing setting keys later can break existing users' settings.
