# Lab 2: Linux Log Investigation

## 1. Overview

This lab covered reading, filtering, and investigating Linux authentication logs — starting from zero prior knowledge of what a log file even is. It combined conceptual fundamentals with two rounds of genuinely hands-on practice: first generating and investigating **real** SSH login-failure log entries on my own WSL (Ubuntu 26.04) environment, then applying the same techniques to a larger, clearly-labeled **synthetic** multi-attacker scenario to practice pattern recognition at a more realistic scale.

## 2. Objectives

- Understand what a log file is, where Linux stores it, and why it doesn't persist forever (log rotation).
- Understand why serious organizations forward logs to a separate system (SIEM) rather than relying only on local retention.
- Read and interpret the structure of a real log line (timestamp, hostname, process/PID, message).
- Use `tail`, `less`, and `cat` appropriately depending on file size and investigative need.
- Filter logs with `grep`, including chaining filters with pipes (`|`).
- Quantify suspicious activity with `wc -l` and reason about **rate/velocity**, not just totals.
- Set up an SSH server from scratch on a fresh WSL install and generate real authentication log data.
- Investigate a multi-actor scenario to distinguish a genuine compromise from background noise and a failed attack attempt.
- Understand the Incident Response lifecycle (Identification → Containment → Eradication → Recovery → Lessons Learned) and the defense-in-depth principle.

## 3. Skills Learned

- Reading the anatomy of a Linux log line and recognizing that exact format (BSD-style vs. `systemd`/ISO 8601) can vary between systems while the underlying fields stay conceptually the same.
- Choosing the right command for the job: `tail`/`tail -f` for recent/live activity, `less` for searching within a large file, `cat` only for genuinely small files.
- Filtering with `grep "pattern" file`, and chaining two filters with `grep "A" file | grep "B"` to narrow results by more than one condition at once.
- Counting matched lines with `grep ... | wc -l` to turn a qualitative suspicion into a quantitative check.
- Reasoning about **rate** (events per unit time) as a stronger brute-force indicator than a raw total count, and that a low, spread-out rate isn't automatically safe ("low and slow" evasion).
- Installing and starting a service from scratch on Ubuntu/WSL (`apt install`, `systemctl start`, verifying with `service ... status`).
- Generating genuine log evidence by deliberately triggering real failed SSH logins against my own machine, rather than fabricating log content.
- Distinguishing a successful compromise from normal human error and from a failed attack attempt, using the sequence of `Failed password` / `Accepted password` lines around a given timestamp.
- Naming and reasoning through the IR lifecycle and the defense-in-depth principle (e.g., disabling direct root SSH login as a second barrier).

## 4. Tools Used

- WSL2 — Ubuntu 26.04.1 LTS ("Resolute Raccoon")
- `openssh-server` (installed fresh during this lab)
- Core CLI tools: `cat`, `tail`, `less`, `grep`, `wc`, `sudo`, `systemctl`, `service`, `dpkg`

## 5. Lab Environment

- Host OS: Windows, with WSL2 running Ubuntu 26.04.1 LTS (hostname `Alexa`, user `alek`)
- `openssh-server` was **not** installed by default — installed and started as part of this lab (see Section 9).
- Real investigation: my own WSL instance's `/var/log/auth.log`.
- Synthetic investigation: a hand-built sample log file (`~/lab2-sample-auth.log`), explicitly created as training data, not a capture from any real system.

## 6. Prerequisites

- Comfort with basic Linux navigation and `sudo` from prior OverTheWire Bandit practice.
- WSL with a working terminal (confirmed before starting).

## 7. Background Knowledge

Before hands-on work, I studied and was quizzed on the following fundamentals:

| Concept | Summary |
|---|---|
| What a log file is | An automatically generated, plain-text record of system/application activity, stored on regular disk under `/var/log/` — not ROM (corrected a real misconception I had going in) |
| Log rotation | Old logs are archived/compressed/deleted on a policy (`logrotate`), commonly a few weeks' retention by default |
| Why centralize logs (SIEM) | Two combined reasons: (1) local retention/capacity limits, and (2) tamper-resistance — an attacker who compromises a host can delete/alter local logs, but not a copy already forwarded elsewhere |
| Log line structure | `<timestamp> <hostname> <process>[<pid>]: <message>` — verified against a real captured line |
| Reading tools | `tail` (last N lines / live with `-f`), `less` (paged, searchable with `/`), `cat` (whole file — only sensible for small files) |
| Filtering | `grep "pattern" file`; chaining multiple conditions with `grep "A" file \| grep "B"` |
| Rate vs. total count | A high count in a short window is a strong brute-force signal; the same count spread over weeks is not automatically safe (possible slow/evasive attack) |

## 8. Lab Scenario

**Part A — Real, self-generated evidence:**
Install and start an SSH server on my own WSL instance, then deliberately trigger failed logins against my own account to produce genuine `auth.log` entries, and verify them using `grep`/`wc -l`.

**Part B — Synthetic multi-actor investigation:**
Given a fictional prompt ("investigate last night's SSH activity on `webserver01` and determine whether an attacker got in"), investigate a hand-built sample log (`~/lab2-sample-auth.log`) containing a mix of normal logins, a failed attack attempt, and one successful brute-force compromise, to identify the real incident and distinguish it from noise.

