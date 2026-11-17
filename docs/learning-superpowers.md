# Learning superpowers: design docs, plans, and what lasts

Notes from a conversation on 2026-11-17 about superpowers 7.0.2 (the installed version). Findings come from reading the plugin's skills: brainstorming, writing-plans, executing-plans, subagent-driven-development, requesting-code-review and finishing-a-development-branch.

## What superpowers writes

Superpowers has two document types and no ADRs. "ADR" and "decision record" do not appear anywhere in the plugin.

| | Design doc (spec) | Plan |
|---|---|---|
| **Written by** | brainstorming | writing-plans |
| **Location** | `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`, committed | `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md` |
| **Answers** | what and why | how, task by task |
| **Reader** | a builder who could plan from it without coming back to you | an implementer who has seen neither the codebase nor the spec |
| **Gate** | you approve it before anything is built | you review it and pick an execution method before implementation |

Your preference for where either file goes overrides the default paths.

### Design doc

- **Contents:** intent, goals, non-goals and anti-goals, constraints, the parts of the how you decided, and what's left to the builder.
- **Decision record:** an Approaches section lists the two or three ways considered, the trade-offs, which one you picked, and why. This is the closest thing to an ADR, but it sits inside the spec.
- **Size:** a small change gets a short version. A project gets the full one, and a fresh subagent reads it (the "builder check") before you see it.
- **Attribution:** calls the agent made are marked as its own, not as things you agreed to.

### Plan

- **Header:** goal, architecture, tech stack, a link to the spec, global constraints copied verbatim from the spec, and a "Review Focus" list of failure modes the spec implies but no task tests.
- **Tasks:** each has exact files, the interfaces it consumes and produces, and checkbox steps in test-first order (failing test, run, implement, run, commit).
- **Level of detail:** it holds decisions, not code, so it has signatures and test assertions but not function bodies. A plan several times longer than its spec is treated as a defect.
- **Execution:** either a fresh subagent per task with a review after each, or inline in the session with one review at the end.

### Real examples

The plugin ships its own specs under `~/.claude/plugins/cache/anthropic-plugin-directory/superpowers/7.0.2-22f0b463e7a5/docs/superpowers/specs/`. `2026-10-05-brainstorming-rebuild-design.md` is the design for the brainstorming skill itself.

## My view on what lasts

From past experience with other agent frameworks:

- Specs get stale.
- Plans rot if not implemented right away.
- Plans are pointless after the change has been merged into main.

What matters long term is the overall system design and the decisions that led to it:

- what product we are building
- the system modules, their boundaries and interfaces
- what makes sense to test and what doesn't
- a system-specific glossary

## How superpowers measures against that

Superpowers only produces the perishable layer. None of the execution, review or branch-finishing skills maintains a system-level document.

- **Plans:** disposable in spirit, since they are dated, tied to one feature and meant to be executed right after approval. Nothing removes or archives them after merge, so they pile up in `docs/superpowers/plans/`.
- **Specs:** per-change snapshots. Nothing updates one when the implementation diverges, and nothing marks an old one as superseded.
- **Decisions:** the "why" is captured in each spec's Approaches section. That is the durable part, but it stays in a dated per-feature file and is never lifted into a record of how the system came to be.
- **System design, boundaries, interfaces, testing policy, glossary:** no artifact for any of these. The plan's file-structure section and per-task interface blocks describe boundaries for one change, then die with the plan.

The plugin's own repo treats old specs and plans as history that is annotated, not rewritten.

Superpowers does consume a durable layer if the project has one. Brainstorming says to read the existing project before asking anything, and plan-writing says to follow established patterns. An architecture doc, glossary or testing policy in the repo would feed every spec and plan. Keeping them current is left to the project.

## Would it update an architecture doc on its own?

Not reliably. No skill tells it to, so it would only happen if the model noticed the doc while planning. Three things make it possible but not guaranteed:

- **Plan file mapping:** writing-plans maps every file to be created or modified before defining tasks. An architecture doc could land on that list, but the instruction is about code decomposition and says nothing about docs.
- **Documentation steps:** the task-sizing rule says to fold "documentation steps into the task whose deliverable needs them". That says where a doc step goes if one exists, not that one must exist.
- **Spec coverage check:** the plan's self-review confirms every spec requirement has a task. If the spec says "update the architecture doc", the plan will carry it.

The last one is the dependable route.

### Claude's suggestion (not something decided)

Superpowers defers to project instructions over its own skills. A line in CLAUDE.md such as "any change that alters a module boundary or interface updates `docs/architecture.md` in the same branch" would be pulled into the spec's constraints and from there into the plan.

## Is it useful to non-technical people building an app?

Claude's assessment: partly. The first half is built for them. The second half works on their behalf but asks for approvals they can't meaningfully give.

### Where it helps

- **Working out what they want:** brainstorming is written to cover non-software projects too, uses plain language matched to the person's vocabulary, and bans process jargon. It asks about outcomes, not technology.
- **Technical choices:** when someone talks only about outcomes, the skill tells the agent to pick the platform, language and libraries itself and state each choice in a line.
- **Seeing before building:** it offers browser mockups and quick throwaway prototypes, which suit someone who can react to a screen but not to a description of one.
- **Quality they can't supply themselves:** test-first development, per-task reviews and "verify before claiming done" give some of the discipline a non-technical person can't enforce.

### Where it doesn't

- **The plan review:** the plan is file paths, function signatures and test code, and the person is asked to confirm it captures what they want. A non-technical reader can only rubber-stamp it.
- **The execution choice:** they are asked to choose between subagent-driven and inline execution, a trade-off that means little without engineering background.
- **The surrounding tooling:** it assumes a git repo, branches, worktrees and a coding agent such as Claude Code running in a terminal or IDE. Nothing covers hosting, deployment, accounts or what to do when the running app breaks.
- **Long-term upkeep:** nothing maintains a picture of the whole system, and a non-technical owner is the person least able to rebuild one from the code.

It would get them a better-specified and better-tested first version than prompting alone. They would still need a technical person for the plan check, deployment and anything that goes wrong later.
