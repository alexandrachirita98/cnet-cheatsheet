# NAT — iptables

This lab uses the **nat** table of iptables on the **host** (gateway) to (1) let private stations reach the Internet via **MASQUERADE** (source NAT) and (2) expose internal services to the outside via **port forwarding** (DNAT).

Every section first explains **what it is about**, then gives a **Solution** subchapter with all the commands. Each code cell contains commands for **one station only**.

---

## Lab Setup

### What this is about

Provision a VM (`scgc_lab<no>_<username>`, **SCGC Template**, flavor **g.medium**+, user `student`), download the lab and boot the topology.

### Solution

On **host**:

```bash
cd work/
wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-iptables-nat.zip
unzip lab-iptables-nat.zip
bash runvm.sh            # boots the host/red/green/blue topology
source ~/.bashrc         # loads helpers: go, start_lab, prepare_lab
prepare_lab              # final setup of the scenario
```

---

## Topology

### What this is about

The **host** is the gateway between the **private** internal network (stations red/green/blue, e.g. `10.10.1.0/24`) and the **outside** world.

```
red / green / blue  <--usernet-->  host (gateway)  <--eth0-->  Internet / fep
```

- `usernet` = host's **internal** interface (toward the stations).
- `eth0` = host's **external** interface (toward the Internet / fep.grid.pub.ro).
- Private addresses can't be routed on the Internet, so the host must **translate** them.

---

## MASQUERADE NAT

### What this is about

Private ranges (e.g. `192.168.0.0/24`, `10.x`) are not routable on the Internet. **MASQUERADE** is source NAT: as packets leave the external interface, the host rewrites their **source IP/port** to its own public address, so replies can come back. The rule goes in **POSTROUTING** (applied just before a packet leaves). NAT also needs **IP forwarding** enabled, or the host won't route at all.

### Solution

On **host** — enable masquerading and verify the rule:

```bash
iptables -t nat -A POSTROUTING -j MASQUERADE   # rewrite source addr of outgoing packets to host's IP
iptables -t nat -L POSTROUTING -n -v           # confirm the rule (and watch its counters)
```

On **red** — first test fails because forwarding is off:

```bash
ping -c 2 8.8.8.8              # fails: host isn't forwarding yet
```

On **host** — check and enable IPv4 forwarding:

```bash
sysctl net.ipv4.ip_forward    # likely prints 0
sysctl -w net.ipv4.ip_forward=1   # turn the host into a router
```

On **red** — now connectivity works:

```bash
ping -c 2 8.8.8.8             # success, traffic is masqueraded by the host
```

On **host** — watch the POSTROUTING counters increase as traffic flows:

```bash
iptables -t nat -L POSTROUTING -n -v
```

---

## Observing NATed packets

### What this is about

Use `tcpdump` on different interfaces to *see* the translation happen: on the **internal** interface the source is the station; on the **external** interface the source has been rewritten to the host.

### Solution

On **red** — generate steady traffic (leave it running):

```bash
ping 8.8.8.8
```

On **host** — capture on the external interface (source shows as the host = translated):

```bash
tcpdump -n -i eth0 ip dst host 8.8.8.8        # outbound packets, source already NATed to host
```

On **host** — capture on the internal interface (source still the original station):

```bash
tcpdump -n -i usernet ip dst host 8.8.8.8     # same packets pre-translation, source = red
```

On **host** — capture on all interfaces at once to see both sides of the translation:

```bash
tcpdump -n -i any ip dst host 8.8.8.8         # request leaving
tcpdump -n -i any ip src host 8.8.8.8         # replies coming back
```

Repeat the same observation driving traffic from **green** instead of red.

---

## Observing NATed packets (TCP)

### What this is about

Same idea for **TCP**, where NAT may also rewrite the **source port** (it keeps the original port when there's no conflict). Here you watch an HTTP request.

### Solution

On **host** — capture HTTP traffic toward the web server (leave it running):

```bash
tcpdump -n -i any ip dst host cs.pub.ro and tcp dst port 80
```

On **red** — generate an HTTP request:

```bash
wget http://cs.pub.ro         # triggers a TCP/80 connection through the NAT
```

Observe in the host capture that the source IP is translated; the source port is usually preserved.

---

## Port forwarding

### What this is about

The reverse direction: let an **outside** client reach an **internal** station. You map a port on the gateway to a station's service using **DNAT** in the **PREROUTING** chain (applied as a packet arrives). First you'll see the rule is too broad (internal hosts can hit it too), then restrict it to the external interface with `-i eth0`.

Goal: reach red's SSH (`10.10.1.2:22`) from the Internet via the gateway's port **10022**.

### Solution

On **host** — add the port-forward rule and verify it:

```bash
iptables -t nat -A PREROUTING -p tcp --dport 10022 -j DNAT --to-destination 10.10.1.2:22
iptables -t nat -L PREROUTING -n -v           # confirm the DNAT rule
```

On an **external host** (e.g. fep.grid.pub.ro) — connect through the forwarded port:

```bash
ssh -l student 10.9.X.Y -p 10022              # 10.9.X.Y = the gateway's external IP
```

On **host** — restrict the rule to the external interface so internal hosts can't use it:

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 10022 -j DNAT --to-destination 10.10.1.2:22
```

(Use `iptables -t nat -D PREROUTING ...` to delete the earlier, unrestricted rule.)

---

## Port forwarding again

### What this is about

Extend port forwarding to multiple stations by mapping a distinct gateway port to each station's SSH port. Replace `<green-ip>` / `<blue-ip>` with the stations' real addresses (check with `ip a` on each).

### Solution

On **host** — forward 20022 → green:22 and 30022 → blue:22:

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20022 -j DNAT --to-destination <green-ip>:22
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 30022 -j DNAT --to-destination <blue-ip>:22
```

On an **external host** (fep.grid.pub.ro) — reach each station:

```bash
ssh -l student 10.9.X.Y -p 20022      # -> green
ssh -l student 10.9.X.Y -p 30022      # -> blue
```

---

## Observing NATed packets (Port Forwarding)

### What this is about

Watch DNAT rewrite the **destination** pair `<gateway-ip, 10022>` into `<10.10.1.2, 22>` by capturing on both interfaces.

### Solution

On **host** — capture incoming connections on the external interface:

```bash
tcpdump -n -i eth0 tcp dst port 10022         # arrives addressed to the gateway:10022
```

On **host** — in a second shell, capture the forwarded traffic internally:

```bash
tcpdump -n -i usernet tcp dst port 22         # same flow, now addressed to red:22
```

On an **external host** (fep.grid.pub.ro) — generate the traffic:

```bash
ssh -l student 10.9.X.Y -p 10022
```

---

## Telnet port forwarding

### What this is about

Same DNAT pattern for **telnet (port 23)**: map a gateway port per station to its telnet service. Adapt the station IPs.

### Solution

On **host** — forward 10023/20023/30023 → red/green/blue telnet:

```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 10023 -j DNAT --to-destination <red-ip>:23
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 20023 -j DNAT --to-destination <green-ip>:23
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 30023 -j DNAT --to-destination <blue-ip>:23
```

On an **external host** (fep.grid.pub.ro) — connect (example for red):

```bash
telnet 10.9.X.Y 10023
```
