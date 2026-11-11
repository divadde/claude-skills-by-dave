# claude-skills-by-dave

Personal marketplace of Claude Code plugins. Each plugin lives in `plugins/<name>/` and is listed in `.claude-plugin/marketplace.json`.

## Available plugins

| Plugin | What it does | Command |
|---|---|---|
| [`orchestra`](plugins/orchestra) | Opus 5.5 plans, delegates and verifies; Sonnet 5.5 and Haiku 5.5 workers do the heavy lifting | `/orchestra:run <task>` |

## Installation

In Claude Code:

```
/plugin marketplace add divadde/claude-skills-by-dave
/plugin install orchestra@orchestra-marketplace
/reload-plugins
```

To pick up new pushes later: `/plugin marketplace update orchestra-marketplace`.

Try it without installing, from a local clone of this repo: `claude --plugin-dir ./plugins/orchestra`.

Requirements:
- An up-to-date Claude Code (`claude update`).
- Access to Opus 5.5, Sonnet 5.5 and Haiku 5.5 on the Anthropic API. The full model IDs (`claude-opus-5-5`, `claude-sonnet-5-5`, `claude-haiku-5-5`) are hardcoded in the plugin files; on Bedrock or Vertex, replace them with the provider's IDs or with the aliases `opus`, `sonnet`, `haiku`.

## Usage: orchestra

Start the session on Opus so it stays on Opus for the whole conversation:

```
claude --model claude-opus-5-5
```

Then, inside the session:

```
/orchestra:run migrate the tests in tests/ from unittest to pytest and make sure the suite passes
```

The skill is manual-only (`disable-model-invocation`): it runs only when you invoke it.

## What orchestra does

The orchestrator (Opus) keeps the judgement; cheaper workers do the volume. The flow:

1. **Preflight.** Checks that it is running on Opus and that `CLAUDE_CODE_SUBAGENT_MODEL` is unset (otherwise it can override the workers' models).
2. **Plan.** Splits the task into units, each with a goal, input files, expected output, an acceptance check and a tier. If the task is small (roughly three tool calls or fewer) it just does it itself.
3. **Routing.** Assigns each unit to a tier:

   | Tier | What goes there |
   |---|---|
   | Opus (orchestrator) | ambiguous requirements, design decisions, non-obvious bugs, security-sensitive changes, final integration and review |
   | `orchestra:worker-sonnet` | well-specified implementation, refactors with tests, writing tests, reading and summarising code or documents, bounded web research, read-only exploration |
   | `orchestra:worker-haiku` | mechanical, verifiable work only: renames, format conversions, boilerplate from a template, an explicit list of edits, find-and-replace you can check with grep |

   Haiku never does open-ended search (its misses are silent) or judgement calls.
4. **Delegate.** Each worker gets a structured brief (goal, context as paths, what to do, what not to touch, acceptance check, return format). Independent units run in parallel on disjoint sets of files.
5. **Verify.** A worker's "DONE" is not enough: the orchestrator runs the acceptance check or inspects the diff. On failure it re-delegates once with the evidence, then takes the unit over. Escalation goes Haiku → Sonnet → Opus.
6. **Report.** At most 10 lines: what was done, how it was verified, what is still open, plus the split (for example `units: 2 haiku, 3 sonnet, 1 self`).

The savings come from moving token volume to the cheaper tiers: the orchestrator generates few tokens, the workers most of them. How much you save depends on how much of the task is delegable.

## Known limitations

- `CLAUDE_CODE_SUBAGENT_MODEL`: if set, it can override per-agent routing. Check with `echo "${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"`.
- The built-in Explore and general-purpose agents inherit the main model (Opus): the skill tells the orchestrator not to use them for heavy work.
- The `model` field in the skill's frontmatter applies while the skill runs; to keep Opus for the whole session use `claude --model claude-opus-5-5` or `/model claude-opus-5-5`.
- Workers cannot spawn other agents.
- To see which model a worker actually used, run `/tasks` while it is active.
- If the plugin's agents are not found, the skill falls back to `general-purpose` with an explicit `model` and reports it.

More details in the [plugin README](plugins/orchestra/README.md).

## Repository layout

```
claude-skills-by-dave/
├── .claude-plugin/
│   └── marketplace.json        # catalogue: one entry per plugin
├── README.md
└── plugins/
    └── orchestra/
        ├── .claude-plugin/plugin.json
        ├── skills/run/SKILL.md          # /orchestra:run
        ├── agents/worker-sonnet.md      # orchestra:worker-sonnet
        ├── agents/worker-haiku.md       # orchestra:worker-haiku
        └── README.md
```

## Adding a new plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` and, as needed, `skills/<skill>/SKILL.md`, `agents/*.md`, `commands/*.md`. Component folders go at the plugin root, not inside `.claude-plugin/`.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
3. Validate: `claude plugin validate .` and `claude plugin validate ./plugins/<name>`.
4. Commit and push, then `/plugin marketplace update orchestra-marketplace` and `/plugin install <name>@orchestra-marketplace`.

A plugin's skills are named `/<plugin>:<folder>`, so they do not collide with other plugins' skills or with personal skills.
