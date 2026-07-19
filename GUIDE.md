# References

## Quick Start

**Start here:** [SKILL.template.md](SKILL.template.md) - Use this as a template when creating a new skill. It includes all required and optional YAML frontmatter fields with explanations.

---

## Official Claude Documentation

### Agent Skills Overview

https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview

### MCP (Model Context Protocol) Documentation

https://modelcontextprotocol.io/

---

## Concepts

### Agent Skills

Agent Skills are modular capabilities that extend Claude's functionality. Each Skill packages instructions, metadata, and optional resources (scripts, templates) that Claude uses automatically when relevant.

**Key characteristics:**

- Progressive disclosure: YAML frontmatter loads first, full instructions load when relevant
- Modular: Multiple skills can work together
- Portable: Work identically across Claude.ai, Claude Code, and API
- Reusable: Package once, benefit every time

### Skill Structure

```
your-skill-name/
├── SKILL.md              # Required - instructions with YAML frontmatter
├── scripts/              # Optional - executable code (Python, Bash, etc.)
├── references/           # Optional - bundled documentation
└── assets/               # Optional - templates, fonts, icons
```

### YAML Frontmatter Fields

**Required:**

- `name` - kebab-case identifier
- `description` - What it does + when to use it (include trigger phrases)

---

## Related Resources

- [Skills Repository (Community & Official Examples)](https://github.com/anthropics/skills)
