# Firewall — iptables (Security)

This lab uses **iptables** on the **host** (acting as a router between **red** and **green**) to filter traffic: you observe which services send credentials in cleartext, then block and selectively allow services with firewall rules.

Every section first explains **what it is about**, then gives a **Solution** subchapter with all the commands. Each code cell contains commands for **one station only** — the heading above it tells you where to run them.

---

## Lab Setup

### What this is about

Provision a VM (`scgc_lab<no>_<username>`, **SCGC Template**, flavor **g.medium**+, user `student`), download the lab and boot the topology.

### Solution

On **host**:

```bash
cd work/
wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-iptables-security.zip
unzip lab-iptables-security.zip
bash runvm.sh            # boots the host/red/green topology
source ~/.bashrc         # loads helpers: go, start_lab, prepare_lab
prepare_lab              # final setup of the scenario
```

Enter a station with `go <name>` (e.g. `go red`); leave it with `exit`.

---

## Topology

### What this is about

The **host** is the router in the middle; **red** and **green** sit on either side. All traffic between red and green is **routed (FORWARD)** through the host, which is exactly where iptables filters it.

```
red  <--->  host (router)  <--->  green
```

- The host's interface toward red is `usernet`.
- Because traffic between stations is *routed*, you filter it in the **FORWARD** chain on the host.

---

## Encrypted and unencrypted traffic

### What this is about

Demonstrate that **telnet (TCP/23)** and **FTP (TCP/21)** send usernames and passwords in **cleartext**, while **SSH (TCP/22)** encrypts everything. You capture the traffic on the host and watch the credentials appear (or not).

### Solution

On **host** — start a capture on the red-facing interface (leave it running in one shell):

```bash
tcpdump -vvv -A -i usernet      # -A prints packet payload as ASCII so you can read cleartext
```

On **red** — connect to green over telnet and log in; watch the host's tcpdump:

```bash
telnet green                    # credentials appear in cleartext in the capture
```

On **red** — same with FTP:

```bash
ftp green                       # username/password again visible in cleartext
```

On **red** — now use SSH and compare:

```bash
ssh -l student green            # payload is encrypted; no readable credentials in the capture
```

---

## Blocking unencrypted services

### What this is about

Learn iptables basics and block telnet (and FTP) to green. iptables inspects packets and applies a verdict: `ACCEPT`, `REJECT` (refuse and notify the sender) or `DROP` (silently discard). Because traffic to green is routed, the rules go in the **FORWARD** chain.

Syntax reminder: `-A` append rule · `FORWARD` chain for routed packets · `-d` destination · `-p` protocol · `--dport` destination port · `-j` jump/verdict.

### Solution

On **host** — block telnet destined for green:

```bash
iptables -A FORWARD -d green -p tcp --dport telnet -j REJECT   # refuse telnet (port 23) to green
```

On **host** — inspect the chain (three increasingly detailed views):

```bash
iptables -L FORWARD             # list rules in the FORWARD chain
iptables -L FORWARD -v          # verbose: also show packet/byte counters
iptables -L FORWARD -v -n       # -n: numeric (don't resolve names/ports — faster, clearer)
```

On **red** — confirm telnet is now refused:

```bash
telnet green                    # connection rejected
```

On **host** — also block FTP to green:

```bash
iptables -A FORWARD -d green -p tcp --dport 21 -j REJECT       # refuse FTP to green
```

---

## Block SSH

### What this is about

Add a rule that blocks **SSH (port 22)** to green as well, then verify it took effect.

### Solution

On **host** — block SSH to green and check the counters:

```bash
iptables -A FORWARD -d green -p tcp --dport 22 -j REJECT       # refuse SSH to green
iptables -L FORWARD -n -v                                      # verify the rule is present
```

On **red** — confirm SSH is refused:

```bash
ssh -l student green            # connection rejected
```

---

## Allow SSH traffic

### What this is about

Demonstrates **rule ordering**: iptables evaluates rules **top to bottom** and stops at the first match. If the REJECT for SSH comes first, a later ACCEPT never runs. The fix is to **insert** (`-I`) the ACCEPT *above* the REJECT instead of appending it.

### Solution

On **host** — first show why appending an ACCEPT fails (the earlier REJECT wins):

```bash
iptables -A FORWARD -d green -p tcp --dport 22 -j ACCEPT       # appended LAST -> never reached
iptables -L FORWARD -n -v                                      # ACCEPT sits below the REJECT
```

On **host** — fix it by inserting the ACCEPT at position 1 (before the block):

```bash
iptables -I FORWARD 1 -d green -p tcp --dport 22 -j ACCEPT     # insert at top so it matches first
iptables -L FORWARD -n -v                                      # ACCEPT now precedes the REJECT
```

On **red** — SSH works again:

```bash
ssh -l student green
```

On **host** — to remove a specific rule (instead of flushing all), use `-D` with the same rule spec:

```bash
iptables -D FORWARD -d green -p tcp --dport 22 -j ACCEPT       # delete that one rule
```

---

## Deleting added rules

### What this is about

Clear all the rules you added so the stations communicate normally again. `-F` flushes (empties) a whole chain.

### Solution

On **host** — flush the FORWARD chain and verify it's empty:

```bash
iptables -F FORWARD             # remove all rules from FORWARD
iptables -L FORWARD -n -v       # chain is now empty
```

On **red** — confirm services work again:

```bash
telnet green                    # works
ssh -l student green            # works
```

---

## Traffic Captures

### What this is about

Capture traffic to a file with `tcpdump`, copy it off the host, and open it in Wireshark for offline analysis. Useful flags: `-i` interface · `-v` verbosity · `-w` write to file (instead of printing).

### Solution

On **host** — capture to a file (stop with Ctrl-C when done):

```bash
tcpdump -i usernet -w capture.pcap        # write raw packets to capture.pcap
```

On your **local machine** — copy the file off the host with scp, then open it:

```bash
scp student@<host-ip>:~/work/capture.pcap .   # download the capture
wireshark capture.pcap                        # analyze offline
```
