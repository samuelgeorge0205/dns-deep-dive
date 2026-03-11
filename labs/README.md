# DNS Deep Dive — Labs

Hands-on exercises to reinforce the theory in `docs/`.

## Prerequisites

```bash
# Install required tools (Ubuntu/Debian)
sudo apt-get install -y dnsutils bind9utils bind9 tcpdump wireshark-cli

# macOS
brew install bind knot-dns tcpdump

# Verify
dig -v
```

## Lab Index

| Lab | Topic | Difficulty | Time |
|-----|-------|------------|------|
| [01](01-basic-dig-queries/) | Basic dig queries | Beginner | 20 min |
| [02](02-trace-resolution/) | Tracing full resolution | Beginner | 30 min |
| [03](03-zone-file-setup/) | Zone file setup with BIND | Intermediate | 45 min |
| [04](04-dnssec-signing/) | DNSSEC signing & validation | Intermediate | 60 min |
| [05](05-cache-poisoning-demo/) | Cache poisoning simulation | Advanced | 60 min |
| [06](06-doh-dot-setup/) | DNS-over-TLS/HTTPS setup | Intermediate | 45 min |
