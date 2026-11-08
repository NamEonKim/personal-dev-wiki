# Multi-Agent Software Development Guide

This guide describes a practical way to build software with an orchestrator agent and focused specialist agents. The goal is not to make every agent autonomous; it is to preserve the user's intent, split work safely, and make the result easy to verify.

The patterns below are general. Copilot clients, MCP hosts, repository settings, and agent runtimes differ, so use the documentation for the exact product when configuring a tool. A prompt or a shared folder is not a substitute for repository permissions, review, or tests.

## 1. Start with a small, reliable workflow

Adopt the workflow incrementally. Keep a human responsible for priorities, design decisions, and final acceptance.

| Stage | What to implement | Ready-to-advance signal |
|---|---|---|
| 1. Single-agent baseline | Add project instructions, a task brief template, and a short validation checklist. Do one feature end to end with one agent. | A new session can find the project conventions and complete a bounded change with passing relevant checks. |
| 2. Repeatable task records | Put the original request, acceptance criteria, constraints, owner, and status in a durable task brief. Require a handoff note when work pauses. | Work can resume from the brief without relying on chat history alone. |
| 3. Orchestrator plus one worker | Have an orchestrator clarify scope, delegate one independent implementation or research task, and integrate its result. | The worker returns a bounded deliverable and the orchestrator verifies it against the brief. |
| 4. Parallel independent work | Add a second worker only for work that can proceed without editing the same files or depending on unreviewed output. Give each worker a separate branch, worktree, or other isolated workspace when supported. | Integration conflicts are rare, ownership is clear, and each result is independently checkable. |
| 5. MCP tools by need | Enable only the MCP servers and operations needed for a specific workflow. Confirm scope, access, and authentication with the host's supported setup. | Each enabled tool has a clear purpose, owner, and least-privilege access. |
| 6. Measure and improve | Track rework, failed checks, handoff gaps, and time spent resolving conflicts. Improve one bottleneck at a time. | More agents or automation demonstrably improve throughput without reducing quality. |

Do not begin with a large agent team, broad write permissions, or automatic merging. A reliable single-agent process is the foundation for safe delegation.

## 2. Choose a directory layout that matches the work

Use the repository's existing conventions first. These examples show where project instructions, durable task records, and agent outputs can live; they are starting points, not required names. Keep task artifacts small and intentional. Avoid putting secrets, credentials, or unreviewed generated files in the repository.

### A. Small feature in one repository

Use this for a contained feature or bug fix where the code and tests are in one repository.

```text
repo/
├── src/
├── tests/
├── docs/
│   └── agent-tasks/
│       └── add-export.md
├── .github/
│   └── copilot-instructions.md
└── README.md
```

Keep stable rules in the project's instruction file (or the equivalent supported by your agent), and put changing requirements for one task in `docs/agent-tasks/<task>.md`. Do not copy the whole task history into permanent project instructions.

### B. Feature spanning multiple modules in a monorepo

Use this when the repository has clear modules with their own tests or conventions.

```text
repo/
├── apps/
│   ├── web/
│   └── api/
├── packages/
│   ├── shared/
│   └── data/
├── docs/
│   └── agent-tasks/
│       └── account-settings/
│           ├── brief.md
│           └── handoff.md
├── tests/
└── .github/
    └── copilot-instructions.md
```

Assign work by module boundary only when interfaces and ownership are clear. For example, an API contract can be agreed first, then separate workers can implement the web and API sides. If the contract is undecided, keep that design decision with the orchestrator and do not let agents independently invent incompatible interfaces.

### C. Product composed of multiple repositories

Use this for a change that genuinely spans separately versioned repositories. Keep each clone independent, and store cross-repository task coordination outside the individual product repos unless the team has an approved shared workspace repository.

```text
product-workspace/
├── service-api/                 # independent Git repository
├── web-client/                  # independent Git repository
├── infrastructure/              # independent Git repository
└── coordination/                # local/private task material; not necessarily versioned
    └── tasks/
        └── subscription-change/
            ├── brief.md
            ├── interface-decisions.md
            └── status.md
```

Record which repository, branch, and commit each result targets. Keep cross-repo contracts and sequencing in the task brief. Do not treat sibling directories as one Git repository, and do not assume an agent working in one repository can see changes made in another.

