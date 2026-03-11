# DNS Record Types — Complete Reference

## All Record Types

| Type | Code | RFC | RDATA Format | Common Use |
|------|------|-----|-------------|------------|
| **A** | 1 | 1035 | 4-byte IPv4 | Host address |
| **NS** | 2 | 1035 | Domain name | Nameserver delegation |
| MD | 3 | 1035 | Domain name | Mail destination (obsolete) |
| MF | 4 | 1035 | Domain name | Mail forwarder (obsolete) |
| **CNAME** | 5 | 1035 | Domain name | Alias |
| **SOA** | 6 | 1035 | MNAME RNAME serial refresh retry expire minimum | Zone authority |
| MB | 7 | 1035 | Domain name | Mailbox (experimental) |
| MG | 8 | 1035 | Domain name | Mail group (experimental) |
| MR | 9 | 1035 | Domain name | Mail rename (experimental) |
| NULL | 10 | 1035 | Anything | Experimental |
| WKS | 11 | 1035 | IP + protocol bitmap | Well-known services (deprecated) |
| **PTR** | 12 | 1035 | Domain name | Reverse DNS |
| **HINFO** | 13 | 1035 | CPU OS | Host info (mostly unused) |
| MINFO | 14 | 1035 | RMAILBX EMAILBX | Mailbox info (experimental) |
| **MX** | 15 | 1035 | priority + name | Mail exchange |
| **TXT** | 16 | 1035 | Text strings | Free text (SPF, DKIM, etc.) |
| RP | 17 | 1183 | mbox-dname txt-dname | Responsible person |
| AFSDB | 18 | 1183 | subtype hostname | AFS database |
| X25 | 19 | 1183 | PSDN-address | X.25 address |
| ISDN | 20 | 1183 | ISDN-address | ISDN address |
| RT | 21 | 1183 | preference host | Route-through |
| NSAP | 22 | 1706 | NSAP | NSAP address |
| SIG | 24 | 2535 | | Old DNSSEC signature (obsolete) |
| KEY | 25 | 2535 | | Old DNSSEC key (obsolete) |
| PX | 26 | 2163 | | X.400 mail mapping |
| GPOS | 27 | 1712 | | Geographic position |
| **AAAA** | 28 | 3596 | 16-byte IPv6 | IPv6 address |
| LOC | 29 | 1876 | lat lon alt | Geographic location |
| NXT | 30 | 2535 | | Old DNSSEC next record (obsolete) |
| EID | 31 | — | | Endpoint identifier |
| NIMLOC | 32 | — | | Nimrod locator |
| **SRV** | 33 | 2782 | priority weight port target | Service location |
| ATMA | 34 | — | | ATM address |
| **NAPTR** | 35 | 3403 | order pref flags service regexp replacement | Name authority pointer |
| KX | 36 | 2230 | preference exchanger | Key exchanger |
| CERT | 37 | 4398 | type key-tag algorithm cert | Certificate |
| DNAME | 39 | 6672 | target | Delegation name (subtree alias) |
| OPT | 41 | 6891 | options | EDNS0 pseudo-record |
| APL | 42 | 3123 | address prefix list | |
| **DS** | 43 | 4034 | key-tag algorithm digest-type digest | DNSSEC delegation signer |
| **SSHFP** | 44 | 4255 | algorithm fp-type fingerprint | SSH key fingerprint |
| IPSECKEY | 45 | 4025 | | IPsec key |
| **RRSIG** | 46 | 4034 | type-covered algorithm labels ttl expiry inception tag signer signature | DNSSEC signature |
| **NSEC** | 47 | 4034 | next-domain type-bitmap | DNSSEC next secure |
| **DNSKEY** | 48 | 4034 | flags protocol algorithm public-key | DNSSEC public key |
| DHCID | 49 | 4701 | identifier-type digest-type digest | DHCP identifier |
| **NSEC3** | 50 | 5155 | algorithm flags iterations salt next-hash types | Hashed NSEC |
| **NSEC3PARAM** | 51 | 5155 | algorithm flags iterations salt | NSEC3 parameters |
| **TLSA** | 52 | 6698 | usage selector matching-type cert-assoc | DANE TLS association |
| SMIMEA | 53 | 8162 | usage selector matching-type cert | S/MIME cert association |
| HIP | 55 | 8005 | | Host identity protocol |
| NINFO | 56 | — | | Zone status info |
| RKEY | 57 | — | | |
| TALINK | 58 | — | | Trust anchor link |
| CDS | 59 | 7344 | | Child DS (for automated rollover) |
| CDNSKEY | 60 | 7344 | | Child DNSKEY |
| OPENPGPKEY | 61 | 7929 | | OpenPGP key |
| CSYNC | 62 | 7477 | | Child-to-parent sync |
| ZONEMD | 63 | 8976 | | Zone message digest |
| **SVCB** | 64 | 9460 | priority target params | Service binding |
| **HTTPS** | 65 | 9460 | priority target params | HTTPS service binding |
| SPF | 99 | 4408 | text | SPF (deprecated, use TXT) |
| UINFO | 100 | — | | User info |
| UID | 101 | — | | User ID |
| GID | 102 | — | | Group ID |
| UNSPEC | 103 | — | | Unspecified |
| NID | 104 | 6742 | | Node identifier |
| L32 | 105 | 6742 | | 32-bit locator |
| L64 | 106 | 6742 | | 64-bit locator |
| LP | 107 | 6742 | | Locator pointer |
| EUI48 | 108 | 7043 | EUI-48 | MAC-48 address |
| EUI64 | 109 | 7043 | EUI-64 | EUI-64 address |
| TKEY | 249 | 2930 | | Transaction key |
| TSIG | 250 | 2845 | | Transaction signature |
| IXFR | 251 | 1995 | | Incremental zone transfer |
| AXFR | 252 | 5936 | | Full zone transfer |
| MAILB | 253 | 1035 | | Mailbox-related records |
| MAILA | 254 | 1035 | | Mail agent records |
| **ANY** | 255 | 1035 | | Any/all types (deprecated for public use) |
| URI | 256 | 7553 | priority weight target | URI |
| **CAA** | 257 | 8659 | flags tag value | CA authorization |
| AVC | 258 | — | | Application visibility |
| DOA | 259 | — | | Digital object architecture |
| AMTRELAY | 260 | 8777 | | AMT relay |
| RESINFO | 261 | 9606 | | Resolver information |
| TA | 32768 | — | | Trust anchor (unofficial) |
| DLV | 32769 | 4431 | | DNSSEC lookaside validation (historic) |

## Most Commonly Used (Quick Reference)

```
A        IPv4 address
AAAA     IPv6 address
CNAME    Alias (must not be at zone apex)
MX       Mail server (priority + hostname)
NS       Nameserver
PTR      Reverse DNS
SOA      Zone authority (one per zone)
SRV      Service location (_service._proto.name)
TXT      Text (SPF, DKIM, DMARC, domain verification)
CAA      Cert authority restriction
DS       DNSSEC delegation signer (in parent zone)
DNSKEY   DNSSEC public key
RRSIG    DNSSEC signature
NSEC3    DNSSEC non-existence proof
HTTPS    HTTPS service binding (ECH, ALPN, IP hints)
TLSA     DANE TLS certificate association
```
