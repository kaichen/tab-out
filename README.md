# Tab Out (modified fork)

**Keep tabs on your tabs.**

This is a modified fork of [zarazhangrui/tab-out](https://github.com/zarazhangrui/tab-out). See [Changes in this fork](#changes-in-this-fork) below for what's different.

Tab Out is a Chrome extension that replaces your new tab page with a dashboard of everything you have open. Tabs are grouped by domain, with homepages (Gmail, X, LinkedIn, etc.) pulled into their own group. Close tabs with a satisfying swoosh + confetti.

No server. No account. No external API calls. Just a Chrome extension.

---

## Install with a coding agent

Send your coding agent (Claude Code, Codex, etc.) this repo and say **"install this"**:

```
https://github.com/kaichen/tab-out
```

The agent will walk you through it. Takes about 1 minute.

---

## Features

- **See all your tabs at a glance** on a clean grid, grouped by domain
- **Homepages group** pulls Gmail inbox, X home, YouTube, LinkedIn, GitHub homepages into one card
- **Close tabs with style** with swoosh sound + confetti burst
- **Duplicate detection** flags when you have the same page open twice, with one-click cleanup
- **Click any tab to jump to it** across windows, no new tab opened
- **Save for later** bookmark tabs to a checklist before closing them
- **Localhost grouping** shows port numbers next to each tab so you can tell your vibe coding projects apart
- **Expandable groups** show the first 8 tabs with a clickable "+N more"
- **100% local** your data never leaves your machine
- **Pure Chrome extension** no server, no Node.js, no npm, no setup beyond loading the extension

---

## Changes in this fork

Compared with the upstream repo ([zarazhangrui/tab-out](https://github.com/zarazhangrui/tab-out)):

- **Custom groups** — aggregate any set of hosts into a single group card. Configure them through a settings modal (gear button in the top-right corner); groups are stored in `chrome.storage.local`. Seeded by default with an **AI** group that bundles 16 AI chat sites (ChatGPT, Claude, Gemini, Kimi, DeepSeek, Grok, Qwen, ChatGLM, Yuanbao, Yiyan, Hailuo, Doubao, ...) into one card.
- **Priority band** — pinned tabs from a few key hosts (GitHub, Gist, X, YouTube, Bilibili) always sit in a top row above all other groups.
- **Implicit groups** — code-defined groups that merge docs/search sites without cluttering the settings UI: GitHub Docs, OpenAI Docs, Claude Docs, Feishu, and Search engines.
- **Built-in homepage patterns** — the landing-page patterns that used to live in an optional `config.local.js` are now built into `app.js`, so no personal config file is needed.
- **Pinned tabs flagged** — Chrome-pinned tabs are marked in the dashboard.

---

## Manual Setup

**1. Clone the repo**

```bash
git clone https://github.com/kaichen/tab-out.git
```

**2. Load the Chrome extension**

1. Open Chrome and go to `chrome://extensions`
2. Enable **Developer mode** (top-right toggle)
3. Click **Load unpacked**
4. Navigate to the `extension/` folder inside the cloned repo and select it

**3. Open a new tab**

You'll see Tab Out.

---

## How it works

```
You open a new tab
  -> Tab Out shows your open tabs grouped by domain
  -> Homepages (Gmail, X, etc.) get their own group at the top
  -> Click any tab title to jump to it
  -> Close groups you're done with (swoosh + confetti)
  -> Save tabs for later before closing them
```

Everything runs inside the Chrome extension. No external server, no API calls, no data sent anywhere. Saved tabs are stored in `chrome.storage.local`.

---

## Tech stack

| What | How |
|------|-----|
| Extension | Chrome Manifest V3 |
| Storage | chrome.storage.local |
| Sound | Web Audio API (synthesized, no files) |
| Animations | CSS transitions + JS confetti particles |

---

## License

MIT

---

Built by [Zara](https://x.com/zarazhangrui)
