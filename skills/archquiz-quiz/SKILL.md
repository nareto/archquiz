---
name: archquiz-quiz
description: >-
  Run a short architecture quiz on the current codebase using the .archquiz/
  area index: pick areas from past scores, write questions and an answer key
  from the current code, ask them, score each area, then review the answers and
  discuss fixes, issues or study notes. Use when the user asks to be quizzed on
  how their system works, or for their regular architecture review.
compatibility: Requires git, an .archquiz/ folder created by the archquiz-index skill, and a coding agent that can read files and run shell commands.
metadata:
  version: "0.2.1"
---

# Architecture quiz

A session takes about ten minutes of the user's answering time. It tests whether the user understands how the system works at the architecture level, scores them per area, and turns every wrong answer into one of three things: learning, a corrected answer key, or a change to the system.

The session runs in six phases: pick areas, prepare questions, ask, grade and review, discuss, record.

## Rules

- The code and configuration at the current commit of each root are the only evidence. Never base an answer on documentation, index summaries, earlier sessions or study notes: they are claims, and claims go out of date. Do not run the system or query live services.
- Ask about architecture: what runs where, who owns which state, how a request, job or piece of data flows end to end, what breaks when something fails, what a change would touch, how code reaches production. Do not ask about single-function details or trivia such as exact port numbers unless they matter to how the system works.
- Write the questions and the full answer key to the session file before asking the first question. Never adjust the key to fit the user's answers; corrections happen only in the discussion phase, with evidence.
- Give no feedback, hints or confirmations until the user has answered every question.
- Everything you store is dated history tied to commits. Never write anything that reads like current documentation.
- The user decides every dispute. Change code, file issues or write study notes only after they agree.

## Before starting

1. Find the repository root with `git rev-parse --show-toplevel` and read `.archquiz/config.yaml` and `.archquiz/index.yaml`. The files under `.archquiz/` are hidden from ripgrep, so open them by path. If either file is missing, if `format_version` is not 1, or if the index has no active leaf areas, stop and tell the user to run the `archquiz-index` skill.
2. Record the current commit of each root with `git -C <root> rev-parse --short HEAD`, and mark a root as dirty if `git status` shows uncommitted changes. Uncommitted changes are part of the evidence.
3. Read `.archquiz/learner/progress.yaml` if it exists. If it is missing but sessions exist, or it disagrees with them, rebuild it from the session files (see Scheduling).
4. If the user asked for particular areas or a different length, follow that instead of the defaults below.

## Phase 1: pick areas

Candidates are the active leaf areas of the index. Picking is your judgement, guided by `session.selection` in the config if it is set, and otherwise by this default mix for a session of four areas:

- two areas never asked before;
- one due area with the lowest last score;
- one due area asked the longest time ago.

An area is due when its `next_due` date is today or earlier. When a slot has no candidate, fill it from the others: due areas first, then new areas, then the areas closest to being due. Among equal candidates, prefer areas from different top-level groups, cross-cutting areas, and areas whose hinted paths changed since they were last asked (`git -C <root> log --since=<last_asked> -- <paths>`).

Create the session file `.archquiz/learner/sessions/<YYYY-MM-DD-HHMM>.md` from the template below, without question blocks, with each picked area and a one-line reason for picking it. Create the `learner/` folders if they don't exist.

## Phase 2: prepare questions

The answer key must stay out of the user's sight. If your agent can run subagents, give this phase to one fresh subagent: point it to this file's Phase 2 and Rules sections, and pass it the session file path, the picked areas with their index entries, the roots and the recorded commits. Tell it to write one block per question into the session file, with the Question and Expected subsections filled in and the Answer and Outcome subsections left empty, and to return only the numbered question texts with their area IDs. Do not read the answer key yourself until Phase 4. If you cannot run subagents, warn the user before you start that the next steps write the answer key, and ask them not to read the tool output until the quiz is over.

Tell the user that preparation takes a few minutes. Then, for each picked area:

1. Explore the current code. Start from the area's hints, then search all roots for other code touching the same concern: other callers, configuration, jobs, deployment files, agent instructions. A second path that bypasses the hinted one is exactly what the quiz should catch. Note any hint that no longer exists.
2. Write the questions, `session.questions_per_area` of them (two or three by default). Mix question types across the session. Each question must be answerable from memory in about a minute and have one answer the code clearly supports. Prefer short free-text answers. Use multiple choice for at most half of the questions, with four options whose wrong answers come from the real system. Skip anything the code cannot settle, such as behaviour that depends on runtime data or external services.
3. Write the answer key for each question: the expected answer, two to four rubric points with point values, and the evidence as `root:path:line` with a short note on what each location shows.
4. Check the key: reopen every evidence location and confirm it supports the rubric. Confirm that the rubric rewards every part the question asks for, so that a correct answer to the question as worded cannot lose points. Avoid open lead-ins such as "what happens" unless the rubric covers that description. Fix or replace any question that fails.

## Phase 3: ask

Tell the user how many questions there are and which areas they cover (titles only), that they should answer from memory without looking at the code, and that "don't know" is a fine answer. Ask the questions one at a time, grouped by area. If the user asks what a question means, rephrase it without hinting at the answer. If they want to stop early, end the phase; questions not asked don't count.

