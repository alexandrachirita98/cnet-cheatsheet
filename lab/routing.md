# Routing

This lab turns the **host** station into a router that connects three stations — **red**, **green**, **blue** — each on its own network segment. You configure IPs (L3), bring interfaces up (L2), add routes, enable forwarding, then extend to ARP, IPv6 and persistence.

Every section first explains **what it is about** (the lab text), then gives a **Solution** subchapter with all the commands fully worked out. Each code cell contains commands for **one station only** — the heading above it tells you where to run them.

---

## Lab Setup

### What this is about

Provision a VM on the faculty cloud and start the lab topology. The VM must be named `scgc_lab<no>_<username>`, created from the **SCGC Template** with flavor **g.medium** or larger. You connect as user `student`, then download and boot the lab.

### Solution

On **host**:

```bash
cd work/
wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-routing.zip
unzip lab-routing.zip
bash runvm.sh            # boots the topology (the host/red/green/blue VMs)
source ~/.bashrc         # loads helper commands: go, start_lab, reboot_vms
```

- `bash runvm.sh` starts the four virtual machines that make up the topology.
- `source ~/.bashrc` makes the helper commands available in your shell.
- Enter a station with `go <name>` (e.g. `go red`); leave it with `exit` to return to the host.

---

## Topology

### What this is about

The **host** acts as the router in the middle. Each station hangs off its own /24 segment:

| Link        | Network         | host address | station address |
|-------------|-----------------|--------------|-----------------|
| host–red    | `10.10.10.0/24` | `10.10.10.1` | `10.10.10.2`    |
| host–green  | `10.10.20.0/24` | `10.10.20.1` | `10.10.20.2`    |
| host–blue   | `10.10.30.0/24` | `10.10.30.1` | `10.10.30.2`    |

On the host, the interface toward **red** is `usernet`. The interfaces toward **green** and **blue** have different names — find them with `ip link` (shown below as `green-link` and `blue-link`; substitute the real names you see).

---

## Configuring and deleting IP addresses

### What this is about

Configure Layer 3 addresses on the **host–red** link (`10.10.10.0/24`) using the `iproute2` tools, bring the interfaces up at Layer 2, validate, and learn how to remove addresses.

### Solution

On **host** — give the host its address on the red link and turn the interface on:

```bash
ip address add 10.10.10.1/24 dev usernet   # assign the L3 address (IP + mask) to the interface
ip link set dev usernet up                 # bring the interface up at L2 so it can carry traffic
```

On **red** (`go red`) — give red its address on the same link:

```bash
ip address add 10.10.10.2/24 dev red-eth0  # red's address in the same /24 subnet
ip link set dev red-eth0 up                # activate red's interface
```

On **host** — check the address you set:

```bash
ip address show dev usernet                # confirm 10.10.10.1/24 is present and the link is UP
```

On **red** — check and test reachability to the host:

```bash
ip address show dev red-eth0               # confirm 10.10.10.2/24
ping -c 3 10.10.10.1                        # red reaches host across the link
```

On **host** — how to wipe an interface's addresses when you need a clean slate:

```bash
ip address flush dev usernet               # remove ALL addresses from the interface
```

---

## Configuring IP addresses

### What this is about

Do the same for the **host–green** link on `10.10.20.0/24`. The host's green-facing interface has a different name than `usernet`, so you discover it with `ip link` first, then verify L2 is up before testing.

### Solution

On **host** — find the green interface, then configure it:

```bash
ip link                                     # list interfaces; spot the one toward green (e.g. green-link)
ip address add 10.10.20.1/24 dev green-link # host's address on the green segment
ip link set dev green-link up               # bring it up at L2
```

On **green** (`go green`) — configure green's side:

```bash
ip address add 10.10.20.2/24 dev green-eth0 # green's address in the same /24
ip link set dev green-eth0 up               # activate the interface
```

On **green** — verify the link is up and test reachability to the host:

```bash
ip link                                     # confirm green-eth0 state is UP
ping -c 3 10.10.20.1                         # green reaches host across the link
```

---

## IP Addressing and Routing

### What this is about

So far each station only reaches the host (its direct neighbor). To let red and green talk **through** the host, two things are needed: the stations must know to send foreign traffic to the host (a default route), and the host must be willing to forward packets between interfaces (IP forwarding).

### Solution

On **red** — tell red how to reach the host and where to send everything else:

