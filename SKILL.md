---
name: ai-news-watcher
description: Monitor AI news via Google News RSS and post breaking stories to Discord with a 2-line summary. Use when you want to be alerted the moment something big drops in AI — model releases, acquisitions, funding rounds, outages.
---

# AI News Watcher

Polls Google News RSS every 2 hours across configurable keywords. Filters out blog fluff. Posts only the top breaking story to your Discord channel with a short "why it matters" summary.

No API keys required. Works out of the box with OpenClaw + Bun.

## Configuration

Before running, fill in these values in the script:

| Placeholder | What to set |
|---|---|
| `YOUR_ANNOUNCEMENTS_CHANNEL_ID` | Discord channel ID to post to (right-click channel → Copy ID) |
| `YOUR_WORKSPACE_PATH` | Full path to your OpenClaw workspace (e.g. `/Users/yourname/.openclaw/workspace`) |
| `YOUR_DISCORD_USER_ID` | Your Discord user ID for @mentions, e.g. `<@123456789>` (optional, leave `""` to disable) |

## Quick Start

**1. Install Bun** (if not already):
```bash
curl -fsSL https://bun.sh/install | bash
```

**2. Create the script**

Save the script below to `YOUR_WORKSPACE_PATH/scripts/news-watcher.ts` and fill in your values.

**3. Create the state directory**
```bash
mkdir -p YOUR_WORKSPACE_PATH/state
```

**4. Test it manually**
```bash
cd YOUR_WORKSPACE_PATH && bun run scripts/news-watcher.ts
```

**5. Schedule the cron** (in OpenClaw or via the CLI):

Tell your OpenClaw agent:
> "Set up a cron that runs `cd YOUR_WORKSPACE_PATH && bun run scripts/news-watcher.ts` every 2 hours using the haiku model"

Or add to your `HEARTBEAT.md`:
```
- Every 2 hours: run scripts/news-watcher.ts and post breaking AI news to Discord
```

## The Script

```typescript
#!/usr/bin/env bun
import { readFileSync, writeFileSync, existsSync } from "fs";

// ── CONFIG ───────────────────────────────────────────────────────────────────
const DISCORD_CHANNEL = "YOUR_ANNOUNCEMENTS_CHANNEL_ID";
const SEEN_FILE = "YOUR_WORKSPACE_PATH/state/news-seen-ids.json";
const USER_TAG = ""; // e.g. "<@YOUR_DISCORD_USER_ID>" or leave empty

const KEYWORDS = [
  "OpenAI",
  "Anthropic",
  "Claude AI",
  "vibe coding",
  "Cursor AI",
  "Windsurf AI",
  "GitHub Copilot",
  "Gemini AI",
  "Claude Code",
  "AI coding agent",
];

const HIGH_SIGNAL = [
  "launches", "release", "acquires", "acquisition", "raises", "funding",
  "announces", "GPT", "model", "update", "new feature", "API", "open source",
  "partnership", "IPO", "breach", "outage", "banned", "lawsuit", "regulation",
  "beats", "surpasses", "replaces", "fires", "hires", "CEO", "billion", "million",
  "introduces", "unveils", "drops", "ships", "integrates",
];

// ── HELPERS ──────────────────────────────────────────────────────────────────
function isHighSignal(title: string): boolean {
  const lower = title.toLowerCase();
  return HIGH_SIGNAL.some(w => lower.includes(w.toLowerCase()));
}

function loadSeen(): Set<string> {
  if (!existsSync(SEEN_FILE)) return new Set();
  try {
    return new Set(JSON.parse(readFileSync(SEEN_FILE, "utf8")));
  } catch { return new Set(); }
}

function saveSeen(seen: Set<string>) {
  writeFileSync(SEEN_FILE, JSON.stringify(Array.from(seen).slice(-500)), "utf8");
}

async function fetchGoogleNews(keyword: string) {
  const url = `https://news.google.com/rss/search?q=${encodeURIComponent(keyword)}&hl=en-US&gl=US&ceid=US:en`;
  const res = await fetch(url, { headers: { "User-Agent": "Mozilla/5.0 (compatible; news-watcher/1.0)" } });
  if (!res.ok) return [];

  const xml = await res.text();
  const items = [];

  for (const match of xml.matchAll(/<item>([\s\S]*?)<\/item>/g)) {
    const block = match[1];
    const titleMatch = block.match(/<title>([\s\S]*?)<\/title>/);
    const guidMatch = block.match(/<guid[^>]*>([\s\S]*?)<\/guid>/);
    const linkMatch = block.match(/<link>([\s\S]*?)<(?:\/link|guid)>/);
    const pubMatch = block.match(/<pubDate>([\s\S]*?)<\/pubDate>/);
    const sourceMatch = block.match(/<source[^>]*>([\s\S]*?)<\/source>/);

    if (!titleMatch || !guidMatch) continue;

    const published = pubMatch ? pubMatch[1].trim() : "";
    if (published) {
      const ageHours = (Date.now() - new Date(published).getTime()) / (1000 * 60 * 60);
      if (ageHours > 3) continue;
    }

    items.push({
      id: guidMatch[1].trim(),
      title: titleMatch[1].replace(/<!\[CDATA\[|\]\]>/g, "").replace(/&amp;/g, "&").replace(/ - [^-]+$/, "").trim(),
      url: linkMatch ? linkMatch[1].trim() : "",
      source: sourceMatch ? sourceMatch[1].replace(/<!\[CDATA\[|\]\]>/g, "").trim() : "Unknown",
    });
  }
  return items;
}

