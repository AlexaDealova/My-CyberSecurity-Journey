# Lab 1: Wireshark Basic Packet Capture

## 1. Overview

This lab was my first hands-on exercise with network packet capture. Using Wireshark on a Windows host, I captured live traffic while browsing to two websites — one using HTTPS (Wikipedia) and one intentionally serving plain HTTP (httpforever.com) — in order to observe, in real traffic, the networking concepts I had studied theoretically beforehand.

## 2. Objectives

- Capture live network traffic on a real interface (Ethernet).
- Identify and read a DNS query/response for a real domain.
- Observe a TCP three-way handshake (SYN → SYN-ACK → ACK) in captured packets.
- Compare unencrypted (HTTP) vs encrypted (HTTPS/TLS) traffic at the byte level.
- Practice isolating relevant traffic from background noise using Wireshark display filters.

## 3. Skills Learned

- Applying Wireshark display filters (`ip.addr == x.x.x.x`, `http`) to isolate relevant traffic from a noisy capture.
- Reading a DNS query/response pair to map a domain name to its resolved IP address.
- Recognizing a TCP 3-way handshake in a packet list (`[SYN]`, `[SYN, ACK]`, `[ACK]`).
- Using **Follow → HTTP Stream** and **Follow → TCP Stream** to reconstruct a full conversation between client and server.
- Identifying TLS **SNI (Server Name Indication)** as unencrypted metadata within an otherwise encrypted TLS handshake.
- Basic operational security when handling capture files (recognizing what is safe vs. unsafe to publish — e.g., own public IP, session tokens, credentials).

## 4. Tools Used

- **Wireshark** (previously installed; verified working, no fresh install needed for this session)
- Native Windows network interface (Ethernet)
- Browser (for generating traffic to test targets)

## 5. Lab Environment

- Host OS: Windows
- Capture interface: Ethernet (native, not WSL — avoids the virtual NAT adapter issue with WSL2)
- No virtual machines or lab platform used; this was a real capture on my own personal network with my own devices.

## 6. Prerequisites

- Wireshark installed and able to capture on an active interface (Npcap driver present).
- Administrator access on the host (confirmed before starting).

## 7. Background Knowledge

Before starting hands-on capture, I studied and was quizzed on the following fundamentals:

| Concept | Summary |
|---|---|
| IP Addressing | Public vs. private ranges (`192.168.x.x`, `10.x.x.x`, `172.16–31.x.x`) |
| DNS | Resolves human-readable domain names to IP addresses |
| Protocols & Ports | One IP can host many services, distinguished by port (80, 443, 53, 22, 3389) |
| TCP vs UDP | TCP = reliable, ordered, handshake-based; UDP = fast, no delivery guarantee |
| TCP 3-Way Handshake | SYN → SYN-ACK → ACK before data transfer begins |
| OSI Model | Layer 2 (MAC/Data Link), Layer 3 (IP/Network), Layer 4 (TCP-UDP/Transport), Layer 7 (HTTP-DNS/Application) |
| MAC Address | Physical, vendor-assigned, local-network-only identifier — distinct from the logical, routable IP address |
| ARP / ARP Spoofing | Local-network protocol resolving IP→MAC with no built-in authentication, making it exploitable |

## 8. Lab Scenario

**Part A — HTTPS baseline (Wikipedia):**
Capture traffic while loading `wikipedia.org` in a browser, isolate it from background noise, and identify the DNS resolution, TCP handshake, and TLS session setup.

**Part B — HTTP vs HTTPS comparison (httpforever.com):**
Capture traffic while loading a deliberately HTTP-only test site (`httpforever.com`), then directly compare the readability of HTTP traffic against the HTTPS traffic captured in Part A.

## 9. Methodology

1. Started a live capture on the Ethernet interface in Wireshark.
2. Browsed to `wikipedia.org`; noted that the first capture attempt included significant background noise (Microsoft Edge telemetry, Google services) and restarted the capture after closing unrelated applications.
3. Applied a DNS filter and confirmed the query/response for `wikipedia.org` and `www.wikipedia.org`.
4. Applied `ip.addr == 103.102.166.224` to isolate only Wikipedia-related traffic and observed two parallel TCP handshakes (one per domain variant requested).
5. Browsed to `http://httpforever.com` (a site intentionally served over plain HTTP) to generate comparison traffic.
6. Applied an `http` filter to isolate the plaintext HTTP requests/responses.
7. Used **Follow → HTTP Stream** on the HTTP traffic and **Follow → TCP Stream** on the TLS traffic to directly compare payload readability.

## 10. Step-by-Step Walkthrough & Findings

### 10.1 DNS Resolution

Captured DNS queries confirmed the resolution flow before any connection was made:

- `wikipedia.org` → `A` record → **103.102.166.224**
- `www.wikipedia.org` → `CNAME` → `dyna.wikimedia.org` → same IP (103.102.166.224)

