# ARP Spoofing & MITM Lab — BetweenUs

A hands-on study of ARP (Address Resolution Protocol) spoofing as a foundation for man-in-the-middle (MITM) attacks on local networks, built around [BetweenUs](https://github.com/o-sec/BetweenUs), a lightweight Python ARP-poisoning tool.

This repo documents the project: why ARP is exploitable, how the attack works end to end, and what it reveals once positioned in the traffic path — run entirely against devices I own on an isolated lab network.

> ⚠️ **For education and authorized testing only.** Intercepting traffic on a network or devices you do not own, or do not have explicit written authorization to test, is illegal in most jurisdictions (in the U.S., under the Computer Fraud and Abuse Act). Everything in this repo was run on an isolated personal lab network with no third-party traffic involved.

## Table of contents

- [Background](#background-why-arp-is-exploitable)
- [Threat model](#threat-model-from-arp-poisoning-to-mitm)
- [Tool architecture](#tool-architecture-betweenus)
- [How it works](#how-it-works)
- [Lab walkthrough](#lab-walkthrough)
- [Traffic analysis](#traffic-analysis)
- [Detection & defenses](#detection--defenses)
- [References](#references)

## Background: Why ARP Is Exploitable

ARP (RFC 826) maps a known IP address to the MAC address a switch needs to deliver an Ethernet frame on a local segment:

1. A host broadcasts an **ARP request**: *"Who has 192.168.1.1? Tell 192.168.1.50."*
2. The IP's owner replies with an **ARP reply** containing its MAC address.
3. Every host caches IP → MAC mappings (the **ARP cache**) to avoid repeating the lookup.

Designed in 1982 for a trusted LAN, ARP has no authentication, no sequence numbers, and no verification step:

- **Replies are trusted unconditionally** — even unsolicited ones (*gratuitous ARP*), and the latest reply overwrites the cache.
- **There's no source verification** — nothing stops a host from claiming an IP it doesn't own.
- **ARP is link-local only** — it never crosses a router, so the attack is confined to one broadcast domain (switch/VLAN/Wi-Fi segment). This is the "one-hop rule" behind every classic coffee-shop or shared-office-LAN MITM scenario.

## Threat Model: From ARP Poisoning to MITM

ARP spoofing is the technique; MITM is the position it buys. The only precondition is sharing a broadcast domain with both targets — no Wi-Fi cracking or endpoint compromise required.

The attacker runs a two-sided spoof:

- To **Victim A**: forged ARP replies claiming *"I am Victim B's MAC address."*
- To **Victim B**: forged ARP replies claiming *"I am Victim A's MAC address."*

Both victims update their caches to point at the attacker's MAC. With IP forwarding enabled on the attacker's host, traffic keeps flowing transparently — this is what makes ARP spoofing a *silent* threat rather than a denial-of-service.

## Tool Architecture: BetweenUs

BetweenUs is a single `BetweenUs.py` script, deliberately minimal:

| Flag | Argument | Purpose |
|---|---|---|
| `-t1` | `TARGET1` | IP of the first victim (e.g., a client machine) |
| `-t2` | `TARGET2` | IP of the second victim (e.g., the default gateway) |
| `-i` | `IFACE` *(optional)* | Interface to attack from; defaults to the active one |
| `-h` | — | Help text |

```bash
git clone https://github.com/o-sec/BetweenUs.git
cd BetweenUs
sudo python3 BetweenUs.py -t1 <victim_ip> -t2 <gateway_ip>
```

## How It Works

Three stages, repeated until stopped:

1. **Reconnaissance** — resolve each target's real MAC via a legitimate ARP request, so forged replies look structurally authentic.
2. **Poisoning loop** — continuously send forged ARP replies to both targets at an interval short enough to outlast the OS's ARP cache timeout, so each believes the attacker's MAC is its peer's.
3. **Forwarding** — enable IP forwarding (`/proc/sys/net/ipv4/ip_forward` on Linux) so poisoned traffic is relayed on to its real destination instead of dead-ending at the attacker.

A well-behaved spoofer also restores both victims' real ARP mappings on exit (`Ctrl-C`), returning the network to normal rather than leaving a stale cache behind.

## Lab Walkthrough

Run on an isolated home lab network — three devices on one subnet: *Victim*, *Gateway* (router), and *Attacker* (BetweenUs host).

**Setup**
1. Confirmed baseline connectivity; recorded each device's legitimate ARP entry (`arp -a`) before the attack.

**Attack**
2. Ran `BetweenUs.py -t1 <victim_ip> -t2 <gateway_ip>` from the attacker machine.
3. Re-checked `arp -a` on the victim — the gateway's IP now resolved to the attacker's MAC.
4. Confirmed connectivity was preserved (IP forwarding on): the victim's browsing continued uninterrupted — the mark of a successful, stealthy MITM versus a crude DoS.

**Teardown**
5. Stopped the tool and verified `arp -a` reverted to the legitimate gateway MAC.

## Traffic Analysis

ARP poisoning only buys *position* — it doesn't expose data by itself. With the attacker's interface in the data path, a passive capture (Wireshark / `tcpdump`) shows:

- **Plaintext protocols are fully readable** — unencrypted HTTP, plain FTP/Telnet credentials, DNS queries.
- **TLS/HTTPS is not readable as plaintext** — the attacker sees ciphertext, IPs, SNI hostnames, and timing, but not content, unless a further attack (cert substitution, downgrade) is layered on top.

This is also the clearest practical case for HTTPS-everywhere and VPN use on shared networks: both remove the value of a won MITM position.

## Detection & Defenses

| Layer | Defense | How it helps |
|---|---|---|
| Host | Static ARP entries for critical IPs | Refuses conflicting replies for that IP |
| Host | ARP-watch monitoring (`arpwatch`, custom sniffers) | Flags changing IP↔MAC mappings in real time |
| Switch | Dynamic ARP Inspection (DAI) + DHCP snooping | Drops forged replies at the switch port |
| Switch | Port security | Locks which MACs can appear on a port |
| Network | VLAN segmentation | Shrinks the attacker's reachable broadcast domain |
| Transport | TLS/HTTPS everywhere | Makes captured traffic unreadable without a further attack |
| Network | VPN on untrusted networks | Encrypts traffic before it reaches the local segment |

On flat home/SOHO networks without a managed switch, detection comes down to the host-level techniques above — which is exactly why ARP spoofing stays practical on public and BYOD networks.

## References

- [o-sec/BetweenUs](https://github.com/o-sec/BetweenUs) — the tool studied here
- [ARP Spoofing & Sniffing: The Silent Threat on Your Local Network](https://medium.com/@cyberarmy629/arp-spoofing-sniffing-the-silent-threat-on-your-local-network-30d7397e80e8)
- [Recreating a MITM Coffee Shop Scenario](https://bedinger.net/blog/recreating-mitm-coffee-shop-scenario)
- [Northeastern CCS — MITM / ARP Lab](https://www.ccs.neu.edu/home/amirali/teaching/Summer14/lab/mitm.html)

## Disclaimer

This project is for educational purposes. I am not responsible for any misuse of the information or techniques described here. Only test on networks and devices you own or are explicitly authorized to test.