## 9. Methodology

### Part A — Real environment
1. Confirmed the WSL distro and version (`cat /etc/os-release` → Ubuntu 26.04.1 LTS).
2. Checked whether `openssh-server` was installed (`dpkg -l | grep openssh-server`) — it was not.
3. Installed it (`sudo apt update && sudo apt install openssh-server -y`).
4. Checked service status (`sudo service ssh status`) — found `inactive (dead)`.
5. Started it (`sudo systemctl start ssh`), then re-confirmed it was `running`.
6. Ran `ssh alek@localhost` and deliberately entered the wrong password 3 times until the connection was closed.
7. Read the real log with `sudo tail -20 /var/log/auth.log` and located the corresponding `Failed password` and PAM authentication-failure lines.
8. Verified the count matched exactly using `sudo grep "Failed password" /var/log/auth.log | wc -l` → **3**, matching the 3 deliberate failed attempts.

### Part B — Synthetic scenario
1. Created `~/lab2-sample-auth.log` from a provided, explicitly-labeled synthetic dataset (not a real capture) containing 26 lines: normal logins, one failed attack attempt (invalid user `admin`), and one successful attack (11 failed attempts against `root` followed by a success), plus one normal human typo.
2. Ran `tail ~/lab2-sample-auth.log` first, and correctly identified that it missed the earliest events (only shows the last 10 lines by default) — self-corrected to `cat`, reasoning that `cat` is appropriate here specifically because the file is small.
3. Read the full file with `cat ~/lab2-sample-auth.log`.
4. Rather than reading every `Failed password` line manually, filtered directly for the outcome that answers the actual question ("did anyone get in?") using `grep "Accepted password" ~/lab2-sample-auth.log`.
5. Compared the four successful logins against what preceded each one in the full log, to separate normal activity from the real compromise.

## 10. Step-by-Step Walkthrough & Findings

### 10.1 Real Log Investigation (Part A)

After 3 deliberate wrong-password attempts, `sudo tail -20 /var/log/auth.log` showed the real entries:

```
sshd-session[1210]: Failed password for alek from 127.0.0.1 port 41040 ssh2
sshd-session[1210]: Failed password for alek from 127.0.0.1 port 41040 ssh2
sshd-session[1210]: Failed password for alek from 127.0.0.1 port 41040 ssh2
sshd-session[1210]: PAM 2 more authentication failures
```

*(See `screenshots/01-wsl-ubuntu-version-check.png` through `screenshots/05-grep-wc-count-verification.png`)*

Note: the timestamp format on this system was `systemd`/ISO 8601 style (`2026-09-22T23:03:45.427382+07:00`) with process name `sshd-session`, rather than the classic BSD-style (`Sep 22 10:14:03`) with plain `sshd` used in the generic teaching example — a useful real-world lesson that the exact format varies by system/OpenSSH version, even though the underlying fields (time, hostname, process, message) stay conceptually the same.

`sudo grep "Failed password" /var/log/auth.log | wc -l` returned **3**, confirming the count matched exactly.

### 10.2 Synthetic Scenario Investigation (Part B)

Full sample log content (`~/lab2-sample-auth.log`, synthetic training data):

```
Sep 21 22:58:02 webserver01 sshd[4102]: Accepted password for budi from 10.0.2.15 port 52011 ssh2
Sep 21 23:14:02–23:14:17 webserver01 sshd[...]: Failed password for invalid user admin from 198.51.100.23 (5 attempts, then connection closed — no success)
Sep 21 23:20:33 webserver01 sshd[4301]: Accepted password for sari from 10.0.2.20 port 60122 ssh2
Sep 21 23:31:09–23:31:59 webserver01 sshd[...]: Failed password for root from 203.0.113.77 (11 attempts, ~1/5 seconds)
Sep 21 23:32:04 webserver01 sshd[4466]: Accepted password for root from 203.0.113.77 port 55221 ssh2
Sep 21 23:45:11 webserver01 sshd[4501]: Failed password for sari from 10.0.2.20 port 39901 ssh2
Sep 21 23:45:20 webserver01 sshd[4502]: Accepted password for sari from 10.0.2.20 port 39905 ssh2
```

*(See `screenshots/06-tail-limitation-demonstrated.png`, `screenshots/07-sample-log-full-cat-output.png`, `screenshots/08-grep-accepted-password-investigation.png`)*

`grep "Accepted password" ~/lab2-sample-auth.log` isolated all 4 successful logins. Comparing each against what preceded it in the full log:

| Successful login | Preceded by | Verdict |
|---|---|---|
| `budi` from `10.0.2.15` | No failures | Normal |
| `sari` from `10.0.2.20` (23:20) | No failures | Normal |
| **`root` from `203.0.113.77`** | **11 failed attempts in ~50 seconds** | **Compromise** |
| `sari` from `10.0.2.20` (23:45) | 1 failure | Normal human typo |

Separately, `198.51.100.23` made 5 failed attempts against a non-existent user (`admin`) and never succeeded — a failed attack attempt, distinct in severity from the successful one.

