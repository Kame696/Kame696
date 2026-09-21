<div align="center">

# 🐢⚡ KAME

### I build things that makes AI more powerful. Hope you like :)

[![Discord](https://img.shields.io/badge/discord-kame055856-5865F2.svg)](#-say-hello)
[![License: MIT](https://img.shields.io/badge/everything-MIT-green.svg)](#)
[![Donate Bitcoin](https://img.shields.io/badge/donate-bitcoin-f7931a.svg)](#-support-the-work)


</div>

---

## 🔑 API Rotation — the main line

Your agent owns fifteen API keys and uses one. That one hits a rate limit and the
turn ends, while fourteen healthy keys sit there untouched. **KAME picks a key per
call, and a failed call moves to the next key instead of ending your turn.**

One engine, one port per host, same version line on both.

| | Repository | What it is |
|---|---|---|
| 🏠 | **[kame-api-rotation](https://github.com/Kame696/kame-api-rotation)** | The front door — what it does, which port to install, how they stay in step |
| 🅰️ | **[kame-api-rotation-for-agent-zero](https://github.com/Kame696/kame-api-rotation-for-agent-zero)** | The Agent Zero plugin, 1.8.1.0 — verified in real sessions on Agent Zero v2.12 |
| 🅷 | **[kame-api-rotation-for-hermes](https://github.com/Kame696/kame-api-rotation-for-hermes)** | The Hermes plugin, 1.8.1.0 — 2,829 tests, catalog security scan safe, zero third-party packages |

**No provider allowlist anywhere in it.** Every decision is made on evidence in
the response, never on who the provider is — so a provider that does not exist yet
is already covered.

---

## 🧪 What else is coming

I work on two things besides rotation, and they will land here as they become
worth other people's time:

- **Chemistry** — the field I actually come from, and the reason I care whether a
  long unattended run survives the night.
- **AI agents** — the tooling around them: what makes one reliable enough to leave
  alone, and what quietly makes one useless.

Nothing ships here until it has been run against something real. Every project
above was built while using it daily and fixed from its own logs.

---

## 👋 Say hello

**Discord** — `kame055856`

Bug report, a log that looks wrong, or an idea for a port to another host: all
welcome. If a plugin of mine broke your agent, I would genuinely rather hear it
than not.

---

## ❤️ Support the work

Free, MIT, no telemetry, nothing to upsell — written by one person against a real
free tier, on real quotas that really run out.

**Bitcoin** — `36BGYhMEVFgY8PLGMVux93pjGt92KVM6dJ`

*Any amount helps, genuinely.* And a ⭐ costs nothing and helps someone else find
the fix for a problem they are currently blaming on themselves.

---

<div align="center">

*Not affiliated with Agent Zero or Nous Research. Just a person with too many API
keys and not enough quota.*

</div>
