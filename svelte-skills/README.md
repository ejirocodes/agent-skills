# svelte-skills

Agent skills for Svelte 5 and SvelteKit development.

> **Early Experiment**
>
> This repository is an early experiment in creating specialized skills for AI agents to enhance their capabilities in Svelte 5 and SvelteKit development. The skills are derived from real-world issues, official documentation, and best practices, but might be incomplete or inaccurate due to hallucinations.
>
> Please give feedback when encountering any issues.

## Installation

```bash
npx add-skill ejirocodes/svelte-skills
```

## Usage

For most reliable results, prefix your prompt with `use svelte skill`:

```
Use svelte skill, <your prompt here>
```

This explicitly triggers the skill and ensures the AI follows the documented patterns. Without the prefix, skill triggering may be inconsistent depending on how closely your prompt matches the skill's description keywords.

## Available Skills

### svelte5-best-practices (24 rules)

Svelte 5 and SvelteKit development best practices covering runes, snippets, TypeScript configuration, component patterns, state management, and SvelteKit-specific patterns.

| Type | Count | Examples |
|------|-------|----------|
| Capability | 19 | $state rune declaration, $props destructuring, snippet syntax, event handler syntax, SvelteKit form actions |
| Efficiency | 5 | Universal reactivity, component testing, avoiding over-reactivity, load waterfalls, streaming optimization |

## Rule Types

Rules are classified into two categories:

- **Capability**: AI *cannot* solve the problem without the skill. These address Svelte 5 breaking changes, new syntax (runes, snippets), version-specific issues, or patterns outside AI's training data.

- **Efficiency**: AI *can* solve the problem but not well. These provide optimal patterns, best practices, and consistent approaches that improve solution quality.

## Methodology

Every skill in this repository is created through a rigorous, evidence-based process:

**1. Real-World Issue Collection**

Skills are sourced from actual developer pain points encountered when migrating to or working with Svelte 5.

**2. Multi-Model Verification**

Each skill undergoes systematic testing:
- **Baseline test**: Verify the model fails to solve the problem *without* the skill
- **Skill test**: Confirm the model correctly solves the problem *with* the skill
- **Deletion criteria**: If both Sonnet AND Haiku pass without the skill, the rule will be deleted

**3. Model Tier Validation**

| Model | Without Skill | With Skill | Action |
|-------|--------------|------------|--------|
| Haiku | Fail | Pass | Keep |
| Sonnet | Fail | Pass | Keep |
| Both | Pass | - | Delete |

**Acceptance criteria**: A skill is only kept if it enables Haiku or Sonnet to solve a problem they couldn't solve without it.

## License

MIT
