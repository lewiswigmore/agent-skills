# VS Code AI customization guide

Use the customization that matches the job: instructions for conventions, prompts for repeatable requests, agents for a role, and skills for reusable task workflows.

## File types and usage

| File Type | Path/Naming Convention | Purpose | Scope |
|-----------|----------------------|---------|-------|
| **copilot-instructions.md** | `.github/copilot-instructions.md` | Project-wide coding conventions | Copilot harness project guidance |
| **Instructions Files** | `.github/instructions/*.instructions.md` | Language/framework-specific rules | Conditionally applied via `applyTo` pattern |
| **Prompt Files** | `.github/prompts/*.prompt.md` | Reusable task definitions | Manually executed |
| **Custom Agents** | `.github/agents/*.agent.md` | Specialised agents | Applied when the agent is selected |
| **Agent Skills** | `.github/skills/<skill-name>/SKILL.md` | Reusable, task-specific workflows with optional supporting resources | Loaded when relevant; can also be invoked as a slash command |
| **AGENTS.md** | Root or subfolder | Project guidance shared across compatible agents | Harness-dependent |
| **CLAUDE.md** | Root or `.claude/` | Claude Code compatibility | Harness-dependent |

Agent Skills became available experimentally in VS Code 1.108 (December 2025) and were later enabled by default. Extensions can bundle a skill through `contributes.chatSkills`; its `path` points to `SKILL.md`, whose `name` must match its parent directory.

```json
{
  "contributes": {
    "chatSkills": [
      { "path": "./skills/my-skill/SKILL.md" }
    ]
  }
}
```

See the [Agent Skills guide](https://code.visualstudio.com/docs/agent-customization/agent-skills) for supported locations, format, and harness differences, and the [chatSkills contribution point](https://code.visualstudio.com/api/references/contribution-points#contributes.chatSkills) for extension packaging details.

## Instructions file format

### Header (YAML frontmatter)

```yaml
---
name: Code Review              # Display name in UI
description: Expert code review guidelines
applyTo: "**/*.{ts,tsx,js,jsx}"  # Auto-apply glob pattern
---
```

### Key properties

| Property | Description |
|----------|-------------|
| `name` | Display name in UI (defaults to filename if unset) |
| `description` | When and why to use the instructions |
| `applyTo` | Auto-apply glob pattern (manual attachment only if unset) |

## VS Code settings

`chat.instructionsFilesLocations` adds editor search locations. Prefer native harness folders for portability: the Copilot CLI and app don't inherit VS Code's editor settings.

```json
{
  "chat.instructionsFilesLocations": {
    ".github/instructions": true,
    ".claude/rules": true
  }
}
```

## Configurable scenarios

| Scenario | Setting Key |
|----------|------------|
| Review selected code | `github.copilot.chat.reviewSelection.instructions` |
| Commit messages (experimental) | `github.copilot.chat.commitMessageGeneration.instructions` |
| PR descriptions (experimental) | `github.copilot.chat.pullRequestDescriptionGeneration.instructions` |

These settings accept inline text or a Markdown file path. For harness-specific discovery and current deprecation details, use the [custom instructions reference](https://code.visualstudio.com/docs/agents/reference/custom-instructions-reference).

## Custom agent format

```markdown
---
name: Code Reviewer
description: Review code for bugs, security issues, and performance problems
tools:
  - read
  - search
---

# Code Reviewer Agent

Review the changes for bugs, security issues, and performance problems.
Report actionable findings with file and line references.
```

## Directory structure example

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

## Official resources

| Resource | URL |
|----------|-----|
| Custom Instructions | https://code.visualstudio.com/docs/agent-customization/custom-instructions |
| Custom Instructions Reference | https://code.visualstudio.com/docs/agents/reference/custom-instructions-reference |
| Prompt Files | https://code.visualstudio.com/docs/agent-customization/prompt-files |
| Custom Agents | https://code.visualstudio.com/docs/agent-customization/custom-agents |
| Agent Skills | https://code.visualstudio.com/docs/agent-customization/agent-skills |
| Awesome Copilot | https://github.com/github/awesome-copilot |

## Tips

- **Glob patterns**: `applyTo: "**/*.py"` applies only to Python files
- **Customizations**: Use **Chat: Open Customizations** to create and manage instruction files
- **Sync**: Settings Sync covers Local user-profile instructions; keep project instruction files in source control
