# 🌐 dns-deep-dive

> A comprehensive, low-level reference repository for understanding the Domain Name System — from ARPANET history to modern DNS-over-QUIC, packet-by-packet.

---

## 📚 What This Repo Covers

| Area | Topics |
|------|--------|
| **Foundations** | History, architecture, namespace hierarchy |
| **Protocol** | Wire format, message headers, EDNS0, transport |
| **Records** | All RR types, zone files, SOA, MX, SRV, DNSSEC records |
| **Resolution** | Full iterative flow, packet traces, CNAME chains, reverse DNS |
| **Security** | Cache poisoning, DNSSEC, amplification attacks, DoT/DoH |
| **Hands-on** | Labs with dig, zone setup, DNSSEC signing, packet captures |

---

## 📂 Repository Structure

```
dns-deep-dive/
├── docs/               # Deep-dive notes (11 topics)
├── reference/          # Quick-reference tables and cheat sheets
├── labs/               # Hands-on exercises
├── packet-captures/    # Annotated .pcap breakdowns
├── zone-examples/      # Real zone file examples
└── diagrams/           # Architecture and flow diagrams (ASCII)
```

---

## 🗂️ Documentation Index

### Core Notes (`docs/`)

| # | File | What You'll Learn |
|---|------|-------------------|
| 01 | [History](docs/01-history.md) | HOSTS.TXT, RFC 882/1034, full timeline |
| 02 | [Architecture](docs/02-architecture.md) | Namespace tree, resolvers, delegation, glue |
| 03 | [Resource Records](docs/03-resource-records.md) | All RR types, wire format, SOA/MX/SRV deep dives |
| 04 | [Message Format](docs/04-message-format.md) | Header bits, question section, name compression, EDNS0 |
| 05 | [Resolution Flow](docs/05-resolution-flow.md) | Iterative resolution, full 8-packet trace, CNAME, reverse DNS |
| 06 | [Zone Management](docs/06-zone-management.md) | Zone files, AXFR/IXFR, NOTIFY, TSIG, DDNS |
| 07 | [Caching](docs/07-caching.md) | TTL mechanics, negative caching, Kaminsky attack |
| 08 | [DNSSEC](docs/08-dnssec.md) | Chain of trust, RRSIG/DS/NSEC, validation, algorithms |
| 09 | [Security & Attacks](docs/09-security-attacks.md) | All attack vectors + mitigations |
| 10 | [Privacy Protocols](docs/10-privacy-protocols.md) | DoT, DoH, DoQ, ODoH — comparison and setup |
| 11 | [Advanced Topics](docs/11-advanced-topics.md) | Split-horizon, anycast, GeoDNS, mDNS, RPZ |

### Reference (`reference/`)

- [RFC Index](reference/rfc-index.md) — Every DNS RFC you need, organized by topic
- [RCODE Table](reference/rcode-table.md) — All response codes explained
- [Record Types Table](reference/record-types-table.md) — All RR types cheat sheet
- [Tools Cheat Sheet](reference/tools-cheatsheet.md) — `dig`, `drill`, `delv`, `nsupdate`

### Labs (`labs/`)

| Lab | Topic |
|-----|-------|
| [Lab 01](labs/01-basic-dig-queries/) | Basic dig queries and reading output |
| [Lab 02](labs/02-trace-resolution/) | Tracing full resolution with `+trace` |
| [Lab 03](labs/03-zone-file-setup/) | Writing and serving a zone file |
| [Lab 04](labs/04-dnssec-signing/) | Signing a zone and validating with `delv` |
| [Lab 05](labs/05-cache-poisoning-demo/) | Cache poisoning simulation |
| [Lab 06](labs/06-doh-dot-setup/) | Setting up DNS-over-TLS and DNS-over-HTTPS |

---

## 🚀 Quick Start

```bash
# Clone the repo
git clone https://github.com/yourusername/dns-deep-dive.git
cd dns-deep-dive

# Start with the basics
cat docs/01-history.md

# Or jump straight to the resolution flow
cat docs/05-resolution-flow.md

# Run your first lab
cd labs/01-basic-dig-queries
cat README.md
```

**Prerequisites for labs:** `dig`, `bind9` (for zone labs), `wireshark`/`tcpdump` (for packet labs)

---

## 🧭 Learning Paths

### "I'm new to DNS"
→ `01-history` → `02-architecture` → `03-resource-records` → `05-resolution-flow` → Lab 01 → Lab 02

### "I want to understand the wire protocol"
→ `04-message-format` → `05-resolution-flow` → `packet-captures/` → Lab 02

### "I want to secure DNS"
→ `07-caching` → `08-dnssec` → `09-security-attacks` → Lab 04 → Lab 05

### "I'm preparing for an interview / exam"
→ All `docs/` → `reference/` → All labs

---

## 📖 Key RFCs at a Glance

| RFC | Title |
|-----|-------|
| RFC 1034 | DNS Concepts and Facilities |
| RFC 1035 | DNS Implementation and Specification |
| RFC 2308 | Negative Caching of DNS Queries |
| RFC 4033–4035 | DNSSEC |
| RFC 7858 | DNS over TLS |
| RFC 8484 | DNS over HTTPS |
| RFC 9250 | DNS over QUIC |

Full list → [reference/rfc-index.md](reference/rfc-index.md)

---

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). All contributions welcome — corrections, new labs, packet captures, diagrams.

---

## 📄 License

MIT — free to use, share, and build on.
