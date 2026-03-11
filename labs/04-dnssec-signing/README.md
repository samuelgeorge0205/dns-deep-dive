# Lab 04 — DNSSEC Signing & Validation

**Goal:** Sign a zone with DNSSEC, validate responses, and understand the chain of trust.

**Prerequisites:** `bind9`, `dnssec-keygen`, `dnssec-signzone`, `delv` installed

**Time:** 60 minutes

---

## Exercise 4.1 — Generate Signing Keys

```bash
cd /tmp/dnssec-lab
mkdir -p /tmp/dnssec-lab && cd /tmp/dnssec-lab

# Copy the base zone file
cp ../../zone-examples/basic.zone mylab.zone

# Generate ZSK (Zone Signing Key)
dnssec-keygen -a ECDSAP256SHA256 -n ZONE mylab.com
# Creates: Kmylab.com.+013+XXXXX.key and Kmylab.com.+013+XXXXX.private

# Generate KSK (Key Signing Key)
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE mylab.com
# Creates: Kmylab.com.+013+YYYYY.key (with 257 flag)

# List generated keys
ls -la Kmylab.com.*
```

**Examine the keys:**
```bash
cat Kmylab.com.+013+XXXXX.key    # Public key (put this in zone as DNSKEY)
# Note: flags field — 256 = ZSK, 257 = KSK

# Which is which?
grep -l "257" Kmylab.com.*.key   # KSK
grep -l "256" Kmylab.com.*.key   # ZSK
```

---

## Exercise 4.2 — Sign the Zone

```bash
# Sign with both keys
dnssec-signzone \
    -A \
    -3 $(head -c 4 /dev/urandom | xxd -p) \
    -N INCREMENT \
    -o mylab.com \
    -t \
    mylab.zone \
    Kmylab.com.+013+<ZSK-tag>.key \
    Kmylab.com.+013+<KSK-tag>.key

# Output: mylab.zone.signed
# Also creates: dsset-mylab.com (DS records to submit to parent)
```

**Examine the signed zone:**
```bash
cat mylab.zone.signed | grep DNSKEY    # public keys in zone
cat mylab.zone.signed | grep RRSIG     # signatures
cat mylab.zone.signed | grep NSEC3     # non-existence proofs
```

**Questions:**
1. How many DNSKEY records are there? Why?
2. Every RRset should have an RRSIG. Find the RRSIG for the A record.
3. What dates does the RRSIG cover (inception/expiry)?

---

## Exercise 4.3 — Serve the Signed Zone

```bash
# BIND config
cat >> /etc/bind/named.conf.local << EOF
zone "mylab.com" {
    type primary;
    file "/tmp/dnssec-lab/mylab.zone.signed";
};
EOF

sudo rndc reload

# Query the signed zone
dig @localhost mylab.com A +dnssec
# Should see RRSIG in answer section

dig @localhost mylab.com DNSKEY
# Should see 2 DNSKEY records

dig @localhost mylab.com DNSKEY +dnssec
# Should see RRSIG(DNSKEY)
```

---

## Exercise 4.4 — Validate with delv

```bash
# delv validates DNSSEC chain
# Without a proper trust anchor for our lab zone, use -a with the KSK

# Export KSK as trust anchor
cat Kmylab.com.+013+<KSK-tag>.key > /tmp/trust-anchor.key

# Validate
delv @localhost -a /tmp/trust-anchor.key +root=mylab.com mylab.com A

# Output should include:
# + fully validated
```

---

## Exercise 4.5 — Observe NSEC3

```bash
# Query for non-existent name
dig @localhost ghost.mylab.com A +dnssec

# Look for NSEC3 records in authority section
# These prove "ghost.mylab.com" doesn't exist without revealing other names
```

**Questions:**
1. What's the hashed name in the NSEC3 record?
2. Could you figure out all names in the zone from NSEC3? (vs NSEC)

---

## Exercise 4.6 — Simulate Signature Expiry

```bash
# Sign with very short validity (already expired!)
dnssec-signzone \
    -e 20200101000000 \
    -s 20190101000000 \
    -o mylab.com \
    -N INCREMENT \
    mylab.zone \
    Kmylab.com.+013+<ZSK>.key \
    Kmylab.com.+013+<KSK>.key

# Load the expired zone and query
# Validators should return SERVFAIL (signatures expired)
dig @localhost mylab.com A +dnssec
```

---

## Exercise 4.7 — Generate DS Record

```bash
# Get DS record to submit to parent zone
dnssec-dsfromkey Kmylab.com.+013+<KSK-tag>.key

# Output looks like:
# mylab.com. IN DS 12345 13 2 ABCDEF...
# This is what you'd paste into the parent zone (or registrar interface)

# Also check dsset file from signzone
cat dsset-mylab.com.
```

---

## Exercise 4.8 — Check Real DNSSEC Chains

```bash
# Check well-known DNSSEC-signed domains
delv www.isc.org A
delv cloudflare.com A

# Check DS records in parent zone
dig cloudflare.com DS @a.gtld-servers.net

# Validate full chain
dig +dnssec +cd cloudflare.com A     # bypass validation
dig +dnssec cloudflare.com A         # with validation
# Compare: both should return same answer; AD bit set in second
```