### D. Avoid premature structure

For a one-off task, a focused brief and the existing project structure may be enough; do not create an elaborate `agents/`, `memory/`, or workflow framework just because agents are involved. Add folders when they solve a repeated retrieval, ownership, or review problem. Prefer isolated worktrees or branches to multiple agents editing the same checkout simultaneously.

## 3. Define roles and orchestration boundaries

An agent's role should describe a responsibility, not grant it broad authority. One agent can hold multiple roles for a small task.

| Role | Owns | Should not do by default |
|---|---|---|
| User / product owner | Goal, priorities, acceptance, and decisions that affect users or scope | Delegate final accountability to an agent |
| Orchestrator | Preserve the request, write the task brief, plan dependencies, assign bounded work, integrate results, and report status | Claim a worker result is correct without checking it |
| Implementer | Make a focused change within assigned files or module boundaries and report files changed and checks run | Expand scope, redefine acceptance, or edit another worker's area without coordination |
| Researcher / designer | Gather evidence, compare options, and record assumptions or an interface proposal | Present an unverified proposal as an approved decision |
| Tester / reviewer | Independently assess behavior, edge cases, and test coverage; report actionable findings | Make unrequested product changes or approve its own implementation |

### Dependency and ownership rules

1. The orchestrator turns the request into acceptance criteria and identifies decisions that need user input (or, in an unattended flow, clearly records the chosen assumption).
2. Split work by deliverable and file/module ownership. Each task has one accountable implementer; avoid concurrent edits to the same files.
3. Express real dependencies explicitly. For example: agree on the API schema before the API and UI implementations; implementation must precede review; review findings must be resolved or consciously accepted before completion.
4. Delegate independent research and isolated implementation in parallel. Keep tightly coupled edits sequential.
5. Workers return a handoff: outcome, files or artifacts, decisions/assumptions, validation, known gaps, and the next dependency. The orchestrator reconciles conflicts and checks that every acceptance criterion is covered.
6. Only the user or an explicitly authorized release process approves scope changes, sensitive access, deployments, and merges.

More agents do not automatically mean more throughput. Every delegation creates context, integration, and review costs.

## 4. Set up MCP tools safely

MCP is a way for a compatible client to connect an agent to external tools or data. It does not by itself guarantee that a tool is trustworthy, available, authenticated, or authorized for every agent. Treat each server and tool as an integration that needs a clear owner and access boundary.

### Decide which scope owns the configuration

The exact scopes and labels vary by client. Commonly, setup may be managed in a user's client settings, at an organization or enterprise level, or through repository-provided configuration interpreted by a compatible client. These scopes are not interchangeable:

- **User/client scope:** A person's installed or personal server connection and sign-in. It may be available across repositories in that client.
- **Organization scope:** Centrally managed availability or policy, when the product supports it.
- **Repository scope:** Configuration or recommendations associated with a repository. It can affect collaborators who use a compatible client; review it like other repository configuration.

Before enabling a server, identify the configuration source, which repositories or users it affects, who maintains it, and how changes are reviewed. Follow the specific Copilot client and MCP server's current official setup instructions. Do not assume that one client's JSON example, server URL, executable command, environment-variable name, or UI label applies to another.

### Authentication and permissions

- Authenticate through the client/server's supported sign-in flow. Do not paste access tokens, passwords, private keys, or other credentials into prompts, task files, source code, or commits.
- Use the minimum account and permissions needed for the task. Prefer read-only access for search or review tasks; grant write or administrative capabilities only when necessary and authorized.
- Keep credentials in the supported secret store or authentication flow, not in versioned repository configuration. Repository configuration should not contain a personal secret.
- Inspect repository-supplied server configuration before enabling it, especially when it comes from an unfamiliar branch or contributor.
- Confirm the server connected and the required tools are available. If sign-in, permissions, policy, or network access fails, report that explicitly; do not silently substitute an unapproved account or bypass the restriction.
- Tell workers which tools are in scope and what actions are permitted. A connected MCP server does not imply permission to publish, modify, or delete external data.

## 5. Keep prompts and context continuous

