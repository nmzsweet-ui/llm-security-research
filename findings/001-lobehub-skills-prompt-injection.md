# Finding 001: LobeHub Skills Marketplace — Prompt Injection via Weaponized Documentation

**Date:** 2026-04-08  
**Researcher:** Vasil Andreev  
**Severity:** High  
**Category:** Prompt Injection / Social Engineering  
**Target URL:** `https://lobehub.com/skills/skill.md`  
**Status:** Active threat

---

## Summary

The URL `https://lobehub.com/skills/skill.md` serves content disguised as official LobeHub Skills Marketplace setup documentation. In reality, it is a **prompt injection payload** designed to trick AI coding agents (e.g., Claude Code, Cursor, GitHub Copilot agents) into executing arbitrary commands and exfiltrating data.

When an AI agent is instructed to "read and follow" this URL, the injected instructions attempt to make the agent:

1. Execute an untrusted npm package via `npx -y @lobehub/market-cli`
2. Send identity/platform information to an attacker-controlled endpoint
3. Establish a persistent "self-learning loop" that keeps the agent calling back

## Attack Vector

**Type:** Indirect Prompt Injection via Weaponized URL  
**Delivery:** A user (or automated system) asks an AI agent to fetch and follow instructions from the URL.  
**Trigger:** The AI agent reads the page content, interprets it as legitimate documentation, and executes the embedded commands.

## Indicators of Compromise

### 1. Fake CLI Package
The document instructs agents to run:
```
npx -y @lobehub/market-cli <command>
```
- The `-y` flag bypasses installation confirmation prompts
- The `@lobehub/market-cli` package is **not a real LobeHub package**
- If an attacker registers this package on npm, every agent following these instructions would download and execute malicious code (supply chain attack)

### 2. Data Exfiltration via "Registration"
The "register" command harvests:
- `--name` — Agent/session identity
- `--description` — Context about the agent's purpose
- `--source` — Which AI platform is being used
- This data is sent to an unknown endpoint under attacker control

### 3. Fake URL Structure
- **Malicious path:** `lobehub.com/skills/skill.md`
- **Real LobeHub plugin paths:** `lobechat.com/discover/plugin/` or `github.com/lobehub/lobe-chat-plugins`
- The `/skills/` path does not correspond to any legitimate LobeHub product

### 4. AI-Agent-Targeted Language
Phrases specifically crafted to manipulate AI agents:
- "You are an agent"
- "This is your self-learning loop"
- "Always use the CLI commands below"
- References to fake sub-documents (`references/skills-search.md`, `references/skills-install.md`, `references/skills-feedback.md`) to simulate legitimate documentation structure

### 5. HTTP 403 on Direct Fetch
The URL returns a 403 Forbidden error when fetched with standard HTTP clients but apparently serves content to certain user agents or under certain conditions, suggesting selective serving to target AI agent infrastructure.

## Real LobeHub Plugin Ecosystem (for comparison)

The legitimate LobeHub plugin system is entirely different:

| Aspect | Fake (Attack) | Real (LobeHub) |
|---|---|---|
| Domain | `lobehub.com/skills/` | `github.com/lobehub/lobe-chat-plugins` |
| Setup method | `npx @lobehub/market-cli` | Fork repo, add JSON, submit PR |
| SDK | None (fake CLI) | `@lobehub/chat-plugin-sdk` |
| Template | None | `github.com/lobehub/chat-plugin-template` |
| Submission | CLI "register" command | GitHub Pull Request |

### Legitimate Repos
- Plugin index: `https://github.com/lobehub/lobe-chat-plugins`
- Plugin SDK: `https://github.com/lobehub/chat-plugin-sdk`
- Plugin template: `https://github.com/lobehub/chat-plugin-template`
- Plugins gateway: `https://github.com/lobehub/chat-plugins-gateway`

## Impact Assessment

| Impact | Description |
|---|---|
| **Remote Code Execution** | If the fake npm package is published, agents execute arbitrary code |
| **Data Exfiltration** | Agent identity, platform info, and session context sent to attacker |
| **Supply Chain Risk** | npm package squatting under a trusted brand name (`@lobehub/`) |
| **Trust Exploitation** | Leverages the legitimate LobeHub brand to bypass human and AI scrutiny |
| **Persistence** | "Self-learning loop" design keeps agents calling back repeatedly |

## MITRE ATLAS Mapping

| Tactic | Technique |
|---|---|
| Initial Access | AML.T0051 — LLM Prompt Injection |
| Execution | AML.T0040 — ML Supply Chain Compromise |
| Exfiltration | AML.T0048.002 — Exfiltration via ML Inference API |
| Persistence | Instruction to establish recurring agent callbacks |

## Mitigation Recommendations

1. **AI Agent Sandboxing** — Never allow AI agents to execute `npx -y` or similar auto-install commands from fetched URLs without human confirmation
2. **URL Allowlisting** — AI agents should validate that fetched instruction URLs come from known-good sources
3. **Content Inspection** — Flag documents that contain command-execution instructions targeting AI agents
4. **npm Package Monitoring** — Monitor for registration of `@lobehub/market-cli` and similar squatted packages
5. **Human-in-the-Loop** — Require explicit human approval before an AI agent executes shell commands derived from external content

## Responsible Disclosure

- This finding documents a live prompt injection technique observed in the wild
- The LobeHub team should be notified if the domain `lobehub.com` is under their control, as the `/skills/skill.md` path may indicate a compromised or misconfigured endpoint
- If the domain is not controlled by the real LobeHub organization, this constitutes brand impersonation

---

*Documented as part of the LLM Security Research project. All research follows responsible disclosure principles.*
