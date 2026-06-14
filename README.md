# CNET / SCGC — Lab Cheatsheets

Summaries with **complete solutions** for the labs on [cybercloud.upb.ro](https://cybercloud.upb.ro/docs). Every file has the same structure: for each section, first **What this is about** (what it does and why), then **Solution** (all commands fully worked out). Each code cell contains commands for **one station only** (the heading above it says where to run them).

> For the exam: search here by **what you want to do** (e.g. "block a port", "NAT", "GPG signature", "systemd timer") — the list below tells you which file and which section.

> Practice papers: see **[practice_exam/](practice_exam/README.md)** — 5 exams (A–E) with tasks + collapsible answer keys.

---

## [Routing](routing.md)

Turn the **host** station into a **router** connecting three stations (red / green / blue), each on its own /24 segment. You build connectivity step by step, from L2 up to persistence.

**Exercises covered:**
- **Configure IP addresses (L3) + bring interfaces up (L2)** — `ip address add IP/mask dev IF`, `ip link set dev IF up`, validate with `ip address show` and `ping`; how to wipe addresses with `ip address flush`.
- **Identify the correct interface** — on the host, the interface toward red is `usernet`, but the ones toward green/blue have different names (find them with `ip link`).
- **Routing between stations** — add a route to the gateway + `default via <gateway>` on each station, and enable **IP forwarding** on the host (`sysctl -w net.ipv4.ip_forward=1`). Watch routed traffic with `tcpdump`.
- **Full connectivity** — add the third segment (blue) and test every pair red↔green↔blue.
- **ARP table** — `ip neighbor show`, how it's populated by traffic, the REACHABLE/STALE/DELAY states.
- **Troubleshoot wrong netmask (ex6)** — the bug is `/32` instead of `/24`; diagnose with `ip r s` / `ip address show`, fix by delete + re-add with the correct mask.
- **Troubleshoot connectivity (ex7)** — wrong/missing default route on a station or forwarding off on the host; method: "read the routing table first".
- **IPv6** — same topology with `2201::/64` etc., `ip -6`, IPv6 forwarding, IPv6 default routes, test with `ping6`.
- **Persistence (ex9)** — make the config permanent with **netplan** (`/etc/netplan/*.yaml`) + persistent forwarding via `/etc/sysctl.d/`, verify with `reboot_vms`.

---

## [Layer 2 — Ethernet, Switching & VLANs](layer2_vlans.md)

A **Cisco Packet Tracer** lab (GUI + Cisco IOS CLI). You watch how a switch learns MAC addresses, then logically split one physical network into separate **VLANs** with access and trunk ports.

**Exercises covered:**
- **Packet Tracer basics** — workspace tools, Realtime vs. Simulation mode, the Add Simple PDU (test packet) tool.
- **Connectivity test** — build 2 switches + 4 PCs, set IPs `192.168.1.x/24` via Desktop → IP Configuration, send a test packet.
- **Switch vs. Hub** — hub floods all ports (L1) vs. switch forwards to the right port (L2); identify the OSI layers.
- **Switching table** — how a switch learns source MAC → port; `enable`, `show mac-address-table`, `clear mac-address-table`; flooding (unknown dest) vs. directed forwarding, observed in Simulation mode.
- **Two-switch analysis** — the inter-switch uplink port accumulates multiple MACs.
- **Removing / reconnecting equipment** — entries disappear when a host is unplugged and are relearned on the new port.
- **VLANs intro** — logical L2 separation via VLAN ID; Config tab → VLAN Database, or `vlan 10/20` + `show vlan brief`.
- **Access ports** — one VLAN per port (`switchport mode access` / `switchport access vlan N`); isolate PCs by VLAN per the Fa1/1…Fa6/1 mapping.
- **Trunk ports** — carry many VLANs between switches (`switchport mode trunk` / `switchport trunk allowed vlan all`), `show interfaces trunk`.
- **Cascading trunks** — VLANs spanning 3 switches; create VLANs on all switches and trunk every inter-switch link.
- **Debugging VLANs** — fix a host port in the wrong access VLAN (Switch0) and a trunk that doesn't allow a VLAN (Switch1).

---

## [Firewall — iptables (Security)](iptables_security.md)

Use **iptables** on the host (router between red and green) to filter traffic: see which services send passwords in cleartext, then block and selectively allow services.

**Exercises covered:**
- **Encrypted vs. unencrypted traffic** — with `tcpdump -A` you see that **telnet (23)** and **FTP (21)** send credentials in cleartext, while **SSH (22)** is encrypted.
- **Blocking unencrypted services** — iptables basics (`-A`, the **FORWARD** chain for routed traffic, `-d`, `-p`, `--dport`, `-j REJECT/DROP/ACCEPT`); block telnet and FTP to green; inspect with `iptables -L FORWARD -v -n`.
- **Block SSH** — add a rule blocking SSH to green.
- **Allow SSH (rule order!)** — an ACCEPT added with `-A` doesn't work because the REJECT is evaluated first; fix by **inserting** at position 1: `iptables -I FORWARD 1 ... -j ACCEPT`. Delete a single rule with `-D`.
- **Deleting rules** — `iptables -F FORWARD` flushes the whole chain; verify services work again.
- **Traffic captures** — `tcpdump -w file.pcap`, transfer with `scp`, analyze in Wireshark.

---

## [NAT — iptables](iptables_nat.md)

Use the **nat** table of iptables on the host (gateway) to give private stations Internet access (**MASQUERADE**) and to expose internal services to the outside (**port forwarding / DNAT**).

**Exercises covered:**
- **MASQUERADE (source NAT)** — `iptables -t nat -A POSTROUTING -j MASQUERADE` rewrites the source address of outgoing packets; you also need **IP forwarding** enabled. Classic troubleshooting: ping fails until you enable forwarding.
- **Observing NATed packets (ICMP)** — with `tcpdump` on different interfaces you see the **translated** source on `eth0` (external) and the **original** on `usernet` (internal); capture on `-i any` for both directions.
- **Observing NATed packets (TCP)** — with TCP the source port may change too; observe an HTTP request (`wget http://cs.pub.ro`).
- **Port forwarding (DNAT)** — expose red's SSH from the Internet via the gateway's port 10022: `iptables -t nat -A PREROUTING -p tcp --dport 10022 -j DNAT --to-destination 10.10.1.2:22`. Restrict with `-i eth0` so only external traffic is forwarded.
- **Port forwarding for multiple stations** — distinct ports per station (20022→green:22, 30022→blue:22), test from `fep.grid.pub.ro`.
- **Observing port forwarding** — `tcpdump` on `eth0` (incoming) and `usernet` (forwarded) to see the translation `<gateway,10022>` → `<red,22>`.
- **Telnet port forwarding** — same DNAT pattern for telnet (ports 10023/20023/30023 → 23).

---

## [Certificates — PKI, X.509, TLS, GPG](certificates.md)

Public-key security: inspect and generate **X.509 certificates** with `openssl`, secure a client/server channel with **TLS**, and use **GPG** for encryption and signatures.

**Exercises covered:**
- **Inspecting local certificates** — `openssl x509 -text/-issuer/-subject/-dates/-pubkey/-modulus`; chain verification with `openssl verify -CAfile`; listing all certs in a chain. Exercise: verify two more certificates by identifying the right issuer.
- **Inspecting remote certificates** — `openssl s_client -connect host:443` to grab a live server's cert; extract + decode with `sed` + `openssl x509`; observe SAN/DNS.
- **Generating certificates** — private key (`genrsa`), CSR (`req -new`), sign with a CA (`openssl ca -config ca.cnf`); verify the **key and certificate match** by comparing the modulus (md5sum).
- **Unencrypted client/server** — `nc` server+client, sniffing with `tcpdump -A` → messages appear in cleartext.
- **SSL/TLS client/server** — `openssl s_server` + `openssl s_client`; without `-CAfile` verification fails, with it → "Verification: OK"; tcpdump shows no readable text.
- **GPG — key setup** — `gpg --full-generate-key` (RSA 4096), `--list-key`; primary key (sign) + subkey (encrypt).
- **GPG — sharing encrypted data** — export/import public key, verify fingerprint, sign the key (`--edit-key` → `fpr`/`sign`/`save`), encrypt with `--encrypt --recipient`, decrypt with `--decrypt`.
- **GPG — verifying files** — detached signature (`--detach-sign`) and verification (`--verify`).
- **GPG — verifying an ISO download** — import official Fedora keys, `--verify` the checksum file, then `sha256sum -c`.

---

## [Containers — Docker](docker.md)

Container-based virtualization: run containers, build images with a **Dockerfile**, wire multiple containers together (manually and with **compose**), persist data with **volumes**.

**Exercises covered:**
- **Concepts** — what a container is, why it's lighter than a VM (shared kernel), advantages and the security trade-off.
- **Starting containers** — interactive (`run -it ... bash`), single command, detached (`run -d`), inspect (`docker ps`), enter (`exec -it`), stop (`stop`). Exercise: Rocky Linux 8 background container + `yum install bind-utils`.
- **Building custom images** — `Dockerfile` (`FROM`/`RUN`), `docker build -t`, `docker image list`. Exercise: `Dockerfile.rocky` with `bind-utils`, tested with `nslookup`.
- **Multi-container applications** — Docker network (`network create`), MySQL + WordPress linked by name, publish a port `-p 8000:80`. Exercise: NextCloud exposed on a port.
- **Docker Compose** — the whole stack in one `docker-compose.yml` (services/networks/environment/ports), `docker-compose up/down`. Exercise: compose file for NextCloud.
- **Persistent storage** — named volumes (`-v vol:/path`), host bind mount (`-v ~/dir:/path`), volumes in compose. Exercise: volume for NextCloud at `/var/www/html`, verified after restart.
- **Security** — process isolation vs. shared kernel; comparison with VMs.

---

## [Services & Task Automation](services_automation.md)

Managing services with **systemd**, reading logs with **journald**, and automating with **cron** and **systemd timers**.

**Exercises covered:**
- **systemd services** — `systemctl status` (enabled/active/PID/logs).
- **Changing service state** — `start`/`stop` (now) vs. `enable`/`disable` (at boot). Exercise 1: stop SSH and observe the effect on reconnect/reboot. Exercise 2: install nginx, check status, toggle enabled/disabled.
- **Tuning service files** — where they live (`/lib/systemd/system`, `/etc/systemd/system`), `systemctl cat`. Exercise 3: override with `systemctl edit` (`LimitNOFILE=5`), lands in `.../override.conf`.
- **journald** — `journalctl` with flags `-b -1`/`-e`/`-x`/`-u`/`-t`/`-n`/`-f`; persistent logs via `Storage=persistent` in `journald.conf`. Exercise 4: follow nginx logs live (`-u nginx -f`) and trigger a config error.
- **cron** — the 5 time fields + command, `crontab -e`, escaping `%`. Exercise 5: a task every 2 minutes counting files in `/usr/share` (`find | wc -l`). Exercise 6: postfix to deliver the output (`/var/mail/student`) or redirect with `>>file 2>&1` / `>/dev/null`.
- **systemd timers** — `name.timer` triggers `name.service`, `systemctl list-timers`. Prep: disable the dynamic MOTD in `/etc/pam.d/sshd`, write `update-motd.sh`. Exercise 7: write `motd-update.service` + `motd-update.timer` (runs every 2 min via `OnCalendar`, enable only the timer). Exercise 8: add `SyslogIdentifier=motd-update` with `systemctl edit --full`.

---

## Notes

A few values are **specific to your instance** and must be taken from the lab, not invented:
- **NAT**: the stations' IPs (`<green-ip>`, `<blue-ip>`) and the gateway's external IP (`10.9.X.Y`).
- **Routing**: the real names of the host interfaces toward green/blue (from `ip link`).
- **Certificates**: the exact Fedora release name for the ISO verification.
