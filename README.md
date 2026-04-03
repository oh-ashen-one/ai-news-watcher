# ai-news-watcher

Get pinged in Discord the moment something big happens in AI — no noise, no blog posts, just breaking product launches, funding rounds, acquisitions, and major releases.

## The Problem

AI moves fast. You find out about GPT-5, a new Claude model, or a billion-dollar acquisition hours after it dropped — because you're busy building.

## The Fix

An OpenClaw skill that polls Google News RSS every 2 hours across 10 AI/dev keywords. Filters for high-signal events only (launches, funding, acquisitions, model releases, outages). Posts the single most newsworthy story to your Discord announcements channel with a 2-line summary of what happened and why it matters.

No API key required. Runs entirely on Google News RSS.

## Setup

See `SKILL.md` for full configuration and the install command.

## Cost

- Model: `haiku` (cheapest available)
- ~$0.001 per run
- ~$0.012/day (runs every 2 hours)

## Schedule

Every 2 hours. Configurable.

## Keywords (default)

OpenAI · Anthropic · Claude AI · vibe coding · Cursor AI · Windsurf AI · GitHub Copilot · Gemini AI · Claude Code · AI coding agent

## License

MIT
