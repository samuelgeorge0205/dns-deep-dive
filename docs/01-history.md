# 01 — History of DNS

## Table of Contents
- [The Pre-DNS Era: HOSTS.TXT](#the-pre-dns-era-hoststxt)
- [Why HOSTS.TXT Failed](#why-hoststxt-failed)
- [Paul Mockapetris and RFC 882/883](#paul-mockapetris-and-rfc-882883)
- [RFC 1034/1035 — The Standard](#rfc-10341035--the-standard)
- [Full Timeline](#full-timeline)

---

## The Pre-DNS Era: HOSTS.TXT

Before DNS existed, every machine on the ARPANET used a single flat text file called **HOSTS.TXT**, maintained by the **Stanford Research Institute Network Information Center (SRI-NIC)**.

```
# Sample HOSTS.TXT entry (simplified)
99.0.0.10    SRI-NIC
10.0.0.51    USC-ISIF
192.5.146.82 MIT-MULTICS
```

Every host would periodically **FTP-download** this file from SRI-NIC. When you wanted to reach a host by name, your system looked up the name in the local copy of HOSTS.TXT.

### Modern Equivalent

The `/etc/hosts` file on Linux/macOS and `C:\Windows\System32\drivers\etc\hosts` on Windows are direct descendants of HOSTS.TXT. They still work the same way and are checked **before** DNS.

---

## Why HOSTS.TXT Failed

By 1982, ARPANET had ~400 hosts. The cracks were already showing:

| Problem | Detail |
|---------|--------|
| **Flat namespace** | No hierarchy — every name globally unique, no delegation possible |
| **Manual updates** | Admins emailed SRI-NIC changes; propagation took days |
| **Consistency** | Different machines had different versions simultaneously |
| **Scalability** | File doubled every ~18 months as ARPANET grew |
| **Bandwidth waste** | Every node repeatedly FTP-polled SRI-NIC |
| **No structure** | No concept of zones, organisations, or authority |

The TCP/IP transition in January 1983 (from NCP) meant the internet was about to explode in size. HOSTS.TXT could never scale.

---

## Paul Mockapetris and RFC 882/883

**Paul Mockapetris** at USC-ISI was tasked with solving the naming problem. He published:

- **RFC 882** — *Domain Names: Concepts and Facilities* (November 1983)
- **RFC 883** — *Domain Names: Implementation and Specification* (November 1983)

His key design decisions:

```
1. Hierarchical distributed namespace
   └── No single central authority; zones delegated to owners

2. Delegation model
   └── .com delegates example.com to its owner

3. Caching
   └── Resolvers cache answers — reduces load dramatically

4. Distributed database
   └── Data spread across thousands of servers worldwide

5. UDP-first transport
   └── Fast, connectionless; TCP fallback for large responses
```

These five ideas are still the foundation of DNS today.

---

## RFC 1034/1035 — The Standard

RFC 882/883 were replaced in **November 1987** by:

- **RFC 1034** — *Domain Names: Concepts and Facilities*
- **RFC 1035** — *Domain Names: Implementation and Specification*

These two RFCs define:
- The DNS message wire format (still in use)
- All original resource record types (A, NS, CNAME, SOA, MX, PTR, TXT)
- The zone file format
- The resolution algorithm (iterative and recursive)
- Name compression in messages

They are still the **authoritative core specification**, amended by hundreds of later RFCs but never replaced.

---

## Full Timeline

| Year | Event | Significance |
|------|-------|--------------|
| 1969 | ARPANET born | 4 nodes; HOSTS.TXT trivially manageable |
| 1971 | HOSTS.TXT introduced | SRI-NIC maintains master file |
| 1982 | HOSTS.TXT crisis | ~400 hosts; file growing unsustainably |
| 1983 Jan | TCP/IP flag day | ARPANET moves from NCP; internet ready to scale |
| 1983 Nov | **RFC 882/883** | Paul Mockapetris designs DNS |
| 1984 | First DNS servers | JEEVES (SRI), ZONESERVER deployed |
| 1984 | BIND 1.0 | Berkeley Internet Name Domain — first major implementation |
| 1987 | **RFC 1034/1035** | Core DNS standard, still canonical today |
| 1990 | BIND 4.8 | Widespread adoption |
| 1993 | InterNIC created | Manages .com, .net, .org registrations |
| 1995 | BIND 8 released | Major rewrite; dominant for a decade |
| 1997 | RFC 2136 | Dynamic DNS updates (DDNS) |
| 1997 | RFC 2181 | DNS clarifications (TTL consistency, CNAME rules) |
| 1998 | ICANN formed | Takes over DNS coordination from US government |
| 1999 | RFC 2535 | DNSSEC (early version) |
| 2000 | BIND 9 released | Current major version |
| 2003 | RFC 3596 | AAAA record for IPv6 |
| 2005 | **RFC 4033–4035** | Modern DNSSEC |
| 2008 | **Kaminsky Attack** | Cache poisoning flaw disclosed; source port randomization adopted |
| 2010 | Root DNSSEC signed | Root zone signed with DNSSEC KSK |
| 2010 | RFC 5966 | DNS over TCP must be supported |
| 2013 | RFC 7043 | EUI-48/EUI-64 records |
| 2015 | RFC 7626 | DNS privacy problem statement |
| 2016 | RFC 7766 | Any DNS response may use TCP |
| 2016 | RFC 7871 | EDNS Client Subnet (ECS) |
| 2016 | Dyn DDoS attack | Mirai botnet takes down major DNS provider |
| 2018 | **RFC 8484** | DNS over HTTPS (DoH) |
| 2019 | **RFC 7858** | DNS over TLS (DoT) widely deployed |
| 2020 | NXNS Attack | Resolver amplification via NS referral flooding |
| 2022 | **RFC 9250** | DNS over QUIC (DoQ) |
| 2023 | KeyTrap vulnerability | DNSSEC resource exhaustion attack (CVE-2023-50387) |

---

## Key People

| Person | Contribution |
|--------|-------------|
| **Paul Mockapetris** | Invented DNS (RFC 882/883, RFC 1034/1035) |
| **Jon Postel** | IANA founder, early internet stewardship |
| **Dan Kaminsky** | Discovered the 2008 cache poisoning vulnerability |
| **Eric Rescorla** | Major contributor to DNS privacy work |
| **Wietse Venema** | BIND security analysis and contributions |

---

*Next: [02 — Architecture →](02-architecture.md)*