*(See `screenshots/02-dns-query-wikipedia.png`)*

### 10.2 TCP Three-Way Handshake

After filtering to `ip.addr == 103.102.166.224`, two separate TCP handshakes were visible (the browser opened parallel connections for `wikipedia.org` and `www.wikipedia.org`):

| Step | Packet | Flag |
|---|---|---|
| 1 | Client → Server | `[SYN]` |
| 2 | Server → Client | `[SYN, ACK]` |
| 3 | Client → Server | `[ACK]` |
| 4 | Client → Server | `TLS Client Hello (SNI=wikipedia.org)` |

*(See `screenshots/03-tcp-handshake-filtered.png`)*

**Observation:** the TLS Client Hello carries the target domain name in plaintext via the **SNI** field, even before the encrypted session is established — this is necessary because a single server IP can host multiple domains (virtual hosting), so the server needs to know which certificate/site to serve.

### 10.3 HTTP Traffic (httpforever.com) — Plaintext

Filtering to `http` isolated six plaintext HTTP request/response pairs (`GET /css/style.css`, `GET /js/theme.js`, `GET /favicon.svg`, plus `304 Not Modified` responses).

*(See `screenshots/04-http-capture-httpforever.png` and `screenshots/05-http-filter-applied.png`)*

Using **Follow → HTTP Stream** on one of these requests revealed the full HTTP conversation in readable plaintext, including:
- Request method, path, and HTTP version
- `Host`, `User-Agent`, `Referer`, `Accept-Language` headers — fully readable
- Full response headers from the server (`Server: cloudflare`, cache headers, security headers)

*(See `screenshots/06-follow-http-stream-plaintext.png`)*

### 10.4 HTTPS/TLS Traffic (Wikipedia) — Encrypted

Using **Follow → TCP Stream** on the equivalent TLS traffic to Wikipedia showed the payload as unreadable binary/ciphertext — with one notable exception: the plaintext string `www.wikipedia.org` was visible mid-stream, corresponding to the unencrypted SNI field sent during the TLS handshake.

*(See `screenshots/07-follow-tls-stream-encrypted.png`)*

## 11. Analysis

The side-by-side comparison directly demonstrates why HTTP is considered insecure for anything sensitive: request headers (which could include cookies, authentication tokens, or form data in other scenarios) are fully readable to anyone able to observe the traffic. HTTPS protects the same category of data through TLS encryption, but does **not** hide the destination domain, since SNI is transmitted in plaintext by design — a detail with direct relevance to network monitoring, discussed below.

## 12. Security Relevance (SOC Analyst Context)

- **Plaintext credentials over HTTP**: any login form submitted over HTTP would expose the username/password in the same readable format observed for the HTTP headers in this lab. This is an immediate, reportable finding in a real environment.
- **SNI as a monitoring signal**: because SNI is unencrypted even in TLS traffic, a SOC analyst can still identify which domains a host is communicating with — without decrypting the traffic — which is useful for detecting connections to known-malicious or unauthorized domains.
- **TCP handshake anomalies**: understanding what a *normal* 3-way handshake looks like is the baseline for recognizing abnormal patterns, such as repeated unacknowledged SYNs consistent with a SYN flood.
- **Traffic noise triage**: the first (unfiltered) capture attempt is itself a realistic lesson — real-world captures are rarely clean, and filtering out irrelevant background traffic (OS telemetry, unrelated apps) is a necessary first skill before any deeper analysis.

## 13. Challenges and Troubleshooting

- The first capture attempt was too noisy to analyze meaningfully (mixed traffic from background applications). Resolved by closing unrelated applications and restarting the capture — a reminder that a clean baseline matters before analysis.
- `neverssl.com` (originally planned as the HTTP-only comparison target) was unreachable at the time of testing; `httpforever.com` was used instead as a working alternative.

## 14. Lessons Learned

- Real captures require active noise filtering — this isn't optional, it's a core skill.
- Concepts that seemed abstract in theory (3-way handshake, SNI, encryption) became immediately clear once observed directly in captured traffic.
- HTTP vs HTTPS is not just a theoretical security talking point — the difference in what an observer can read is stark and immediately obvious once compared side by side.

## 15. Conclusion

Core objectives for this lab — DNS resolution, TCP handshake observation, and HTTP vs HTTPS comparison — were completed with real, verified captures. Further exploration (Protocol Hierarchy statistics, TCP RST analysis, QUIC traffic, and live ARP requests) was identified as valuable follow-up but is **not yet completed** and will be picked up in a future session or lab extension.

## 16. References

- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [httpforever.com](http://httpforever.com) — intentionally HTTP-only test site
- [MDN: HTTP overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [Cloudflare: What is SNI?](https://www.cloudflare.com/learning/ssl/what-is-sni/)

---

**Status:** Core objectives complete · Extensions (Protocol Hierarchy, RST/QUIC/ARP deep-dive) pending
