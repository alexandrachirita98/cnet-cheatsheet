# Practice Exams

Five practical exam papers in the SCGC style, drawn from the lab cheatsheets in this repo. Each is self-contained: a scenario, the **tasks (prompts)** first, then a collapsible **answer key** per task so you can attempt it cold and check after.

| Exam | Focus | Cheatsheets |
|------|-------|-------------|
| [Exam A](exam_A.md) | Routing — addressing, routing, ARP, IPv6, persistence, troubleshooting | [routing](../routing.md) |
| [Exam B](exam_B.md) | Layer 2 — switching table & VLANs (Cisco Packet Tracer) | [layer2_vlans](../layer2_vlans.md) |
| [Exam C](exam_C.md) | Firewall + NAT — filtering, MASQUERADE, port forwarding | [iptables_security](../iptables_security.md), [iptables_nat](../iptables_nat.md) |
| [Exam D](exam_D.md) | Containers — run, build, multi-container, compose, volumes | [docker](../docker.md) |
| [Exam E](exam_E.md) | Certificates/GPG + Services & automation | [certificates](../certificates.md), [services_automation](../services_automation.md) |

**How to use:** read the tasks, solve on a real lab VM / Packet Tracer, then expand the answer keys. Each task notes the points and the cheatsheet section it tests.

> Values like interface names (`green-link`), station IPs (`<green-ip>`), and the gateway's external IP (`10.9.X.Y`) depend on your lab instance — read them with `ip link` / `ip a`, don't assume the examples.
