# Routing

This lab turns the **host** station into a router that connects three stations — **red**, **green**, **blue** — each on its own network segment. You configure IPs (L3), bring interfaces up (L2), add routes, enable forwarding, then extend to ARP, IPv6 and persistence.

---

## Lab Setup

On the cloud:

1. Create a VM named `scgc_lab<no>_<username>` from the **SCGC Template**, flavor **g.medium** or larger.
2. Connect as user `student`.

Then download and start the lab:

```bash
cd work/
wget https://repository.grid.pub.ro/cs/scgc/laboratoare/lab-routing.zip
unzip lab-routing.zip
bash runvm.sh
source ~/.bashrc
```

- `bash runvm.sh` → boots the topology (host/red/green/blue).
- `source ~/.bashrc` → loads helper commands (`go`, `start_lab`, `reboot_vms`).

To enter a station use `go <name>`, e.g. `go red`.

---

## Configuring and deleting IP addresses

Configure the **host–red** link on `10.10.10.0/24`.

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

- `ip address add IP/mask dev IF` → assign an L3 address.
- `ip link set dev IF up` → activate the interface (L2).

Validate and test:

```bash
ip address show dev usernet      # on host
ip address show dev red-eth0     # on red
ping -c 3 10.10.10.2             # from host to red
```

To remove all addresses from an interface:

```bash
ip address flush dev usernet
```

---

## Configuring IP addresses

Now do the same for the **host–green** link on `10.10.20.0/24`.

On **host** (interface toward green — check the real name with `ip link`):

```bash
ip address add 10.10.20.1/24 dev <green-link-if>
ip link set dev <green-link-if> up
```

On **green** (`go green`):

```bash
ip address add 10.10.20.2/24 dev green-eth0
ip link set dev green-eth0 up
```

Before testing, verify L2 is up:

```bash
ip link                  # confirm interfaces are UP
ping -c 3 10.10.20.2     # from host to green
```

---

## IP Addressing and Routing

Stations only reach the host so far. Add routes on the stations and enable forwarding on the host.

On **red**:

```bash
ip route add 10.10.10.1 dev red-eth0     # route to the gateway (host)
ip route add default via 10.10.10.1      # everything else goes via host
ip route show                            # validate
```

Repeat the same logic on **green** (gateway `10.10.20.1`).

On **host**, enable IPv4 forwarding:

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward               # validate -> should be 1
```

Watch traffic while testing:

```bash
tcpdump -n -i usernet                     # on host, in a separate shell
ping -c 3 10.10.20.2                       # from red toward green
```

---

## Complete connectivity setup

Add the third segment, **host–blue** on `10.10.30.0/24`.

On **host**:

```bash
ip address add 10.10.30.1/24 dev <blue-link-if>
ip link set dev <blue-link-if> up
```

On **blue** (`go blue`):

```bash
ip address add 10.10.30.2/24 dev blue-eth0
ip link set dev blue-eth0 up
ip route add default via 10.10.30.1       # host as default gateway
```

Now test connectivity between **all** stations (red↔green, red↔blue, green↔blue).

---

## ARP table

ARP maps IP ↔ MAC on a segment.

```bash
ip neighbor show           # view current neighbors
ping -c 1 10.10.10.2       # populate by pinging a neighbor
ip neighbor show           # entry now appears
```

Repeat for red, green, and blue to populate entries for each.

---

## Troubleshoot IP address configuration problem

```bash
start_lab ex6
go red
```

Diagnose:

```bash
ip r s                     # show routing table (= ip route show)
ip address show            # check the netmask carefully
```

The bug here is a wrong mask (`/32` instead of `/24`). Fix it by deleting the bad address and re-adding it correctly:

```bash
ip address delete 10.10.7.1/32 dev usernet
ip address add 10.10.7.1/24 dev usernet
```

---

## Troubleshooting connectivity problem

```bash
start_lab ex7
```

Method:

1. First, **display the routing table** on each station (`ip route show`).
2. Test connectivity between all stations to see which links fail.
3. Identify the misconfiguration (missing/wrong route or missing forwarding) and fix it, then re-test.

---

## IPv6

On **host**:

```bash
ip -6 address add 2201::1/64 dev usernet
ip -6 address show dev usernet            # validate
```

Configure the other segments:

- **host–blue** → `2202::/64`
- **host–green** → `2203::/64`

Enable IPv6 forwarding on host:

```bash
sysctl -w net.ipv6.conf.all.forwarding=1
```

Add IPv6 default routes on red, green, blue (via the host's IPv6 on each link), then test:

```bash
ping6 -c 3 2201::1
```

---

## Persistent setup

```bash
start_lab ex9
ip address flush dev usernet
```

Make the IPv6 config from the previous exercise **permanent** by editing the Ubuntu network config (netplan, `/etc/netplan/*.yaml`) instead of using temporary `ip` commands. Also persist routing by enabling forwarding via sysctl config (e.g. `/etc/sysctl.conf` or a file in `/etc/sysctl.d/`).

Verify it survives a reboot:

```bash
reboot_vms
```

After reboot, the addresses, routes and forwarding should still be present.
