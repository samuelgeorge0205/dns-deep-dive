# RFC Index — DNS Reference

## Core DNS

| RFC | Title | Key Content |
|-----|-------|-------------|
| **1034** | Domain Names — Concepts and Facilities | Architecture, namespace, resolution algorithm |
| **1035** | Domain Names — Implementation and Specification | Wire format, record types, zone files |
| 1101 | DNS Encoding of Network Names and Other Types | Early network records |
| 1123 | Requirements for Internet Hosts | Clarifies DNS behavior for hosts |
| 1183 | New DNS RR Definitions | AFSDB, RP, X25, ISDN, RT records |
| 1995 | Incremental Zone Transfer (IXFR) | IXFR mechanism |
| 1996 | DNS NOTIFY | Zone change notification |
| **2136** | Dynamic Updates in DNS (DNS UPDATE) | DDNS protocol |
| **2181** | Clarifications to DNS Specification | TTL consistency, CNAME rules, RRset semantics |
| **2308** | Negative Caching of DNS Queries | NXDOMAIN/NODATA caching, SOA MINIMUM |
| 2317 | Classless IN-ADDR.ARPA Delegation | Sub-/24 reverse DNS delegation |
| 2535 | Domain Name System Security Extensions | Early DNSSEC (obsoleted) |
| **2782** | DNS SRV Records | SRV record format and semantics |
| **2845** | Secret Key Transaction Authentication (TSIG) | TSIG for zone transfer auth |
| 3007 | Secure Dynamic DNS Update | DDNS + DNSSEC |
| 3225 | Indicating Resolver Support of DNSSEC | DO bit |
| 3226 | DNSSEC and IPv6 AAAA Queries to DNS | |
| **3596** | DNS Extensions to Support IPv6 | AAAA record |
| 3845 | DNS Security (DNSSEC) NextSECure (NSEC) RR | NSEC record format |

## DNSSEC

| RFC | Title |
|-----|-------|
| **4033** | DNS Security Introduction and Requirements |
| **4034** | Resource Records for DNS Security Extensions |
| **4035** | Protocol Modifications for DNS Security Extensions |
| 4398 | Storing Certificates in DNS |
| 4509 | SHA-256 in DNS Delegation Signer (DS) RRs |
| 5011 | Automated DNSSEC Trust Anchor Updates |
| 5155 | DNS Security (DNSSEC) Hashed Authenticated Denial (NSEC3) |
| 5702 | SHA-2 Algorithms with RSA in DNSKEY and RRSIG |
| 6605 | ECDSA for DNSSEC |
| 6781 | DNSSEC Operational Practices |
| 6840 | Clarifications and Implementation Notes for DNSSEC |
| 7344 | Automating DNSSEC Delegation Trust Maintenance (CDS/CDNSKEY) |
| 8080 | EdDSA for DNSSEC (Ed25519, Ed448) |
| 8198 | Aggressive Use of DNSSEC-Validated Cache |
| 9364 | DNSSEC Resolvers and Validators |

## Privacy and Security

| RFC | Title |
|-----|-------|
| **7858** | DNS over TLS (DoT) |
| **8484** | DNS Queries over HTTPS (DoH) |
| **9230** | Oblivious DNS over HTTPS (ODoH) |
| **9250** | DNS over Dedicated QUIC Connections (DoQ) |
| 7626 | DNS Privacy Considerations |
| 7816 | DNS Query Name Minimisation |
| 7830 | The EDNS(0) Padding Option |
| 7873 | Domain Name System Cookies |
| 8310 | Usage Profiles for DNS over TLS and DNS over DTLS |
| 8932 | Recommendations for DNS Privacy Service Operators |

## Extensions

| RFC | Title |
|-----|-------|
| **6891** | Extension Mechanisms for DNS (EDNS0) |
| 7828 | TCP Keepalive EDNS0 Option |
| **7871** | Client Subnet in DNS Queries (ECS) |
| 5001 | DNS Name Server Identifier (NSID) Option |
| 1317 | Dynamic DNS Update (DDNS) |
| 7766 | DNS Transport over TCP |
| 8499 | DNS Terminology |
| 8767 | Serving Stale Data to Improve DNS Resiliency |

## Special Use and Other

| RFC | Title |
|-----|-------|
| 6761 | Special-Use Domain Names (.local, .localhost, .example, etc.) |
| 6762 | Multicast DNS (mDNS) |
| 6763 | DNS-Based Service Discovery |
| **6698** | DANE / TLSA |
| 7929 | DNS-Based Authentication of Named Entities for OpenPGP |
| 8552 | Scoped Interpretation of DNS Resource Records using SVCB |
| **9460** | HTTPS and SVCB RRs |
| **8659** | DNS Certification Authority Authorization (CAA) |
| **8482** | Providing Minimal-Sized Responses to DNS ANY Queries |
| 7553 | URI Resource Record |
| 1876 | Location Information in DNS (LOC record) |

## Obsoleted / Historic

| RFC | Obsoleted By | Topic |
|-----|-------------|-------|
| 882 | 1034/1035 | Original DNS proposal |
| 883 | 1034/1035 | Original DNS spec |
| 2535 | 4033–4035 | Early DNSSEC |
| 2671 | 6891 | Original EDNS |
| 3755 | 4034 | DNSSEC record types |