Chat history is useful context, but it is not a dependable task database. Long conversations can be truncated, sessions can end, and separate agents do not automatically share one another's context. Store essential intent in a concise source of truth and pass the relevant parts directly to each worker.

### Preserve what the user actually asked for

For every substantial task, record:

1. **Original request:** Include the user's request verbatim (or the exact relevant excerpt for a delegated subtask). Do not silently rewrite an ambiguous request into a different requirement.
2. **Task brief:** Summarize the outcome, scope, non-goals, constraints, acceptance criteria, target repository/branch, and relevant links or files.
3. **Decisions and assumptions:** Record who decided, what was decided, and any unresolved question. Distinguish user-approved decisions from agent assumptions.
4. **Task state:** Track owners, dependencies, status, changed files, checks, blockers, and next action. Update this as work progresses; do not rely on an agent's implicit memory.
5. **Handoff:** Before pausing or transferring work, capture what is done, what is not done, where the source of truth lives, and the precise next step.

Stable project instructions should contain enduring conventions, architecture facts, and safety requirements. A task brief should contain the current request and acceptance criteria. A progress log or handoff should contain transient execution state. Avoid duplicating contradictory versions of the same requirement across multiple files; name one canonical brief and link to it from other artifacts.

### Example task brief

```markdown
# Task: Add CSV export

## Original user request
> Add an export button to the report page so I can download the currently filtered rows as CSV.

## Outcome
Allow a signed-in user to download the rows matching the active report filters.

## Acceptance criteria
- The downloaded file contains the currently filtered rows and column headings.
- Values containing commas, quotes, or newlines are validly escaped.
- Empty results are handled without an error.
- Automated tests cover normal and quoted values.

## Scope and non-goals
- In scope: report page and its export path.
- Out of scope: new report filters, background exports, and unrelated formatting.

## Constraints and source of truth
- Follow the existing export and authentication patterns.
- Relevant code: `src/reports/` and `tests/reports/`.
- Project conventions: `.github/copilot-instructions.md`.

## Ownership and dependencies
- Implementer: Agent A, report/export module.
- Reviewer: Agent B, after implementation and tests.
- Do not edit the shared report filter model without coordinating with the orchestrator.

## Status and handoff
- Status: in progress
- Last verified commit: <commit>
- Checks run: <commands and results>
- Remaining work / next action: <specific action>
- Decisions or assumptions: <decision, owner, or none>
```

Replace the example request with the actual user's words. Do not invent acceptance criteria that conflict with the request; surface important uncertainty instead.

### Context package for each agent

Send each agent a compact, complete packet, not just “continue” or “implement this”:

- the verbatim request or relevant excerpt;
- the canonical task brief and acceptance criteria;
- project instructions and only the code/docs needed for its assignment;
- assigned files, ownership boundaries, dependencies, and non-goals;
- available tools and allowed external actions;
- expected deliverable and handoff format.

Agents should report uncertainty instead of filling missing context with invented requirements. When a session resumes, reload the brief and current task state, inspect the live repository status, and confirm the current branch before continuing. Treat the brief as the source of truth if chat history or a remembered summary conflicts with it.

## 6. Prompt templates

Adapt these templates to the agent runtime. Include the relevant context in the actual message; do not rely on a child agent having access to the orchestrator's private conversation.

### Orchestrator

```text
You are the orchestrator for this task.

Original user request (verbatim):
<paste the request>

Canonical task brief:
<path or contents>

First, restate the intended outcome and acceptance criteria without changing
their meaning. Identify any unresolved decisions. Propose a dependency-aware
task breakdown with one owner and a clear file/module boundary per task.
Delegate only independent bounded tasks. Keep changes to shared interfaces
sequential until their contract is agreed. Give every worker the context packet
it needs. Track status and handoffs in the task brief.

Before reporting completion, inspect the integrated diff, map every acceptance
criterion to evidence, run the relevant checks, and report remaining risks.
Do not merge, publish, deploy, delete external data, or broaden permissions
unless explicitly authorized.
```

### Implementer

```text
Implement only this assigned task: <task and owner>.

Original user request (verbatim or relevant excerpt):
<paste it>

Acceptance criteria:
<criteria>

Read <project instructions> and <canonical task brief>. Own only these files
or module boundaries: <paths>. Do not change the agreed interface, expand
scope, or edit another worker's area without coordinating with the
orchestrator. If requirements conflict or are insufficient, stop and report
the exact ambiguity rather than inventing a product decision.

Deliver a focused change, relevant tests, and a handoff with files changed,
decisions/assumptions, checks and results, known gaps, and the next action.
```

