# VS Code AI Customization Guide

Comprehensive guide to AI customization in VS Code.

## File Types and Usage

| File Type | Path/Naming Convention | Purpose | Scope |
|-----------|----------------------|---------|-------|
| **copilot-instructions.md** | `.github/copilot-instructions.md` | Project-wide coding conventions | Auto-applied to all chat requests |
| **Instructions Files** | `*.instructions.md` | Language/framework-specific rules | Conditionally applied via `applyTo` pattern |
| **Prompt Files** | `*.prompt.md` | Reusable task definitions | Manually executed |
| **Custom Agents** | `*.agent.md` | Specialised agents | Applied when the agent is selected |
| **AGENTS.md** | Root or subfolder | Multi-agent environments | Auto-applied to all chats |
| **CLAUDE.md** | Root, `.claude/`, or `~/` | Claude Code compatibility | Auto-applied to all chats |

## Instructions File Format

### Header (YAML Frontmatter)

```yaml
---
name: Code Review              # Display name in UI
description: Expert code review guidelines
applyTo: "**/*.{ts,tsx,js,jsx}"  # Auto-apply glob pattern
---
```

### Key Properties

| Property | Description |
|----------|-------------|
| `name` | Display name in UI (defaults to filename if unset) |
| `description` | Description text |
| `applyTo` | Auto-apply glob pattern (manual attachment only if unset) |

## VS Code Settings

```json
{
  "chat.instructionsFilesLocations": {
    ".github/instructions": true,
    ".claude/rules": true
  },
  "github.copilot.chat.reviewSelection.instructions": [
    { "text": "Review for bugs, security, and performance." },
    { "file": ".github/instructions/code-review.instructions.md" }
  ],
  "github.copilot.chat.commitMessageGeneration.instructions": [
    { "text": "Use Conventional Commits format." }
  ]
}
```

> **Note:** `github.copilot.chat.codeGeneration.useInstructionFiles` is deprecated since VS Code 1.102. Use file-based `.instructions.md` files instead.

## Configurable Scenarios

| Scenario | Setting Key |
|----------|------------|
| Code review | `github.copilot.chat.reviewSelection.instructions` |
| Commit messages | `github.copilot.chat.commitMessageGeneration.instructions` |
| PR title/description | `github.copilot.chat.pullRequestDescriptionGeneration.instructions` |

## Custom Agent Format

```markdown
---
name: Code Reviewer
description: Expert code reviewer
tools:
  - codebase
  - terminal
  - githubRepo
---

# Code Reviewer Agent

You are a senior code reviewer...

## Your Role
- Conduct thorough code reviews
- Identify bugs, security issues, and performance problems
```

## Directory Structure Example

```
.github/
├── copilot-instructions.md      # Project-wide
├── instructions/
│   ├── code-review.instructions.md
│   ├── typescript.instructions.md
│   └── security.instructions.md
├── prompts/
│   ├── review-pr.prompt.md
│   └── refactor-file.prompt.md
└── agents/
    ├── code-reviewer.agent.md
    └── planner.agent.md
```

## Official Resources

| Resource | URL |
|----------|-----|
| Custom Instructions | https://code.visualstudio.com/docs/copilot/customization/custom-instructions |
| Prompt Files | https://code.visualstudio.com/docs/copilot/customization/prompt-files |
| Custom Agents | https://code.visualstudio.com/docs/copilot/customization/custom-agents |
| Agent Skills | https://code.visualstudio.com/docs/copilot/customization/agent-skills |
| Awesome Copilot | https://github.com/github/awesome-copilot |

## Tips

- **Glob patterns**: `applyTo: "**/*.py"` applies only to Python files
- **Tool references**: Use `#tool:githubRepo` in the body to reference tools
- **Generate command**: Use `Chat: Configure Instructions` to auto-generate Instructions files
- **Sync**: Use Settings Sync to share Instructions files across devices