## 11. Analysis

**Incident summary:** A brute-force SSH attack against the `root` account on `webserver01`, originating from `203.0.113.77`, succeeded on the 12th attempt within a roughly one-minute window (23:31:09–23:32:04, 21 September). A separate, unrelated attack attempt from `198.51.100.23` targeting a non-existent `admin` account failed and did not result in access.

The distinguishing factor between the real compromise and normal activity was not the fact that a login succeeded, but the **pattern immediately preceding it** — a high number of failures in a short window, immediately followed by success. A single prior failure (as with `sari`'s second login) is ordinary human error and not, on its own, a red flag.

## 12. Security Relevance (SOC Analyst Context)

- **Rate over raw count**: 11 failures in under a minute is a much stronger signal than the same 11 failures spread across a month — velocity of attempts matters as much as the total.
- **"Low and slow" evasion**: a spread-out, low-rate pattern is not automatically safe; sophisticated attackers deliberately slow down to stay under rate-based detection thresholds. This means a full assessment needs more than one signal (e.g., unfamiliar IP reputation, targeting a nonexistent or highly privileged account) to be conclusive either way.
- **Direct root SSH login as a risk factor**: this scenario was made worse by root being directly reachable over SSH. Disabling `PermitRootLogin` (a standard hardening setting) would not make such an attack impossible, but would force an attacker to first compromise an ordinary user account and then escalate via `sudo` — an example of **defense in depth** (multiple independent barriers, so breaking one is not enough).
- **`sudo` actions are themselves logged**: verified directly in my own real `auth.log`, e.g. `sudo: alek : TTY=/dev/pts/0 ; ... ; COMMAND=/usr/bin/systemctl start ssh` — meaning privilege escalation via `sudo` leaves its own separate audit trail, giving a SOC analyst a second chance to detect an intrusion even after an attacker gets past the first barrier.
- **Log format is not universal**: real-world log format varies by distro/OpenSSH version (confirmed firsthand — my own system used `systemd`-style timestamps and `sshd-session`, not the generic textbook example). Investigation skills need to generalize to the underlying *fields*, not a memorized exact format.
- **Centralized logging (SIEM) rationale, revisited with evidence in hand**: having now generated and read a real local log, it's concrete why relying solely on it is risky — the same access that let me (the legitimate owner) read `/var/log/auth.log` with `sudo` is exactly the level of access a successful attacker (like the `root` compromise in Part B) would also have, meaning they could edit or delete these entries unless a copy already exists elsewhere.

## 13. Incident Response Lifecycle Applied

| Stage | What it means | Applied to this scenario |
|---|---|---|
| Identification | Confirm the incident is real | Done — verified via `grep`/pattern analysis above |
| Containment | Stop further damage | Block `203.0.113.77` (e.g., firewall rule) |
| Eradication | Remove the root cause | Reset the `root` password; check for anything the attacker left behind |
| Recovery | Return to normal, with closer monitoring | Restore normal operation; watch for repeat attempts |
| Lessons Learned | Prevent recurrence | Recommend disabling direct root SSH login (`PermitRootLogin no`) going forward |

## 14. Challenges and Troubleshooting

- Initially misunderstood log storage as being on ROM rather than regular disk with a retention policy — corrected during the fundamentals discussion.
- First instinct for getting an overview of the synthetic log file was `tail`, which silently missed the earliest events (only shows the last 10 lines by default); self-corrected to `cat` after recognizing the file was small enough for that to make sense.
- Initial guess for isolating "did anyone succeed" was to filter by IP address — recognized on reflection that this doesn't work if the suspicious IP isn't known yet, and the correct filter was the status keyword (`Accepted password`) shared across all outcomes.

## 15. Lessons Learned

- A log line's *fields* (time, host, process, message) are a more durable mental model than any single exact format, since format varies by system.
- Counting matches (`wc -l`) turns "this looks like a lot" into a verifiable number — the same evidence-first habit practiced during OverTheWire Bandit.
- Finding a compromise is only "Identification" — a real SOC response continues through Containment, Eradication, Recovery, and Lessons Learned.
- Security controls work in layers (defense in depth); a single misconfiguration (root SSH login allowed) can turn a survivable attack attempt into a full compromise.

## 16. Conclusion

Core objectives for this lab were completed: log fundamentals, real self-generated SSH failure evidence, and a full multi-actor investigation on a clearly-labeled synthetic dataset, ending with a correct identification of the compromised account, attacker IP, and time window, and a decision-relevant understanding of what a SOC analyst does next.

## 17. References

- [Ubuntu Server Documentation — OpenSSH](https://ubuntu.com/server/docs/openssh-server)
- [NIST SP 800-61: Computer Security Incident Handling Guide](https://csrc.nist.gov/pubs/sp/800/61/r2/final)
- `man sshd_config` (for `PermitRootLogin` and related hardening settings)

---

**Status:** Core objectives complete · Part B used a clearly-labeled synthetic dataset, not a real capture · No fabricated results — every log line shown was either generated live on my own system or explicitly created as labeled training data before analysis.
