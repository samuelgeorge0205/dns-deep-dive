# 08 — DNSSEC

## Table of Contents
- [Overview](#overview)
- [DNSSEC Record Types](#dnssec-record-types)
- [Chain of Trust](#chain-of-trust)
- [Signing a Zone](#signing-a-zone)
- [Validation Process](#validation-process)
- [Algorithm Reference](#algorithm-reference)
- [NSEC vs NSEC3](#nsec-vs-nsec3)
- [Key Rollover](#key-rollover)
- [Common Failures](#common-failures)

---

## Overview

DNSSEC (DNS Security Extensions) adds **cryptographic signatures** to DNS data.

| What DNSSEC provides | What it does NOT provide |
|----------------------|--------------------------|
| Data origin authentication | Confidentiality (no encryption) |
| Data integrity | Availability |
| Authenticated denial of existence | DDoS protection |

RFCs: 4033 (overview), 4034 (records), 4035 (protocol), 4509 (SHA-256 DS), 5155 (NSEC3), 6840 (clarifications), 8080 (EdDSA)

---

## DNSSEC Record Types

### DNSKEY — Public Key Record

```
example.com.  3600  IN  DNSKEY  256  3  13  <base64-public-key>
                              ↑    ↑  ↑
                           flags ver alg

; Flags:
;   256 = Zone Signing Key (ZSK)  — signs all zone RRsets
;   257 = Key Signing Key  (KSK)  — signs only DNSKEY RRset

; Protocol: always 3 (DNSSEC)
; Algorithm: 13 = ECDSA P-256/SHA-256 (recommended)
```

**KSK vs ZSK:**

| | KSK (257) | ZSK (256) |
|-|-----------|-----------|
| Signs | DNSKEY RRset only | All other RRsets |
| Key size | Larger (more secure) | Smaller (performance) |
| Rollover | Rarely (requires DS update in parent) | More frequently |
| In DS record | Yes | No |

### RRSIG — Signature Record

```
www.example.com.  300  IN  RRSIG  A  13  3  300  (
    20240201000000  ; signature expiration
    20240101000000  ; signature inception
    12345           ; key tag (identifies DNSKEY)
    example.com.    ; signer name
    <base64-signature>
)
; type covered = A
; algorithm = 13
; labels = 3 (www.example.com has 3 labels)
; original TTL = 300
```

### DS — Delegation Signer

Stored in the **parent zone**. Links parent to child zone's KSK:

```
; In .com TLD zone:
example.com.  3600  IN  DS  12345  13  2  <hex-sha256-hash-of-KSK>
                          ↑       ↑   ↑
                       key tag   alg  digest type (1=SHA-1, 2=SHA-256)
```

DS record = `SHA-256(key_tag || algorithm || DNSKEY_RDATA)`

### NSEC — Next Secure

```
; Lists next name alphabetically + types present at current name
www.example.com.  3600  IN  NSEC  zzz.example.com.  A AAAA RRSIG NSEC
                              ↑                      ↑
                         next name              types at www
```

Used to prove non-existence without querying authoritative.

### NSEC3 — Hashed NSEC

```
; Hash of name, not name itself — prevents zone enumeration
<hash>.example.com.  IN  NSEC3  1  0  10  AB12  <next-hash>  A AAAA RRSIG
                                ↑  ↑   ↑   ↑
                              alg flags iter salt
```

### NSEC3PARAM

Zone configuration for NSEC3:

```
example.com.  IN  NSEC3PARAM  1  0  10  AB12
;                             ↑  ↑   ↑   ↑
;                           alg flg iter salt
```

---

## Chain of Trust

```
. (Root)
│  DNSKEY (KSK, ZSK) — root KSK is hardcoded in resolvers as trust anchor
│  RRSIG(DNSKEY) — signed by root KSK
│  DS for .com — signed by root ZSK
│
└── com.
│   DNSKEY (KSK, ZSK)
│   RRSIG(DNSKEY) — signed by .com KSK
│   DS for example.com — signed by .com ZSK
│   RRSIG(DS) — signed by .com ZSK
│
└── example.com.
    DNSKEY (KSK, ZSK)
    RRSIG(DNSKEY) — signed by example.com KSK
    www A 93.184.216.34 — signed by example.com ZSK
    RRSIG(A) — signed by example.com ZSK
```

**Trust anchor:** Resolver trusts root KSK fingerprint (hardcoded in software). IANA publishes it at https://data.iana.org/root-anchors/

### How a Resolver Validates

```
1. Receive: www.example.com A + RRSIG(A)
2. Need DNSKEY(example.com) to verify RRSIG(A)
3. Fetch: example.com DNSKEY + RRSIG(DNSKEY)
4. Need to trust that DNSKEY — check DS in .com
5. Fetch: example.com DS from .com zone + RRSIG(DS)
6. Need to trust .com DNSKEY — check DS in root
7. Fetch: com DS from root zone + RRSIG(DS)
8. Verify root DNSKEY against hardcoded trust anchor ← root of trust
9. Chain verified ✓
```

---

## Signing a Zone

### Generate Keys

```bash
# Generate ZSK (ECDSA P-256)
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com
# Creates: Kexample.com.+013+12345.key and .private

# Generate KSK
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.com
# Creates: Kexample.com.+013+67890.key and .private
```

### Sign the Zone

```bash
# Sign zone file (validity: 30 days, re-sign before expiry)
dnssec-signzone -A -3 $(head -c 4 /dev/urandom | xxd -p) \
    -N INCREMENT -o example.com -t \
    example.com.zone \
    Kexample.com.+013+12345.key \
    Kexample.com.+013+67890.key

# Output: example.com.zone.signed
```

### Submit DS to Registrar

```bash
# Get DS record to submit to parent zone (registrar)
dnssec-dsfromkey Kexample.com.+013+67890.key
# example.com. IN DS 67890 13 2 <hash>

# Submit this DS record through your domain registrar's interface
```

### Automated with BIND inline-signing

```
zone "example.com" {
    type primary;
    file "example.com.zone";
    dnssec-policy default;   # BIND 9.16+ — auto-signs and rolls keys
    inline-signing yes;
};
```

---

## Validation Process

```bash
# Validate with delv (DNSSEC-aware dig replacement)
delv www.example.com A
# + fully validated → DNSSEC chain is intact
# - SERVFAIL → validation failed

# Check with dig (AD bit = authenticated data)
dig www.example.com A +dnssec
# AD flag in response = resolver validated successfully

# Check a specific record type
dig example.com DNSKEY +dnssec

# Test negative response validation
dig ghost.example.com A +dnssec
# Should include NSEC/NSEC3 + RRSIG to prove non-existence

# Online validators
# https://dnssec-analyzer.verisignlabs.com/
# https://dnsviz.net/
```

---

## Algorithm Reference

| Number | Name | Key Size | Status |
|--------|------|----------|--------|
| 5 | RSA/SHA-1 | 1024–4096 | **Deprecated** — weak hash |
| 7 | RSASHA1-NSEC3 | 1024–4096 | **Deprecated** |
| 8 | RSA/SHA-256 | 2048–4096 | Widely supported |
| 10 | RSA/SHA-512 | 2048–4096 | Large signatures |
| 13 | **ECDSA P-256/SHA-256** | 256 | **Recommended** — small, fast |
| 14 | ECDSA P-384/SHA-384 | 384 | High security |
| 15 | **Ed25519** | 256 | **Recommended** — modern, fast |
| 16 | Ed448 | 448 | Maximum security |

Algorithm 13 or 15 recommended for new deployments.

---

## NSEC vs NSEC3

| Feature | NSEC | NSEC3 |
|---------|------|-------|
| Name in record | Plaintext | Hashed |
| Zone enumeration | **Possible** — walk NSEC chain | Prevented (hashes) |
| Denial of existence | Proves exact range | Proves hash range |
| Performance | Faster | Slightly slower |
| Complexity | Simpler | More complex (salt, iterations) |

### NSEC Zone Walking

```bash
# Enumerate all names in a DNSSEC zone using NSEC
# (Only works if zone uses NSEC, not NSEC3)
dig +dnssec +noall +answer example.com NSEC
# Returns next name → query that name's NSEC → repeat until back to start
```

---

## Key Rollover

### ZSK Rollover (Pre-publish method)

```
Phase 1 (30 days): Publish new ZSK alongside old ZSK
                   Sign zone with BOTH keys
                   Caches learn new ZSK

Phase 2 (0 days):  Switch signing to new ZSK only
                   Remove old ZSK from DNSKEY RRset

Phase 3 (TTL):     Wait for old key to expire from caches
                   Remove old RRSIG records
```

### KSK Rollover (Double-DS method)

KSK rollover is more complex because it requires updating the DS record in the parent zone:

```
1. Generate new KSK
2. Publish new KSK (both KSKs in DNSKEY)
3. Submit new DS to parent registrar
4. Wait for DS TTL to expire in caches
5. Switch signing to new KSK
6. Remove old KSK from DNSKEY
7. Remove old DS from parent
```

---

## Common Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| SERVFAIL on signed domain | RRSIG expired | Re-sign zone; set up auto-renewal |
| SERVFAIL on validation | DS/DNSKEY mismatch | Check DS matches KSK; re-submit DS |
| `ad` bit missing | Resolver doesn't validate | Use validating resolver |
| Bogus response | Signature invalid or missing | Check zone signing; check NSEC coverage |
| SERVFAIL after key rollover | Premature key removal | Follow rollover timing carefully |

```bash
# Debug DNSSEC validation
dig www.example.com A +dnssec +cd  # +cd = checking disabled (bypass validation)
# If answer comes back with +cd but not without → validation failing

# Check if DS record is published in parent
dig example.com DS @a.gtld-servers.net

# Check DNSKEY in zone
dig example.com DNSKEY +dnssec

# Use DNSViz for visual chain of trust
# https://dnsviz.net/d/example.com/dnssec/
```

---

*Previous: [07 — Caching](07-caching.md) | Next: [09 — Security Attacks →](09-security-attacks.md)*
