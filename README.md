# archquiz

Two [agent skills](https://agentskills.io/specification) that quiz you on how your codebase works, so you find the parts you no longer understand, your [comprehension debt](https://codemanship.wordpress.com/2025/09/30/comprehension-debt-the-ticking-time-bomb-of-llm-generated-code/), before you have to fix them.

- **`archquiz-index`** proposes the areas of your system worth understanding (checkout, payments, order export, deployment, ...) and where in the code each one starts. It names areas but never describes them, so it doesn't go stale.
- **`archquiz-quiz`** runs a ten-minute quiz with questions written from the current code, scores each area, and turns every wrong answer into study notes, a corrected answer key or a fix. Areas come back on a spaced schedule.

Both skills write only to `.archquiz/` in your repository. Your quiz history is gitignored by default.

## Install

Copy the two folders in `skills/` to where your agent loads skills, such as `~/.claude/skills/` for Claude Code. Then ask it to set up architecture quizzes, and later to quiz you.

## Status

Early: used on one codebase so far.

## License

[Apache 2.0](LICENSE)
