# SOC Analyst Learning Journey

Documenting my journey from zero prior knowledge to (eventually) a Junior SOC Analyst role. Everything in this repository reflects work I have actually completed — no fabricated results, no skills claimed before they're demonstrated.

## About

I'm learning Cyber Security from scratch with the goal of becoming a **SOC (Security Operations Center) Analyst**. This repo is my public log: daily notes, hands-on labs, and the reasoning behind what I learned — not just polished final answers.

## Structure

```
soc-analyst-journey/
├── daily-logs/          # Short logs for each study session (even without a full lab)
├── labs/                 # One folder per hands-on lab, each with its own README + screenshots
│   └── lab01-wireshark-basic-capture/
└── notes/                # Standalone concept notes / cheat sheets
```

## Labs Completed

| # | Lab | Status | Key Skills |
|---|-----|--------|------------|
| 1 | [Wireshark Basic Packet Capture](labs/lab01-wireshark-basic-capture/README.md) | Core objectives complete | DNS resolution, TCP 3-way handshake, HTTP vs HTTPS traffic analysis |
| 2 | [Linux Log Investigation](labs/lab02-linux-log-investigation/README.md) | Core objectives complete | Log fundamentals, `grep`/`tail`/`less` filtering, real + synthetic SSH brute-force investigation, IR lifecycle |

## Extra Practice

- [OverTheWire: Bandit](notes/overthewire-bandit.md) — side practice on Linux command-line fundamentals (levels 0–4 so far), outside the main lab roadmap.

## Fundamentals Covered So Far

- IP addressing (private vs public)
- DNS resolution
- Protocols & ports (HTTP/80, HTTPS/443, DNS/53, SSH/22, RDP/3389)
- TCP vs UDP, including the 3-way handshake (SYN / SYN-ACK / ACK)
- OSI model (Layers 2, 3, 4, 7 in particular)
- MAC addresses vs IP addresses
- ARP and ARP spoofing (conceptual)
- Linux log fundamentals (log rotation, centralized/SIEM logging rationale, log line anatomy)
- Log investigation with `grep`, `tail`, `less`, and `wc -l` (including rate vs. total-count reasoning)
- Incident Response lifecycle and defense in depth
- Security+ Domain 1 (General Security Concepts): CIA Triad, AAA, non-repudiation, Zero Trust, honeypots/deception technology, physical security controls, gap analysis, change management

## A Note on Honesty

This repo will move slowly at times, and that's intentional. I'd rather document real, verified progress than inflate it. If a lab folder says "in progress," it means exactly that.

## Contact

Feel free to connect on [LinkedIn](https://www.linkedin.com/in/alexa-alfito-dealova).
