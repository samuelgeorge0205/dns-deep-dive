# Lab 06 — DNS over TLS and DNS over HTTPS

**Goal:** Set up and verify DoT and DoH, understand the differences, detect DNS leaks.

**Prerequisites:** `curl`, `kdig` (knot-dnsutils), `openssl`, `unbound`

**Time:** 45 minutes

---

## Exercise 6.1 — Test DoT with kdig

```bash
# Install knot-dnsutils
sudo apt install knot-dnsutils   # Debian/Ubuntu
brew install knot                # macOS

# Query via DoT (port 853)
kdig -d @1.1.1.1 +tls-ca +tls-host=cloudflare-dns.com example.com A
kdig @8.8.8.8 +tls-ca +tls-host=dns.google example.com A
kdig @9.9.9.9 +tls-ca +tls-host=dns.quad9.net example.com A
```

**Observe:**
- TLS handshake details in `-d` output
- Certificate presented
- Total query time vs plain DNS (run `dig @1.1.1.1 example.com A` to compare)

---

## Exercise 6.2 — Inspect the TLS Certificate

```bash
# See the TLS certificate for Cloudflare's DoT server
openssl s_client -connect 1.1.1.1:853 -servername cloudflare-dns.com < /dev/null 2>/dev/null | \
    openssl x509 -noout -text | grep -A2 "Subject:"

# Check certificate validity dates
openssl s_client -connect 1.1.1.1:853 -servername cloudflare-dns.com < /dev/null 2>/dev/null | \
    openssl x509 -noout -dates

# View full certificate
echo | openssl s_client -connect 1.1.1.1:853 2>/dev/null | openssl x509 -noout -text
```

---

## Exercise 6.3 — Test DoH with curl

```bash
# DoH uses standard HTTPS — queries as POST with binary DNS messages
# curl has built-in DoH support:

curl --doh-url https://cloudflare-dns.com/dns-query https://example.com -I

# Or send a raw DoH query:
# First, encode the DNS query as base64url
# Query: example.com A IN
QUERY=$(printf '\x00\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x01\x07example\x03com\x00\x00\x01\x00\x01\x00\x00)\x10\x00\x00\x00\x00\x00\x00\x00' | base64 | tr '+/' '-_' | tr -d '=')

curl -s "https://cloudflare-dns.com/dns-query?dns=$QUERY" \
    -H "Accept: application/dns-message" | xxd | head -20
```

---

## Exercise 6.4 — Configure systemd-resolved for DoT

```bash
# Edit /etc/systemd/resolved.conf
sudo tee /etc/systemd/resolved.conf << EOF
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google
FallbackDNS=9.9.9.9#dns.quad9.net
DNSOverTLS=yes
DNSSEC=yes
EOF

# Restart resolved
sudo systemctl restart systemd-resolved

# Test
resolvectl status
resolvectl query example.com

# Verify DoT is being used
sudo tcpdump -i any -n 'tcp port 853' -c 5 &
dig example.com A
# Should see TCP:853 traffic
```

---

## Exercise 6.5 — Set Up Unbound with DoT Forwarding

```bash
# Install Unbound
sudo apt install unbound

# Configure DoT forwarding
sudo tee /etc/unbound/unbound.conf.d/dot.conf << EOF
server:
    verbosity: 1
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    tls-cert-bundle: /etc/ssl/certs/ca-certificates.crt
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: yes

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853#cloudflare-dns.com
    forward-addr: 1.0.0.1@853#cloudflare-dns.com
    forward-addr: 8.8.8.8@853#dns.google
EOF

# Check config
unbound-checkconf

# Start Unbound
sudo systemctl start unbound

# Test
dig @127.0.0.1 -p 5335 example.com A

# Verify it uses TLS upstream
sudo tcpdump -i any -n 'tcp port 853' -c 5 &
dig @127.0.0.1 -p 5335 github.com A
```

---

## Exercise 6.6 — DNS Leak Test

```bash
# 1. Find your current public IP
curl -s https://api.ipify.org

# 2. Find what DNS server is being used
dig +short myip.opendns.com @resolver1.opendns.com
dig +short whoami.akamai.net

# 3. Check if DNS is leaking outside VPN/proxy
# Visit https://dnsleaktest.com (or use the API)
curl -s https://bash.ws/dnsleak/test/$RANDOM | python3 -m json.tool

# 4. Compare resolvers being used before/after enabling DoT
```

---

## Exercise 6.7 — Packet Capture Comparison

```bash
# Terminal 1: Capture plain DNS
sudo tcpdump -i any -n 'udp port 53' -w /tmp/plain-dns.pcap &

# Make queries
dig @8.8.8.8 example.com A
dig @8.8.8.8 google.com A
kill %1

# Terminal 1: Capture DoT
sudo tcpdump -i any -n 'tcp port 853' -w /tmp/dot-dns.pcap &

# Make DoT queries
kdig @1.1.1.1 +tls-ca example.com A
kdig @1.1.1.1 +tls-ca google.com A
kill %1

# Analyze captures
# Plain DNS: you can read query names
tcpdump -r /tmp/plain-dns.pcap -A | grep -a "example\|google"

# DoT: encrypted, unreadable
tcpdump -r /tmp/dot-dns.pcap -A | grep -a "example\|google"
# (should find nothing readable)
```

**Questions:**
1. What can an ISP see in the plain DNS capture?
2. What can they see in the DoT capture?
3. Can they tell you're making DNS queries via DoT? (Yes — port 853 is visible even if content isn't)
4. How does DoH differ in visibility? (Blends with port 443 HTTPS traffic)
