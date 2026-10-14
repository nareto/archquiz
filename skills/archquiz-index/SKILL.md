---
name: archquiz-index
description: >-
  Create or update the architecture area index in .archquiz/ that the
  archquiz-quiz skill uses to quiz people on how a codebase works. Use when
  setting up architecture quizzes in a repository, when a quiz reports missing
  or uncovered areas, or when the user asks to refresh the area index.
compatibility: Requires git and a coding agent that can read files and run shell commands.
metadata:
  version: "0.2.0"
---

# Architecture quiz index

This skill maintains `.archquiz/` at the root of the current Git repository. The folder holds the quiz configuration and the index of **areas**: the parts of the system someone working on it should understand, arranged as a tree (for example Orders, then Checkout and Payments under it). The `archquiz-quiz` skill picks areas from this index and writes questions about them from the current code each time, so the index itself must contain nothing that can go out of date when the code changes.

## Rules

- The index names areas and says where to start looking. It never says how anything works or who owns it: no data flows, dependencies, owners, technologies, versions, counts or behaviour. "Payments: how an order is charged, refunded and reconciled" is right. "Payments: the API charges the card before it saves the order" is wrong, because it becomes false when the code changes while the area stays.
- Only code and configuration inside the configured roots count as evidence that an area exists. Documentation, diagrams and READMEs can suggest where to look, but never justify an area on their own.
- Area IDs are permanent. Never change an ID or reuse a retired one; quiz history is keyed by ID.
- Never delete an area. Retire it so its history stays readable.
- Never treat `.archquiz/` as part of the system.
- Do not inspect running systems. Everything comes from files in the roots.
- Show the user the proposed index or the proposed changes, and write them only after they confirm.

## What counts as the system

Everything in the roots that runs, deploys, stores data or instructs agents:

- application source, entry points and interfaces between components;
- configuration and infrastructure as code: container and orchestration files, provisioning, service units and timers, scheduled jobs, proxies, networking;
- build, packaging, CI/CD workflows and deployment scripts;
- data: schemas, migrations, storage layout, backup and retention settings;
- agent instructions and automation: AGENTS.md and similar files, skills, prompts, hooks, scheduled agent jobs and helper scripts.

## Folder layout

```text
.archquiz/
  config.yaml     scope and session settings, owned by the user
  index.yaml      area tree, owned by this skill
  .ignore         contains "*": hides the folder from ripgrep-based agent searches
  .gitignore      contains "learner/" when learner data is local
  learner/        quiz history, written by archquiz-quiz
    <name>/       one folder per person quizzed
      profile.yaml
      progress.yaml
      sessions/
      notes/
```

The `.ignore` file keeps quiz records and study notes out of ordinary agent searches, so no agent mistakes them for documentation. Skills read these files by explicit path.

## config.yaml

```yaml
format_version: 1
roots:                    # every Git repository that is part of the system
  - name: shop            # short name used in hints and evidence, e.g. shop:api/server.py
    path: .               # relative to the repository that contains .archquiz/
  - name: infra
    path: ../shop-infra
exclude:                  # root:path prefixes that are not part of the system
  - shop:vendor/
  - shop:docs/
learner_data: local       # local: learner/ is gitignored; commit: learner/ is tracked
session:
  minutes: 10             # target answering time
  areas: 4                # areas per session
  questions_per_area: 2-3
  selection: ""           # optional free-text guidance that replaces the quiz's default area mix
```

List every Git repository as its own root, including repositories nested inside another root. A root never covers a nested repository, because each root's commit is recorded separately.

## index.yaml

```yaml
format_version: 1
updated_at: 2026-10-13T09:12:00+02:00
commits:                  # HEAD of each root when the index was last built or updated
  shop: 1a2b3c4
  infra: 5d6e7f8
areas:
  - id: orders
    title: Orders
    summary: How orders are placed, paid for and fulfilled.
    added_at: 2026-10-13
    children:
      - id: payments
        title: Payments
        summary: How an order is charged, refunded and reconciled.
        added_at: 2026-10-13
        hints:
          - shop:api/payments/
          - shop:worker/refunds.py
          - infra:terraform/payments.tf
      - id: legacy-checkout
        title: Legacy checkout
        summary: The previous checkout flow.
        added_at: 2026-10-13
        status: retired
        retired_at: 2026-11-02
        reason: Its routes were removed.
        replaced_by: [checkout]
```

