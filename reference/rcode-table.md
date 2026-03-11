# RCODE Reference Table

## Standard RCODEs (4-bit, in header)

| Code | Name | Meaning | Common Causes |
|------|------|---------|---------------|
| 0 | **NOERROR** | Success | Answer found (or NODATA if answer section empty) |
| 1 | **FORMERR** | Format Error | Malformed query, unknown EDNS version |
| 2 | **SERVFAIL** | Server Failure | DNSSEC validation failed, upstream timeout, config error |
| 3 | **NXDOMAIN** | Non-Existent Domain | Name does not exist in DNS |
| 4 | **NOTIMP** | Not Implemented | Opcode or feature not supported |
| 5 | **REFUSED** | Query Refused | ACL blocks query; open recursion disabled |
| 6 | YXDOMAIN | Name Should Not Exist | DNS UPDATE: name exists but shouldn't |
| 7 | YXRRSET | RRset Should Not Exist | DNS UPDATE: RRset exists but shouldn't |
| 8 | NXRRSET | RRset Should Exist | DNS UPDATE: RRset doesn't exist but should |
| 9 | NOTAUTH | Not Authoritative | DNS UPDATE: server not auth for zone |
| 10 | NOTZONE | Name Not In Zone | DNS UPDATE: name not in specified zone |
| 11–15 | (unassigned) | | |

## Extended RCODEs (via EDNS0 OPT TTL, 12-bit total)

| Code | Name | Meaning |
|------|------|---------|
| 16 | BADSIG / BADVERS | TSIG signature failed, or EDNS version not supported |
| 17 | BADKEY | Key not recognized (TSIG) |
| 18 | BADTIME | Timestamp out of range (TSIG) |
| 19 | BADMODE | Bad TKEY mode |
| 20 | BADNAME | Duplicate key name |
| 21 | BADALG | Algorithm not supported |
| 22 | BADTRUNC | Bad truncation (TSIG) |
| 23 | BADCOOKIE | Bad/missing DNS Cookie (RFC 7873) |

## NOERROR vs NXDOMAIN — Critical Distinction

```
NXDOMAIN (RCODE=3):  The NAME does not exist at all.
  dig ghost.example.com A → NXDOMAIN

NODATA (RCODE=0, ANCOUNT=0): The NAME exists, but no records of that TYPE.
  dig example.com AAAA → NOERROR (if only A record exists)

Both cache using SOA MINIMUM as the negative TTL.
```

## Troubleshooting by RCODE

```bash
# SERVFAIL — check DNSSEC
dig +cd example.com A   # disable DNSSEC checking
# If answer returns with +cd but not without → DNSSEC failure

# REFUSED — check ACLs
# Server is configured to refuse recursion for your IP

# FORMERR — check EDNS
dig +noedns example.com  # disable EDNS0
# If works without EDNS → server has EDNS bug

# NXDOMAIN — verify name
dig example.com SOA     # check if zone exists at all
dig @authoritative example.com A  # query authoritative directly
```
