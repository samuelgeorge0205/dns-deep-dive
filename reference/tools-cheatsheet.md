# DNS Tools Cheat Sheet

## dig

### Basic Queries
```bash
dig example.com                        # A record (default)
dig example.com A                      # explicit A query
dig example.com AAAA                   # IPv6
dig example.com MX                     # mail servers
dig example.com NS                     # nameservers
dig example.com TXT                    # text records
dig example.com SOA                    # start of authority
dig example.com ANY                    # all records (deprecated)
dig example.com CAA                    # CA authorization
dig example.com HTTPS                  # HTTPS service binding
```

### Targeting Servers
```bash
dig @8.8.8.8 example.com              # query specific resolver
dig @ns1.example.com example.com      # query specific nameserver
dig @192.0.2.1 example.com            # query by IP
dig @1.1.1.1 +tcp example.com         # force TCP
```

### Output Control
```bash
dig +short example.com                 # just the answer value(s)
dig +noall +answer example.com         # answer section only
dig +noall +authority example.com      # authority section only
dig +ttlid example.com                 # show TTL
dig +multiline example.com SOA        # human-readable SOA
dig +stats example.com                 # query timing + server info
dig +noquestion example.com            # hide question section
```

### DNSSEC
```bash
dig +dnssec example.com A             # request DNSSEC records
dig +cd example.com A                  # disable checking (bypass validation)
dig example.com DNSKEY                 # fetch public keys
dig example.com DS                     # fetch delegation signer
dig example.com RRSIG                  # fetch signatures
```

### Tracing
```bash
dig +trace example.com                 # full iterative trace from root
dig +trace +additional example.com    # trace with glue records shown
```

### Zone Transfer
```bash
dig @ns1.example.com example.com AXFR # full zone transfer
dig @ns1.example.com example.com IXFR=2024010101  # incremental
```

### Reverse DNS
```bash
dig -x 93.184.216.34                   # reverse IPv4 lookup
dig -x 2606:2800:220:1:248:1893:25c8:1946  # reverse IPv6
dig 34.216.184.93.in-addr.arpa PTR    # explicit PTR query
```

### Special
```bash
dig +short myip.opendns.com @resolver1.opendns.com   # your public IP
dig @a.root-servers.net hostname.bind TXT CH          # root server instance
dig +subnet=1.2.3.0/24 example.com    # test GeoDNS with specific subnet
dig +bufsize=4096 example.com          # set EDNS buffer size
dig +ignore example.com                # ignore TC bit (don't retry TCP)
```

### Reading dig Output
```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1234
;; flags: qr rd ra;           ← qr=response rd=recursion-desired ra=recursion-available
;;        ^^^ aa              ← aa=authoritative (set if from authoritative server)
;;            ad              ← ad=authenticated-data (DNSSEC validated)
;; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;www.example.com.    IN  A    ← what was asked

;; ANSWER SECTION:
www.example.com.  300  IN  A  93.184.216.34  ← answer, TTL=300 remaining

;; Query time: 12 msec        ← round-trip time
;; SERVER: 8.8.8.8#53         ← which server answered
;; MSG SIZE rcvd: 56          ← packet size in bytes
```

---

## delv (DNSSEC-validating dig)

```bash
delv www.example.com A                 # validate with system trust anchors
delv +rtrace www.example.com           # show resolution trace
delv +vtrace www.example.com           # show validation trace
delv +multiline example.com DNSKEY    # human-readable keys
delv @8.8.8.8 www.example.com A       # use specific resolver

# Output indicators:
# + fully validated (AD bit set, chain verified)
# - BOGUS (validation failed)
# (unsigned) no DNSSEC records
```

---

## drill

```bash
drill example.com                      # A record
drill -t MX example.com               # MX query
drill -D example.com                   # DNSSEC (like +dnssec)
drill -T example.com                   # trace (like +trace)
drill -S example.com                   # chase DNSSEC chain
drill -k /etc/unbound/root.key example.com  # validate with key file
```

---

## nslookup

```bash
nslookup example.com                   # A record
nslookup -type=MX example.com         # MX record
nslookup -type=ANY example.com        # all records
nslookup example.com 8.8.8.8          # specific server
nslookup -debug example.com            # verbose output

# Interactive mode:
nslookup
> server 8.8.8.8
> set type=NS
> example.com
```

---

## host

