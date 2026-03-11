# 03 — Resource Records (RRs)

## Table of Contents
- [Wire Format](#wire-format)
- [Name Encoding & Compression](#name-encoding--compression)
- [All Record Types](#all-record-types)
- [Record Deep Dives](#record-deep-dives)
- [RRsets](#rrsets)
- [TTL Rules](#ttl-rules)

---

## Wire Format

Every DNS resource record has the same on-wire structure (RFC 1035 §3.2.1):

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           NAME                                |
|                        (variable)                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           TYPE            |           CLASS                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           TTL                                 |
|                        (32 bits)                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         RDLENGTH          |           RDATA                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                                   |
|                        (variable)                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Size | Description |
|-------|------|-------------|
| NAME | variable | Owner name — encoded as labels or compressed pointer |
| TYPE | 2 bytes | Record type (e.g., A=1, NS=2, CNAME=5, MX=15) |
| CLASS | 2 bytes | Class: IN=1 (Internet), CH=3 (Chaosnet), ANY=255 |
| TTL | 4 bytes | Unsigned 32-bit integer, seconds to cache |
| RDLENGTH | 2 bytes | Byte length of RDATA |
| RDATA | variable | Record-specific data |

---

## Name Encoding & Compression

### Label Encoding

A domain name is encoded as a sequence of **length-prefixed labels**, terminated by a zero byte:

```
www.example.com  →  03 77 77 77        (3, "www")
                     07 65 78 61 6d 70 6c 65  (7, "example")
                     03 63 6f 6d        (3, "com")
                     00                 (root terminator)
```

Total: 17 bytes for `www.example.com.`

### Message Compression (RFC 1035 §4.1.4)

To save space, repeated names use a **pointer** — two high bits set to 1 (`0xC0`) followed by a 14-bit offset from message start:

```
Pointer byte format:
  1 1 x x x x x x  x x x x x x x x
  ↑ ↑ └──────────────────────────┘
  pointer flag         offset (0–16383)

Example: C0 0C = pointer to byte offset 12
```

```
# Hex dump example
Header (12 bytes)
Question: 03 77 77 77  07 65 78 61 6d 70 6c 65  03 63 6f 6d  00  # www.example.com.
Answer NAME: C0 0C      # pointer back to offset 12 (www.example.com. again)
```

If a byte starts with `0xC0`, the next byte gives the offset. If it starts with `0x00`–`0x3F`, it's a label length.

---

## All Record Types

### Standard Types

| Type | Code | RDATA | Description |
|------|------|-------|-------------|
| **A** | 1 | 4 bytes IPv4 | IPv4 address |
| **NS** | 2 | domain name | Authoritative nameserver |
| **CNAME** | 5 | domain name | Canonical name alias |
| **SOA** | 6 | see below | Start of Authority |
| **PTR** | 12 | domain name | Reverse DNS pointer |
| **HINFO** | 13 | two strings | Host info (historical) |
| **MX** | 15 | uint16 + name | Mail exchanger |
| **TXT** | 16 | string(s) | Free text (SPF, DKIM, etc.) |
| **AAAA** | 28 | 16 bytes IPv6 | IPv6 address |
| **SRV** | 33 | uint16 × 3 + name | Service location |
| **NAPTR** | 35 | complex | Naming authority pointer (VoIP) |
| **DS** | 43 | hash | DNSSEC delegation signer |
| **SSHFP** | 44 | fingerprint | SSH key fingerprint |
| **RRSIG** | 46 | signature | DNSSEC RRset signature |
| **NSEC** | 47 | name + bitmap | DNSSEC next secure record |
| **DNSKEY** | 48 | flags + key | DNSSEC public key |
| **NSEC3** | 50 | hashed | DNSSEC hashed NSEC |
| **NSEC3PARAM** | 51 | params | NSEC3 parameters |
| **TLSA** | 52 | cert assoc | DANE TLS certificate association |
| **SVCB** | 64 | service bind | Service binding |
| **HTTPS** | 65 | service bind | HTTPS service binding |
| **CAA** | 257 | flags + tag + value | Certification Authority Authorization |

### Meta / Pseudo Types

| Type | Code | Description |
|------|------|-------------|
| **OPT** | 41 | EDNS0 pseudo-record (not a real RR) |
| **AXFR** | 252 | Zone transfer request |
| **ANY** | 255 | Query all types (deprecated for public use, RFC 8482) |

---

## Record Deep Dives

### A Record
```
; Wire: NAME TYPE CLASS TTL RDLENGTH RDATA
www.example.com.  300  IN  A  93.184.216.34

; RDATA = 4 bytes: 5D B8 D8 22
```

### AAAA Record
```
www.example.com.  300  IN  AAAA  2606:2800:220:1:248:1893:25c8:1946

; RDATA = 16 bytes (full 128-bit IPv6 address)
```

### SOA Record
```
example.com.  3600  IN  SOA  ns1.example.com.  admin.example.com. (
    2024010101  ; SERIAL   — version, usually YYYYMMDDNN
    3600        ; REFRESH  — secondary polls primary every N seconds
    900         ; RETRY    — retry after N seconds if refresh fails
    604800      ; EXPIRE   — secondary stops answering after N seconds
    300         ; MINIMUM  — negative cache TTL (RFC 2308)
)
```

| Field | Meaning |
|-------|---------|
| MNAME | Primary nameserver hostname |
| RNAME | Admin email (@ replaced with .) |
| SERIAL | Zone version — must increment on every change |
| REFRESH | How often secondary checks for updates |
| RETRY | How long secondary waits before retrying failed refresh |
| EXPIRE | How long secondary serves zone without contacting primary |
| MINIMUM | RFC 2308: negative caching TTL (was originally minimum record TTL) |

**Serial number convention:** `YYYYMMDDNN` — e.g., `2024010102` = January 1 2024, 2nd change that day.

### MX Record
```
; Format: priority  mail-server-hostname
example.com.  3600  IN  MX  10  mail1.example.com.
example.com.  3600  IN  MX  20  mail2.example.com.
example.com.  3600  IN  MX  30  backup-mail.otherprovider.net.
```

- Lower priority number = higher preference
- Multiple same-priority records = random selection (load balancing)
- MX must point to an A/AAAA record, **never a CNAME**

### TXT Record
```
; Multiple strings, each max 255 bytes
; Multiple strings are concatenated
example.com.  TXT  "v=spf1 ip4:192.0.2.0/24 include:_spf.google.com ~all"
example.com.  TXT  "google-site-verification=abc123"
_dmarc.example.com.  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

Common TXT record uses:

| Use Case | Prefix/Name | Example Content |
|----------|------------|-----------------|
| SPF | @ | `v=spf1 ip4:... -all` |
| DKIM | `selector._domainkey` | `v=DKIM1; k=rsa; p=<pubkey>` |
| DMARC | `_dmarc` | `v=DMARC1; p=reject` |
| Domain verification | @ or specific | `google-site-verification=...` |
| ACME DNS-01 | `_acme-challenge` | random token |

### SRV Record
```
; _service._proto.name.  TTL  IN  SRV  priority  weight  port  target
_sip._tcp.example.com.     3600  IN  SRV  10  20  5060  sip1.example.com.
_sip._tcp.example.com.     3600  IN  SRV  10  80  5060  sip2.example.com.
_http._tcp.example.com.    3600  IN  SRV  0   0   80    www.example.com.
_xmpp-client._tcp.jabber.org.  IN  SRV  5   0   5222  xmpp.jabber.org.
```

| Field | Range | Meaning |
|-------|-------|---------|
| priority | 0–65535 | Lower = more preferred |
| weight | 0–65535 | Among same priority; higher = more traffic |
| port | 0–65535 | Service port |
| target | FQDN | Hostname (must have A/AAAA) — `.` means unavailable |

**Weight selection algorithm:** Sum all weights at same priority. Pick random number 0–sum. Walk records accumulating weight until number exceeded.

### CAA Record
```
; Flags  Tag   Value
example.com.  CAA  0  issue    "letsencrypt.org"
example.com.  CAA  0  issuewild "pki.goog"
example.com.  CAA  0  iodef    "mailto:security@example.com"
```

| Tag | Meaning |
|-----|---------|
| `issue` | CA allowed to issue single-name certs |
| `issuewild` | CA allowed to issue wildcard certs |
| `iodef` | Where to report policy violations |

### HTTPS / SVCB Records (RFC 9460)
```
; Priority 0 = alias mode (like CNAME for services)
; Priority 1+ = service mode
example.com.  HTTPS  1  .  alpn="h3,h2" ipv4hint="1.2.3.4" ech=<base64>

; Alias mode
example.com.  HTTPS  0  svc.example.net.
```

Encodes: ALPN (protocol negotiation), IP hints, ECH (Encrypted Client Hello) keys — all in DNS.

### PTR Record
```
; Reverse DNS
34.216.184.93.in-addr.arpa.  PTR  www.example.com.

; IPv6 reverse (nibble format)
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa.  PTR  www.example.com.
```

---

## RRsets

An **RRset** is the set of all RRs with the same owner name, class, and type:

```
; This is ONE RRset (www.example.com A)
www.example.com.  300  IN  A  1.2.3.4
www.example.com.  300  IN  A  5.6.7.8
www.example.com.  300  IN  A  9.10.11.12
```

Rules:
- All records in an RRset **must have the same TTL** (RFC 2181 §5.2)
- RRsets are signed as a unit in DNSSEC (RRSIG covers the whole RRset)
- Resolvers cache and return whole RRsets, not individual records

---

## TTL Rules

- TTL is set by the **authoritative server**, not the resolver
- Resolvers **decrement** TTL as time passes and must re-query when it hits 0
- TTL = 0 means **do not cache** — always re-query
- Negative TTL (NXDOMAIN/NODATA caching) comes from **SOA MINIMUM** field
- RFC 8767 recommends minimum TTL of 5 seconds in resolvers to prevent abuse

```bash
# Check a record's current TTL (decremented value from cache)
dig www.example.com A +ttlid

# The TTL in the response shows how long the resolver has LEFT in its cache
# (not the original TTL from the authoritative server)
```

---

*Previous: [02 — Architecture](02-architecture.md) | Next: [04 — Message Format →](04-message-format.md)*
