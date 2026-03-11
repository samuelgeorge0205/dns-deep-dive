# 09 — DNS Security Attacks & Mitigations

| Attack | Vector | Mitigation |
|--------|--------|------------|
| Cache Poisoning | Forged responses | DNSSEC, port randomization |
| DNS Amplification DDoS | Open resolvers | RRL, block open recursion |
| NXDOMAIN Flood | Query exhaustion | Negative caching, rate limiting |
| Zone Walking | NSEC chain | NSEC3 |
| DNS Hijacking | Registrar compromise | Registry lock, DNSSEC |
| Subdomain Takeover | Dangling CNAME | Audit CNAMEs regularly |
| NXNS Attack | Malicious NS referrals | RFC 8198, NS fan-out limit |
| DNS Rebinding | JS + low TTL | Bind to localhost, TTL minimums |
| Slow Drip DoS | Deep delegation chains | Query limits per client |
| BGP Hijacking | Route injection | RPKI, route filtering |

---

## Cache Poisoning

See [07-caching.md](07-caching.md#cache-poisoning) for full details.

**Quick summary:**
- Forge DNS responses with correct Transaction ID
- Inject false A records pointing to attacker infrastructure
- All resolver clients affected until TTL expires

**Mitigations:** DNSSEC (cryptographic), source port randomization (entropy), 0x20 encoding

---

## DNS Amplification DDoS

**Attack:**
```
Attacker spoofs victim's IP → queries open resolvers with ANY/TXT/DNSKEY
Open resolver sends large response to victim
Amplification factor: 50–100x (small query → large response)

Example:
  Query: 40 bytes (ANY isc.org)
  Response: 4000+ bytes
  Amplification: 100x
```

**Real-world scale:** Millions of open resolvers × 100x amplification = terabit-scale DDoS.

**Mitigations:**
```bash
# 1. Never run open recursive resolver (restrict to known clients)
# BIND: allow-recursion { 192.168.0.0/16; 10.0.0.0/8; };

# 2. Response Rate Limiting (RRL) — RFC 8020
rate-limit {
    responses-per-second 5;
    referrals-per-second 5;
    nodata-per-second 5;
    nxdomains-per-second 5;
    slip 2;
    window 5;
};

# 3. BCP38 — ISPs should filter spoofed source IPs (ingress filtering)
# https://www.bcp38.info/

# 4. Disable ANY queries (RFC 8482)
```

---

## NXDOMAIN Flood (Pseudo-Random Subdomain Attack)

```
Attack: Send millions of queries for random non-existent subdomains
  rand1.example.com, rand2.example.com, ... (endless unique names)

Effect:
  - Cache can't help (every query unique)
  - Authoritative server gets flooded
  - Legitimate queries time out
  - SERVFAIL cascades to all clients
```

**Mitigations:**
```
1. Rate limit NXDOMAIN responses per client IP
2. Negative caching (RFC 2308) — already cached NXDOMAINs don't hit authoritative
3. Synthesize NXDOMAIN from NSEC (RFC 8198)
4. Upstream filtering by recursive resolver operators
```

---

## Zone Walking / Enumeration

```
Attack on NSEC-protected zones:
1. Query example.com NSEC → returns "next name" = aaa.example.com
2. Query aaa.example.com NSEC → returns "next" = bbb.example.com
3. Continue until full zone enumerated

Result: Complete list of all hostnames in zone (intelligence gathering)
```

**Mitigation:** Use NSEC3 with salt and high iterations.

```bash
# Check if zone uses NSEC or NSEC3
dig example.com NSEC
dig example.com NSEC3PARAM   # if this returns data → zone uses NSEC3
```

---

## DNS Hijacking

**Attack vectors:**
```
1. Registrar account compromise → NS records changed
2. Nameserver compromise → zone data modified
3. BGP hijack of authoritative server's IP range
4. Authoritative server software vulnerability
```

**Real-world examples:**
- Sea Turtle (APT): hijacked NS records for government domains (2017-2019)
- DNSpionage: targeted Middle East registrars
- GoDaddy breach (2020): credentials for 1.2M accounts exposed

**Mitigations:**
```
1. Registry Lock (ServerDeleteProhibited, ServerTransferProhibited, ServerUpdateProhibited)
   → Requires out-of-band verification (phone call) to change NS records

2. Multi-factor authentication on registrar account

3. DNSSEC
   → Even if NS records changed, resolver validation fails without valid KSK
   → Provides detection even if hijack occurs

4. RPKI (Resource Public Key Infrastructure)
   → Prevents BGP hijacking of authoritative server IPs

5. Monitor NS records with external tools (Zabbix, Nagios, commercial)
```

---

## Subdomain Takeover

```
Attack on dangling CNAME records:

1. www.example.com CNAME → myapp.somecloud.io
2. Company removes myapp.somecloud.io service but forgets CNAME
3. Attacker claims myapp.somecloud.io on same cloud provider
4. Now controls content served at www.example.com

High-risk services: GitHub Pages, Heroku, Azure, AWS Elastic Beanstalk, S3
```

**Detection:**
```bash
# Find dangling CNAMEs
# Get all CNAME records
dig example.com CNAME
dig +short example.com ANY | grep CNAME

# Check if CNAME target exists
dig CNAME-target.somecloud.io A
# NXDOMAIN = potential takeover vector!

# Automated tools:
# subjack, subzy, can-i-take-over-xyz (GitHub)
```

**Mitigation:**
```
1. Remove CNAME records BEFORE deprovisioning cloud resources
2. Regularly audit all CNAME records
3. Use CAA records to restrict certificate issuance
4. Automate CNAME validation in infrastructure-as-code pipelines
```

---

## NXNS Attack (2020)

Discovered by Herzberg & Shulman. Amplifies resolver work using malicious NS referrals.

```
Attack:
1. Attacker controls malicious authoritative server for attack.com
2. Attacker sends query to victim resolver: rand.attack.com
3. Attacker's server returns referral with many NS records:
   attack.com  NS  ns1.victim.com.    ← points at victim
   attack.com  NS  ns2.victim.com.
   attack.com  NS  ns3.victim.com.
   ... (up to 1500 NS records)
4. Resolver queries ALL NS targets to find answers
5. Victim nameserver flooded with queries from legitimate resolvers
Amplification: 1580x reported
```

**Mitigations:**
- RFC 8198: Aggressive negative caching
- Limit NS records followed per referral (implemented by BIND, Unbound post-disclosure)
- Rate limit queries to individual authoritative servers

---

## DNS Rebinding

**Bypasses browser same-origin policy:**

```
Attack:
1. Victim visits attacker.com (DNS TTL = 1 second)
2. Browser caches attacker's IP briefly
3. After TTL, DNS for attacker.com changes to 192.168.1.1 (internal router)
4. Browser JavaScript on attacker.com now makes requests to 192.168.1.1
5. Browser thinks it's talking to attacker.com (same origin)
6. Attacker reads internal services: router admin, IoT devices, localhost APIs
```

**Mitigations:**
```
1. Browsers: Pin DNS for session duration (not just TTL)
2. Services: Check Host header, reject unexpected values
3. DNS resolver: Reject responses with private IPs for public names (DNS rebind protection)
4. Minimum TTL enforcement: Unbound/BIND enforce TTL >= 5 seconds
```

```bash
# Unbound: reject responses with private IPs
server:
    private-address: 192.168.0.0/16
    private-address: 10.0.0.0/8
    private-address: 172.16.0.0/12
```

---

## KeyTrap (2024 — CVE-2023-50387)

DNSSEC resource exhaustion attack:

```
Attack:
1. Attacker-controlled zone has many DNSKEY records + many RRSIG records
2. All combinations must be tried during validation
3. Exponential work for resolver: O(n²) cryptographic operations
4. Single query can consume 100% CPU for minutes
5. Resolver becomes unavailable (DoS)
```

**Impact:** Affected nearly all DNSSEC-validating resolvers. Patched in Feb 2024.

**Mitigation:** Update resolver software. BIND ≥ 9.18.25, Unbound ≥ 1.19.1.

---

## Summary: Defense Layers

```
Layer 1 — Transport:    DoT/DoH (prevent eavesdropping/tampering in transit)
Layer 2 — Data:         DNSSEC (cryptographic integrity of records)
Layer 3 — Infrastructure: Registry Lock, MFA (prevent hijacking)
Layer 4 — Network:      RPKI, BCP38 (prevent BGP hijacking, IP spoofing)
Layer 5 — Application:  Audit CNAMEs, Host header validation (prevent takeover)
Layer 6 — Rate limiting: RRL, per-client limits (prevent DoS/amplification)
```

---

*Previous: [08 — DNSSEC](08-dnssec.md) | Next: [10 — Privacy Protocols →](10-privacy-protocols.md)*
