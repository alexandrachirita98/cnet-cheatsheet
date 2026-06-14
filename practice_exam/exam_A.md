# Practice Exam A — Routing

**Lab:** `lab-routing` · **Topology:** host (router) + red / green / blue · **Time:** 60 min · **Total:** 100 pts

## Scenario

The **host** station must route between three stations on separate /24 segments. Boot the topology and complete the tasks. Prove each result (show the command output that confirms it).

```bash
# on host
cd work/ && wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-routing.zip
unzip lab-routing.zip && bash runvm.sh && source ~/.bashrc
```

| Link | Network | host | station |
|------|---------|------|---------|
| host–red | `10.10.10.0/24` | `.1` | `.2` |
| host–green | `10.10.20.0/24` | `.1` | `.2` |
| host–blue | `10.10.30.0/24` | `.1` | `.2` |

---

## Tasks (prompts)

1. **(10 pts)** Configure the **host–red** link (both sides), bring the interfaces up, and prove host can ping red. *[addressing L2/L3]*
2. **(10 pts)** Configure the **host–green** and **host–blue** links. The host's green/blue interfaces are **not** `usernet` — find their real names. *[interface discovery]*
3. **(20 pts)** Make **red, green and blue reach each other**. Add the needed routes on the stations and the needed setting on the host. Prove red↔green and red↔blue. *[routing + forwarding]*
4. **(10 pts)** From red, ping green and blue, then display red's **ARP/neighbor table** and explain the entry states you see. *[ARP]*
5. **(20 pts)** Add **IPv6** on all three links (`2201::/64` red, `2202::/64` blue, `2203::/64` green), enable IPv6 forwarding, add IPv6 default routes on the stations, and prove red can `ping6` green **through** the host. *[IPv6]*
6. **(15 pts)** Make the IPv6 addresses **and** forwarding **survive a reboot**. Reboot and prove they persisted. *[persistence]*
7. **(15 pts) Troubleshooting:** run `start_lab ex6`, then `go red`. Connectivity is broken. Diagnose the cause, fix it, and re-test. State what was wrong. *[troubleshooting]*

---

## Answer key

<details><summary>Task 1 — host–red link</summary>

On **host**:
```bash
ip address add 10.10.10.1/24 dev usernet
ip link set dev usernet up
```
On **red** (`go red`):
```bash
ip address add 10.10.10.2/24 dev red-eth0
ip link set dev red-eth0 up
```
Prove (on **host**):
```bash
ping -c 3 10.10.10.2
```
</details>

<details><summary>Task 2 — green & blue links</summary>

On **host** (discover the interfaces first):
```bash
ip link                                       # spot the green/blue-facing devices
ip address add 10.10.20.1/24 dev green-link
ip link set dev green-link up
ip address add 10.10.30.1/24 dev blue-link
ip link set dev blue-link up
```
On **green**:
```bash
ip address add 10.10.20.2/24 dev green-eth0
ip link set dev green-eth0 up
```
On **blue**:
```bash
ip address add 10.10.30.2/24 dev blue-eth0
ip link set dev blue-eth0 up
```
</details>

<details><summary>Task 3 — routing + forwarding</summary>

On **red**:
```bash
ip route add default via 10.10.10.1
```
On **green**:
```bash
ip route add default via 10.10.20.1
```
On **blue**:
```bash
ip route add default via 10.10.30.1
```
On **host** (the key step — without it nothing routes):
```bash
sysctl -w net.ipv4.ip_forward=1
```
Prove (on **red**):
```bash
ping -c 2 10.10.20.2     # red -> green
ping -c 2 10.10.30.2     # red -> blue
```
</details>

<details><summary>Task 4 — ARP table</summary>

On **red**:
```bash
ping -c 1 10.10.20.2
ping -c 1 10.10.30.2
ip neighbor show
```
Explanation: `REACHABLE` = confirmed recently; `STALE` = cached but not re-confirmed (still valid); `DELAY` = traffic just sent, kernel will probe shortly. Entries cycle REACHABLE → STALE → DELAY → REACHABLE with use.
</details>

<details><summary>Task 5 — IPv6</summary>

On **host**:
```bash
ip -6 address add 2201::1/64 dev usernet
ip -6 address add 2203::1/64 dev green-link
ip -6 address add 2202::1/64 dev blue-link
sysctl -w net.ipv6.conf.all.forwarding=1
```
On **red**:
```bash
ip -6 address add 2201::2/64 dev red-eth0
ip -6 route add default via 2201::1
```
On **green**:
```bash
ip -6 address add 2203::2/64 dev green-eth0
ip -6 route add default via 2203::1
```
On **blue**:
```bash
ip -6 address add 2202::2/64 dev blue-eth0
ip -6 route add default via 2202::1
```
Prove (on **red**):
```bash
ping6 -c 3 2203::2       # red -> green through the host
```
</details>

<details><summary>Task 6 — persistence</summary>

On **host** — netplan `/etc/netplan/01-netcfg.yaml`:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    usernet:    { addresses: [10.10.10.1/24, 2201::1/64] }
    green-link: { addresses: [10.10.20.1/24, 2203::1/64] }
    blue-link:  { addresses: [10.10.30.1/24, 2202::1/64] }
```
```bash
netplan apply
```
On **host** — forwarding `/etc/sysctl.d/99-routing.conf`:
```ini
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```
```bash
sysctl --system
reboot_vms
# after reboot, prove:
ip address show ; ip -6 route show ; sysctl net.ipv6.conf.all.forwarding
```
</details>

<details><summary>Task 7 — troubleshooting (ex6)</summary>

On **red**:
```bash
ip address show          # the address has mask /32 instead of /24 -> no on-link route
ip r s
```
Fix:
```bash
ip address delete 10.10.7.1/32 dev usernet
ip address add 10.10.7.1/24 dev usernet
ping -c 3 10.10.7.2
```
What was wrong: the interface address was configured with a `/32` netmask, so the station had no route to its own subnet.
</details>
