# Angi Skill

A Claude skill that gives developers expert, hallucination-free guidance for [`@angi-ai/angi`](https://www.npmjs.com/package/@angi-ai/angi) — the AI-native React UI library that lets end users control interface components through natural language.

## What this skill covers

- **Fresh project setup** — server route, layout provider, chat bubble, first component
- **Making any component AI-addressable** — `<Angi>` boundary + `useAngiComponent`
- **Read-only components** — expose state to Claude without allowing modifications (`permissions={["read"]}`)
- **Action design** — granularity principles, schema types, async `execute`, descriptions that work
- **Programmatic usage** — triggering Angi from custom UI with `useAngi()`
- **Import reference** — correct paths for `@angi-ai/angi/client` vs `@angi-ai/angi/server`

Excludes: building custom adapters, debugging internals.

## When it triggers

The skill activates when a developer:

- Mentions Angi or `@angi-ai/angi`
- Wants to make a React component AI-addressable or controllable by Claude
- Asks about `useAngiComponent`, `AngiAgent`, `AngiChatBubble`, `AngiNextProvider`, `<Angi>`
- Says things like "I want an AI component", "I want Claude to fill this form", "make this component readable by Claude", "set up Angi", "how do I expose state to Claude", "I want users to control my UI with natural language"

## Why this skill exists

Without this skill, Claude consistently halluccinates the Angi API surface. Eval results from iteration 1 (10 runs, 5 test cases):

| | Score |
|---|---|
| **With skill** | **100%** (34/34 assertions) |
| **Without skill** | **65%** (22/34 assertions) |
| **Improvement** | **+35 percentage points** |

Common hallucinations the skill fixes:
- `createAngiHandler` from `@angi-ai/angi/next` (doesn't exist)
- `useAngi({ apiRoute })` returning `{ ref, input, setInput }` (doesn't exist)
- `data-angi-id` DOM attributes (doesn't exist)
- `state:` instead of `getState:`
- `parameters` + `handler` instead of `schema` + `execute`
- Importing from `@angi-ai/angi` instead of `@angi-ai/angi/client`

## Files

```
angi/
├── SKILL.md          # The skill Claude reads (developer guide + code examples)
├── README.md         # This file
└── evals/
    └── evals.json    # 5 test cases with assertions for benchmarking
```

Eval workspace (generated, not committed):
```
~/.claude/skills/angi-workspace/
└── iteration-1/
    ├── benchmark.json
    ├── eval_viewer.html
    └── {eval-name}/
        ├── eval_metadata.json
        ├── with_skill/outputs/response.md
        └── without_skill/outputs/response.md
```

## Usage

The skill is loaded automatically by Claude when a trigger phrase is detected. No manual invocation needed.

To run evals after editing `SKILL.md`, use the `skill-creator` skill:
```
run evals on the angi skill
```
