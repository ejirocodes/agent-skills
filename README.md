# Agent Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

A curated collection of AI agent skills that enhance coding assistants with specialized domain knowledge.

## What Are Agent Skills?

Agent skills are structured knowledge bases that help AI coding assistants (like Claude, GPT, Cursor, etc.) understand framework-specific patterns, best practices, and common pitfalls. They bridge the gap between an AI's general knowledge and the specific, up-to-date expertise needed for modern development.

**Why agent skills matter:**

- **Stay current** - Frameworks evolve faster than AI training data. Skills provide the latest patterns and syntax.
- **Reduce errors** - Skills teach AI about breaking changes, deprecations, and migration paths.
- **Best practices** - Encode community-vetted solutions to common problems.
- **Type safety** - Include TypeScript patterns and proper typing for modern codebases.

## Available Skills

| Skill | Description | Rules | Install |
|-------|-------------|-------|---------|
| [svelte-skills](./svelte-skills) | Svelte 5 runes, snippets, and SvelteKit patterns | 24 | `npx add-skill ejirocodes/agent-skills/svelte-skills` |

## Installation

Install a skill using the `add-skill` CLI:

```bash
npx add-skill ejirocodes/agent-skills/<skill-name>
```

For example, to install Svelte skills:

```bash
npx add-skill ejirocodes/agent-skills/svelte-skills
```

## Usage

For best results, prefix your prompt with `use <skill> skill`:

```
Use svelte skill, help me create a form component with validation
```

This explicitly triggers the skill and ensures the AI follows the documented patterns. Without the prefix, skill triggering depends on how closely your prompt matches the skill's keywords.

## Skill Structure

Each skill follows this structure:

```
<skill-name>/
├── README.md              # Skill overview and usage
├── LICENSE                # MIT License
└── skills/
    └── <skill-set>/
        ├── SKILL.md       # Skill metadata and rule index
        └── rules/
            └── *.md       # Individual rule files
```

### Rule Types

Rules are classified into two categories:

- **Capability** - AI *cannot* solve the problem without the skill. These address breaking changes, new syntax, or patterns outside AI's training data.

- **Efficiency** - AI *can* solve the problem but not optimally. These provide best practices and consistent approaches.

### Rule Format

Each rule includes:

- **Impact level** (HIGH/MEDIUM)
- **Symptoms** - What the developer experiences
- **Root cause** - Why the problem occurs
- **Fix** - Correct patterns with code examples
- **References** - Links to official documentation

## Contributing

Contributions are welcome! To add a new skill:

1. Create a folder for your skill (e.g., `react-skills/`)
2. Follow the structure above
3. Include comprehensive rules with:
   - Clear problem descriptions
   - Code examples (incorrect vs correct)
   - Links to official documentation
4. Submit a pull request

### Guidelines

- Focus on problems AI assistants commonly get wrong
- Prioritize recent framework changes and breaking updates
- Include TypeScript patterns where applicable
- Test rules with multiple AI models (Claude, GPT, etc.)
- Delete rules if AI consistently solves them without help

## Roadmap

Future skills planned:

- **Frontend** - React 19, Vue 3, Angular, Solid
- **Backend** - Node.js, Deno, Bun, Go, Rust
- **Databases** - PostgreSQL, MongoDB, Redis, Prisma
- **DevOps** - Docker, Kubernetes, CI/CD
- **Testing** - Vitest, Playwright, Jest
- **AI/ML** - LangChain, Vector DBs, RAG patterns

## License

MIT - see [LICENSE](./LICENSE)

---

Created by [Ejiro Asiuwhu](https://github.com/ejirocodes)
