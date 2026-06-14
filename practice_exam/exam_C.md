# Practice Exam C — Firewall & NAT (iptables)

**Labs:** `lab-iptables-security` + `lab-iptables-nat` · **Topology:** host (router/gateway) + red/green/blue · **Time:** 75 min · **Total:** 100 pts

## Scenario

The **host** is a router/gateway. Part 1 filters traffic between stations; Part 2 translates addresses to/from the outside. Prove each result with a capture or a connection test.

```bash
# on host (use the archive for the part you are working on)
cd work/ && wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-iptables-security.zip
unzip lab-iptables-security.zip && bash runvm.sh && source ~/.bashrc && prepare_lab
```

`usernet` = internal interface (toward stations); `eth0` = external interface (toward Internet/fep).

---

## Tasks (prompts)

### Part 1 — Firewall (security)

1. **(10 pts)** Prove that **telnet to green leaks credentials in cleartext** but **SSH does not**, using a single capture on the host. *[encrypted vs unencrypted]*
2. **(15 pts)** Block **telnet and FTP** to green (routed traffic). Prove telnet is now refused. Show the rules with counters. *[FORWARD filtering]*
3. **(15 pts)** Block **SSH to green**, then make SSH work again **without removing the block on telnet/FTP** and **without flushing**. Explain why a plain `-A ... ACCEPT` would not work. *[rule ordering]*
4. **(5 pts)** Flush all firewall rules and confirm services work again. *[flush]*

### Part 2 — NAT

5. **(15 pts)** Give red Internet access via **MASQUERADE**. Ping `8.8.8.8` from red — if it fails, diagnose and fix. *[MASQUERADE + forwarding]*
6. **(15 pts)** Prove the translation: capture so you see red's **original** source on the internal interface and the **translated** source on the external interface. *[observing NAT]*
7. **(15 pts)** Port-forward the gateway's port **10022** to red's SSH (`:22`), accepting **only from the external interface**. Test from an external host. *[DNAT port forwarding]*
8. **(10 pts)** Add forwarding for green (**20022**) and blue (**30022**) to their SSH ports. *[multi-host DNAT]*

---

## Answer key

<details><summary>Task 1 — cleartext vs encrypted</summary>

On **host** (leave running):
```bash
tcpdump -vvv -A -i usernet
```
On **red**:
```bash
telnet green            # username/password visible in the capture
ssh -l student green    # payload encrypted, nothing readable
```
</details>

<details><summary>Task 2 — block telnet & FTP</summary>

On **host**:
```bash
iptables -A FORWARD -d green -p tcp --dport telnet -j REJECT
iptables -A FORWARD -d green -p tcp --dport 21 -j REJECT
iptables -L FORWARD -v -n
```
On **red**: `telnet green` → refused.
</details>

<details><summary>Task 3 — block then allow SSH (ordering)</summary>

On **host**:
```bash
iptables -A FORWARD -d green -p tcp --dport 22 -j REJECT     # block
iptables -I FORWARD 1 -d green -p tcp --dport 22 -j ACCEPT   # insert ABOVE the block
iptables -L FORWARD -v -n
```
Why `-A ... ACCEPT` fails: iptables evaluates top-to-bottom and stops at the first match; an appended ACCEPT sits *below* the REJECT, so it's never reached. `-I FORWARD 1` puts it first.
</details>

<details><summary>Task 4 — flush</summary>

On **host**:
```bash
iptables -F FORWARD
iptables -L FORWARD -v -n
```
On **red**: telnet/ssh to green work again.
</details>

<details><summary>Task 5 — MASQUERADE</summary>

On **host**:
```bash
iptables -t nat -A POSTROUTING -j MASQUERADE
```
On **red**:
```bash
ping -c 2 8.8.8.8        # fails if forwarding is off
```
On **host** (fix):
```bash
sysctl net.ipv4.ip_forward      # 0?
sysctl -w net.ipv4.ip_forward=1
```
On **red**: `ping -c 2 8.8.8.8` → success.
</details>

<details><summary>Task 6 — observe NAT</summary>

On **red** (leave running): `ping 8.8.8.8`

On **host**, two captures:
```bash
tcpdump -n -i usernet ip dst host 8.8.8.8    # source = red (original)
tcpdump -n -i eth0    ip dst host 8.8.8.8    # source = host (translated)
```
</details>

<details><summary>Task 7 — port forward to red:22</summary>

On **host**:
```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 10022 -j DNAT --to-destination 10.10.1.2:22
iptables -t nat -L PREROUTING -n -v
```
From an **external host** (fep.grid.pub.ro):
```bash
ssh -l student 10.9.X.Y -p 10022     # 10.9.X.Y = gateway external IP
```
`-i eth0` ensures internal hosts can't use the forward.
</details>

<details><summary>Task 8 — green & blue forwards</summary>

On **host**:
```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20022 -j DNAT --to-destination <green-ip>:22
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 30022 -j DNAT --to-destination <blue-ip>:22
```
Test from fep: `ssh -l student 10.9.X.Y -p 20022` / `-p 30022`.
</details>
