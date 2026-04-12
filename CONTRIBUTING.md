# Contributing

Thanks for your interest in contributing to agent-skills!

## Adding a New Skill

1. Create a directory under `skills/` with your skill name
2. Include at minimum:
   - `SKILL.md` — with YAML frontmatter (`name`, `description`, `license`) and skill content
   - `LICENSE.txt` — appropriate license for the skill
3. Optionally add a `references/` directory for supporting documents

## Skill Guidelines

- Skills should be focused on a single topic
- Keep files under 100KB each (skills should be lightweight)
- Include attribution if adapting from another source
- All content must be in English

## Pull Requests

- One skill per PR
- Include a brief description of the skill's purpose
- Ensure the validation CI passes
