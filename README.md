# Daily 2026 Skills

Auto-generated agent skills from GitHub's trending open source projects, by [ara.so](https://ara.so).

Every hour, a GitHub Actions workflow checks what's trending on GitHub and creates a comprehensive agent skill for the top project. Skills are published to [skills.sh](https://skills.sh/Arasoai/daily-2026-skills) and installable by any AI coding agent.

## Install all skills

```bash
npx skills add Arasoai/daily-2026-skills
```

## Install a specific skill

```bash
npx skills add Arasoai/daily-2026-skills --skill lightpanda-browser
```

## Skills Index

| Skill | Description | Source | Date |
|-------|-------------|--------|------|
| [lightpanda-browser](skills/lightpanda-browser/) | Headless browser built in Zig for AI and automation | [lightpanda-io/browser](https://github.com/lightpanda-io/browser) | 2026-03-15 |
<!-- SKILL_INDEX -->

## How it works

1. GitHub Actions cron runs every hour
2. Fetches the hottest new repos (created in the last 7 days, sorted by stars)
3. Skips repos that already have a skill
4. Uses the Claude API to generate a comprehensive SKILL.md from the repo's README
5. Commits, pushes, and registers on skills.sh

## By ara.so

[Ara](https://ara.so) — instant AI agent environments in the cloud. This repo is part of our commitment to the open source AI ecosystem.

## License

MIT

