# Home Lab 01: Home Network Hardening

Type: Lab write-up  
Channel: GitHub portfolio  
Status: ready_for_review  
Slug: home-lab-01-network-hardening

Author: Cory Franek  
Completed: November 7, 2026 (America/Phoenix)

## Overview

My home network mixed personal computers, cameras, streaming devices, and older smart-home hardware on the same trusted LAN. I wanted to reduce that exposure and cut down on advertising and tracking traffic without breaking everyday use.

I inventoried the network, hardened the router, created an isolated guest network for IoT, and installed AdGuard Home on a Mac mini. This was a first pass, not a claim that the network is fully segmented or that tracking has stopped.

## Goals

- Find out what was connected and flag devices that need attention.
- Reduce easy paths into the router and trusted devices.
- Give IoT devices a separate wireless network.
- Block known advertising, tracking, and malicious domains at DNS.
- Test normal browsing and blocked domains before calling the setup usable.

## Environment

- Netgear Nighthawk RAX30 consumer router.
- Home LAN represented only as `192.168.1.0/24`.
- 30 inventory entries, including the router: one recorder, six cameras, 14 other IoT devices, seven trusted devices, and one unidentified device.
- Cameras: five wired Hanwha Wisenet cameras and one Aosu cloud camera; an Alarm.com NVR records video.
- IoT includes four end-of-life Sonos ZP100 speakers, Chromecast, two Apple TVs, an onn Roku TV, a Samsung device, Hubspace, Roomba, Nintendo, and two thermostats.
- Trusted devices include iPhones, iPads, a MacBook Pro, and the Mac mini used as the DNS host.
- AdGuard Home v0.107.80 runs natively on the Mac mini as a service.

The [sanitized inventory](docs/inventory-sanitized.csv) uses category labels instead of hostnames. It contains no device IP addresses, MAC addresses, serial numbers, SSIDs, passwords, owner fields, or room locations. Target zones are plans, not proof that every device has moved.

## What I did

### 1. Discovery and inventory

I started with the router's client list, then used a ping/ARP sweep from the Mac mini to cross-check devices visible on the local network. I grouped the entries into trusted devices, cameras, IoT, core infrastructure, and an unknown device.

This was asset discovery, not a vulnerability scan. Sleeping devices, isolated clients, and devices that don't answer probes can be missed. The router's connection labels are observations, not independently verified radio capabilities.

### 2. Router hardening

I updated the router firmware, changed the administrator and Wi-Fi passwords, and disabled UPnP. I checked the WPS controls too. This firmware doesn't expose a full WPS off switch; PIN pairing wasn't exposed, but push-button pairing remained available. I haven't recorded WPS as fully disabled.

### 3. IoT isolation

I created a guest wireless network using WPA2-PSK with AES for compatibility with older devices. Guest-to-LAN access and guest-to-guest access were disabled through the router's combined isolation setting.

Creating the network is complete. Moving every IoT device and testing the isolation boundary are separate tasks still to finish. The wired cameras don't become isolated just because a guest Wi-Fi network exists. Casting, speaker discovery, and local control may also stop working across that boundary.

### 4. DNS filtering

I installed AdGuard Home on the Mac mini and configured a DHCP reservation so the router keeps assigning it the same address. The router uses AdGuard as primary DNS and Quad9 as secondary DNS.

Inside AdGuard, I configured Quad9 and Cloudflare DNS-over-HTTPS upstreams and enabled DNSSEC validation. Encrypted upstream DNS protects the lookup transport between AdGuard and the resolver. It doesn't hide all browsing activity or make either resolver unable to see queries.

I enabled these blocklists:

- AdGuard DNS filter
- HaGeZi Pro
- OISD Big
- HaGeZi Apple Tracker
- HaGeZi Threat Intelligence Feeds

The displayed list counts totaled about 2.1 million rules at setup time. Lists overlap, so that isn't a count of unique blocked domains or a measure of threats prevented.

I also applied custom rules for Samsung advertising and ACR, Roku advertising and logging, Amazon device metrics, Alphonso, and Samba TV:

```text
! Samsung advertising and viewing telemetry
||samsungads.com^
||samsungacr.com^
||samsungcloudsolution.com^
! Roku advertising and logging
||ads.roku.com^
||logs.roku.com^
||scribe.logs.roku.com^
! Amazon device metrics
||device-metrics-us.amazon.com^
||device-metrics-us-2.amazon.com^
! Viewing telemetry
||alphonso.tv^
||samba.tv^
```

