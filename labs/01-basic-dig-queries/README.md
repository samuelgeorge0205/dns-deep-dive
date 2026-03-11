# Lab 01 — Basic dig Queries

**Goal:** Get comfortable reading DNS responses and understanding all sections of dig output.

**Prerequisites:** `dig` installed (`sudo apt install dnsutils`)

**Time:** 20 minutes

---

## Setup

No server required. All queries go to public DNS.

---

## Exercises

### Exercise 1.1 — Your First Query

```bash
dig example.com A
```

**Questions:**
1. What is the IP address returned?
2. What is the TTL?
3. What resolver answered (look at `SERVER:` line)?
4. What is the query time in milliseconds?
5. Run it again. Did the TTL change? Why?

---

### Exercise 1.2 — Short Output

```bash
dig +short example.com A
dig +short example.com AAAA
dig +short example.com MX
dig +short example.com NS
dig +short example.com TXT
```

**Task:** For each type, describe what the answer means.

---

### Exercise 1.3 — Reading the Full Response

```bash
dig example.com MX
```

Identify in the output:
- [ ] Transaction ID
- [ ] QR, RD, RA flags
- [ ] QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT
- [ ] The priority value in the MX answer
- [ ] Query time and server

---

### Exercise 1.4 — EDNS0

```bash
# With EDNS0 (default)
dig example.com A

# Without EDNS0
dig +noedns example.com A
```

**Questions:**
1. What appears in the `OPT PSEUDOSECTION` with EDNS0?
2. What UDP payload size is advertised?
3. What changes in the response size?

---

### Exercise 1.5 — Querying Different Servers

```bash
# Google
dig @8.8.8.8 example.com A

# Cloudflare
dig @1.1.1.1 example.com A

# Quad9
dig @9.9.9.9 example.com A

# A root server
dig @a.root-servers.net example.com A
```

**Questions:**
1. How do the answers from Google and Cloudflare compare?
2. What does the root server return for `example.com A`? Is it an answer or a referral?
3. Look at the `flags:` line for the root server response — is `aa` set?

---

### Exercise 1.6 — Different Record Types

```bash
# SOA record
dig example.com SOA +multiline

# TXT records
dig google.com TXT +short

# CAA records
dig google.com CAA +short

# SRV (Jabber/XMPP)
dig _xmpp-client._tcp.jabber.org SRV
```

For the SOA: identify SERIAL, REFRESH, RETRY, EXPIRE, MINIMUM.

---

### Exercise 1.7 — Reverse DNS

```bash
# Reverse lookup
dig -x 8.8.8.8 +short
dig -x 1.1.1.1 +short
dig -x 9.9.9.9 +short

# Find your own IP's PTR
dig +short myip.opendns.com @resolver1.opendns.com
# Then reverse that IP:
dig -x <your-ip> +short
```

---

### Exercise 1.8 — NXDOMAIN and NODATA

```bash
# NXDOMAIN — name doesn't exist
dig definitely-does-not-exist.example.com A

# NODATA — name exists, but not this type
dig example.com AAAA   # example.com has no AAAA
```

**Questions:**
1. What RCODE do you get for each?
2. Both have empty answer sections — how do you tell them apart?
3. What record is in the Authority section? Why?
4. What TTL will the negative response be cached for?

---

## Expected Answers

<details>
<summary>Click to reveal hints</summary>

- Exercise 1.5: Root server returns a **referral** (AA=0, empty answer, authority section has NS records for .com)
- Exercise 1.8: NXDOMAIN has RCODE=3; NODATA has RCODE=0 (NOERROR) with empty answer. Both include SOA in authority for negative caching TTL.

</details>
