# 11 — Advanced DNS Topics

## Topics
- [Split-Horizon DNS](#split-horizon-dns)
- [Anycast DNS](#anycast-dns)
- [GeoDNS & DNS Load Balancing](#geodns--dns-load-balancing)
- [DNS Response Policy Zones (RPZ)](#dns-response-policy-zones-rpz)
- [Multicast DNS (mDNS)](#multicast-dns-mdns)
- [LLMNR](#llmnr)
- [DANE / TLSA](#dane--tlsa)
- [DNS Long-Term Operation](#dns-long-term-operation)

---

## Split-Horizon DNS

Returns different answers based on who's asking. Also called split-brain or split-view DNS.

```
Internal client (10.x.x.x) queries mail.example.com:
    → returns 10.0.0.50 (internal mail server)

External client (public IP) queries mail.example.com:
    → returns 93.184.216.50 (public MX IP / load balancer)
```

### BIND Views

```
// named.conf
acl "internal" { 10.0.0.0/8; 192.168.0.0/16; };

view "internal" {
    match-clients { internal; };
    recursion yes;
    zone "example.com" {
        type primary;
        file "/etc/bind/zones/internal/example.com.zone";
    };
};

view "external" {
    match-clients { any; };
    recursion no;
    zone "example.com" {
        type primary;
        file "/etc/bind/zones/external/example.com.zone";
    };
};
```

### Pitfalls

- DNSSEC is very complex with split-horizon (different RRSIGs for internal/external)
- Zone transfers must be configured per-view
- EDNS Client Subnet can leak internal topology info

---

## Anycast DNS

One IP address announced from multiple geographic locations via BGP.

```
IP: 1.1.1.1

   Client in Europe ──BGP──► Frankfurt instance
   Client in Asia   ──BGP──► Singapore instance
   Client in US     ──BGP──► New York instance

All use same IP (1.1.1.1), BGP routes to nearest
```

### How It Works

```
1. Multiple data centers each have a BGP session
2. All announce the same IP prefix (e.g., 1.1.1.0/24) to their upstream peers
3. Internet routes client to nearest BGP path
4. "Nearest" = shortest AS path (not always geographic)
5. Failover: if one DC goes down, BGP withdraws its route; traffic shifts automatically
```

### Who Uses Anycast

| Service | IPs |
|---------|-----|
| Cloudflare DNS | 1.1.1.1, 1.0.0.1 |
| Google DNS | 8.8.8.8, 8.8.4.4 |
| Quad9 | 9.9.9.9 |
| All 13 root servers | Hundreds of anycast nodes each |

### Root Server Anycast Count

```bash
# See which root server instance you're hitting (Chaosnet query)
dig @a.root-servers.net hostname.bind TXT CH +short
# Returns: "iad.l.root-servers.org" (IAD = Dulles, VA)
```

---

## GeoDNS & DNS Load Balancing

### Round-Robin DNS

```
; Multiple A records → resolver rotates through them
www.example.com.  A  1.2.3.4
www.example.com.  A  5.6.7.8
www.example.com.  A  9.10.11.12

; No health checking — dead servers still returned
; TTL=0 recommended for round-robin
```

### Weighted Round-Robin

Implemented via SRV records or via DNS management API (Route 53, Cloudflare):

```
; Via SRV
_http._tcp.example.com  SRV  0  10  80  server1.example.com.  ; 10/(10+90) = 10%
_http._tcp.example.com  SRV  0  90  80  server2.example.com.  ; 90/(10+90) = 90%
```

### Health-Check Based Routing (Route 53 Style)

```
1. Monitoring service checks each endpoint
2. If endpoint fails health check → remove its A record from DNS
3. If passes → add back
4. Requires very short TTL (60s or less) for fast failover
```

### GeoDNS with EDNS Client Subnet

```
Client in EU → query reaches resolver with ECS /24 prefix
Authoritative receives ECS data → returns EU CDN node

Client in US → authoritative receives US ECS prefix → returns US CDN node
```

```bash
# Test GeoDNS response for different client subnets
dig @8.8.8.8 www.example.com A +subnet=1.2.3.0/24   # simulate client from 1.2.3.x
dig @8.8.8.8 www.example.com A +subnet=200.0.0.0/24  # simulate client from Brazil
```

---

## DNS Response Policy Zones (RPZ)

Allows resolvers to override DNS responses for policy enforcement — malware blocking, parental controls, corporate filtering.

```
Policy zone structure (special zone format):
  rpz.example.internal.  SOA ...
  rpz.example.internal.  NS  localhost.

  ; Block malware.evil.com → NXDOMAIN
  malware.evil.com.rpz.example.internal.  CNAME  .

  ; Redirect ads.tracker.com → 0.0.0.0
  ads.tracker.com.rpz.example.internal.  A  0.0.0.0

  ; Block entire evil.com zone
  *.evil.com.rpz.example.internal.  CNAME  .
```

### BIND RPZ Config

```
response-policy {
    zone "rpz.example.internal" policy nxdomain;
    zone "ads-rpz.example.internal";
} recursive-only yes;
```

### RPZ Triggers

| Trigger | Blocks when... |
|---------|---------------|
| QNAME | Query name matches |
| CLIENT-IP | Query comes from specific IP |
| RESPONSE-IP | Response would contain specific IP |
| NSDNAME | NS hostname matches |
| NSIP | NS IP matches |

---

## Multicast DNS (mDNS)

RFC 6762. Zero-configuration name resolution on local network without a server.

```
Protocol:
  Multicast address: 224.0.0.251 (IPv4), FF02::FB (IPv6)
  Port: 5353 UDP
  Domain: .local (by convention)
  No central server — devices announce themselves
```

### How It Works

```
Device wants to find printer.local:
1. Sends multicast query to 224.0.0.251:5353
2. All .local devices receive it
3. Device named "printer" responds with its IP
4. Questioner caches response

Device connects to network:
1. Announces its name via multicast (Probing + Announcement)
2. If name conflict → appends (2), (3), etc.
```

### mDNS Implementations

| Platform | Implementation |
|----------|---------------|
| macOS | Bonjour (mDNSResponder) |
| Linux | avahi-daemon |
| iOS/iPadOS | Built-in |
| Windows | WSD (Web Services for Devices) for some |

```bash
# Browse mDNS on Linux
avahi-browse -a                    # list all services
avahi-browse _http._tcp            # list HTTP services
avahi-resolve -n hostname.local    # resolve a .local name
```

### Security Note

mDNS is link-local only (not routed). Should not be allowed across network segments. Attackers on local network can spoof mDNS responses.

---

## LLMNR

Link-Local Multicast Name Resolution. Microsoft's alternative to mDNS.

```
Port: 5355 UDP/TCP
Multicast: 224.0.0.252 (IPv4), FF02::1:3 (IPv6)
Used when: DNS fails, for unqualified names on local network
```

### Security Risk: Responder Attack

LLMNR (and NetBIOS) are frequently exploited in Windows environments:

```
1. Windows machine queries LLMNR: who is "fileserver"?
2. Attacker tool (Responder) responds: "I am fileserver, here's my IP"
3. Victim sends NTLMv2 authentication attempt to attacker
4. Attacker captures NTLMv2 hash → crack offline or relay

Tool: impacket-ntlmrelayx, Responder
```

**Mitigation:**
```powershell
# Disable LLMNR via Group Policy
# Computer Configuration → Administrative Templates →
# Network → DNS Client → Turn off multicast name resolution = Enabled

# Or via registry:
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" `
    -Name "EnableMulticast" -Value 0
```

---

## DANE / TLSA

DNS-Based Authentication of Named Entities (RFC 6698). Associates TLS certificates with DNS names, stored as TLSA records.

```
; Format: _port._proto.host  TLSA  usage selector matching-type data
_443._tcp.www.example.com.  TLSA  3  1  1  <sha256-of-cert-pubkey>

; Usage:
;   0 = CA constraint (cert must chain to this CA)
;   1 = Service certificate constraint
;   2 = Trust anchor assertion  
;   3 = Domain-issued certificate (DANE-EE, most common)

; Selector:
;   0 = Full certificate
;   1 = SubjectPublicKeyInfo only (recommended)

; Matching type:
;   0 = Exact match
;   1 = SHA-256 hash
;   2 = SHA-512 hash
```

```bash
# Check TLSA record
dig _443._tcp.www.example.com TLSA

# Verify DANE with openssl + TLSA
# (requires DNSSEC on the zone)
```

**Benefit:** Certificate pinning via DNS — attacker can't MITM even with rogue CA cert, if DNSSEC protects the TLSA record.

---

## DNS Long-Term Operation

### Monitoring Checklist

```bash
# 1. Check SOA serial consistency across all nameservers
for ns in $(dig +short example.com NS); do
    echo -n "$ns: "; dig @$ns +short example.com SOA
done

# 2. Check DNSSEC signature expiry
dig +dnssec example.com A | grep RRSIG
# Check expiration date in RRSIG record

# 3. Verify zone transfers working
dig @secondary-ns.example.com example.com SOA

# 4. Test negative response
dig nonexistent.example.com A
# Should return NXDOMAIN, not SERVFAIL

# 5. Check TTLs are reasonable
dig example.com NS    # Should be 3600+
dig www.example.com A # Should be 300+

# 6. Verify CAA records exist
dig example.com CAA
```

### Operational Runbook

| Task | Frequency | Command |
|------|-----------|---------|
| Check serial sync | Daily | `for ns in $(dig +short NS); do dig @$ns SOA +short; done` |
| Check RRSIG expiry | Weekly | `dig +dnssec A \| grep RRSIG` |
| Test resolution | Hourly (automated) | `dig +time=2 +tries=1 www.example.com A` |
| Validate DNSSEC chain | Weekly | `delv www.example.com A` |
| Audit CNAME targets | Monthly | Check all CNAMEs resolve |
| Review TTLs | Quarterly | Ensure TTLs appropriate for change frequency |

---

*Previous: [10 — Privacy Protocols](10-privacy-protocols.md)*