```bash
host example.com                       # A + MX records
host -t MX example.com                # MX only
host -a example.com                    # all records (ANY)
host -v example.com                    # verbose
host 93.184.216.34                     # reverse lookup
host -l example.com ns1.example.com   # zone transfer (if allowed)
```

---

## nsupdate (Dynamic DNS)

```bash
# Interactive mode
nsupdate
> server ns1.example.com
> zone example.com.
> update add newhost.example.com. 3600 A 192.0.2.100
> send
> quit

# With TSIG authentication
nsupdate -k /etc/bind/keys/ddns.key
nsupdate -y hmac-sha256:keyname:base64secret

# From file
cat > /tmp/update.txt << EOF
server ns1.example.com
zone example.com.
update add host1.example.com. 300 A 10.0.0.1
update delete oldhost.example.com. A
send
EOF
nsupdate /tmp/update.txt

# Useful prereq conditions
> prereq nxdomain host.example.com.      # fail if exists
> prereq yxdomain host.example.com.      # fail if doesn't exist
> prereq yxrrset host.example.com. A     # fail if A record absent
```

---

## whois (domain registration)

```bash
whois example.com                      # registrar, nameservers, dates
whois 93.184.216.34                    # IP WHOIS (ARIN/RIPE/APNIC)
```

---

## DNSSEC-specific Tools

```bash
# Generate DNSSEC keys
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com          # ZSK
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.com   # KSK
dnssec-keygen -a Ed25519 -n ZONE example.com                   # Ed25519 ZSK

# Sign a zone
dnssec-signzone -A -3 $(head -c4 /dev/urandom | xxd -p) \
    -N INCREMENT -o example.com -t \
    example.com.zone \
    Kexample.com.+013+12345.key \
    Kexample.com.+013+67890.key

# Generate DS record from KSK
dnssec-dsfromkey Kexample.com.+013+67890.key
dnssec-dsfromkey -2 Kexample.com.+013+67890.key  # SHA-256 only

# Check zone signatures
dnssec-verify -o example.com example.com.zone.signed

# BIND management
rndc sign example.com           # re-sign zone
rndc loadkeys example.com       # load new keys from key directory
rndc dnssec -status example.com # show DNSSEC status
```

---

## Packet Capture

```bash
# Capture DNS traffic
tcpdump -i eth0 -w dns.pcap 'udp port 53 or tcp port 53'

# Capture DoT
tcpdump -i eth0 'tcp port 853'

# Read capture
tcpdump -r dns.pcap -n

# Wireshark display filters
# udp.port == 53
# dns.qry.name == "www.example.com"
# dns.flags.rcode == 3    (NXDOMAIN)
# dns.flags.aa == 1       (authoritative)
```

---

## BIND (named) Operations

```bash
# Check config syntax
named-checkconf /etc/bind/named.conf
named-checkzone example.com /etc/bind/zones/example.com.zone

# Runtime management
rndc reload                      # reload all zones
rndc reload example.com          # reload specific zone
rndc flush                       # flush all caches
rndc flushname www.example.com   # flush specific name
rndc status                      # server status
rndc stats                       # dump statistics
rndc querylog on                 # enable query logging

# Logs
tail -f /var/log/named/queries.log
```

---

## Unbound Operations

```bash
# Runtime management
unbound-control status
unbound-control flush www.example.com
unbound-control flush_zone example.com
unbound-control flush_bogus
unbound-control reload
unbound-control dump_cache > cache.txt
unbound-control load_cache < cache.txt

# Check config
unbound-checkconf /etc/unbound/unbound.conf
```

---

## Online Tools

| Tool | URL | Use |
|------|-----|-----|
| MXToolbox | https://mxtoolbox.com | Email DNS, blacklists |
| DNSViz | https://dnsviz.net | DNSSEC chain visualization |
| DNSSEC Analyzer | https://dnssec-analyzer.verisignlabs.com | DNSSEC validation |
| IntoDNS | https://intodns.com | DNS health check |
| DNS Checker | https://dnschecker.org | Global propagation check |
| Zonemaster | https://zonemaster.net | Zone quality check |
| SPF Validator | https://www.kitterman.com/spf/validate.html | SPF syntax check |
| DMARC Analyzer | https://dmarcian.com | DMARC setup |
| DNS Leak Test | https://dnsleaktest.com | VPN/DoH leak detection |