function whyItMatters(title: string, keyword: string): string {
  const t = title.toLowerCase();
  if (t.includes("acqui") || t.includes("buys")) return `${keyword} is making a major business move — could reshape the competitive landscape.`;
  if (t.includes("billion") || t.includes("million") || t.includes("funding") || t.includes("raises")) return `Big money moving into ${keyword} — signals strong investor confidence or major expansion.`;
  if (t.includes("launch") || t.includes("release") || t.includes("ships") || t.includes("drops") || t.includes("unveil")) return `New ${keyword} product or feature just dropped — worth knowing before everyone else does.`;
  if (t.includes("api") || t.includes("open source") || t.includes("model")) return `${keyword} just expanded what developers can build — dev ecosystem impact incoming.`;
  if (t.includes("outage") || t.includes("breach") || t.includes("banned") || t.includes("lawsuit")) return `${keyword} is in hot water — could affect trust, regulation, or market position.`;
  if (t.includes("ceo") || t.includes("hires") || t.includes("fires")) return `Leadership change at ${keyword} — signals a shift in direction or strategy.`;
  return `Significant ${keyword} development worth tracking.`;
}

// ── MAIN ─────────────────────────────────────────────────────────────────────
async function main() {
  const seen = loadSeen();
  const candidates: any[] = [];

  for (const keyword of KEYWORDS) {
    try {
      const items = await fetchGoogleNews(keyword);
      for (const item of items) {
        if (seen.has(item.id)) continue;
        seen.add(item.id);
        if (!isHighSignal(item.title)) continue;
        const score = HIGH_SIGNAL.filter(w => item.title.toLowerCase().includes(w.toLowerCase())).length;
        candidates.push({ keyword, ...item, score });
      }
    } catch (e) { console.error(`Error: ${keyword}`, e); }
    await new Promise(r => setTimeout(r, 400));
  }

  saveSeen(seen);
  if (candidates.length === 0) { console.log("Nothing new."); return; }

  candidates.sort((a, b) => b.score - a.score);
  const top = candidates[0];
  const tag = USER_TAG ? `${USER_TAG} ` : "";

  const message = [
    `📡 **Breaking AI News** ${tag}`,
    ``,
    `**${top.title}**`,
    `*${top.source}*`,
    `<${top.url}>`,
    ``,
    `**Why it matters:** ${whyItMatters(top.title, top.keyword)}`,
    candidates.length > 1 ? `**Also trending:** ${candidates.length - 1} other stories this cycle.` : "",
  ].filter(Boolean).join("\n");

  console.log(message);

  Bun.spawnSync([
    "openclaw", "message", "send",
    "--channel", "discord",
    "--target", DISCORD_CHANNEL,
    "--message", message,
  ]);
}

main().catch(e => { console.error(e); process.exit(1); });
```

## Customizing Keywords

Edit the `KEYWORDS` array to track anything — company names, technologies, product names, competitor names. Google News RSS supports any search query.

## Output Format

```
📡 Breaking AI News @you

**Microsoft Introduces 3 Foundational AI Models To Take on OpenAI**
*The Next Web*
<https://...>

Why it matters: Big tech is moving on OpenAI — competition that shifts the power dynamic.
Also trending: 4 other stories this cycle.
```

## How It Works

1. Fetches Google News RSS for each keyword
2. Filters stories older than 3 hours (catches only fresh news)
3. Scores each headline by how many high-signal words it contains
4. Posts the top-scoring story with a "why it matters" summary
5. Deduplicates via a local JSON file — you'll never see the same story twice
