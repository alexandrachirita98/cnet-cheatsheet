# Layer 2 — Ethernet, Switching & VLANs

This lab runs in **Cisco Packet Tracer** (a GUI network simulator). You build topologies of switches/hubs/PCs, watch how a switch learns MAC addresses into its **switching table**, then logically split one physical network into separate **VLANs** using access and trunk ports.

Every section first explains **what it is about**, then gives a **Solution** subchapter. Because this is Packet Tracer, the Solution mixes **GUI steps** (where the lab is point-and-click) and **Cisco IOS CLI commands** (entered in the switch's *CLI* tab). Each command block is for **one device only** — the heading says which.

> CLI note: select a switch → **CLI** tab → press **Enter** once, then type. `enable` enters privileged mode; `configure terminal` enters config mode; `end` returns to privileged mode.

---

## Setup

### What this is about

Get Packet Tracer (no cloud VM for this lab). You download it from the **NetAcad** website, which requires a free account (an institutional email works).

### Solution

1. Register / log in at the NetAcad site.
2. Download and install **Cisco Packet Tracer**.
3. For each exercise, open the provided downloadable `.pkt` topology, or build it yourself.

---

## Packet Tracer intro

### What this is about

Learn the workspace before configuring anything:
- Device tools: **Router**, **PC (PC-PT)**, **Switch (Switch-PT)**, **Hub (Hub-PT)**.
- **Cable** tool — pick a cable type, or use auto-select to let it choose the right one.
- **Delete** button — clears placed devices / packet history.
- **Status pane** (lower right) — shows results of sent packets.
- **Modes**: *Realtime* (default) and *Simulation* (step through packets, with Auto Capture / Play).
- **Add Simple PDU** tool — sends a test packet between two L3 devices.

### Solution

No commands — explore the toolbars, switch between **Realtime** and **Simulation** mode, and try the **Add Simple PDU** (envelope) tool.

---

## Connectivity test

### What this is about

Build a small topology and prove basic L2 connectivity by giving the PCs IPs in the same subnet and sending a test packet.

### Solution — topology

Place **2 switches**, link them, and connect **2 PCs to each switch**.

### Solution — on each PC

Configure the IP via **Desktop → IP Configuration**:

```
IP address : 192.168.1.1  (then .2, .3, .4, .5 on the others — unique per PC)
Subnet mask: 255.255.255.0
```

Then use the **Add Simple PDU** tool to send a packet between two PCs and watch the status pane report *Successful*.

---

## Switch vs Hub

### What this is about

Compare a **hub** (L1 — repeats every frame out all ports) with a **switch** (L2 — forwards only to the port that owns the destination MAC). The exercise asks you to identify the differences and at which OSI layer each device operates.

### Solution

1. Open the provided hub-vs-switch topology.
2. **Wait for the switch ports to turn green** (not orange) — orange means STP is still converging.
3. Enter **Simulation** mode and send a packet; observe that the **hub floods all ports**, while the **switch forwards only to the destination port** (after it has learned the MAC).

Answer: a **hub = Layer 1** (physical, dumb repeater); a **switch = Layer 2** (data link, MAC-aware).

---

## Populating the switching table

### What this is about

A switch builds a **switching (MAC address) table** by learning the **source MAC** of every frame and the port it arrived on. Here you view and clear that table from the CLI.

### Solution — on the switch (CLI tab)

```cisco
enable                        ! enter privileged mode
show mac-address-table        ! view learned MAC -> port entries
clear mac-address-table       ! wipe the table to start fresh
```

---

## The switching table

### What this is about

Understand the table's structure: each entry maps a **MAC address → port**. A port can host **many MACs**, but each **MAC appears only once**. The switch learns from source addresses and forwards (or **floods**, if the destination is unknown).

### Solution — exercise sequence

On the **switch**, start clean:

```cisco
enable
clear mac-address-table
```

Then, using **Add Simple PDU** in **Realtime**:
1. Send **PC0 → PC1**, then `show mac-address-table` — entries for PC0 and PC1 appear.
2. Send **PC1 → PC2**, check the table again — note only the new MAC is added.
3. Send **PC2 → PC3**, check again.

```cisco
show mac-address-table        ! run after each send to watch the table grow
```

---

## The switching table reloaded

### What this is about

Repeat the learning behavior in **Simulation** mode so you can *see* flooding vs. directed forwarding frame by frame. With an empty table the switch **floods** an unknown destination; once learned, it forwards only to the right port.

### Solution

On the **switch**:

```cisco
enable
clear mac-address-table
```

Then in **Simulation** mode:
1. Send **PC1 → PC3** — destination unknown, so the frame is **flooded** to all ports.
2. Send **PC1 → PC3 again** — now PC3's MAC is known, so it's **forwarded only** to PC3's port.
3. `clear mac-address-table`, resend, and inspect the table at **each step** to confirm the learning.

```cisco
show mac-address-table
```

---

## Switching table analysis (two switches)

### What this is about

Extend to a **two-switch** topology with hosts in pairs per switch. The key insight: the port that connects to the **other switch** ends up associated with **multiple MAC addresses** (all the hosts behind that switch).

### Solution

On **each switch**:

```cisco
enable
clear mac-address-table
show mac-address-table        ! after sending traffic, the inter-switch port lists several MACs
```

Send packets between hosts on different switches and observe that the uplink port accumulates multiple MAC entries.

---

## Removing equipment

### What this is about

Show that the switching table is dynamic: unplugging a host removes its learned entry (entries also age out over time).

### Solution

1. Unplug **PC1** from **Switch0** (delete the cable).
2. On **Switch0**:

```cisco
enable
show mac-address-table        ! PC1's entry is gone
```

---

## Reconnecting equipment

### What this is about

Reconnect a host to a **different port** and confirm the switch relearns it on the new port.

### Solution

1. Reconnect **PC1** to a **different port** on **Switch0** using a **Copper Straight-Through** cable.
2. Send **PC1 → PC0**.
3. On **Switch0**:

```cisco
enable
show mac-address-table        ! PC1 now appears on its new port
```

---

## VLANs intro

### What this is about

A **VLAN (Virtual LAN)** logically splits one physical LAN into multiple independent subnets. Separation happens at **L2** by tagging frames with a **VLAN ID**; hosts only see their own VLAN's network. Example assignment: PC0 & PC2 → **VLAN 10**, PC1 & PC3 → **VLAN 20**.

### Solution

Inspect/configure VLANs via the switch's **Config tab → VLAN Database** (GUI), or via CLI:

On the **switch**:

```cisco
enable
configure terminal
vlan 10
 name vlan10
vlan 20
 name vlan20
end
show vlan brief               ! list VLANs and which ports belong to each
```

---

## Access ports

### What this is about

An **access port** carries a **single VLAN** (untagged toward the host). Goal: isolate PC1 & PC3 from PC2 & PC4 by putting them in different VLANs. Mapping:

| Port    | Host | VLAN |
|---------|------|------|
| Fa1/1   | PC1  | 10   |
| Fa2/1   | PC2  | 20   |
| Fa6/1   | PC3  | 10   |
| Fa3/1   | PC4  | 20   |

### Solution — on the switch (CLI tab)

Create the VLANs, then assign each port as an access port in its VLAN:

```cisco
enable
configure terminal

vlan 10
 name vlan10
vlan 20
 name vlan20

interface FastEthernet1/1      ! PC1
 switchport mode access
 switchport access vlan 10

interface FastEthernet6/1      ! PC3
 switchport mode access
 switchport access vlan 10

interface FastEthernet2/1      ! PC2
 switchport mode access
 switchport access vlan 20

interface FastEthernet3/1      ! PC4
 switchport mode access
 switchport access vlan 20

end
show vlan brief                ! confirm each port sits in the right VLAN
```

Verify: PC1↔PC3 (same VLAN 10) and PC2↔PC4 (same VLAN 20) work; PC1↔PC2 (different VLANs) does **not**.

---

## Trunk ports

### What this is about

An **access** port carries one VLAN; a **trunk** port carries **many VLANs** between switches by **tagging** frames (802.1Q). To make a VLAN span two switches, the link between them must be a trunk. Without it, same-VLAN hosts on different switches can't talk.

### Solution — topology

Connect **Switch0** and **Switch1** with a **fiber optic** link on the **Fa4/1** ports of each. Same-VLAN connectivity across the switches **fails** until the link is a trunk.

### Solution — on Switch0

```cisco
enable
configure terminal
interface FastEthernet4/1
 switchport mode trunk
 switchport trunk allowed vlan all     ! carry all VLANs (1 default, 10, 20, mgmt 100)
end
show interfaces trunk                  ! confirm trunk + allowed VLANs
```

### Solution — on Switch1

```cisco
enable
configure terminal
interface FastEthernet4/1
 switchport mode trunk
 switchport trunk allowed vlan all
end
show interfaces trunk
```

Verify: same-VLAN hosts on different switches now communicate; different VLANs stay isolated.

---

## Cascading trunk ports

### What this is about

Chain **three switches** (Switch0 — Switch1 — Switch2) so VLANs span the whole chain. Every inter-switch link must be a trunk, and the VLANs must exist on **all** switches. Hosts: VLAN 10 = PC0 & PC2; VLAN 20 = PC1 & PC3.

### Solution

First, in the no-VLAN state, send packets to confirm everything is connected. Then enforce L2 separation.

On **each of Switch0, Switch1, Switch2** — create the VLANs and trunk the inter-switch links:

```cisco
enable
configure terminal

vlan 10
 name vlan10
vlan 20
 name vlan20

interface FastEthernet4/1               ! the inter-switch link(s) on this switch
 switchport mode trunk
 switchport trunk allowed vlan all

! repeat 'interface ... switchport mode trunk' for every port that connects to another switch

end
show vlan brief
show interfaces trunk
```

On the **access switches** — put each host port in its VLAN (as in *Access ports*):

```cisco
configure terminal
interface FastEthernet1/1
 switchport mode access
 switchport access vlan 10               ! VLAN 20 for PC1/PC3
end
```

Verify: VLAN 10 hosts reach each other across all switches; VLAN 20 likewise; the two VLANs stay isolated.

---

## Debugging VLANs

### What this is about

A pre-broken topology: **5 hosts** in 2 VLANs (VLAN 10 = PC0, PC1, PC3; VLAN 20 = PC2, PC4) with connectivity failures to fix. Two typical bugs: a host port in the **wrong access VLAN**, and a **trunk that doesn't allow** a needed VLAN.

### Solution — diagnose first

On the relevant switch:

```cisco
enable
show vlan brief                ! is each host port in the VLAN it should be?
show interfaces trunk          ! does the trunk carry the VLANs that must cross it?
```

### Solution — fix: PC0 ↔ PC1 (check Switch0)

A host port is in the wrong VLAN — put it back in VLAN 10:

```cisco
configure terminal
interface FastEthernet1/1      ! the offending host port
 switchport mode access
 switchport access vlan 10
end
show vlan brief
```

### Solution — fix: PC2 ↔ PC4 (check Switch1 trunk)

The trunk isn't carrying VLAN 20 — allow it (or allow all):

```cisco
configure terminal
interface FastEthernet4/1      ! the trunk between the switches
 switchport mode trunk
 switchport trunk allowed vlan all
end
show interfaces trunk
```

Re-test each failing pair after the fix.
