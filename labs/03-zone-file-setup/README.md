# Lab 03 — Zone File Setup with BIND

**Goal:** Create a zone file, serve it with BIND, and query it.

**Prerequisites:** `bind9` installed (`sudo apt install bind9 bind9utils`)

**Time:** 45 minutes

---

## Setup

```bash
# Check BIND is installed
named -v

# Check config syntax tool
named-checkconf --help
named-checkzone --help
```

---

## Exercise 3.1 — Validate the Example Zone File

The repo includes `zone-examples/basic.zone`. Validate it:

```bash
named-checkzone lab.example.com ../../zone-examples/basic.zone
```

Expected output: `zone lab.example.com/IN: loaded serial XXXXXXXXXX` then `OK`.

Fix any errors before proceeding.

---

## Exercise 3.2 — Create Your Own Zone

Create `/tmp/myzone.com.zone` with the following requirements:

- [ ] SOA record with `ns1.myzone.com.` as primary
- [ ] Two NS records
- [ ] A record for the apex (`@`)
- [ ] A records for `www`, `mail`, `ftp`
- [ ] `ftp` should be a CNAME to `www`
- [ ] Two MX records with different priorities
- [ ] A TXT record with `v=spf1 a mx ~all`
- [ ] A wildcard `*` A record
- [ ] A subdomain delegation for `dev.myzone.com.` to fictional NS records

Validate:
```bash
named-checkzone myzone.com /tmp/myzone.com.zone
```

---

## Exercise 3.3 — Serve the Zone Locally

**Warning:** This modifies BIND config. Use a VM or container if possible.

```bash
# Add to /etc/bind/named.conf.local:
zone "myzone.com" {
    type primary;
    file "/tmp/myzone.com.zone";
};

# Check config
named-checkconf

# Reload BIND
sudo rndc reload

# Or restart
sudo systemctl restart named
```

Query your local zone:
```bash
dig @localhost myzone.com A
dig @localhost www.myzone.com A
dig @localhost ftp.myzone.com A         # CNAME resolution
dig @localhost mail.myzone.com A
dig @localhost myzone.com MX
dig @localhost myzone.com SOA +multiline
dig @localhost ghost.myzone.com A       # wildcard
dig @localhost nonexistent.myzone.com A # should NOT match wildcard if name doesn't exist
```

---

## Exercise 3.4 — Update the Zone

Increment the serial and add a new record:

```bash
# Edit the zone file:
# 1. Increment serial (last 2 digits)
# 2. Add: vpn  IN  A  10.0.0.1

# Reload the zone
sudo rndc reload myzone.com

# Verify
dig @localhost vpn.myzone.com A
dig @localhost myzone.com SOA +short   # confirm serial changed
```

---

## Exercise 3.5 — Zone Transfer

Set up a secondary that transfers from your primary:

```bash
# Add to named.conf.local on the SAME machine (simulating secondary):
zone "myzone.com" {
    type secondary;
    primaries { 127.0.0.1; };
    file "/tmp/myzone.com.secondary";
};

# In the primary zone config, allow transfer:
zone "myzone.com" {
    type primary;
    file "/tmp/myzone.com.zone";
    allow-transfer { 127.0.0.1; };  # add this line
};

# Reload and trigger transfer
sudo rndc reload

# Verify secondary received zone
dig @127.0.0.1 -p 5353 myzone.com SOA  # if running on different port
ls -la /tmp/myzone.com.secondary        # zone should be written here
```

---

## Exercise 3.6 — Observe AXFR

```bash
# Attempt zone transfer
dig @localhost myzone.com AXFR

# Note: starts and ends with SOA record
# Count the records in the transfer
```

---

## Cleanup

```bash
# Remove zones from named.conf.local
# Reload BIND
sudo rndc reload
```
