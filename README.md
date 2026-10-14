# archquiz

Two [agent skills](https://agentskills.io/specification) that quiz you on how your codebase works:

1. **Map your system:** run the `archquiz-index` skill. It proposes the areas worth understanding and writes them to `.archquiz/` once you confirm. Run it again monthly, or when a quiz says the index is out of date.
2. **Get quizzed:** run the `archquiz-quiz` skill. Ten minutes of questions written from the current code, scored per area; areas you get wrong come back sooner.

## Install

Copy the two folders in `skills/` to where your agent loads skills, such as `~/.claude/skills/` for Claude Code.

## Why

Agents keep adding parts you never learned. The quiz finds the parts of your system you no longer understand, your [comprehension debt](https://codemanship.wordpress.com/2025/09/30/comprehension-debt-the-ticking-time-bomb-of-llm-generated-code/), before you have to fix them.

## How it works

- The index names areas (checkout, payments, order export, deployment, ...) and where in the code each one starts, but never describes them, so it doesn't go stale.
- Every quiz is written from the current code, and every wrong answer becomes study notes, a corrected answer key or a fix.
- If your repository has a CODEOWNERS file, each person is quizzed on the parts they own, including those owned by their teams. Without one, you're quizzed on the whole system.
- Both skills write only to `.archquiz/`. Quiz history is gitignored by default.

## Status

Early: used on one codebase so far.

## License

[Apache 2.0](LICENSE)