### Independent reviewer

```text
Review this change against the original request and acceptance criteria.

Request / task brief:
<paste or link the canonical source>

Review the diff for concrete defects, regressions, missing edge cases, and
security or privacy risks. Do not rewrite the implementation. For each
finding, give severity, file/line, impact, and evidence. Separate confirmed
issues from questions and optional improvements. If there are no actionable
findings, say so and note what you could not verify.
```

## 7. Validate, review, and recover

### Before delegation

- Confirm repository, branch, worktree status, and the canonical task brief.
- Break down acceptance criteria into independently verifiable work.
- Check that parallel tasks do not own the same files or depend on an undecided contract.
- Give each worker only the access and MCP tools it needs.

### At integration

- Inspect actual diffs and changed files; do not accept a summary as proof of the change.
- Run targeted tests, lint, type checks, or builds appropriate to the touched code. Record commands and outcomes in the task state.
- Review security and privacy implications, especially new external tools, credentials, permissions, user data, and generated outputs.
- Resolve each review finding explicitly: fix, verify as not applicable with evidence, or defer with an owner and reason.
- Check every acceptance criterion against concrete evidence. A passing test suite does not prove a requirement that has no test or other verification.

### When something goes wrong

| Situation | Recovery |
|---|---|
| Agent loses context or a session ends | Reload the canonical brief and latest handoff; inspect the current branch/status and recent changes before resuming. Re-send the exact request and acceptance criteria. |
| Two agents edit overlapping files | Pause one owner, compare both diffs, agree on a single owner, and integrate the intended changes deliberately. Do not overwrite or reset another agent's work. |
| A worker changes scope or makes an unapproved assumption | Stop dependent work, compare the change with the verbatim request, and ask the user or orchestrator to decide. Update the brief before continuing. |
| A check fails | Capture the exact command and failure, determine whether it is caused by the change or is a known baseline issue, fix or report it, and rerun the relevant check. Do not label the task complete with an unresolved failure. |
| MCP connection or permission fails | Report which server/tool and permission step is blocked. Use the supported authentication/settings path or proceed without that capability if possible; never copy secrets into prompts or disable policy controls. |
| Integration conflict or incompatible outputs | Revisit dependency order and agreed interfaces. Have the orchestrator choose one coherent result, record the decision, and rerun tests/review on the integrated diff. |
| Work must be paused | Update status, changed files/commit, checks, unresolved questions, and one precise next action in the handoff. Preserve unfinished work; do not discard it to make the workspace appear clean. |

## 8. A realistic first implementation

For an initial rollout, implement just this loop:

1. Add one project-level instruction file using a format your agent client actually supports.
2. Create a reusable task brief with the original request, acceptance criteria, non-goals, ownership, and a handoff section.
3. Ask one orchestrator agent to plan the task, then use one implementer for a bounded change.
4. Work in a clean branch or isolated worktree. Run the existing targeted checks and inspect the diff.
5. Ask a separate reviewer only if the change warrants it; a reviewer should inspect rather than silently rewrite the implementation.
6. Record what was lost, misunderstood, or duplicated, then improve the brief or workflow based on evidence.

Add multi-agent parallelism, MCP integrations, automation, and richer status tracking only when a repeated need justifies each one. The durable brief, explicit ownership, and verified handoff are the core system; the number of agents is secondary.

## 9. Completion checklist

- [ ] The actual user request and acceptance criteria are available in the canonical task brief.
- [ ] Each delegated agent received the relevant request, instructions, constraints, and ownership boundary.
- [ ] Dependencies and shared-interface decisions were handled in the right order.
- [ ] The integrated change and changed-file list were inspected.
- [ ] Relevant checks passed, or remaining failures are accurately disclosed.
- [ ] Review and security findings were resolved or explicitly documented.
- [ ] The task state includes the outcome, verification, remaining risks, and handoff or next step.
- [ ] Any external action (publishing, merging, deployment, or data modification) was explicitly authorized.
