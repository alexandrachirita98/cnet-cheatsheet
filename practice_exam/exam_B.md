# Practice Exam B — Layer 2: Switching & VLANs

**Tool:** Cisco Packet Tracer · **Time:** 60 min · **Total:** 100 pts

## Scenario

You work in Packet Tracer. Build the topologies described, configure the switches, and prove behavior in **Simulation** mode and via the switch **CLI** tab. (CLI: select switch → CLI tab → Enter; `enable` → privileged, `configure terminal` → config, `end` → back.)

---

## Tasks (prompts)

1. **(10 pts)** Build: **2 switches** linked, **2 PCs per switch**. Give the PCs `192.168.1.1–.4 /24`. Prove any PC can reach any other. *[connectivity]*
2. **(10 pts)** On a switch, show the **MAC address table**, clear it, then send PC0→PC1 and show that the table learned both MACs. *[switching table]*
3. **(15 pts)** In **Simulation** mode with an empty table, send PC1→PC3 and describe what the switch does; send the **same** packet again and describe the difference. Explain why. *[flooding vs forwarding]*
4. **(10 pts)** Unplug a PC and show its table entry disappears; reconnect it to a **different port** and show the entry is relearned on the new port. *[dynamic learning]*
5. **(20 pts)** On one switch isolate PC1 & PC3 (VLAN 10) from PC2 & PC4 (VLAN 20) using **access ports**: Fa1/1→PC1, Fa2/1→PC2, Fa6/1→PC3, Fa3/1→PC4. Prove same-VLAN works and cross-VLAN fails. *[access ports]*
6. **(20 pts)** Link two switches and make VLAN 10 (and 20) span both using a **trunk** on Fa4/1. Prove a VLAN-10 host on Switch0 reaches a VLAN-10 host on Switch1. *[trunk ports]*
7. **(15 pts) Troubleshooting:** open the debug topology (VLAN 10 = PC0/PC1/PC3, VLAN 20 = PC2/PC4). PC0↔PC1 and PC2↔PC4 fail. Find and fix both. *[debugging VLANs]*

---

## Answer key

<details><summary>Task 1 — connectivity</summary>

Place 2 Switch-PT + 4 PC-PT, link them. On each PC: **Desktop → IP Configuration** → `192.168.1.1` (`.2/.3/.4`), mask `255.255.255.0`. Use **Add Simple PDU** between PCs → status *Successful*.
</details>

<details><summary>Task 2 — switching table</summary>

On the **switch** (CLI):
```cisco
enable
show mac-address-table
clear mac-address-table
```
Send PC0→PC1, then:
```cisco
show mac-address-table     # entries for PC0 and PC1 with their ports
```
</details>

<details><summary>Task 3 — flooding vs forwarding</summary>

On the **switch**: `enable` → `clear mac-address-table`. In **Simulation**:
- First PC1→PC3: destination unknown → frame is **flooded** to all ports.
- Second PC1→PC3: PC3's MAC is now learned → frame is **forwarded only** to PC3's port.

Why: the switch learns the source MAC of the first frame; until the destination MAC is in the table it must flood.
</details>

<details><summary>Task 4 — dynamic learning</summary>

Delete the cable to a PC → `show mac-address-table` shows its entry gone. Reconnect with a **Copper Straight-Through** cable to a different port, send a packet from it, then `show mac-address-table` → entry reappears on the new port.
</details>

<details><summary>Task 5 — access ports</summary>

On the **switch** (CLI):
```cisco
enable
configure terminal
vlan 10
 name vlan10
vlan 20
 name vlan20
interface FastEthernet1/1
 switchport mode access
 switchport access vlan 10
interface FastEthernet6/1
 switchport mode access
 switchport access vlan 10
interface FastEthernet2/1
 switchport mode access
 switchport access vlan 20
interface FastEthernet3/1
 switchport mode access
 switchport access vlan 20
end
show vlan brief
```
Prove: PC1↔PC3 works, PC2↔PC4 works, PC1↔PC2 fails.
</details>

<details><summary>Task 6 — trunk ports</summary>

Connect Switch0 Fa4/1 ↔ Switch1 Fa4/1 (fiber). On **Switch0**:
```cisco
enable
configure terminal
interface FastEthernet4/1
 switchport mode trunk
 switchport trunk allowed vlan all
end
show interfaces trunk
```
On **Switch1**: identical config. Prove a VLAN-10 host on each switch can now ping across.
</details>

<details><summary>Task 7 — debugging</summary>

Diagnose on the relevant switch:
```cisco
enable
show vlan brief          # is each host port in the right VLAN?
show interfaces trunk    # does the trunk allow the needed VLAN?
```
Fix PC0↔PC1 (Switch0) — host port in wrong VLAN:
```cisco
configure terminal
interface FastEthernet1/1
 switchport mode access
 switchport access vlan 10
end
```
Fix PC2↔PC4 (Switch1) — trunk not carrying VLAN 20:
```cisco
configure terminal
interface FastEthernet4/1
 switchport mode trunk
 switchport trunk allowed vlan all
end
```
</details>