These are the rules applied during this lab, not a universal recommendation. Broad vendor-domain blocks, especially `samsungcloudsolution.com`, can affect legitimate device services. If something breaks, I'll check the query log and make a narrow exception rather than turn off all filtering.

## Results

I used `dig` against the AdGuard resolver to check both allowed and blocked domains. This table summarizes the recorded checks; it isn't a raw terminal transcript.

| Domain | Observed result |
| --- | --- |
| `example.com` | Resolved normally |
| `apple.com` | Resolved normally |
| `netflix.com` | Resolved normally |
| `doubleclick.net` | Returned `0.0.0.0` |
| `samsungads.com` | Returned `0.0.0.0` |
| `logs.roku.com` | Returned `0.0.0.0` |
| `ads.roku.com` | Returned `0.0.0.0` |
| `samba.tv` | Returned `0.0.0.0` |

That confirms DNS filtering for those queries through AdGuard. It doesn't prove that every device uses it or that all ads are blocked. I haven't measured a before-and-after traffic reduction or broker data collection.

## Lessons and limits

**A server needs a stable address.** After the firmware reboot, DHCP reassigned the Mac mini's address and the original setup link stopped working. The DHCP reservation fixes that dependency. If I change its network interface or private-address behavior, I'll need to check the reservation again.

**Secondary DNS is an availability tradeoff, not strict failover.** Quad9 provides another resolver if the Mac mini is unavailable, but clients and routers can use secondary DNS while primary DNS is healthy. That can bypass AdGuard filtering. The outage behavior still needs testing. A second filtering resolver would be a better long-term answer.

**Guest isolation can also block the DNS server.** The Mac mini is on the trusted LAN. Isolated guest clients may not be able to reach it directly. Router DNS forwarding might provide a working path, but I still need to test this from an actual guest client without opening general guest-to-LAN access.

**DNS blocking isn't an advertising cloak.** Devices can use hardcoded resolvers, encrypted DNS, VPNs, IPv6 DNS settings, or direct connections. Tracking on the same domains as useful content can remain. Account activity, device identifiers, and server-side tracking aren't solved by a DNS blocklist.

**Consumer isolation has limits.** This router doesn't provide the VLAN controls I need for separate wired camera, IoT, management, and trusted networks. The guest network is a useful interim step, not the final architecture.

## Security+ mapping

I mapped the work to CompTIA Security+ SY0-701 domains as a study aid, not a certification claim.

| Domain | Work in this lab |
| --- | --- |
| 1. General Security Concepts | Defense in depth, least privilege through guest isolation, and availability versus privacy tradeoffs |
| 2. Threats, Vulnerabilities, and Mitigations | End-of-life hardware findings, firmware maintenance, disabling UPnP, and reducing exposure |
| 3. Security Architecture | Trusted/IoT separation, DNS security design, and identifying a single filtering-host dependency |
| 4. Security Operations | Asset inventory, configuration changes, DNS logging, blocklist maintenance, and functional validation |
| 5. Security Program Management and Oversight | Documenting residual risks, keeping sensitive inventory private, and prioritizing remediation |

## Next steps

1. Finish moving wireless IoT devices and test local control, casting, guest isolation, and guest DNS resolution.
2. Move the wired cameras and recorder off the trusted LAN. Review their firmware and credentials separately; that work isn't claimed complete here.
3. Retire or replace the four Sonos ZP100 speakers.
4. Identify the unknown device and confirm the Samsung device's role.
5. Check IPv6 DNS, browser/device encrypted DNS, and actual resolver paths for bypasses.
6. Test a Mac mini outage, prevent sleep from interrupting DNS, and consider Ethernet plus a second filtering resolver.
7. Plan an OPNsense firewall with managed switching and VLANs for trusted devices, cameras, IoT, guests, and management. Start with default-deny inter-VLAN rules and explicit service exceptions.
8. Add Suricata for network detection and Wazuh for endpoint monitoring after segmentation is working. Neither is deployed in this lab yet.

## Notes for review

This write-up is based on the November 7 setup record, DNS test summary, and local inventory. Raw command output and screenshots aren't included. Before publication, review any claims that need fresh evidence, especially guest DNS reachability, actual device migrations, and outage behavior. No secrets or original inventory identifiers belong in this repository.
