# OpenClaw Hardening

**Part of the [OpenClaw Security Suite](#the-suite) — Tool 4 of 4**

Step-by-step guided security hardening. The other tools tell you what's broken. This one walks you through fixing it — one command at a time, in plain language, with verification after each step.

Based on ["Don't Let the Claw Grip Your Hand"](https://arxiv.org/abs/2603.10387) (Shan et al., 2026).

For your convenience: 

Tool 1: https://github.com/SaifAlYounan/OpenClawSecurityPostureAssessment

Tool 2: https://github.com/SaifAlYounan/OpenClawSecurityPostureAssessment---Part-2

Tool 3: https://github.com/SaifAlYounan/OpenClawSecurityPostureAssessment---Part-3

Tool 4: https://github.com/SaifAlYounan/OpenClawSecurityPostureAssessment---Part-4

---

## Getting Started

**Install:** Send the link to this repo to your agent and ask it to install it as a skill.

**Run (best with reports from the other tools):**
```
Harden my agent
```
```
Apply the fixes from my security audit
```

The skill asks for your HTML reports from Tools 1-3. Send them in chat. If you don't have reports, it runs its own diagnostic.

**What happens:** The agent walks you through 10 fixes, one at a time. It explains each command, tells you whether you paste it (root) or the agent runs it, waits for your go-ahead, and verifies it worked. At the end, it re-checks your posture and shows before/after.

---

## The Suite

```
  1. openclaw-security-audit    What does your posture look like?
  2. openclaw-red-team          How hard do single attacks have to try?
  3. openclaw-attack-chains     Can innocent requests compose into a breach?
→ 4. openclaw-hardening        Fix everything, step by step
```

**Best workflow:**
1. Run Tool 1 (audit) → get your posture report
2. Run Tool 2 (red team) → get your resilience score
3. Run Tool 3 (chains) → test multi-step attack paths
4. Send all reports to this tool → it prioritizes fixes based on your specific findings

**Shortcut:** Skip tools 2-3 and jump straight here with just the audit report. Or skip everything and just say "harden my agent" — it runs its own diagnostic.

---

## The 10 Fixes

| # | Fix | Who runs it | Impact |
|---|-----|-------------|--------|
| 1 | Non-root user | You (root) | Eliminates entire attack categories |
| 2 | Remove sudo | You (root) | Blocks privilege escalation |
| 3 | Outbound network lock | You (root) | Kills exfiltration — biggest single fix |
| 4 | HITL defense layer | You (root) | Blocks pattern-matched attacks |
| 5 | File permissions | You (root) | Restricts what agent can read |
| 6 | Container isolation | You | Enforces sandbox at OS level |
| 7 | API key protection | You | Prevents key theft |
| 8 | SSH key hygiene | You | Removes unnecessary secrets |
| 9 | Audit logging | Agent or you | Enables post-incident forensics |
| 10 | Startup script protection | You (root) | Prevents persistence attacks |

The skill checks what's already applied and skips it. If you're already non-root with no sudo, it starts at fix #3.

---

## What a Fix Looks Like

```
Fix #3: Outbound Network Restriction
=====================================

What this does: Blocks your agent from sending data to any server
except the ones it actually needs (Anthropic for Claude, Telegram
for your messages).

Why it matters: Right now your agent can reach any website. If a
poisoned document tricks it into uploading your data, nothing stops it.

What could break: If your agent uses other services (email, web
browsing), those stop until you add them to the allowlist.

You need to run this — I can't do it myself because it requires
root privileges, and that's the point.

📋 Paste this in your SSH terminal:

    dig +short api.anthropic.com
    dig +short api.telegram.org

Tell me what IPs come back and I'll generate the rules.
```

One fix. One explanation. One confirmation. Then the next.

---

## Works With or Without Reports

**With reports (recommended):** Send HTML reports from Tools 1-3. The skill reads them, skips fixes already applied, and prioritizes fixes that address the most failures across all reports.

**Without reports:** Runs a quick diagnostic, then uses general best-practice priority. Works fine — just less tailored.

---

## Every Fix Is Reversible

The skill tells you how to undo each fix before you apply it.

---

## Safety

The skill never runs a command without your explicit approval. For root-level fixes, it shows you the command and you paste it yourself — the agent never has root access.

---

## Requirements

`sessions_spawn` is **NOT** required. This skill runs diagnostics directly and guides you through manual commands. Works on any OpenClaw installation.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Guided hardening workflow |
| `references/fixes.md` | 10 fixes with commands, verification, and reversal |

## Citation

Shan, Z., Xin, J., Zhang, Y., & Xu, M. (2026). "Don't Let the Claw Grip Your Hand." arXiv:2603.10387.

## License

MIT

## Author

Alexios van der Slikke-Kirillov
