# 07 — DNS Caching & TTL

## Table of Contents
- [TTL Mechanics](#ttl-mechanics)
- [Cache Behavior](#cache-behavior)
- [Negative Caching (RFC 2308)](#negative-caching-rfc-2308)
- [Cache Poisoning](#cache-poisoning)
- [Kaminsky Attack (2008)](#kaminsky-attack-2008)
- [Mitigations](#mitigations)

---

## TTL Mechanics

TTL (Time to Live) is a **32-bit unsigned integer** in seconds on the wire. It controls how long a resolver may cache a record.

| TTL Value | Typical Use |
|-----------|-------------|
| `0` | Never cache — always re-query (used for failover, round-robin) |
| `60–300` | Short cache — for records changing frequently or during migrations |
| `3600` (1hr) | Default — most A, AAAA, MX records |
| `86400` (1 day) | Stable records — NS, SOA |
| `604800` (1 week) | Very stable — rarely changing infrastructure |

### Who Controls TTL?

```
Authoritative server sets TTL → Resolver caches for that duration
                                  → Resolver decrements TTL every second
                                    → Returns remaining TTL to clients
                                      → Re-queries when TTL = 0
```

**Clients see the remaining TTL**, not the original:

```bash
# First query — TTL = 300 (set by authoritative)
dig www.example.com A +ttlid
; www.example.com.  300  IN  A  93.184.216.34

# 90 seconds later — cached, TTL decremented
dig www.example.com A +ttlid
; www.example.com.  210  IN  A  93.184.216.34
```

### TTL Planning for DNS Changes

Before making DNS changes, **lower TTL in advance**:

```
T-48h:  Lower TTL from 3600 → 300
T-0:    Make the DNS change
T+5m:   Old record expires from all caches (within 300s)
T+24h:  Raise TTL back to 3600
```

If you change TTL and the old high-TTL is cached, you must wait for the old TTL to expire first.

---

## Cache Behavior

### RRset Caching

Resolvers cache complete **RRsets** (all records of same name+type):

```
; These three are ONE RRset — cached together, expire together
www.example.com.  300  IN  A  1.2.3.4
www.example.com.  300  IN  A  5.6.7.8
www.example.com.  300  IN  A  9.10.11.12
```

### Cache Hierarchy

```
Browser cache
    └── OS stub resolver cache
            └── Recursive resolver cache
                    └── Authoritative server (source of truth)
```

Each layer has its own TTL tracking. Browser caches often ignore DNS TTL and use their own timers.

### Cache Flushing

```bash
# Flush OS DNS cache
# Linux (systemd-resolved):
sudo systemd-resolve --flush-caches

# macOS:
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# Windows:
ipconfig /flushdns

# Flush BIND resolver cache
rndc flush
rndc flushname www.example.com

# Unbound:
unbound-control flush www.example.com
unbound-control flush_zone example.com
```

### Prefetching

Some resolvers (BIND, Unbound) support **cache prefetching** — re-querying records before they expire to avoid latency for popular names:

```
# Unbound prefetch config
server:
    prefetch: yes
    prefetch-key: yes   # also prefetch DNSKEY records
```

---

## Negative Caching (RFC 2308)

Negative responses (NXDOMAIN, NODATA) are also cached to prevent hammering authoritative servers with repeated failed queries.

### Negative Cache TTL

The negative cache TTL comes from the **SOA MINIMUM field** (RFC 2308):

```
; SOA record
example.com.  SOA  ns1.example.com. admin.example.com. (
    2024010101  ; serial
    3600        ; refresh
    900         ; retry
    604800      ; expire
    300         ; ← MINIMUM = negative cache TTL
)

; An NXDOMAIN/NODATA response for this zone is cached for 300 seconds
```

RFC 2308 introduced a change: MINIMUM used to mean the minimum TTL for all records in the zone. Now it means negative cache TTL only. Most zones set it to 300–3600 seconds.

### SERVFAIL Caching

RFC 7766 and some implementations also cache SERVFAIL responses briefly (usually 5–30 seconds) to prevent amplification of error states.

---

## Cache Poisoning

Cache poisoning attacks inject **false records** into a resolver's cache. All clients using that resolver then receive attacker-controlled answers.

### Classic Attack (Pre-2008)

```
1. Attacker queries resolver: evil.example.com A
2. Resolver queries authoritative for example.com
3. While resolver waits, attacker floods forged responses:
   - Source IP spoofed as authoritative server
   - Response answers www.example.com A (not what was asked)
   - Attacker guesses 16-bit Transaction ID
4. If guess correct → resolver accepts false A record
5. All users of that resolver get attacker IP for www.example.com
```

**Why easy to exploit:** 16-bit TX ID = only 65,536 possibilities. Could brute-force in seconds with fast connection.

---

## Kaminsky Attack (2008)

Dan Kaminsky disclosed a dramatically more effective variant in July 2008.

### The Key Insight

Instead of poisoning a single record, poison the **NS delegation**:

```
Classic: try to poison www.example.com A
         → must win race for ONE specific query
         → attacker can make ONE attempt per query

Kaminsky: query rand1.example.com, rand2.example.com, ... (non-existent names)
          → resolver MUST query authoritative for each (cache miss every time)
          → flood responses claiming NS for example.com is attacker-controlled
          → win once → control ALL future queries for example.com
          → can make thousands of attempts per second
```

### Attack Flow

```
1. Attacker queries: aaaa.example.com, aaab.example.com, ...  (random subdomains)
2. Each misses cache → resolver queries example.com authoritative
3. For each: attacker floods 65,536 forged responses, all claiming:
     Authority: example.com NS ns.attacker.com
     Additional: ns.attacker.com A <attacker IP>
4. When TX ID guessed correctly → NS record poisoned
5. Now ALL queries for anything.example.com go to attacker
```

**Time to exploit:** ~10 seconds on a fast link. Disclosed to vendors in secret in March 2008; patched before public disclosure in August 2008.

---

## Mitigations

### 1. Source Port Randomization (Immediate fix, 2008)

Randomize **both** Transaction ID (16 bits) AND source UDP port (16 bits):

```
Pre-2008:  Attacker must guess 16 bits  = 65,536 attempts needed
Post-2008: Attacker must guess 32 bits  = 4,294,967,296 attempts needed
```

All modern resolvers implement this. Check:

```bash
# Check your resolver uses random source ports
sudo tcpdump -n -i any 'udp and port 53'
# Source ports should vary widely (not always 53→53)
```

### 2. 0x20 Encoding (Case Randomization)

Randomize case of query name; server must echo back same case:

```
Query:    wWw.ExAmPlE.cOm A
Response: wWw.ExAmPlE.cOm A (must match)

Attacker can't predict case → much harder to forge valid response
```

### 3. DNSSEC (The Real Fix)

Cryptographic signatures on all records — forged responses fail validation.

See [08-dnssec.md](08-dnssec.md).

### 4. Response Rate Limiting (RRL)

Limits how many similar responses an authoritative server sends per second — mitigates amplification attacks using your server.

```
# BIND RRL config
rate-limit {
    responses-per-second 5;
    window 5;
};
```

### 5. RFC 8198 — Aggressive Negative Caching

Use NSEC/NSEC3 records to synthesize negative answers without querying, preventing attackers from forcing cache misses.

---

*Previous: [06 — Zone Management](06-zone-management.md) | Next: [08 — DNSSEC →](08-dnssec.md)*