```bash
ip route add 10.10.10.1 dev red-eth0       # explicit route to the gateway (host) on this interface
ip route add default via 10.10.10.1        # send all non-local traffic to the host
ip route show                              # check the routing table
```

On **green** — same idea, but green's gateway is `10.10.20.1`:

```bash
ip route add 10.10.20.1 dev green-eth0     # route to the gateway (host)
ip route add default via 10.10.20.1        # all other traffic via the host
ip route show                              # check the routing table
```

On **host** — allow the kernel to route packets between interfaces:

```bash
sysctl -w net.ipv4.ip_forward=1            # enable IPv4 forwarding (turns the host into a router)
sysctl net.ipv4.ip_forward                 # validate -> should print 1
```

On **host** — watch traffic pass through while you test (run in a second shell):

```bash
tcpdump -n -i usernet                       # capture packets on the red-facing interface
```

On **red** — generate traffic toward green; you should see it in the host's tcpdump:

```bash
ping -c 3 10.10.20.2                         # red -> green, routed by the host
```

---

## Complete connectivity setup

### What this is about

Add the third segment, **host–blue** on `10.10.30.0/24`, configured the same way as the others. Then verify that every station can reach every other station, proving the host routes between all three networks.

### Solution

On **host** — find the blue interface and configure it:

```bash
ip link                                     # spot the interface toward blue (e.g. blue-link)
ip address add 10.10.30.1/24 dev blue-link  # host's address on the blue segment
ip link set dev blue-link up                # bring it up
```

On **blue** (`go blue`) — configure blue and point its default route at the host:

```bash
ip address add 10.10.30.2/24 dev blue-eth0  # blue's address in the /24
ip link set dev blue-eth0 up                # activate the interface
ip route add 10.10.30.1 dev blue-eth0       # route to the gateway (host)
ip route add default via 10.10.30.1         # host as default gateway
```

Forwarding is already on from the previous section, so test the full matrix.

On **red**:

```bash
ping -c 2 10.10.20.2                         # red -> green
ping -c 2 10.10.30.2                         # red -> blue
```

On **green**:

```bash
ping -c 2 10.10.10.2                         # green -> red
ping -c 2 10.10.30.2                         # green -> blue
```

On **blue**:

```bash
ping -c 2 10.10.10.2                         # blue -> red
ping -c 2 10.10.20.2                         # blue -> green
```

---

## ARP table

### What this is about

ARP maps IP ↔ MAC addresses on a segment so the kernel knows which hardware address to put on a frame. You can view the cache and watch it fill up as you generate traffic.

### Solution

On **host** — view the cache, create traffic, then look again:

```bash
ip neighbor show                            # current ARP/neighbor cache (may be empty/STALE)
ping -c 1 10.10.10.2                         # force resolution of red's MAC
ip neighbor show                            # red now appears, state REACHABLE
```

Repeat the same three steps on **green** and **blue** (pinging their own neighbors) to populate each cache. Entry states cycle naturally over time: `REACHABLE` → `STALE` → (on next use) `DELAY` → `REACHABLE`.

---

## Troubleshoot IP address configuration problem

### What this is about

A pre-broken scenario where connectivity fails because of a wrong netmask: the address was added with `/32` instead of `/24`. With a `/32` the station thinks it is the only host on the link, so it has no on-link route to its segment and can't reach its neighbors.

### Solution

On **host** — load the broken scenario:

```bash
start_lab ex6                               # resets the topology with the injected bug
```

On **red** (`go red`) — diagnose by reading the tables:

```bash
ip r s                                       # routing table (short for: ip route show)
ip address show                              # inspect the netmask -> it is /32, not /24
```

On **red** — fix it: delete the wrong address, add it back with the correct mask:

```bash
ip address delete 10.10.7.1/32 dev usernet   # remove the bad /32 address
ip address add 10.10.7.1/24 dev usernet      # re-add with the correct /24 mask
ip r s                                        # confirm the /24 on-link route now exists
ping -c 3 10.10.7.2                           # re-test connectivity
```

---

## Troubleshooting connectivity problem

### What this is about

A pre-broken scenario where stations can reach the host but not each other. The cause is a routing/forwarding misconfiguration: a missing or wrong default route on a station, or IP forwarding disabled on the host. The method is to read the routing tables first, then localize the break, then fix only what is actually wrong.

