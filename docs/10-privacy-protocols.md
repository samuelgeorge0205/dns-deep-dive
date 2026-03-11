# 10 — DNS Privacy Protocols

## Comparison

| Protocol | Port | Transport | Encryption | Standard | Notes |
|----------|------|-----------|------------|----------|-------|
| Plain DNS | 53 | UDP/TCP | None | RFC 1035 | Visible to ISP, on-path attackers |
| DNS over TLS (DoT) | 853 | TCP | TLS 1.2+ | RFC 7858 | Easily blocked on port 853 |
| DNS over HTTPS (DoH) | 443 | TCP | TLS via HTTPS | RFC 8484 | Blends with web traffic |
| DNS over QUIC (DoQ) | 853 | QUIC/UDP | TLS 1.3 | RFC 9250 | Low latency, 0-RTT |
| Oblivious DoH (ODoH) | 443 | HTTPS | Double-hop | RFC 9230 | Resolver never sees client IP |
| DNSCrypt | 443 | UDP/TCP | Curve25519 | Not IETF | Authenticates resolver |

---

## DNS over TLS (DoT)

RFC 7858. Full TLS session on TCP port 853.

```
Client ──TCP:853──► Server
        TLS handshake
        DNS messages (each with 2-byte length prefix)
```

### Setup: System Stub (Linux/systemd-resolved)

```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google
DNSOverTLS=yes
```

### Setup: Unbound

```
server:
    tls-upstream: yes
    tls-cert-bundle: /etc/ssl/certs/ca-certificates.crt

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853#cloudflare-dns.com
    forward-addr: 8.8.8.8@853#dns.google
```

### Test DoT

```bash
# kdig (from knot-dnsutils)
kdig -d @1.1.1.1 +tls-ca +tls-host=cloudflare-dns.com example.com

# openssl s_client
openssl s_client -connect 1.1.1.1:853

# Check certificate
echo | openssl s_client -connect 1.1.1.1:853 2>/dev/null | openssl x509 -noout -subject
```

---

## DNS over HTTPS (DoH)

RFC 8484. DNS messages transported over HTTPS POST or GET requests.

```
POST /dns-query HTTP/2
Host: cloudflare-dns.com
Content-Type: application/dns-message
Accept: application/dns-message

[binary DNS message]
```

Or GET with base64url-encoded message:
```
GET /dns-query?dns=AAABAAABAAAAAAAAA3d3dwdleGFtcGxlA2NvbQAAAQAB
```

### DoH Providers

| Provider | URL | Notes |
|----------|-----|-------|
| Cloudflare | `https://cloudflare-dns.com/dns-query` | 1.1.1.1 |
| Google | `https://dns.google/dns-query` | 8.8.8.8 |
| Quad9 | `https://dns.quad9.net/dns-query` | 9.9.9.9 (malware blocking) |
| NextDNS | `https://dns.nextdns.io/<id>` | Custom filtering |
| AdGuard | `https://dns.adguard.com/dns-query` | Ad blocking |

### Setup: Firefox

```
about:preferences → General → Network Settings → Enable DNS over HTTPS
```

### Setup: curl

```bash
curl --doh-url https://cloudflare-dns.com/dns-query https://example.com
```

### Setup: Custom DoH Server (Nginx + dnsdist)

```nginx
server {
    listen 443 ssl http2;
    server_name doh.example.com;
    
    location /dns-query {
        proxy_pass http://127.0.0.1:8053;
        proxy_set_header Content-Type application/dns-message;
    }
}
```

---

## DNS over QUIC (DoQ)

RFC 9250. Uses QUIC transport (UDP + TLS 1.3). Designed to reduce latency.

**Advantages over DoT:**
- No TCP head-of-line blocking
- 0-RTT session resumption (faster reconnect)
- Connection migration (mobile networks)
- Built-in multiplexing

```bash
# Test DoQ with kdig
kdig @dns.adguard.com +quic example.com

# Wireshark filter for DoQ
udp.port == 853
```

---

## Oblivious DoH (ODoH) — RFC 9230

Two-hop design: neither the relay nor the resolver knows both the client IP and the query:

```
Client → [Relay] → [Resolver]
          ↑            ↑
    Knows client IP  Knows DNS query
    but NOT query    but NOT client IP
```

```
1. Client encrypts query with Resolver's public key (HPKE)
2. Client sends to Relay (which knows client IP but can't decrypt)
3. Relay forwards ciphertext to Resolver
4. Resolver decrypts, resolves, encrypts response
5. Relay returns encrypted response to client
6. Neither Relay nor Resolver has complete picture
```

Requires trusting that Relay and Resolver don't collude.

---

## Privacy Considerations

### What Your DNS Provider Sees

| Provider | Sees Query? | Sees Client IP? | Logs? |
|----------|-------------|-----------------|-------|
| ISP | Yes | Yes | Often |
| Google 8.8.8.8 | Yes | Yes | Yes (anonymized) |
| Cloudflare 1.1.1.1 | Yes | Yes | 24h then deleted |
| Quad9 9.9.9.9 | Yes | Yes | No |
| ODoH Resolver | Yes | No (relay masks it) | Varies |

### EDNS Client Subnet (ECS) — Privacy Concern

ECS (RFC 7871) passes your /24 subnet to authoritative servers for GeoDNS:

```
# Disable ECS in Unbound for better privacy
server:
    send-client-subnet: 0.0.0.0/0  # send nothing
```

### DNS Leaks

```bash
# Check if DNS queries are leaking outside your VPN/DoH setup
# Use: https://dnsleaktest.com
# Or:
dig +short myip.opendns.com @resolver1.opendns.com
```

---

*Previous: [09 — Security Attacks](09-security-attacks.md) | Next: [11 — Advanced Topics →](11-advanced-topics.md)*
