# 06 — Zone Management

## Table of Contents
- [Zone File Format](#zone-file-format)
- [Zone File Directives](#zone-file-directives)
- [Complete Zone File Example](#complete-zone-file-example)
- [Zone Transfers](#zone-transfers)
- [NOTIFY Protocol](#notify-protocol)
- [TSIG Authentication](#tsig-authentication)
- [Dynamic DNS (DDNS)](#dynamic-dns-ddns)

---

## Zone File Format

Zone files are text files read by authoritative nameservers. Format defined in RFC 1035 §5.

### Basic Syntax

```
; This is a comment
name  [TTL]  [class]  type  rdata

; Examples:
www   3600  IN  A  192.0.2.1
www         IN  A  192.0.2.1   ; TTL omitted → uses $TTL default
www   3600      A  192.0.2.1   ; class omitted → inherits previous
      3600  IN  A  192.0.2.1   ; name omitted → uses previous name
@     3600  IN  A  192.0.2.1   ; @ = zone origin ($ORIGIN)
```

### Special Names

| Symbol | Meaning |
|--------|---------|
| `@` | Zone origin (same as $ORIGIN) |
| (empty/blank) | Same as previous name |
| `*` | Wildcard — matches any label not explicitly defined |

---

## Zone File Directives

| Directive | Example | Description |
|-----------|---------|-------------|
| `$ORIGIN` | `$ORIGIN example.com.` | Sets the default domain appended to unqualified names |
| `$TTL` | `$TTL 3600` | Sets default TTL for records without explicit TTL |
| `$INCLUDE` | `$INCLUDE /etc/bind/sub.zone` | Include another file |
| `$GENERATE` | `$GENERATE 1-10 host$ A 192.0.2.$` | Generate sequential records |

### $GENERATE Example

```
$GENERATE 1-5 host$ IN A 192.0.2.$

; Expands to:
host1  IN  A  192.0.2.1
host2  IN  A  192.0.2.2
host3  IN  A  192.0.2.3
host4  IN  A  192.0.2.4
host5  IN  A  192.0.2.5
```

---

## Complete Zone File Example

```
; ─────────────────────────────────────────────────────────
; Zone: example.com
; File: /etc/bind/zones/example.com.zone
; ─────────────────────────────────────────────────────────

$ORIGIN example.com.
$TTL 3600

; ── SOA (required, exactly one) ──────────────────────────
@  IN  SOA  ns1.example.com.  hostmaster.example.com. (
       2024010102  ; serial   YYYYMMDDNN
       3600        ; refresh  1 hour
       900         ; retry    15 min
       604800      ; expire   7 days
       300         ; minimum  5 min (negative cache TTL)
   )

; ── Name Servers (required, minimum 2) ───────────────────
@          IN  NS   ns1.example.com.
@          IN  NS   ns2.example.com.

; ── Nameserver A records (in-zone = needed for glue) ─────
ns1        IN  A    203.0.113.1
ns1        IN  AAAA 2001:db8::1
ns2        IN  A    203.0.113.2
ns2        IN  AAAA 2001:db8::2

; ── Apex records ─────────────────────────────────────────
@          IN  A    93.184.216.34
@          IN  AAAA 2606:2800:220:1:248:1893:25c8:1946

; ── Mail ─────────────────────────────────────────────────
@          IN  MX   10  mail1.example.com.
@          IN  MX   20  mail2.example.com.
mail1      IN  A    93.184.216.50
mail2      IN  A    93.184.216.51

; ── Web ──────────────────────────────────────────────────
www        IN  A    93.184.216.34
www        IN  AAAA 2606:2800:220:1:248:1893:25c8:1946
ftp        IN  CNAME www.example.com.

; ── Email authentication ─────────────────────────────────
@          IN  TXT  "v=spf1 ip4:93.184.216.0/24 include:_spf.google.com ~all"
@          IN  CAA  0 issue "letsencrypt.org"
@          IN  CAA  0 issuewild ";"   ; no wildcard certs
_dmarc     IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
; DKIM selector (key managed by mail provider):
mail._domainkey  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0G..."

; ── Services ─────────────────────────────────────────────
_https._tcp  IN  HTTPS  1  .  alpn="h2,h3"
_sip._tcp    IN  SRV   10  10  5060  sip.example.com.
_xmpp-client._tcp  IN  SRV  5  0  5222  xmpp.example.com.

; ── Wildcard ─────────────────────────────────────────────
*          IN  A    93.184.216.34   ; catch-all for undefined subdomains

; ── Subdomain delegation ─────────────────────────────────
dev        IN  NS   ns1.dev.example.com.
dev        IN  NS   ns2.dev.example.com.
ns1.dev    IN  A    10.0.0.1    ; glue record (in-zone NS)
ns2.dev    IN  A    10.0.0.2    ; glue record
```

---

## Zone Transfers

Zone transfers synchronize zone data from primary to secondary nameservers.

### AXFR — Full Zone Transfer (RFC 5936)

Transfers the **complete zone**. Always uses **TCP**. QTYPE = AXFR (252).

```
; Response format:
SOA (opening)
RR1
RR2
...
RRn
SOA (closing — same as opening, marks end of transfer)
```

```bash
# Attempt AXFR (should be restricted!)
dig @ns1.example.com example.com AXFR

# Using nslookup
nslookup
> server ns1.example.com
> set type=AXFR
> example.com
```

**Security:** Always restrict AXFR to known secondary IPs via ACL or TSIG:

```
# BIND zone config
zone "example.com" {
    type primary;
    file "example.com.zone";
    allow-transfer { 203.0.113.2; key tsig-key; };
};
```

### IXFR — Incremental Zone Transfer (RFC 1995)

Transfers **only changes** since the secondary's current serial number:

```
; IXFR query includes current SERIAL in authority section SOA
; Server computes diff and returns:

SOA (new)
SOA (old) ← delete following records
deleted_record1
deleted_record2
SOA (new) ← add following records
new_record1
new_record2
SOA (new) ← end marker
```

If the diff is too large (or server doesn't support IXFR), server falls back to full AXFR.

```bash
# Force IXFR
dig @ns1.example.com example.com IXFR=2024010101
#                                       ↑ provide current serial
```

### Zone Serial Number Management

Serial **must increase** for secondaries to fetch updates:

```
; Common formats:
2024010102    ; YYYYMMDDNN — date-based (most common)
1704067202    ; Unix timestamp (works but resets in 2038)
42            ; Monotonically increasing integer (simple)

; IMPORTANT: Zone serial is unsigned 32-bit
; Secondaries compare using RFC 1982 serial arithmetic
; Wraps around: 2^32 = 4,294,967,296

; SOA serial comparison:
; i1 < i2 if (i2 - i1) mod 2^32 < 2^31
```

```bash
# Check serial on all nameservers (should match after propagation)
for ns in ns1.example.com ns2.example.com; do
    echo -n "$ns: "; dig @$ns example.com SOA +short
done
```

---

## NOTIFY Protocol (RFC 1996)

Primary sends NOTIFY when zone changes → secondaries immediately poll for updates instead of waiting for REFRESH interval.

```
Primary changes zone → sends NOTIFY to all secondaries
                               │
                    OPCODE=NOTIFY (4)
                    Question: example.com SOA
                               │
                         Secondary receives
                               │
                    Queries primary: SOA check
                               │
                    Serial changed? → IXFR/AXFR
                               │
                    Responds with NOTIFY ACK
```

```
# BIND primary config
zone "example.com" {
    type primary;
    also-notify { 203.0.113.2; 203.0.113.3; };
    notify yes;
};
```

---

## TSIG Authentication (RFC 2845)

Transaction SIGnatures — HMAC-based authentication for zone transfers and dynamic updates.

### How TSIG Works

```
1. Both servers share a secret key (HMAC-SHA256 or similar)
2. Signer computes HMAC over: request ID + time + key name + algorithm + message
3. Adds TSIG RR to Additional section of every message
4. Receiver verifies HMAC before processing
```

### TSIG Wire Format

```
; TSIG RR (in Additional section, not in zone file)
key-name.  0  ANY  TSIG  algorithm-name  time-signed  fudge  MAC  original-id  error  other
```

| Field | Description |
|-------|-------------|
| key-name | Name of the key (must match on both sides) |
| algorithm | e.g., `hmac-sha256.` |
| time-signed | Unix timestamp |
| fudge | Time allowance (seconds) — prevents replay attacks |
| MAC | HMAC bytes |
| original-id | Original message ID |
| error | RCODE for TSIG-specific errors |

### TSIG Setup in BIND

```bash
# Generate a TSIG key
tsig-keygen -a hmac-sha256 transfer-key

# Output:
key "transfer-key" {
    algorithm hmac-sha256;
    secret "base64encodedkeyhere==";
};
```

```
# named.conf on both primary and secondary
key "transfer-key" {
    algorithm hmac-sha256;
    secret "base64encodedkeyhere==";
};

zone "example.com" {
    type secondary;
    primaries { 203.0.113.1 key transfer-key; };
};
```

---

## Dynamic DNS (DDNS — RFC 2136)

Allows authorized clients to add/modify/delete records without editing zone files.

### Use Cases

- DHCP server updates DNS when clients get leases
- Let's Encrypt ACME DNS-01 challenge automation
- Dynamic IP hosts
- Kubernetes ExternalDNS

### UPDATE Message Format

DNS UPDATE uses `OPCODE=5`. Has same 4-section structure but sections renamed:

```
Header (same)
Zone section    (what zone to update)
Prerequisite   (conditions that must be true before update)
Update section  (records to add/delete)
Additional     (TSIG for authentication)
```

### nsupdate Examples

```bash
# Interactive nsupdate
nsupdate -k /etc/bind/transfer-key.conf
> server ns1.example.com
> zone example.com.
> update add newhost.example.com. 3600 A 192.0.2.50
> send

# Delete a record
> update delete oldhost.example.com. A

# Delete specific record
> update delete oldhost.example.com. A 192.0.2.30

# Multiple updates in one transaction
> update add host1.example.com. 3600 A 10.0.0.1
> update add host2.example.com. 3600 A 10.0.0.2
> send

# With TSIG key
nsupdate -y hmac-sha256:keyname:base64secret
```

### Prerequisite Section

Conditions checked before update executes:

```bash
# Only update if name doesn't already exist
> prereq nxdomain host1.example.com.

# Only update if name exists
> prereq yxdomain host1.example.com.

# Only update if specific record exists
> prereq yxrrset host1.example.com. A

# Only update if specific value exists
> prereq yxrrset host1.example.com. A 192.0.2.1
```

---

*Previous: [05 — Resolution Flow](05-resolution-flow.md) | Next: [07 — Caching →](07-caching.md)*
