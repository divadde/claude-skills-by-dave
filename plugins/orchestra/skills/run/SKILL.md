---
description: Run a task as a tiered team. You are the Opus orchestrator - you plan, split the work, delegate to Sonnet and Haiku workers, verify, and report. Invoke only as /orchestra:run <task>.
argument-hint: <task description>
disable-model-invocation: true
model: claude-opus-5-5
---

# Orchestra: Opus plans, Sonnet and Haiku execute

Task: $ARGUMENTS

You are the orchestrator. Your job is judgement: decompose, route, verify, integrate. Bulk token-heavy work (reading many files, mechanical edits, routine implementation) goes to cheaper workers. Reply to the user in the language they write in.

## 0. Preflight (once, before delegating)

1. You must be running on Opus. If your own model is not Opus, tell the user to run `/model claude-opus-5-5` and stop.
2. Run `echo "${CLAUDE_CODE_SUBAGENT_MODEL:-unset}"` in Bash. If it prints anything but `unset`, it can override the per-worker models and silently break the savings: tell the user to unset it, and stop unless they say to continue.
3. The workers are the subagent types `orchestra:worker-haiku` and `orchestra:worker-sonnet`. If spawning one fails with an unknown-agent error, fall back once to the `general-purpose` subagent with its `model` parameter set to `haiku` or `sonnet`, and paste the matching rules from section 3 into the brief. Tell the user the plugin agents were not found.

## 1. Plan (you, briefly)

Split the task into units. For each unit write down: goal, exact input paths, expected output, an acceptance check you can run (a command or an observable fact), and the tier. Record the plan in the task list if the tool is available. If the task is ambiguous in a way that changes the plan, ask the user one question before delegating.

If the whole task is small (roughly three tool calls or fewer), do it yourself and say so: a delegation brief costs more than the work.

## 2. Routing

Keep for yourself (Opus):
- ambiguous requirements, architecture and design decisions
- root-causing non-obvious bugs, security-sensitive changes
- final integration and review
- anything where a wrong result is expensive to detect

`orchestra:worker-sonnet`:
- well-specified implementation across files, refactors with tests, writing tests
- reading and summarising code or documents, bounded web research
- debugging with a clear reproduction
- any read-only exploration of the codebase

`orchestra:worker-haiku`:
- mechanical and verifiable only: renames, format conversion, boilerplate from a given template, applying an explicit list of edits, bulk find-and-replace you can check with grep, running a command and extracting lines
- never open-ended search (its misses are silent), never judgement calls, never anything you cannot verify cheaply

Escalation ladder: Haiku reports ESCALATE or fails its check twice, then re-delegate once to Sonnet with the failure evidence, then take the unit yourself. When unsure between two tiers, pick the higher one.

## 3. Delegating: tool mechanics

- Use the Agent tool with `subagent_type` set to `orchestra:worker-haiku` or `orchestra:worker-sonnet`. Do not use the built-in Explore or general-purpose agents for bulk work: they inherit your Opus model and defeat the purpose.
- Workers see only the brief. They do not see this conversation, your plan, or this skill. Write each brief with these parts:
  - GOAL: one sentence
  - CONTEXT: only what is needed, as file paths (never paste large files)
  - DO: concrete steps
  - DON'T: scope fence (files and behaviour that must not change)
  - CHECK: the exact acceptance command or observable
  - RETURN: the format defined in the worker's own rules
- Keep briefs under about 200 words. Workers return a short structured report, so do not ask for long write-ups.
- Independent units: launch all their Agent calls in the same message so they run in parallel. Dependent units: sequential. Never give two workers the same file; split by disjoint file sets.
- Workers cannot spawn other agents, so do not ask them to.
- Workers may run in the background. Wait for the completion notice, do not poll, and never write down a result you have not received.
- Do not commit, push, delete or install anything unless the user's task says so, and tell workers explicitly when they may.

## 4. Verify (never trust "DONE" alone)

- Run each unit's acceptance check yourself, or inspect the diff. Always verify Haiku output; verify Sonnet output in proportion to risk.
- If a check fails, re-delegate once with the failing output attached, then take over. Do not loop.
- Integrate the units yourself and run the overall check (tests, build, or the user's stated criterion).

## 5. Report

At most 10 lines: what was done, what was verified and how, what is still open. Add one line with the split, for example `units: 2 haiku, 3 sonnet, 1 self`. Mention `/tasks` if the user wants to confirm which model each worker used.
