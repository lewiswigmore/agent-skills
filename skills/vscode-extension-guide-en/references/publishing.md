# Publishing to Marketplace

Package and publish a VS Code extension to the Marketplace.

## Prerequisites

1. **Publisher account** at [marketplace.visualstudio.com/manage](https://marketplace.visualstudio.com/manage)
2. **vsce CLI**: use `npx @vscode/vsce` or install `@vscode/vsce`

For CI publishing, prefer federated identity instead of a stored Personal Access Token (PAT). GitHub Actions supports Marketplace Trusted Publishing with OIDC; Azure Pipelines supports Microsoft Entra ID. Azure DevOps global PATs are scheduled for retirement on **December 1, 2026**. Follow the [current secure publishing guide](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#secure-automated-publishing-to-visual-studio-marketplace) for setup and provider requirements.

For GitHub Actions, create a Trusted Publishing policy, then grant the publishing job `id-token: write`:

```yaml
name: Publish extension
on:
  release:
    types: [published]
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npx @vscode/vsce publish --oidc
```

## Creating a publisher

1. Go to [marketplace.visualstudio.com/manage](https://marketplace.visualstudio.com/manage)
2. Sign in with your Microsoft account
3. Click "Create publisher"
4. Fill in:
   - **ID**: Unique identifier (used in extension ID)
   - **Name**: Display name
   - **Description**: Optional

## Package and publish

```bash
# Create a VSIX for local testing or distribution
npx @vscode/vsce package

# Publish a new version using the configured trusted identity
npx @vscode/vsce publish --oidc
```

For Azure Pipelines, use the Microsoft Entra ID flow documented in the [secure publishing guide](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#publish-from-azure-pipelines-with-microsoft-entra-id). For interactive or other CI workflows, check the current authentication options in the official guide rather than copying a long-lived PAT into a script or environment.

## Pre-publish checklist

| Item                        | Check                               |
| --------------------------- | ----------------------------------- |
| `publisher` in package.json | Matches your publisher ID           |
| `version`                   | Incremented from previous           |
| `README.md`                 | Exists with that casing and has content |
| `LICENSE`                   | Included                            |
| `icon`                      | PNG at least 128x128, path in package.json |
| `.vscodeignore`             | Excludes unnecessary files          |

## package.json requirements

```json
{
  "name": "my-extension",
  "displayName": "My Extension",
  "description": "Brief description for Marketplace",
  "version": "1.0.0",
  "publisher": "your-publisher-id",
  "icon": "images/icon.png",
  "repository": {
    "type": "git",
    "url": "https://github.com/user/repo"
  },
  "categories": ["Other"],
  "keywords": ["keyword1", "keyword2"]
}
```

## Manifest categories

Use a value supported by the [`categories` manifest field](https://code.visualstudio.com/api/references/extension-manifest#fields), such as `Programming Languages`, `Snippets`, `Linters`, `Themes`, `Debuggers`, `Formatters`, `Keymaps`, `SCM Providers`, `Other`, `Extension Packs`, `Language Packs`, `Data Science`, `Machine Learning`, `Visualization`, `Notebooks`, `Education`, or `Testing`. Marketplace search filters may expose additional categories that aren't valid manifest values.

## Stable and pre-release versions

- Valid: `1.0.0`, `1.2.3`, `0.0.1`
- Invalid: `1.0.0-beta.1`, `1.0.0-rc1` (prerelease tags rejected)
- Publish a Marketplace pre-release channel with `npx @vscode/vsce publish --pre-release`; use a distinct numeric version from the stable release.
- See the official [pre-release extension guidance](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#pre-release-extensions) for version ordering and upgrade behavior.

## Inspect the package before publishing

```bash
# List files that will be included
npx @vscode/vsce ls

# Create VSIX without publishing (for inspection)
npx @vscode/vsce package
```

## .vscodeignore

Keep only files needed at runtime. Exclude `node_modules` only when runtime dependencies are bundled or otherwise included:

```ignore
**
!package.json
!README.md
!LICENSE
!CHANGELOG.md
!out/**
!images/icon.png

src/**
test/**
node_modules/**
*.ts
tsconfig*.json
.github/**
.vscode/**
*.vsix
```

## Updating published extensions

```bash
# Increment version and publish
npx @vscode/vsce publish patch

# Or manually update version first
npm version patch
npx @vscode/vsce publish
```

## Unpublishing

```bash
# Unpublish specific version
npx @vscode/vsce unpublish <publisher>.<extension> --version <version>

# Unpublish entire extension (use with caution!)
npx @vscode/vsce unpublish <publisher>.<extension>
```

## Common errors

| Error                      | Cause                        | Fix                                |
| -------------------------- | ---------------------------- | ---------------------------------- |
| `Missing publisher`        | No publisher in package.json | Add `"publisher": "your-id"`       |
| `Personal Access Token...` | Legacy PAT invalid or expired | Prefer Trusted Publishing or Entra ID; see the [authentication guide](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#secure-automated-publishing-to-visual-studio-marketplace) |
| `version already exists`   | Same version published       | Increment version number           |
| `README not found`         | File missing or wrong case   | Create `README.md` with that casing |
| `invalid prerelease`       | SemVer suffix like `1.0.0-beta` | Use a numeric `x.y.z` version and pass `--pre-release` for the Marketplace pre-release channel |

## Marketplace URLs

- **Your extensions**: `https://marketplace.visualstudio.com/manage/publishers/<publisher-id>`
- **Published extension**: `https://marketplace.visualstudio.com/items?itemName=<publisher>.<extension>`
- **Statistics**: Available in manage portal after publish

## Recommended .vscodeignore exclusions

Exclude development files that the extension does not need at runtime:

```ignore
# Source & config (already compiled to out/)
src/**
**/tsconfig.json
**/.eslintrc.json
**/*.map
**/*.ts
!out/**

# Dev tooling
.vscode/**
.vscode-test/**
.github/**
node_modules/**

# Dev-only content (never ship to users)
output_sessions/**
research/**
session/**
FULL_SPECIFICATION.md
AGENTS.md

# Large or unnecessary assets
images/demo-animated.gif
*.vsix
```

> **Tip**: Run `npx @vscode/vsce ls` to preview exactly what will be packaged
> before running `vsce package` or `vsce publish`.
