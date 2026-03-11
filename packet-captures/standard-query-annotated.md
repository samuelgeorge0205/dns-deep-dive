# Standard DNS Query — Annotated Packet Breakdown

This document annotates the bytes of a standard DNS query and response for:
**`www.example.com A IN`** queried against `8.8.8.8`

---

## Query Packet (43 bytes)

```
Offset  Hex                          Decoded
──────────────────────────────────────────────────────────────────────
00      45 00 00 43                  IPv4 header (IHL=5, TOS=0, len=67)
04      AB CD 40 00                  ID=0xABCD, DF bit set, frag=0
08      40 11 xx xx                  TTL=64, proto=17(UDP), checksum
0C      C0 A8 01 0A                  src=192.168.1.10
10      08 08 08 08                  dst=8.8.8.8

── UDP Header ────────────────────────────────────────────────────────
14      CC B5                        src port = 52405 (random ephemeral)
16      00 35                        dst port = 53
18      00 2F                        UDP length = 47 bytes
1A      xx xx                        UDP checksum

── DNS Header (12 bytes) ─────────────────────────────────────────────
1C      1A 2B                        Transaction ID = 0x1A2B
1E      01 20                        Flags: 0x0120
                                       QR=0 (query)
                                       Opcode=0 (QUERY)
                                       AA=0, TC=0
                                       RD=1 (recursion desired)
                                       RA=0, Z=0, AD=0, CD=0
                                       RCODE=0
20      00 01                        QDCOUNT = 1
22      00 00                        ANCOUNT = 0
24      00 00                        NSCOUNT = 0
26      00 01                        ARCOUNT = 1 (OPT record)

── Question Section ──────────────────────────────────────────────────
28      03                           label length = 3
29      77 77 77                     "www"
2C      07                           label length = 7
2D      65 78 61 6D 70 6C 65        "example"
34      03                           label length = 3
35      63 6F 6D                     "com"
38      00                           root terminator (end of name)
39      00 01                        QTYPE = A (1)
3B      00 01                        QCLASS = IN (1)

── Additional Section: OPT (EDNS0) ───────────────────────────────────
3D      00                           NAME = . (root, empty)
3E      00 29                        TYPE = OPT (41)
40      10 00                        CLASS = 4096 (max UDP payload size)
42      00 00 80 00                  TTL:
                                       extended RCODE = 0
                                       EDNS version = 0
                                       DO bit = 1 (DNSSEC OK)
                                       Z = 0
46      00 00                        RDLENGTH = 0 (no EDNS options)
```

---

## Response Packet

```
── DNS Header ────────────────────────────────────────────────────────
Offset  Hex        Decoded
1C      1A 2B      Transaction ID = 0x1A2B  ← matches query
1E      81 80      Flags: 0x8180
                     QR=1 (response)
                     Opcode=0
                     AA=0 (not authoritative — from cache/recursive)
                     TC=0
                     RD=1 (recursion desired, echoed from query)
                     RA=1 (recursion available — server supports it)
                     Z=0, AD=0, CD=0
                     RCODE=0 (NOERROR)
20      00 01      QDCOUNT = 1
22      00 01      ANCOUNT = 1  ← one answer
24      00 00      NSCOUNT = 0
26      00 01      ARCOUNT = 1  (OPT)

── Question Section (echoed) ─────────────────────────────────────────
(same as query question section)

── Answer Section ────────────────────────────────────────────────────
3D      C0 0C      NAME: pointer → offset 0x0C = www.example.com. (compression!)
3F      00 01      TYPE = A (1)
41      00 01      CLASS = IN (1)
43      00 00 01 2C  TTL = 300 seconds
47      00 04      RDLENGTH = 4
49      5D B8 D8 22  RDATA = 93.184.216.34
```

### Name Compression Explained

```
C0 0C  →  0xC0 = 11000000 (two high bits set = pointer indicator)
           0x0C = offset 12 from start of DNS message
           At offset 12: 03 77 77 77 07 65 78 61 6d 70 6c 65 03 63 6f 6d 00
           = www.example.com.

Instead of repeating 17 bytes, the pointer uses only 2 bytes.
```

---

## Timing Analysis

| Event | Time |
|-------|------|
| Query sent | T+0ms |
| Response received | T+12ms |
| Round-trip time | 12ms |

For a cache hit, this would be ~1ms (local resolver).
For a cold cache (full resolution), expect 50–200ms.