Keep the answers in the conversation and write them to the session file at the start of Phase 4, so the file is not opened while the answer key is still hidden.

## Phase 4: grade and review

1. Write each answer verbatim into its question's Answer subsection, then read the answer key.
2. Score each answer against its rubric. Give partial credit, judge the substance rather than the wording, and score "don't know" as zero. Under the answer, record the score, what earned points and what was missed.
3. Score each area as points earned over points possible. Score each top-level group as the average of its areas asked in this session. Fill in the Scores table in the session file. Lead with the area scores and show each area's previous score from `progress.yaml` next to it. Give an overall score only as a one-line total.
4. Review every question in the conversation in the same shape, whatever its score: a one-line summary of the user's answer, the score, what earned points, what was missed, the expected answer, the evidence locations, and one sentence on why it matters for the system. Everything the session file records about grading must also appear in the conversation. The user should never have to open the file to learn why they lost points.
5. If the user said they don't know, left the answer blank, or said they don't recognise a term in the question, replace the summary, credited, missed and expected lines with a short plain-language lesson: what the thing is, why it exists, then the answer. Start from the term they did not recognise. Keep it informal and self-contained, and list the evidence after it.
6. Compare with history, now that the key is final: read the most recent earlier session for each area. If its answer key states something today's code contradicts, run `git log` and `git diff` between the two recorded commits on the evidence paths. If the code changed, report it as a change in the system since that date and ask whether it was intended. If the code did not change, one of the two keys was wrong: say so, and say which one the evidence supports.

## Phase 5: discuss

Invite the user to challenge grades, ask follow-up questions or dig deeper into an area. For each wrong answer they raise, agree on one outcome and record it:

- **The user was wrong.** Explain using the code, and offer study notes.
- **The key was wrong.** Re-check the code, correct the key and the score in the session file, and mark the correction. This includes correct content that the question invited but the rubric did not reward.

Record each outcome in the question's Outcome subsection.
- **The code is wrong**, because it does not do what the user intends. Offer to fix it now through the repository's normal workflow, to file an issue in the project's tracker, or just to record it. If the intent can be enforced by a deterministic check, such as a test, a dependency rule or a CI check, suggest adding one so the code cannot drift back.

For each area scored below 80%, offer study notes. Write them only if the user agrees, to `.archquiz/learner/notes/<YYYY-MM-DD>-<area-id>.md`. Build them from the current code rather than from the session's answer key, and cite evidence for every claim. Notes can include a Mermaid diagram, a reading path through the code, and a few practice questions. Start each note with the header template below.

## Phase 6: record

1. Finish the session file: per-area scores in the frontmatter and the Scores table, `status` set to `complete` or `partial`, each question's Outcome from Phase 5, the notes written, and any index problems found (missing hints, code that no area covers).
2. Update `progress.yaml` for every area with at least one question asked (see Scheduling).
3. If you found index problems, tell the user to run `archquiz-index` to update the index.
4. Do not commit. Tell the user which files changed.

## Scheduling

Each area sits in a box from 1 to 6. The boxes have these intervals: 1, 3, 7, 14, 30 and 60 days.

- First time asked: a score of 80% or more puts the area in box 3, 50% to 79% in box 2, below 50% in box 1.
- Later sessions: 80% or more moves it up one box (at most box 6), 50% to 79% keeps it in its box, below 50% sends it back to box 1.
- `next_due` is the session date plus the interval of the new box.

To rebuild `progress.yaml`, replay the area scores from the session frontmatter, oldest session first.

## File formats

### Session file

```markdown
---
format_version: 1
started_at: 2026-10-13T08:02+02:00
status: in-progress        # in-progress, complete or partial
commits:
  shop: 1a2b3c4
  infra: 5d6e7f8 dirty
areas:
  - id: payments
    reason: never asked
    score: 0.67            # filled in Phase 4; null if no question was asked
---

# Architecture quiz, 2026-10-13 08:02

Record of a quiz on the code at the commits above. Not documentation.

## Scores

| Area | Score | Previous |
|---|---|---|
| payments | 2/3 (0.67) | 0.4 |

Total: 2/3 (0.67).

## Q1 (payments)

### Question
Question text.

### Expected
The answer.

Rubric:
- (2) first point
- (1) second point

Evidence:
- `shop:api/payments/charge.py:14`: what this location shows.

### Answer
> The user's answer, verbatim.

Score: 2/3. Credited: what earned points. Missed: what did not.

### Outcome
Key corrected: what changed and why. Or: code wrong, issue filed: link. Empty if nothing was raised.

## Changes since earlier sessions

## Notes written

## Index problems
```

### progress.yaml

```yaml
format_version: 1
areas:
  payments:
    times_asked: 2
    last_asked: 2026-10-13
    last_score: 0.67
    box: 2
    next_due: 2026-10-16
    history:
      - date: 2026-10-01
        score: 0.4
        session: sessions/2026-10-01-0800.md
      - date: 2026-10-13
        score: 0.67
        session: sessions/2026-10-13-0802.md
```

### Study note header

```markdown
---
created_at: 2026-10-13
area: payments
session: sessions/2026-10-13-0802.md
commits:
  shop: 1a2b3c4
---

# Payments: study notes, 2026-10-13

Study material written after a quiz, from the code at the commits above. It is not documentation and not evidence; check the code before relying on it.
```
