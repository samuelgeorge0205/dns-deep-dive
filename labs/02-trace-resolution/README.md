# Lab 02 — Tracing Full DNS Resolution

**Goal:** Observe every step of iterative resolution from root to authoritative.

**Time:** 30 minutes

---

## Exercise 2.1 — The +trace Flag

```bash
dig +trace www.github.com A
```

Walk through the output and label each section:
1. Which servers are the **root servers**? (first set of NS records)
2. Which server is the **.com TLD** server?
3. Which server is **authoritative for github.com**?
4. How many total network hops (query/response pairs) were made?
5. Where do you see **glue records**?

---

## Exercise 2.2 — Trace with Additional Section

```bash
dig +trace +additional www.github.com A
```

**Question:** What extra information appears? Why are those records included?

---

## Exercise 2.3 — Compare Trace vs Cached

```bash
# First, query without trace to populate cache
dig www.github.com A

# Now trace — resolver uses its cache for TLD/root, skips some steps
dig +trace www.github.com A
```

**Note:** With `+trace`, dig performs the full resolution itself (bypasses resolver cache). Compare query times between each step.

---

## Exercise 2.4 — Follow a CNAME Chain

```bash
dig +trace www.amazon.com A
```

**Questions:**
1. Does `www.amazon.com` have a CNAME?
2. What does the CNAME point to?
3. How many resolution steps does the CNAME chain add?

---

## Exercise 2.5 — Observe a Referral Packet

```bash
# Query the root directly — you'll get a referral
dig @a.root-servers.net www.github.com A
```

Identify:
- [ ] AA bit (is it set?)
- [ ] ANCOUNT (how many answers?)
- [ ] NSCOUNT (how many authority RRs?)
- [ ] The NS records in the authority section
- [ ] The glue A records in the additional section

---

## Exercise 2.6 — Compare Root Server Responses

```bash
# The root doesn't know the answer — it refers you to TLD
dig @a.root-servers.net google.com A
dig @a.root-servers.net bbc.co.uk A   # two-level TLD
```

**Question:** For `bbc.co.uk`, does the root refer to `.uk` servers or `.co.uk` servers directly?

---

## Exercise 2.7 — Time Each Step

Manually replicate what `+trace` does:

```bash
# Step 1: Query root for www.example.com
dig @198.41.0.4 www.example.com A       # a.root-servers.net

# Note the TLD server in the referral, then:
# Step 2: Query .com TLD server
dig @192.5.6.30 www.example.com A       # a.gtld-servers.net

# Note the authoritative NS, then:
# Step 3: Query authoritative
dig @<NS from step 2> www.example.com A
```

**Record:** Query time for each step. Which step is slowest? Why?
