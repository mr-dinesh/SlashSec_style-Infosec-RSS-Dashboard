# //SlashSec — RSS Dashboard

> **News for Defenders. Stuff that Matters.**

A self-contained, single-file cybersecurity news aggregator styled after the Slashdot magazine format. Pulls live RSS feeds from five authoritative infosec sources, routes them through Groq's LLM for AI-generated 3–4 sentence summaries, severity ratings, and witty department lines — then renders everything as a scannable, filterable newsroom.

![Single HTML File](https://img.shields.io/badge/Single%20HTML%20File-amber) ![No Build Step](https://img.shields.io/badge/No%20Build%20Step-green) ![Groq Powered](https://img.shields.io/badge/Groq%20Powered-indigo) ![5 Live RSS Feeds](https://img.shields.io/badge/5%20Live%20RSS%20Feeds-blue) ![Zero Dependencies](https://img.shields.io/badge/Zero%20Dependencies-green)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Features](#2-features)
3. [Quick Start](#3-quick-start)
4. [News Sources](#4-news-sources)
5. [Architecture](#5-architecture)
6. [Groq Integration](#6-groq-integration)
7. [UI Guide](#7-ui-guide)
8. [Troubleshooting](#8-troubleshooting)
9. [FAQ](#9-faq)

---

## 1. Overview

SlashSec is a browser-based infosec intelligence dashboard that aggregates real-time cybersecurity news from five trusted sources and uses the **Groq LLM API** to summarise, score, and categorise every article — all inside a single `.html` file you open locally or host anywhere.

There is no server, no database, no build pipeline, and no tracking. Your Groq API key is stored only in `sessionStorage` and never leaves your browser except to call Groq's own API endpoint.

> **Philosophy:** All article links come directly from the RSS feeds themselves. Groq never generates URLs. Every "Read More" link carries a green **✓ RSS source** badge confirming it is a verified, canonical link from the originating publication.

---

## 2. Features

| Feature | Description |
|---|---|
| 📡 **5 Live RSS Feeds** | SANS ISC, Schneier, Krebs, Bleeping Computer, and This Week in Security — fetched concurrently on every refresh |
| 🤖 **Groq AI Summaries** | `llama-3.3-70b-versatile` writes a 3–4 sentence summary, assigns severity, generates tags, and coins a witty dept. line per article |
| 🗞 **Slashdot Layout** | Masthead, sticky nav, two-column main + sidebar, bylines, dept. lines, score notation — faithful to the Slashdot format |
| 🚦 **Severity Filtering** | Filter the feed by High, Medium, or Low severity with one click. Stat cards in Dashboard view jump to filtered results |
| 🎨 **Source Colour Coding** | Each source has a distinct accent colour applied to story bars, badges, tags, and sidebar indicators |
| ☾ **Dark & Bright Themes** | Toggle between a deep terminal dark theme and a warm newsprint bright theme. Persisted across sessions |
| 📊 **Intelligence Dashboard** | Stat cards, severity distribution bar, per-source article counts, and a top-tag cloud |
| 🔐 **Key Never Leaves Browser** | Your Groq API key lives only in `sessionStorage`. Sent exclusively to `api.groq.com` — never to any proxy |

---

## 3. Quick Start

No installation. No npm. No build step.

### Step 1 — Get a free Groq API key

Sign up at [console.groq.com](https://console.groq.com). The free tier is generous — each dashboard refresh uses roughly 8–12 API calls (one batch per 8 articles). A key looks like `gsk_abc123…`.

### Step 2 — Open the HTML file

Download `Infosec_Intel_Dashboard_v7_slashdot.html` and open it in any modern browser (Chrome, Firefox, Safari, Edge). No web server required — a local `file://` URL works fine.

### Step 3 — Paste your key and click Fetch Stories

The sticky API bar is always visible below the navigation. Paste your `gsk_…` key into the input field, then click **▶ Fetch Stories**. Live feed status indicators show each source loading in real time. AI summarisation runs automatically once feeds are loaded.

> ⚠️ **RSS feeds require CORS proxies.** The dashboard tries three independent proxies in sequence (`corsproxy.io` → `allorigins.win/raw` → `codetabs.com`). On corporate or restricted networks, all three may be blocked. If feeds fail, try a personal network or browser without strict CSP enforcement. See [Troubleshooting](#8-troubleshooting) for details.

---

## 4. News Sources

Five sources are fetched on every refresh, each pulling up to 8 recent articles.

| Icon | Source | Author / Org | Colour | Primary RSS |
|---|---|---|---|---|
| 📡 | **SANS ISC** | SANS Internet Storm Center | `#e8a020` Amber | `isc.sans.edu/rssfeed_full.xml` |
| 🔐 | **Schneier on Security** | Bruce Schneier | `#8888dd` Indigo | `schneier.com/feed/atom/` |
| 🕵 | **Krebs on Security** | Brian Krebs | `#dd6655` Coral | `krebsonsecurity.com/feed/` |
| 💻 | **Bleeping Computer** | Bleeping Computer | `#44aa77` Emerald | `bleepingcomputer.com/feed/` |
| 📰 | **This Week in Security** | Zack Whittaker | `#4499cc` Sky Blue | `this.weekinsecurity.com/feed` |

Each source has a fallback RSS/Atom URL tried if the primary fails. Both RSS 2.0 and Atom feed formats are parsed correctly.

---

## 5. Architecture

The pipeline runs entirely in the browser on every Refresh click.

```
┌─────────────────────────────────────────────────────┐
│              BROWSER (single HTML file)              │
└─────────────────────────────────────────────────────┘

STEP 1 — Parallel RSS Fetch
  Five feeds fetched concurrently via Promise.allSettled()
  Each tries: direct fetch → corsproxy.io → allorigins.win/raw → codetabs.com
  Failed feeds marked ✗, successful feeds marked ✓

STEP 2 — XML Parsing
  DOMParser handles both RSS 2.0 (<item>) and Atom (<entry>)
  Extracts: title, excerpt, URL, date, source metadata
  URLs taken exclusively from RSS feed — never generated by AI

STEP 3 — Groq Batch Summarisation
  Articles chunked into batches of 8
  Each batch → single POST to api.groq.com/openai/v1/chat/completions
  Model: llama-3.3-70b-versatile  |  temp: 0.25  |  max_tokens: 2048
  Returns per article: summary · severity · tags[] · dept line

STEP 4 — Sort & Render
  Sorted: High → Medium → Low, then newest-first within tier
  Rendered as Slashdot-style story list + sidebar + dashboard
```

### Fetch Fallback Chain

RSS feeds from external domains cannot be fetched directly due to browser CORS restrictions. The dashboard tries four routes in sequence, stopping at the first that succeeds:

```javascript
// 1. Direct (works if the site sets permissive CORS — rare)
fetch(url)

// 2. corsproxy.io — primary proxy
fetch(`https://corsproxy.io/?url=${encodeURIComponent(url)}`)

// 3. allorigins.win /raw — returns body directly, no wrapper
fetch(`https://api.allorigins.win/raw?url=${encodeURIComponent(url)}`)

// 4. codetabs.com — fully independent third fallback
fetch(`https://api.codetabs.com/v1/proxy?quest=${encodeURIComponent(url)}`)
```

---

## 6. Groq Integration

Groq is called **only for article enrichment** — never for URL generation. The model receives article titles and excerpts (capped at 480 characters each) and returns structured JSON.

### System Prompt

```
You are a cybersecurity editor writing for a Slashdot-style infosec magazine.
Given a JSON array of articles, return a JSON array where each element has exactly:

{
  "idx":      number — same as input,
  "summary":  string — exactly 3-4 sentences: what happened, who is affected,
              recommended action. Punchy and direct.,
  "severity": "high" | "medium" | "low",
  "tags":     string[] — 2-4 short tags,
  "dept":     string — witty 2-5 word Slashdot dept. line (e.g. "yet-another-breach dept")
}

Respond ONLY with raw JSON array. No markdown, no fences, no explanation.
```

### Token Usage Estimate

| Stage | Approx. tokens per batch of 8 |
|---|---|
| Input (titles + excerpts) | ~1,200 |
| Output (summaries + tags + dept) | ~800 |
| **Total per full 40-article run (5 batches)** | **~10,000** |

> 💡 Groq's free tier handles a full refresh comfortably. A typical run completes in 8–15 seconds and costs well under $0.01 USD.

### Switching Models

Find and edit the constant near the top of the `<script>` block:

```javascript
const GROQ_MODEL = "llama-3.3-70b-versatile";
```

| Model | Speed | Quality | Best for |
|---|---|---|---|
| `llama-3.3-70b-versatile` | Fast | ★★★★★ | Default — best quality |
| `llama-3.1-8b-instant` | Very fast | ★★★☆☆ | Higher rate limits, lower cost |
| `mixtral-8x7b-32768` | Fast | ★★★★☆ | Longer context windows |

---

## 7. UI Guide

### Sticky API Bar

The amber-accented bar below the navigation is always visible, even when scrolled. It contains:

- **Groq key input** — paste your `gsk_…` key here
- **👁 show / 🙈 hide** — toggle key visibility to verify it was pasted correctly
- **▶ Fetch Stories** — triggers the full pipeline; greys out during loading

### Feed Status Pips

During a refresh, per-source status pips appear below the API bar and update in real time:

| Pip | Meaning |
|---|---|
| 🟢 `live` | Feed loaded and parsed successfully |
| 🔴 `fail` | All proxy attempts failed for this source |
| ⚪ `…` | Loading / awaiting proxy response |

### Story Anatomy

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   ← 3px source-colour accent bar
Headline of the Article (IBM Plex Serif)
📡 SANS ISC · Mon 20 Mar 2026 · Score: High Alert · from the yet-another-breach dept.
  ↑ source     ↑ date              ↑ Groq severity    ↑ Groq-written wit

3–4 sentence AI summary: what happened, who is affected,
and what defenders should do. Written by llama-3.3-70b.

[CVE]  [Patch]  [Ransomware]              Read More… ✓
 ← Groq tags →                    ← RSS-verified link →
```

### Views

| View | How to access | What it shows |
|---|---|---|
| **All Stories** | Click "All Stories" in nav | Full Slashdot story feed, filterable by severity. Sidebar with live stats, source counts, and tags |
| **Dashboard** | Click "Dashboard" in nav | Severity stat cards (clickable → jump to filtered feed), distribution bar, per-source counts, tag cloud |

### Theme Toggle

The **☾ / ☀** button in the top-right masthead switches between dark (terminal) and bright (newsprint) themes. Saved in `sessionStorage`.

---

## 8. Troubleshooting

| Problem | Cause & Fix |
|---|---|
| All feeds show 🔴 `fail` | All three CORS proxies are blocked by your network. Try: (1) a personal hotspot, (2) a different browser with no extensions, (3) disable VPN/proxy |
| Groq API error `401` | Invalid or expired API key. Regenerate at [console.groq.com](https://console.groq.com) |
| Groq API error `429` | Rate limit hit. Wait 60 seconds and retry. Consider switching to `llama-3.1-8b-instant` which has higher free-tier limits |
| Typed key is invisible | Click **👁 show** next to the input to toggle visibility |
| Fetch button is greyed out | A refresh is in progress. Wait for it to complete, or reload the page if stuck for >30s |
| Some feeds load, others fail | Normal — some sources block CORS proxies. The dashboard loads whatever it can and shows a partial-load warning |
| Want summaries in another language | Edit the system prompt in `groqBatch()`. Add `"Respond in [language]."` to the end of the system message |
| Want more/fewer articles per source | In `parseItems()`, change `.slice(0, 8)` to your preferred count. More = more tokens and longer load time |

---

## 9. FAQ

**Is my Groq API key safe?**
Yes. The key is stored in `sessionStorage` (cleared when the tab closes) and sent only to `api.groq.com`. No proxy or third-party service ever sees it.

**Can I add my own RSS feeds?**
Yes. Edit the `FEEDS` array in the `<script>` block. Each entry needs: `id`, `name`, `icon`, `color`, `cls`, and a `urls` array (primary + optional fallback). Add a matching `.src-yourname { --sc: your-color; }` CSS rule to apply source colouring.

**Why Slashdot style?**
Slashdot pioneered the "posted by · from the X dept." format that makes dense technical news scannable at pace. The byline carries who, what, when, and tone in a single line — ideal for a security briefing where cognitive load matters.

**Does it work offline?**
No. RSS feeds require network access through CORS proxies, and Groq requires API connectivity. The HTML file opens offline, but Fetch Stories will fail without a connection.

**Can I host this on a web server?**
Yes — it's a static HTML file. Drop it into any web host, S3 bucket, GitHub Pages, or Cloudflare Pages and it works immediately. No server-side code, no build step, no environment variables.

**Can I change the number of articles per source?**
Yes. In `parseItems()`, change `.slice(0, 8)` to your preferred count. Note: more articles means more Groq token usage and a longer load time.

---

## Source Colour Reference

```
📡  SANS ISC             #e8a020  ████  Amber
🔐  Schneier on Security #8888dd  ████  Indigo
🕵  Krebs on Security    #dd6655  ████  Coral Red
💻  Bleeping Computer    #44aa77  ████  Emerald
📰  This Week in Security#4499cc  ████  Sky Blue
```

---

*SlashSec · v7 · Powered by Groq · Single HTML file · No build step · No tracking*

*"The security of a system is only as strong as its most exhausted defender at 3am on a Friday."*
*— Every SOC on-call rotation, everywhere*
