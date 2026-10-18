<div align="center">

# 🐢⚡ KAME

### I build free tools that make AI agents faster and harder to stop.

[![KAME API key rotation](https://img.shields.io/badge/KAME-free%20API%20key%20rotation-2ea44f.svg)](https://github.com/Kame696/kame-api-rotation)
[![Discord](https://img.shields.io/badge/discord-kame055856-5865F2.svg)](#-say-hello)
[![License: MIT](https://img.shields.io/badge/everything-MIT-green.svg)](#)

</div>

---

## 🔑 KAME — the best free API key rotation for AI agents

Join the quota of many API keys, projects and accounts into one. Your agent owns fifteen API keys and uses one. That one hits a rate limit, a daily
quota or an "overloaded" provider, and the turn ends while fourteen healthy keys sit
there untouched. **KAME uses all of them:** the healthiest key for every call, the next
key when a call fails, and — if you keep a list of fallback models — the next model when
your provider is busy. Your agent keeps going.

| | Repository | What it is |
|---|---|---|
| 🏠 | **[kame-api-rotation](https://github.com/Kame696/kame-api-rotation)** | Start here: what KAME does and which port to install |
| 🅰️ | **[kame-api-rotation-for-agent-zero](https://github.com/Kame696/kame-api-rotation-for-agent-zero)** | The Agent Zero plugin — **[v1.8.2.8](https://github.com/Kame696/kame-api-rotation-for-agent-zero/releases/latest)** |
| 🅷 | **[kame-api-rotation-for-hermes](https://github.com/Kame696/kame-api-rotation-for-hermes)** | The Hermes Agent plugin — **[v1.8.2.8](https://github.com/Kame696/kame-api-rotation-for-hermes/releases/latest)** |

**What you get, on both:**

- ⚡ **Every key used.** The least-loaded healthy key per call; a refused call moves to the next key at once.
- 🧠 **Every error read, never guessed.** The provider's own retry time, per-minute vs daily quota, out-of-credit, a revoked key — 82 providers learned from their own documentation.
- 🔀 **Your fallback models, one big pool.** Every reply starts on your best model; a busy, spent or slow one hands the reply to the next model of your list in seconds.
- 🏎️ **Speed first.** A late reply races a backup key, then the next model — the first to answer is shown; a cut answer is continued, not restarted. Median reply on a bad Gemini day: 2.6 s.
- 👀 **You see what happened.** Which model answered each reply and how long it took; a live panel with every key, every rest, every switch.
- 🔒 **Free, MIT, no telemetry.** Keys never leave your machine and never appear in a log.

No provider allowlist anywhere: every decision is made on the evidence in the response,
so a provider that does not exist yet is already covered.

---

## 🧪 What else is coming

I work on two things besides rotation, and they will land here as they become worth
other people's time:

- **Chemistry** — the field I actually come from, and the reason I care whether a long
  unattended run survives the night.
- **AI agents** — the tooling around them: what makes one reliable enough to leave
  alone, and what quietly makes one useless.

Nothing ships here until it has been run against something real. Every project above
was built while using it daily and fixed from its own logs.

---

## 👋 Say hello

**Discord** — `kame055856`

Bug report, a log that looks wrong, or an idea for a port to another host: all welcome.
If a plugin of mine broke your agent, I would genuinely rather hear it than not.

---

## ⭐ Like it?

Free, MIT, nothing to upsell — written by one person against a real free tier, on real
quotas that really run out. A ⭐ costs nothing and helps someone else find the fix for a
problem they are currently blaming on themselves.

---

<div align="center">

*Not affiliated with Agent Zero or Nous Research. Just a person with too many API keys
and not enough quota.*

</div>
