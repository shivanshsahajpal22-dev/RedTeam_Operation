# 🎯 Red Team Operations

Notes, methodology, and write-ups on **red team engagements** — adversary emulation, attack path planning, Active Directory tradecraft, and lateral movement. This repo documents *process and reasoning*, not plug-and-play exploits.

> ⚠️ **For educational and authorized-engagement use only.** Everything here assumes a signed rules-of-engagement (RoE) and explicit client scope. Unauthorized access to systems you don't own or have permission to test is illegal.

---

## 📖 What's in here

| Area | Description |
|---|---|
| **Engagement Planning** | Scoping, RoE structure, objective-based operation design |
| **Reconnaissance** | OSINT methodology, external/internal recon workflows |
| **Initial Access** | Notes on common entry vectors and how they're typically evaluated |
| **Active Directory** | AD enumeration methodology, trust/privilege mapping concepts, common misconfig patterns |
| **Lateral Movement** | How operators reason about pivoting between hosts and escalating access |
| **Persistence & C2** | High-level concepts around command-and-control structure and operational security |
| **Reporting** | How findings are documented and communicated to clients/blue teams |

---

## 🧠 Philosophy

Red teaming isn't about running tools — it's about **thinking like an adversary within defined boundaries**. This repo leans toward:

- Documenting *why* an attack path works, not just *that* it works
- Mapping techniques back to frameworks like **MITRE ATT&CK**
- Writing with the blue team's perspective in mind — every offensive note here is meant to make detection easier too

---

## 🗂️ Structure

```
red-team-ops/
├── recon/           # OSINT & reconnaissance methodology
├── active-directory/ # AD enumeration & attack path notes
├── lateral-movement/ # Pivoting & privilege escalation concepts
├── c2-notes/          # C2 architecture & opsec concepts
└── reporting/           # Report templates & documentation style
```

---

## 🔗 More of my work

Part of a broader collection — see the full hub: **[shivanshsahajpal22-dev.github.io](https://shivanshsahajpal22-dev.github.io/shivanshsahajpal22-dev/)**

- CTF Write-ups
- Web Exploitation
- Exploit Dev
- Malware Analysis
- Social Engineering

---

> "Discipline is the only path to mastery." — *Inspired by Miyamoto Musashi*