- `id`: lowercase letters, digits and hyphens, unique across the whole tree. It does not encode the parent, so an area can move without changing its ID.
- `title` and `summary`: one line each. The summary names a concern, never a behaviour.
- `hints`: `root:path` entries (files, folders or globs) where exploration should start. Hints are starting points, not boundaries; the quiz searches beyond them.
- `children`: groups have children; leaves have hints. Only leaves are quizzed. Group scores are rolled up from their leaves.
- `status`: `active` (the default, omitted) or `retired`, with `retired_at`, `reason` and optionally `replaced_by`.

## Designing areas

- Use at most three levels. Pick 4 to 10 top-level groups by concern. When a concern spans several repositories, group by concern rather than by repository.
- Size each leaf so it supports two or three architecture-level questions answerable from memory in about three minutes. Split a leaf that would need more than about six questions to cover; merge one that cannot support two.
- Keep areas at the architecture level: components and their boundaries, how they interact, who owns which state, how things fail, what a change touches, how code reaches production. A single function is never an area.
- Include cross-cutting areas wherever the code has them, because they span several components and are the easiest to misunderstand: end-to-end request or data flows, secrets and configuration, authentication and authorization, deployment and rollback, monitoring and alerting, failure and recovery, scheduled jobs, agent workflows.
- Use the names the code uses.
- If a root has a CODEOWNERS file, follow its ownership boundaries where the code allows, so that most leaves sit inside one owner's paths. The quiz uses CODEOWNERS to give each person the areas they own. Never copy owners into the index: ownership changes, so the quiz reads CODEOWNERS fresh each time.

## Exploring

- List files with `git ls-files` in each root, so generated and vendored files stay out, and apply `exclude`.
- Start from structure: directory layout, package manifests, entry points, CI workflows, deployment and infrastructure files, service units, agent instruction files, CODEOWNERS.
- Read enough code to know which concern each part serves. The index does not need to know how it works.
- Treat documentation only as a suggestion of where to look, and confirm every suggested area in code.
- For a large scope, if your agent can run subagents, give each root or each top-level group to a fresh subagent that returns candidate areas with hints. Merge the results yourself: cross-cutting areas only show up in the merged view.

## Coverage check

After drafting or updating, list the significant code and configuration that no hint and no exclusion covers. Significant means anything that runs, deploys, stores data or instructs agents. For each item, the user decides whether it becomes a new area, a hint on an existing area, or an exclusion.

## Creating the index

1. Find the repository root with `git rev-parse --show-toplevel`. If `.archquiz/` already exists, follow "Updating the index" instead.
2. Propose the scope: the repository itself, any Git repositories nested inside it, and repositories that its code or deployment files clearly depend on. Ask the user which to include, which paths to exclude, and whether learner data stays local or is committed. Write `config.yaml`.
3. Explore every root and draft the area tree.
4. Check every leaf against the rules above and run the coverage check.
5. Show the user the tree as an indented list of titles and summaries, together with the coverage report. Apply their corrections.
6. Write `index.yaml` with the current commit of each root, `.ignore` containing `*`, and, if learner data is local, `.gitignore` containing `learner/`.
7. Tell the user they can now run the `archquiz-quiz` skill. Do not commit.

## Updating the index

1. Read `config.yaml` and `index.yaml`. If `format_version` is higher than 1, stop and tell the user to update this skill.
2. For each root, list what changed since the recorded commit with `git log --stat <commit>..HEAD`, plus uncommitted changes from `git status`. If the recorded commit no longer exists, treat the whole root as changed.
3. Explore the changed paths first, then run the coverage check over the whole scope: new code anywhere can create a new concern or a second way of doing an existing one.
4. Include any problems the user brings from quiz sessions, such as hints that no longer exist or code no area covers.
5. Propose the changes as a list:
   - new areas;
   - retired areas whose code is gone;
   - fixed hints;
   - merges and splits, which retire the old areas with `replaced_by` and add new IDs;
   - edited titles or summaries and moved areas, which keep their IDs.
6. After the user confirms, write `index.yaml` with the new commits and `updated_at`. Do not commit.
