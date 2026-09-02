# Lab 02: ARP Spoofing Attack & Wireshark Analysis

## Objective
Demonstrate an ARP spoofing (ARP cache poisoning) attack against a target VM, capture 
the attack traffic with Wireshark, and confirm impact by inspecting the victim's poisoned 
ARP table. Conducted entirely within an isolated host-only virtual network with no 
internet-facing or external-network interfaces involved.

## Topology
Kali Linux — Attacker (192.168.56.20) <--> VirtualBox Host-Only Network (192.168.56.0/24) <--> Ubuntu Server — Victim (192.168.56.10)


## Tools Used
- `arpspoof` (dsniff package) — Kali
- Wireshark — packet capture and analysis
- `ip neigh` — inspecting the ARP/neighbor table on Ubuntu

## Steps

1. **Captured baseline traffic.** With the network idle, ran a Wireshark capture on Kali 
   filtered to `arp` to establish what normal ARP resolution looks like between the two VMs.

2. **Attempted spoofing Kali's own IP first.** Initially ran 
   `arpspoof -i eth0 -t 192.168.56.10 192.168.56.20`, targeting Ubuntu while "spoofing" 
   Kali's own address. Result: no visible impact, since Kali claiming to own 192.168.56.20 
   is true — there was no impersonation for Ubuntu's ARP table to reflect.

3. **Corrected the attack to target an unclaimed IP.** Re-ran the attack as:
  sudo arpspoof -i eth0 -t 192.168.56.10 192.168.56.1

   This makes Kali falsely claim ownership of `192.168.56.1`, an IP that does not 
   otherwise exist on the network — a true impersonation.

4. **Forced Ubuntu to resolve the spoofed IP.** Since Ubuntu's kernel only accepted the 
   forged reply for an address it had actively queried, ran `ping -c 3 192.168.56.1` on 
   Ubuntu while the attack was live to trigger an ARP request.

5. **Captured the attack traffic** in Wireshark on Kali, filtered to `arp`.

6. **Verified impact on the victim** by checking Ubuntu's ARP table with `ip neigh` 
   before and during the attack.

## Findings

**Ubuntu's `ip neigh` before the attack:**
Only a single legitimate entry existed, for Kali (`192.168.56.20`), with its real MAC 
address (`08:00:27:5a:87:bc`).

**Ubuntu's `ip neigh` during the attack:**
A new entry appeared for `192.168.56.1` — pointing to the *same* MAC address as Kali 
(`08:00:27:5a:87:bc`). Since `192.168.56.1` was never a real device on this network, this 
confirms Kali successfully convinced Ubuntu that a nonexistent gateway lives at Kali's MAC. 
Notably, the triggering `ping -c 3 192.168.56.1` showed 100% packet loss — Ubuntu never 
received a real reply, yet its ARP table was poisoned anyway by Kali's forged replies alone.

**Wireshark baseline capture (filtered `arp`):**
4 packets total — two clean request/reply pairs, one per direction, each with correct 
source/destination MACs. Normal, request-driven ARP behavior.

**Wireshark attack capture (filtered `arp`):**
49 of 69 total packets were ARP (71% of traffic) — a large spike compared to baseline. 
Every ARP packet showed the identical, repeating pattern: Kali broadcasting 
`"192.168.56.1 is at 08:00:27:5a:87:bc"` to Ubuntu roughly every 2 seconds, with **no 
corresponding request** preceding each reply. Unsolicited, repeated replies (rather than 
request-triggered ones) are a key signature of ARP spoofing.

**Kali terminal (`arpspoof` output):**
Confirmed the attack tool was continuously sending the forged replies directly to 
Ubuntu's MAC address, matching what was observed on the wire and in Ubuntu's poisoned table.

## Detection & Defense Notes
This lab highlights why ARP is inherently vulnerable — it has no built-in authentication, 
so any host can claim to own any IP on the local segment. Real-world mitigations include:
- **Static ARP entries** for critical infrastructure (gateway, servers) — prevents 
  overwrite by forged replies, though not scalable for large networks
- **Dynamic ARP Inspection (DAI)** — a switch-level feature that validates ARP packets 
  against a trusted binding table (commonly paired with DHCP snooping)
- **ARP monitoring tools** like `arpwatch`, which alert on unexpected IP-to-MAC changes
- **Network segmentation** — limits the blast radius of ARP spoofing to a single broadcast 
  domain

## Screenshots
- `screenshots/baseline-arp-filtered.png` — Normal ARP traffic before the attack
- `screenshots/arpspoof-terminal.png` — Attack command running on Kali
- `screenshots/arp-spoof-filtered.png` — Wireshark capture of the forged reply flood
- `screenshots/ubuntu-ip-neigh-before.png` — Ubuntu's clean ARP table
- `screenshots/ubuntu-ip-neigh-during.png` — Ubuntu's poisoned ARP table showing the spoofed entry
