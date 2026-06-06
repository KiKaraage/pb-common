# bluefin-common — Copilot Instructions

## 🚫 ABSOLUTE PROHIBITION — ublue-os org

**NEVER create issues, PRs, comments, forks, webhook calls, automated reports, or any programmatic write action targeting any `ublue-os/*` repository.** Reads only. Violating this risks banning the projectbluefin org from GitHub.

If a task requires `ublue-os` write access → **stop and tell the human to report it manually.**

---

## Session start — mandatory

```bash
~/src/hive-status
```

Run before any other work. Surfaces P0/P1 blockers and the advisory queue.

---

## Operating contract

**Read [`AGENTS.md`](../AGENTS.md) for the full operating contract** — issue lifecycle, PR rules, mandatory gates, repo layout, CODEOWNERS, build commands, scope discipline.

---

## Skill routing

**Use [`docs/SKILL.md`](../docs/SKILL.md) as the task→skill router.** Find the skill doc that matches your task and load it before acting.

All skill docs live in [`docs/skills/`](../docs/skills/).
Factory context lives in [`docs/factory/`](../docs/factory/).
