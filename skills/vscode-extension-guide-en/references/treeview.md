# TreeView implementation

Create sidebar views for your VS Code extension.

## package.json configuration

```json
{
  "contributes": {
    "viewsContainers": {
      "activitybar": [{
        "id": "myExtContainer",
        "title": "My Extension",
        "icon": "images/icon.svg"
      }]
    },
    "views": {
      "myExtContainer": [{
        "id": "myExtView",
        "name": "Items"
      }]
    }
  }
}
```

**View locations:**

| Container  | Description             |
| ---------- | ----------------------- |
| `explorer` | File Explorer sidebar   |
| `scm`      | Source Control sidebar  |
| `debug`    | Debug sidebar           |
| `test`     | Testing sidebar         |
| Custom ID  | Activity bar (new icon) |

## TreeDataProvider implementation

```typescript
import * as vscode from "vscode";

// Tree item class
class MyItem extends vscode.TreeItem {
  constructor(
    public readonly label: string,
    public readonly collapsibleState: vscode.TreeItemCollapsibleState,
    public readonly children?: MyItem[],
  ) {
    super(label, collapsibleState);
    this.tooltip = this.label;
    this.contextValue = "myItem"; // For context menu
  }
}

// Provider class
class MyTreeProvider implements vscode.TreeDataProvider<MyItem> {
  // Event emitter for refresh
  private _onDidChangeTreeData = new vscode.EventEmitter<MyItem | undefined>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  private data: MyItem[] = [
    new MyItem("Parent", vscode.TreeItemCollapsibleState.Expanded, [
      new MyItem("Child 1", vscode.TreeItemCollapsibleState.None),
      new MyItem("Child 2", vscode.TreeItemCollapsibleState.None),
    ]),
  ];

  refresh(): void {
    this._onDidChangeTreeData.fire(undefined);
  }

  dispose(): void {
    this._onDidChangeTreeData.dispose();
  }

  getTreeItem(element: MyItem): vscode.TreeItem {
    return element;
  }

  getChildren(element?: MyItem): MyItem[] {
    return element ? element.children || [] : this.data;
  }
}
```

## Register the view

```typescript
export function activate(context: vscode.ExtensionContext) {
  const provider = new MyTreeProvider();

  // createTreeView registers the provider and returns a disposable view.
  const treeView = vscode.window.createTreeView<MyItem>("myExtView", {
    treeDataProvider: provider,
    showCollapseAll: true,
  });
  context.subscriptions.push(provider, treeView);

  context.subscriptions.push(
    vscode.commands.registerCommand("myExt.refresh", () => provider.refresh()),
  );
}
```

Use `registerTreeDataProvider` instead when you don't need a `TreeView` handle; don't register both for the same view.

## Item customization

```typescript
class MyItem extends vscode.TreeItem {
  constructor(label: string, isFolder: boolean) {
    super(
      label,
      isFolder
        ? vscode.TreeItemCollapsibleState.Collapsed
        : vscode.TreeItemCollapsibleState.None,
    );

    // Icon (codicon or file path)
    this.iconPath = new vscode.ThemeIcon(isFolder ? "folder" : "file");

    // Click action
    this.command = {
      command: "myExt.openItem",
      title: "Open",
      arguments: [this],
    };

    // Description (gray text after label)
    this.description = "(modified)";
  }
}
```

## Context menu

```json
{
  "contributes": {
    "menus": {
      "view/item/context": [{
        "command": "myExt.delete",
        "when": "view == myExtView && viewItem == myItem"
      }]
    }
  }
}
```
