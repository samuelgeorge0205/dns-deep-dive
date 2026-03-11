# 05 — DNS Resolution Flow

## Table of Contents
- [Overview](#overview)
- [Step-by-Step Iterative Resolution](#step-by-step-iterative-resolution)
- [All 8 Packets Annotated](#all-8-packets-annotated)
- [CNAME Chain Resolution](#cname-chain-resolution)
- [Reverse DNS Resolution](#reverse-dns-resolution)
- [Negative Responses](#negative-responses)
- [Cache Hit Flow](#cache-hit-flow)

---

## Overview

Resolving `www.example.com A` from scratch involves up to **8 messages** (4 query/response pairs):

```
CLIENT          STUB          RECURSIVE          ROOT         TLD (.com)    AUTHORITATIVE
  │               │           RESOLVER            │               │          (example.com)
  │──── Q1 ──────►│                               │               │               │
  │               │──────────── Q2 ──────────────►│               │               │
  │               │◄─────────── R2 (referral) ────│               │               │
  │               │──────────────────── Q3 ───────────────────────►│               │
  │               │◄──────────────────── R3 (referral) ────────────│               │
  │               │────────────────────────────── Q4 ─────────────────────────────►│
  │               │◄─────────────────────────────── R4 (answer) ───────────────────│
  │◄──── R1 ──────│                               │               │               │
```

---

## Step-by-Step Iterative Resolution

### Pre-query: Local Checks

Before any network query, the stub resolver checks:

```
1. /etc/hosts  (highest priority, overrides DNS)
2. Local resolver cache
3. mDNS (.local names only)
4. → Only then: send to recursive resolver
```

### Step 1: Client → Recursive Resolver

The browser calls `getaddrinfo("www.example.com")` → OS stub resolver → sends UDP query to configured recursive resolver (e.g., `8.8.8.8`).

```
UDP  CLIENT:random_port → 8.8.8.8:53
ID=0x1A2B  QR=0  RD=1  QDCOUNT=1
Question: www.example.com. A IN
Additional: OPT (EDNS0, UDP=4096, DO=1)
```

### Step 2: Recursive Resolver → Root

Recursive resolver checks its cache — miss. It has **root hints** pre-loaded. Picks one root server randomly:

```
UDP  RESOLVER:49152 → 198.41.0.4:53  (a.root-servers.net)
ID=0x3C4D  QR=0  RD=0  QDCOUNT=1
Question: www.example.com. A IN
```

Note `RD=0` — resolver queries root iteratively (doesn't ask root to recurse).

### Step 3: Root → Recursive (Referral)

Root doesn't know `www.example.com`, but it knows `.com` TLD servers:

```
UDP  198.41.0.4:53 → RESOLVER:49152
ID=0x3C4D  QR=1  AA=0  RCODE=NOERROR
ANCOUNT=0  NSCOUNT=13  ARCOUNT=13

;; AUTHORITY SECTION:
com.  172800  NS  a.gtld-servers.net.
com.  172800  NS  b.gtld-servers.net.
...  (13 NS records for .com TLD)

;; ADDITIONAL SECTION:
a.gtld-servers.net.  172800  A  192.5.6.30    ← glue
b.gtld-servers.net.  172800  A  192.33.14.30  ← glue
...
```

This is a **referral** — AA=0, empty Answer, Authority + Additional sections populated.

### Step 4: Recursive → TLD (.com)

```
UDP  RESOLVER:49153 → 192.5.6.30:53  (a.gtld-servers.net)
ID=0x7E8F  QR=0  RD=0  QDCOUNT=1
Question: www.example.com. A IN
```

### Step 5: TLD → Recursive (Referral)

TLD knows the NS records for `example.com`:

```
UDP  192.5.6.30:53 → RESOLVER:49153
ID=0x7E8F  QR=1  AA=0  RCODE=NOERROR
ANCOUNT=0  NSCOUNT=2  ARCOUNT=2

;; AUTHORITY SECTION:
example.com.  172800  NS  ns1.example.com.
example.com.  172800  NS  ns2.example.com.

;; ADDITIONAL SECTION:
ns1.example.com.  3600  A  205.251.196.1   ← glue
ns2.example.com.  3600  A  205.251.197.1   ← glue
```

### Step 6: Recursive → Authoritative

```
UDP  RESOLVER:49154 → 205.251.196.1:53  (ns1.example.com)
ID=0xAB12  QR=0  RD=0  QDCOUNT=1
Question: www.example.com. A IN
```

### Step 7: Authoritative → Recursive (Answer)

```
UDP  205.251.196.1:53 → RESOLVER:49154
ID=0xAB12  QR=1  AA=1  RCODE=NOERROR
ANCOUNT=1  NSCOUNT=2  ARCOUNT=0

;; ANSWER SECTION:
www.example.com.  300  IN  A  93.184.216.34

;; AUTHORITY SECTION:
example.com.  3600  NS  ns1.example.com.
example.com.  3600  NS  ns2.example.com.
```

`AA=1` — authoritative answer! Resolver caches this for 300 seconds.

### Step 8: Recursive → Client (Final Answer)

```
UDP  8.8.8.8:53 → CLIENT:random_port
ID=0x1A2B  QR=1  RA=1  RCODE=NOERROR
ANCOUNT=1

;; ANSWER SECTION:
www.example.com.  300  IN  A  93.184.216.34
```

---

## All 8 Packets Annotated

```
Pkt  Dir                  Src→Dst              Key Flags          Notes
─────────────────────────────────────────────────────────────────────────────
 1   Client→Resolver      CLIENT→8.8.8.8:53    QR=0 RD=1         Recursive query
 2   Resolver→Root        RESOLVER→Root:53     QR=0 RD=0         Iterative query
 3   Root→Resolver        Root→RESOLVER        QR=1 AA=0 NOAN    Referral to .com
 4   Resolver→TLD         RESOLVER→TLD:53      QR=0 RD=0         Iterative query
 5   TLD→Resolver         TLD→RESOLVER         QR=1 AA=0 NOAN    Referral to example.com NS
 6   Resolver→Auth        RESOLVER→NS:53       QR=0 RD=0         Iterative query
 7   Auth→Resolver        NS→RESOLVER          QR=1 AA=1         Final authoritative answer
 8   Resolver→Client      8.8.8.8→CLIENT       QR=1 RA=1         Cached + returned to client
```

---

## CNAME Chain Resolution

CNAME (Canonical Name) records redirect to another name. Resolvers must **chase** the chain.

### Example: www.example.com → CDN

```
;; Step 1: Query www.example.com A
;; Authoritative returns:
www.example.com.     300  IN  CNAME  cdn.example.net.

;; Resolver must now resolve cdn.example.net A
;; (starts new resolution from root if not cached)
;; Authoritative for cdn.example.net returns:
cdn.example.net.     60   IN  A      1.2.3.4

;; Final response to client includes FULL CHAIN:
www.example.com.     300  IN  CNAME  cdn.example.net.
cdn.example.net.     60   IN  A      1.2.3.4
```

### CNAME Rules

| Rule | Detail |
|------|--------|
| Cannot coexist | CNAME cannot share a name with any other record type (except RRSIG, NSEC) |
| Only one | An RRset can contain only one CNAME record |
| No MX/NS target | MX and NS records must **never** point to a CNAME |
| Chain limit | No formal RFC limit; resolvers typically stop at 8–10 hops |
| Loop detection | Resolvers must detect and break CNAME loops |

### DNAME Record (RFC 6672)

DNAME is like CNAME but for entire subtrees:

```
; Delegate entire old.example.com subtree to new.example.com
old.example.com.  DNAME  new.example.com.

; www.old.example.com → www.new.example.com (automatic)
; Any sub-label under old.example.com is mapped
```

---

## Reverse DNS Resolution

Maps IP addresses → hostnames using special zones.

### IPv4 Reverse

```
IP: 93.184.216.34

1. Reverse the octets: 34.216.184.93
2. Append .in-addr.arpa.: 34.216.184.93.in-addr.arpa.
3. Query PTR type

dig -x 93.184.216.34
# Equivalent to:
dig 34.216.184.93.in-addr.arpa. PTR

# Response:
34.216.184.93.in-addr.arpa.  86400  IN  PTR  www.example.com.
```

### IPv6 Reverse

```
IP: 2001:0db8:0000:0000:0000:0000:0000:0001

1. Expand to full form: 20010db8000000000000000000000001
2. Split into nibbles: 2 0 0 1 0 d b 8 ...
3. Reverse: 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 8 b d 0 1 0 0 2
4. Dot-separate and append .ip6.arpa.:
   1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa.

dig -x 2001:db8::1
```

### Reverse Zone Delegation

ISPs receive IP blocks and delegate reverse zones:

```
# ARIN delegates 93.184.216.0/24 to Edgecast
# In ARIN's zone (93.in-addr.arpa):
216.184.93.in-addr.arpa.  NS  rdns1.edgecast.com.
216.184.93.in-addr.arpa.  NS  rdns2.edgecast.com.
```

### Sub-/24 Delegation (RFC 2317)

When an ISP assigns a /28 (16 IPs) to a customer:

```
# ISP delegates /28 (192.0.2.32–47) using CNAME trick
32.2.0.192.in-addr.arpa.  CNAME  32.32-47.2.0.192.in-addr.arpa.
33.2.0.192.in-addr.arpa.  CNAME  33.32-47.2.0.192.in-addr.arpa.
...
47.2.0.192.in-addr.arpa.  CNAME  47.32-47.2.0.192.in-addr.arpa.

32-47.2.0.192.in-addr.arpa.  NS  ns1.customer.com.

# Customer's zone:
32.32-47.2.0.192.in-addr.arpa.  PTR  host1.customer.com.
```

---

## Negative Responses

### NXDOMAIN (Name Does Not Exist)

```
dig ghost.example.com A

;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1

;; AUTHORITY SECTION:
example.com.  300  IN  SOA  ns1.example.com. admin.example.com. (
    2024010101 3600 900 604800 300 )
    ↑
    SOA MINIMUM (300) = negative cache TTL
```

### NODATA (Name Exists, Type Doesn't)

```
dig example.com AAAA  (only A record exists)

;; ->>HEADER<<- status: NOERROR, ANCOUNT=0
;; AUTHORITY SECTION:
example.com.  300  IN  SOA  ...
```

### Distinguishing NXDOMAIN vs NODATA

| | NXDOMAIN | NODATA |
|-|----------|--------|
| RCODE | 3 | 0 (NOERROR) |
| Answer section | Empty | Empty |
| Authority section | SOA | SOA |
| Meaning | Name doesn't exist | Name exists, no records of that type |

Both are **negatively cached** using the SOA MINIMUM TTL.

---

## Cache Hit Flow

When resolver has a cached answer:

```
Client → Recursive Resolver
             │
             ├─ Check cache → HIT
             │   (TTL still valid)
             │
             └─ Return cached answer immediately
                (no queries to root/TLD/authoritative)

Response will have:
  AA=0  (not from authoritative — from cache)
  RA=1  (recursion available)
  TTL = remaining cache time (not original TTL)
```

```bash
# First query — full resolution, TTL=300
dig www.example.com A
; TTL = 300

# 60 seconds later — cache hit, TTL decremented
dig www.example.com A
; TTL = 240
```

---

*Previous: [04 — Message Format](04-message-format.md) | Next: [06 — Zone Management →](06-zone-management.md)*
