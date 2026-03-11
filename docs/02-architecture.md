# 02 — DNS Architecture

## Table of Contents
- [The DNS Namespace](#the-dns-namespace)
- [Domain Name Anatomy](#domain-name-anatomy)
- [Label Rules](#label-rules)
- [The DNS Actors](#the-dns-actors)
- [Delegation](#delegation)
- [Glue Records](#glue-records)
- [Root Servers](#root-servers)

---

## The DNS Namespace

DNS is a **hierarchical, distributed database** shaped like an inverted tree.

```
                          . (Root)
           ┌──────────────┼──────────────────┐
          com            net                 uk
     ┌─────┴─────┐                      ┌────┴────┐
  example     google                   co         me
  ┌────┴───┐                            │
 www      mail                       example
                                        │
                                       www
```

- **Root** (`.`) — the top of the tree; empty label
- **TLDs** — `.com`, `.net`, `.uk`, `.io`, etc.
- **SLDs** — `example.com`, `google.com`
- **Subdomains** — `www.example.com`, `mail.example.com`

---

## Domain Name Anatomy

```
     mail  .  corp  .  example  .  co  .  uk  .
      │        │          │        │      │    │
      │        │          │        │      │    └── Root (empty label)
      │        │          │        │      └─────── ccTLD
      │        │          │        └────────────── ccSLD
      │        │          └─────────────────────── SLD
      │        └────────────────────────────────── Subdomain
      └─────────────────────────────────────────── Host label
```

A **Fully Qualified Domain Name (FQDN)** includes the trailing dot (root label):
```
www.example.com.   ← FQDN (absolute)
www.example.com    ← common usage (relative, trailing dot implied)
```

---

## Label Rules (RFC 1035)

Each component between dots is a **label**:

| Rule | Detail |
|------|--------|
| Length | 1–63 characters per label |
| Characters | Letters (a-z), digits (0-9), hyphens (-) |
| Start/End | Cannot start or end with a hyphen |
| Case | Case-insensitive — `EXAMPLE.COM` = `example.com` |
| Total FQDN | Max 255 bytes (including length octets in wire format) |
| IDN | Internationalized names encoded as Punycode: `xn--` prefix |

```bash
# Punycode examples
münchen.de   → xn--mnchen-3ya.de
中文.com      → xn--fiq228c.com
```

---

## The DNS Actors

### 1. DNS Client / Stub Resolver

The minimal DNS component built into every OS:

```
Application → getaddrinfo() → Stub Resolver → Recursive Resolver
```

- Reads `/etc/resolv.conf` (Linux) or system DNS settings
- Sends queries to configured recursive resolver
- Does **NOT** do iterative resolution itself
- Almost always uses **UDP port 53**
- Only checks local cache and `/etc/hosts` first

```bash
# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search example.com      # appended if name is relative
options ndots:5         # dots before trying as absolute
```

### 2. Recursive Resolver (Full-Service Resolver)

Also called: *caching resolver*, *recursive nameserver*, *full-service resolver*

```
Stub → [Recursive Resolver] → Root → TLD → Authoritative
                    ↑
              Caches results
```

- Does the heavy lifting — follows referrals from root down
- Caches all results per TTL
- Returns `RA=1` (Recursion Available) in responses
- Examples: `8.8.8.8` (Google), `1.1.1.1` (Cloudflare), ISP resolvers

### 3. Authoritative Nameserver

Has definitive, master data for a zone:

| Type | Description |
|------|-------------|
| **Primary** | Has the master zone file; accepts DDNS updates |
| **Secondary** | Receives zone via AXFR/IXFR from primary; read-only |
| **Stealth/Hidden Primary** | Not listed in public NS records; only secondaries are public |

Key characteristics:
- Sets `AA=1` (Authoritative Answer) bit in responses
- Does **NOT** recurse for clients
- Returns `NXDOMAIN` (name doesn't exist) or `NODATA` (no records of that type)
- Returns `REFUSED` if queried for zones it's not authoritative for

### 4. Forwarder

A resolver that forwards queries to another resolver rather than resolving iteratively:

```bash
# BIND forwarder config
options {
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;
};
```

Used in: enterprise networks (forward to internal resolver), split-horizon setups.

---

## Resolution Types

### Recursive Query
Client asks resolver: *"Give me the answer, whatever it takes."*
- `RD=1` in query
- Resolver does all the work, returns final answer
- Most client→resolver queries are recursive

### Iterative Query
Resolver asks server: *"What do you know? I'll follow referrals myself."*
- `RD=0` in query
- Server returns best answer it has (referral or answer)
- Most resolver→nameserver queries are iterative

```
Client ──recursive──► Recursive Resolver
                              │
                    iterative │
                              ▼
                         Root Server  (returns referral to .com TLD)
                              │
                    iterative │
                              ▼
                         TLD Server   (returns referral to example.com NS)
                              │
                    iterative │
                              ▼
                    Authoritative NS  (returns answer)
```

---

## Zones vs Domains

These are often confused:

| Concept | Definition |
|---------|-----------|
| **Domain** | A subtree of the DNS namespace (e.g., `example.com` and everything under it) |
| **Zone** | The portion of that domain managed by one entity — stops at delegations |

```
example.com zone owns:
  example.com.
  www.example.com.
  mail.example.com.
  *.example.com.

But NOT:
  dev.example.com.    ← if delegated to a separate zone
  shop.example.com.   ← if delegated to a separate zone
```

A zone always starts with an **SOA record** and must have **NS records**.

---

## Delegation

When a zone owner hands off authority for a subdomain:

```
example.com zone file:
  dev    IN NS   ns1.dev.example.com.
  dev    IN NS   ns2.dev.example.com.
```

Now `dev.example.com` is a **separate zone** managed by its own nameservers. Queries for anything under `dev.example.com` are **referred** to `ns1.dev.example.com` or `ns2.dev.example.com`.

The delegating (parent) zone keeps **only the NS records** for the child. All other records for the child zone live in the child zone's nameservers.

---

## Glue Records

**Problem:** If `example.com` is served by `ns1.example.com`, how do you resolve `ns1.example.com` to query it? Circular dependency!

**Solution:** Glue records — A/AAAA records for in-zone nameservers placed in the **parent zone**:

```
# In the .com TLD zone:
example.com.        NS    ns1.example.com.
example.com.        NS    ns2.example.com.
ns1.example.com.    A     205.251.196.1     ← GLUE
ns2.example.com.    A     205.251.197.1     ← GLUE
```

Glue records are returned in the **Additional section** of referral responses.

**When are glue records required?**
- Nameserver hostname is **within** the delegated zone → glue required
- Nameserver hostname is **outside** the delegated zone → no glue needed

```
example.com  NS  ns1.example.com.    ← IN-ZONE: glue required
example.com  NS  ns1.otherdns.net.   ← OUT-OF-ZONE: no glue needed
```

---

## Root Servers

There are **13 logical root server addresses** (A through M root), but **not 13 physical machines**:

| Server | Operator | IPv4 | IPv6 |
|--------|----------|------|------|
| a.root-servers.net | Verisign | 198.41.0.4 | 2001:503:ba3e::2:30 |
| b.root-servers.net | USC-ISI | 199.9.14.201 | 2001:500:200::b |
| c.root-servers.net | Cogent | 192.33.4.12 | 2001:500:2::c |
| d.root-servers.net | UMaryland | 199.7.91.13 | 2001:500:2d::d |
| e.root-servers.net | NASA | 192.203.230.10 | 2001:500:a8::e |
| f.root-servers.net | ISC | 192.5.5.241 | 2001:500:2f::f |
| g.root-servers.net | US DoD | 192.112.36.4 | 2001:500:12::d0d |
| h.root-servers.net | ARL | 198.97.190.53 | 2001:500:1::53 |
| i.root-servers.net | Netnod | 192.36.148.17 | 2001:7fe::53 |
| j.root-servers.net | Verisign | 192.58.128.30 | 2001:503:c27::2:30 |
| k.root-servers.net | RIPE NCC | 193.0.14.129 | 2001:7fd::1 |
| l.root-servers.net | ICANN | 199.7.83.42 | 2001:500:9f::42 |
| m.root-servers.net | WIDE | 202.12.27.33 | 2001:dc3::35 |

**Why only 13?** The original DNS protocol had a 512-byte UDP limit. 13 NS records + their IP addresses fit in 512 bytes. With EDNS0 this limit is gone, but the 13 addresses remain for historical compatibility.

**Anycast:** Each of the 13 addresses is actually announced from **hundreds of physical locations worldwide** via BGP anycast. There are 1500+ root server instances. Your query goes to the nearest one.

```bash
# See which root instance you're hitting
dig @a.root-servers.net hostname.bind TXT CH
```

### Root Hints

Resolvers bootstrap using a **root hints file** — a hardcoded list of root server IPs:

```bash
# View root hints in BIND
cat /usr/share/dns/root.hints

# Or fetch the latest
dig @a.root-servers.net . NS > root.hints
```

---

## The Bailiwick Principle

A nameserver's **bailiwick** is the set of names it is authoritative for. Resolvers **ignore** any out-of-bailiwick data in responses — this is a critical security measure.

```
Query: www.example.com A → ns1.example.com

Legitimate response includes:
  www.example.com A 1.2.3.4           ← IN bailiwick ✓

Attacker might try to include:
  google.com A 5.6.7.8                ← OUT of bailiwick ✗ (ignored)
  ns1.google.com A 5.6.7.8           ← OUT of bailiwick ✗ (ignored)
```

---

*Previous: [01 — History](01-history.md) | Next: [03 — Resource Records →](03-resource-records.md)*
