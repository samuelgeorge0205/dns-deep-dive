# Lab 05 — Cache Poisoning Demo

**Goal:** Understand cache poisoning by simulating it in a controlled lab environment.

**Prerequisites:** Python 3, Scapy (`pip3 install scapy`), root/sudo, isolated network or VM

**⚠️ WARNING:** Only perform this on isolated lab networks you own. Never on production networks.

**Time:** 60 minutes

---

## Background

Cache poisoning works by:
1. Sending a query to a resolver
2. Racing to send a forged response before the real one arrives
3. The forged response must match: source IP, destination IP, Transaction ID, Question section

---

## Exercise 5.1 — Observe Transaction IDs

```bash
# Run Wireshark/tcpdump to capture DNS traffic
sudo tcpdump -i lo -n 'udp port 53' -X

# In another terminal, make DNS queries
dig @127.0.0.1 example.com A
dig @127.0.0.1 google.com A

# Observe: are Transaction IDs random?
# Are source ports randomized? (should be since 2008)
```

---

## Exercise 5.2 — Count the Entropy

```bash
# Make 20 queries and capture Transaction IDs
for i in $(seq 1 20); do
    dig +short @8.8.8.8 test$i.example.com A 2>/dev/null
done

# Capture and analyze IDs with tcpdump
sudo tcpdump -i any -n 'udp and dst port 53' -c 20 2>/dev/null | \
    grep -oP 'id \K[0-9]+'

# Are the IDs truly random?
# With source port randomization: ~32 bits of entropy = 4 billion combinations
```

---

## Exercise 5.3 — Theoretical Poisoning Probability

**Calculate:** Given:
- 16-bit Transaction ID: 65,536 possibilities
- 16-bit source port: ~55,000 ephemeral ports (~16 bits)
- Total: ~32 bits = ~4 billion combinations

With a 100Mbps link and 100-byte response packets:
```
Packets per second = 100,000,000 / (100 * 8) = 125,000 packets/second
Expected guesses needed = 4,294,967,296 / 2 = 2,147,483,648
Expected time = 2,147,483,648 / 125,000 ≈ 17,180 seconds ≈ 4.7 hours
```

Compare to pre-2008 (only 16-bit TX ID):
```
Expected guesses = 32,768
Time = 32,768 / 125,000 ≈ 0.26 seconds
```

---

## Exercise 5.4 — DNS Spoofing in Isolated Lab

**Requires:** Two machines or network namespaces (victim resolver + attacker).

```bash
# Create isolated network namespaces (no internet access)
sudo ip netns add victim
sudo ip netns add attacker

# This is a simplified demo — see full guide in exercises.md
# The key concept: attacker floods forged responses with random TX IDs
# until one matches

# Python script that demonstrates the concept (safe, offline):
python3 - << 'EOF'
import random
import time

def simulate_poisoning():
    # Real TX ID the resolver is waiting on
    real_tx_id = random.randint(0, 65535)
    real_src_port = random.randint(1024, 65535)
    
    attempts = 0
    start = time.time()
    
    # Attacker floods guesses
    while True:
        attempts += 1
        guess_tx = random.randint(0, 65535)
        guess_port = random.randint(1024, 65535)
        
        if guess_tx == real_tx_id and guess_port == real_src_port:
            elapsed = time.time() - start
            print(f"Poisoned after {attempts:,} attempts ({elapsed:.2f}s)")
            print(f"TX ID: {real_tx_id}, Port: {real_src_port}")
            break
        
        # Simulate realistic speed
        if attempts % 100000 == 0:
            print(f"  ... {attempts:,} attempts, {time.time()-start:.1f}s elapsed")
        
        if attempts > 5000000:  # stop after 5M for demo
            print(f"Stopped at {attempts:,} attempts")
            break

simulate_poisoning()
EOF
```

---

## Exercise 5.5 — Verify Mitigations Work

```bash
# Test 1: Source port randomization
sudo tcpdump -i any -n 'udp and dst port 53' -c 50 &
for i in $(seq 1 50); do dig +short example.com A @8.8.8.8 &>/dev/null; done
# Check: source ports should vary widely

# Test 2: Verify your resolver uses DNSSEC validation
dig +dnssec cloudflare.com A | grep -c "ad"   # should be 1
dig +dnssec cloudflare.com A | grep "flags"   # should include "ad"

# Test 3: Try to poison Unbound (it should reject)
# Unbound rejects responses that don't match source port
```

---

## Key Takeaways

1. Pre-2008: 16-bit TX ID = poisonable in seconds
2. Source port randomization: adds ~16 bits = ~4 billion combinations
3. DNSSEC: forged responses fail cryptographic validation regardless of TX ID
4. Real protection = DNSSEC (not just randomization)
