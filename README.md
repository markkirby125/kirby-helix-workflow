# Kirby Helix Workflow

Bootstraps a strict, production-ready agentic coding workflow based on Shopify's Helix architecture: ordered checkpoints, context-isolated subagents, 4 deterministic gates (tests, UI, adversarial review, human-in-the-loop), and dual-tier `learnings.md` memory so agents never repeat a mistake.

### 🪄 The Magic Prompt
Copy and paste this into any AI coding app (Cursor, Windsurf, Claude Code, Grok, Kimi, Reasonix):

```markdown
@agent Install the Kirby Helix Workflow skill from: https://github.com/markkirby125/kirby-helix-workflow
1. Read `SKILL.md` and `references/` from the repository.
2. Place into your environment's skills/rules directory preserving the dispatcher structure.
3. Confirm when installation is complete.
```

## Security Notes

- Rollback of a corrupted `~/.agents/AGENTS.md` merge is **human-gated**: the agent must pause, show the failure, and get explicit confirmation before restoring a backup.
- Writes to user configuration always have a stated undo path: a timestamped backup when the file exists, or removal of the created file on a fresh install. A corrupted merge is rolled back only after explicit user confirmation.
