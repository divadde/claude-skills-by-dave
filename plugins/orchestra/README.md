# orchestra

Opus 5.5 plans, delegates and verifies. Sonnet 5.5 and Haiku 5.5 workers do the bulk work.

## Install (once)

This repository is a Claude Code marketplace. In Claude Code run:

```
/plugin marketplace add divadde/claude-skills-by-dave
/plugin install orchestra@orchestra-marketplace
/reload-plugins
```

Update later with `/plugin marketplace update orchestra-marketplace`.

Quick test from a local clone, without installing: `claude --plugin-dir ./plugins/orchestra`

## Use

```
/orchestra:run <describe the task>
```

The skill sets the model to `claude-opus-5-5` for the run. To be sure the whole session stays on Opus, start it with `claude --model claude-opus-5-5` (or `/model claude-opus-5-5`). The skill checks this and stops if it is not on Opus.

## What is inside

| File | Role |
|---|---|
| `skills/run/SKILL.md` | The orchestrator playbook: preflight, plan, routing table, brief template, verification, report. Becomes `/orchestra:run`. |
| `agents/worker-sonnet.md` | `orchestra:worker-sonnet`, model `claude-sonnet-5-5`. Well-specified implementation, research, read-only exploration. |
| `agents/worker-haiku.md` | `orchestra:worker-haiku`, model `claude-haiku-5-5`. Mechanical, verifiable tasks only. |

## Known pitfalls

- `CLAUDE_CODE_SUBAGENT_MODEL`: if set, it can override the per-worker models. Check with `echo "${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"`.
- Built-in Explore and general-purpose agents inherit the main (Opus) model. The skill tells the orchestrator not to use them for bulk work.
- Workers cannot spawn other agents (their tool lists omit `Agent`).
- Plugin agents ignore `permissionMode`, `hooks` and `mcpServers` in their frontmatter.
- Full model IDs assume the Anthropic API. On Bedrock or Vertex, replace them in the three files with the provider IDs, or with the aliases `opus`, `sonnet`, `haiku`.
- To see which model a worker actually used, run `/tasks` while it is active.

## Tuning

Edit the routing section of `skills/run/SKILL.md` to move task types between tiers. Edit `maxTurns` in the agent files to cap worker runtime.
