# Packet Captures

Annotated DNS packet breakdowns for learning and reference.

## Contents

| File | Description |
|------|-------------|
| `standard-query-annotated.md` | Plain UDP DNS query/response byte-by-byte |
| `axfr-transfer-annotated.md` | Zone transfer over TCP |
| `dnssec-query-annotated.md` | DNSSEC-signed response with RRSIG |

## Capturing Your Own

```bash
# Capture DNS traffic
sudo tcpdump -i any -n 'udp port 53 or tcp port 53' -w capture.pcap

# Read and analyze
tcpdump -r capture.pcap -n -v

# Open in Wireshark for visual analysis
wireshark capture.pcap
```

## Wireshark DNS Filters

```
dns                              # all DNS traffic
dns.qry.name == "example.com"   # specific query name
dns.flags.rcode == 3            # NXDOMAIN responses
dns.flags.aa == 1               # authoritative responses
dns.flags.tc == 1               # truncated responses
dns.qry.type == 1               # A record queries
dns.qry.type == 28              # AAAA record queries
tcp.port == 53                  # DNS over TCP only
tcp.port == 853                 # DNS over TLS (DoT)