> The fix below is an **example** for when **blue** has a wrong default route. Run the diagnosis first — the broken station and the correct gateway may differ. Gateways per station: red → `10.10.10.1`, green → `10.10.20.1`, blue → `10.10.30.1`.

### Solution

On **host** — load the scenario:

```bash
start_lab ex7
```

On **host** — confirm the router itself is forwarding and knows all segments:

```bash
sysctl net.ipv4.ip_forward                  # must be 1; if 0 that's (part of) the bug
ip route show                               # all three /24 segments should be directly connected
```

On **each station** (`go red`, `go green`, `go blue`) — check its default route:

```bash
ip route show                               # is there a default route via the correct gateway?
ping -c 2 10.10.30.1                          # can this station reach its own gateway (host)?
```

On **host** — if forwarding was off, turn it on:

```bash
sysctl -w net.ipv4.ip_forward=1             # re-enable routing on the host
```

On the **broken station** — if its default route was wrong/missing (example shown for blue):

```bash
ip route del default                         # remove the wrong route (only if one exists)
ip route add default via 10.10.30.1          # add the correct gateway for this station
ip route show                                # confirm
```

Then re-test the full matrix from *Complete connectivity setup*.

---

## IPv6

### What this is about

Repeat the addressing and routing using IPv6 (the `-6` flag on `ip`). Each link gets its own /64, the host forwards IPv6 between them, and each station gets an IPv6 default route via the host.

| Link        | IPv6 network | host address | station address |
|-------------|--------------|--------------|-----------------|
| host–red    | `2201::/64`  | `2201::1`    | `2201::2`       |
| host–blue   | `2202::/64`  | `2202::1`    | `2202::2`       |
| host–green  | `2203::/64`  | `2203::1`    | `2203::2`       |

### Solution

On **host** — add an IPv6 address per link and enable IPv6 forwarding:

```bash
ip -6 address add 2201::1/64 dev usernet     # red link
ip -6 address add 2202::1/64 dev blue-link   # blue link
ip -6 address add 2203::1/64 dev green-link  # green link
ip -6 address show                           # validate the three addresses
sysctl -w net.ipv6.conf.all.forwarding=1     # enable IPv6 forwarding
```

On **red** — add its IPv6 address and default route via the host:

```bash
ip -6 address add 2201::2/64 dev red-eth0
ip -6 route add default via 2201::1
```

On **blue**:

```bash
ip -6 address add 2202::2/64 dev blue-eth0
ip -6 route add default via 2202::1
```

On **green**:

```bash
ip -6 address add 2203::2/64 dev green-eth0
ip -6 route add default via 2203::1
```

On **red** — test IPv6 reachability (to the host, then through it to green):

```bash
ping6 -c 3 2201::1                            # red -> host over IPv6
ping6 -c 3 2203::2                            # red -> green, routed by the host
```

---

## Persistent setup

### What this is about

The `ip` and `sysctl -w` commands are lost on reboot. Make the IPv6 configuration and forwarding **permanent** using Ubuntu's network config (netplan) plus a sysctl drop-in file, then confirm everything survives a reboot.

### Solution

On **host** — load the scenario and clear the temporary config:

```bash
start_lab ex9
ip address flush dev usernet                 # remove the runtime addresses; netplan will own them now
```

On **host** — persist addresses with netplan. Edit `/etc/netplan/01-netcfg.yaml` (or the existing file under `/etc/netplan/`) so each interface carries both its IPv4 and IPv6 address:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    usernet:
      addresses:
        - 10.10.10.1/24
        - 2201::1/64
    green-link:
      addresses:
        - 10.10.20.1/24
        - 2203::1/64
    blue-link:
      addresses:
        - 10.10.30.1/24
        - 2202::1/64
```

On **host** — apply and check the netplan config:

```bash
netplan apply                                # apply the YAML; addresses are now persistent
ip address show                              # confirm the addresses are present
```

On **host** — persist forwarding with a sysctl drop-in. Create `/etc/sysctl.d/99-routing.conf`:

```ini
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

On **host** — load all sysctl config files so the setting takes effect now:

```bash
sysctl --system
```

On **host** — verify it survives a reboot:

```bash
reboot_vms                                   # reboot the topology
```

On **host** — after reboot, the config should still be there:

```bash
ip address show                              # addresses persisted
ip route show                                # routes/segments present
sysctl net.ipv4.ip_forward                   # still 1
```
