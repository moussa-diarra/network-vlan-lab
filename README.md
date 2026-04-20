# Network Segmentation Lab — VLAN & Firewall

![Debian](https://img.shields.io/badge/-Debian-A81D33?style=flat-square&logo=debian&logoColor=white)
![Alpine Linux](https://img.shields.io/badge/-Alpine%20Linux-0D597F?style=flat-square&logo=alpinelinux&logoColor=white)
![VirtualBox](https://img.shields.io/badge/-VirtualBox-183A61?style=flat-square&logo=virtualbox&logoColor=white)
![Networking](https://img.shields.io/badge/-TCP%2FIP%20%2F%20iptables-1BA0D7?style=flat-square&logo=cisco&logoColor=white)

A network segmentation lab built on VirtualBox simulating a real-world VLAN architecture with a Linux-based firewall. Two isolated VLANs communicate through a Debian firewall running iptables — with granular rules controlling exactly which traffic is allowed between segments and to the internet.

---

## Architecture

```
  VLAN 10                 Debian Firewall                  VLAN 20
  ┌─────────────┐       ┌───────────────────┐       ┌─────────────┐
  │ machine1    │ eth0  │ enp0s3            │ eth0  │ machine3    │
  │ 10.10       │──────►│ 192.168.10.1      │◄──────│ 20.10       │
  │ machine2    │       │                   │       │ machine4    │
  │ 10.20       │       │ enp0s8            │       │ 20.20       │
  │ (Alpine)    │       │ 192.168.20.1      │       │ (Alpine)    │
  │ GW: 10.1    │       │                   │       │ GW: 20.1    │
  └─────────────┘       │ enp0s10 (NAT/WAN) │       └─────────────┘
                        │ ──────────────────│──────► Internet
                        │                   │
                        │ IP Forward: on    │
                        │ iptables: stateful│
                        │ NAT: MASQUERADE   │
                        └───────────────────┘

  Réseau interne VirtualBox : LabNet10 (VLAN10) · LabNet20 (VLAN20) · NAT
  Clients Alpine : interface eth0  —  Firewall Debian : enp0s3 / enp0s8 / enp0s10
```

**VMs:** 5 total — 1 Debian firewall + 4 Alpine Linux clients (2 per VLAN)
**Why Alpine?** Minimal footprint (~130MB) — ideal for lab client machines

---

## Traffic policy

| Source | Destination | Protocol | Action | Reason |
|---|---|---|---|---|
| VLAN10 | VLAN20 | Any (NEW) | DROP | Full isolation — VLAN10 cannot initiate connections to VLAN20 |
| VLAN20 | VLAN10 | ICMP echo | DROP | VLAN20 cannot ping VLAN10 |
| VLAN20 | VLAN10 | TCP/22 (SSH) | ACCEPT | VLAN20 can SSH into VLAN10 — simulates an IT/admin VLAN |
| VLAN10 | Internet | Any | ACCEPT | via NAT MASQUERADE |
| VLAN20 | Internet | Any | ACCEPT | via NAT MASQUERADE |
| Any | Any | ESTABLISHED/RELATED | ACCEPT | Return traffic for established sessions |
| Any | Any | (default) | DROP | Deny-all default policy |

**Design rationale:** VLAN20 acts as an admin/IT segment that can reach VLAN10 over SSH but not ping it (reduces attack surface while preserving management access). VLAN10 is fully isolated from VLAN20 — it can only reach the internet.

---

## Firewall rules (iptables)

```bash
# Allow return traffic for established/related sessions
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block VLAN10 → VLAN20 (new connections only)
iptables -A FORWARD -s 192.168.10.0/24 -d 192.168.20.0/24 -m state --state NEW -j DROP

# Allow VLAN20 → VLAN10 SSH only
iptables -A FORWARD -p tcp --dport 22 -s 192.168.20.0/24 -d 192.168.10.0/24 -j ACCEPT

# Block VLAN20 → VLAN10 ICMP ping
iptables -A FORWARD -p icmp -s 192.168.20.0/24 -d 192.168.10.0/24 --icmp-type echo-request -j DROP

# Allow VLAN20 → internet
iptables -A FORWARD -s 192.168.20.0/24 -o enp0s9 -j ACCEPT

# NAT — masquerade outbound traffic on WAN interface
iptables -t nat -A POSTROUTING -o enp0s9 -j MASQUERADE

# Explicit deny-all at end of chain (after all allow rules)
iptables -A FORWARD -j DROP
```

**Default policy vs explicit DROP:**
`-A FORWARD -j DROP` en dernière règle est plus sûr en lab que `-P FORWARD DROP`. La politique par défaut s'applique immédiatement et coupe tout — y compris le trafic légitime pas encore explicitement autorisé. Une règle DROP explicite en fin de chaîne ne bloque que ce qui n'a pas matché les règles précédentes, ce qui permet de construire et tester les règles progressivement.

**Key concept — stateful vs stateless filtering:**
The `ESTABLISHED,RELATED` rule is what makes the firewall stateful. Without it, blocking return traffic would break every outbound connection — a ping to 8.8.8.8 would send the request but the ICMP reply would be dropped. Stateful inspection tracks connection state and automatically allows legitimate return packets.

---

## IP forwarding

For the Debian firewall to route packets between interfaces, kernel IP forwarding must be enabled:

```bash
# Temporary (resets on reboot)
echo 1 > /proc/sys/net/ipv4/ip_forward

# Permanent
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

Without this, the firewall receives packets destined for other subnets and silently drops them — no routing occurs regardless of iptables rules.

---

## Validation tests

| Test | From | Command | Expected result |
|---|---|---|---|
| Intra-VLAN connectivity | machine1 | `ping 192.168.10.20` |  Success |
| VLAN isolation (10→20) | machine1 | `ping 192.168.20.10` |  Blocked |
| Internet access (VLAN10) | machine1 | `ping 8.8.8.8` |  Success |
| VLAN20 SSH → VLAN10 | machine3 | `ssh machine1-vlan10@192.168.10.10` |  Success |
| VLAN20 ping → VLAN10 | machine3 | `ping 192.168.10.10` |  Blocked |
| Internet access (VLAN20) | machine3 | `ping 8.8.8.8` |  Success |

---

## Screenshots

| Step | Screenshot |
|---|---|
| Firewall interfaces — `ip a` | `screenshots/fw-ip-a.png` |
| VLAN10 — ping 8.8.8.8 success, VLAN20 blocked | `screenshots/vlan10-ping-test.png` |
| VLAN20 — SSH to VLAN10 success,ping VLAN10 blocked, 8.8.8.8 success | `screenshots/vlan20-ping-and-ssh-test.png` |

---

## What I learned

- How IP forwarding works at the kernel level and why it's the prerequisite for any routing
- The difference between stateful and stateless packet filtering — why `ESTABLISHED,RELATED` is essential
- iptables rule ordering — rules are evaluated top to bottom, first match wins. The `ESTABLISHED,RELATED` accept must come before any DROP rules or return traffic gets blocked
- NAT with MASQUERADE — how a private subnet reaches the internet through a single public-facing interface
- Why a default DROP policy (`-P FORWARD DROP`) is the correct baseline — explicit allow is safer than explicit deny on an otherwise-permissive chain
- VLAN segmentation as a containment strategy — a compromised machine in VLAN10 cannot laterally move to VLAN20 regardless of what it tries