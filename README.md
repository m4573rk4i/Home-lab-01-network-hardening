# Home Lab 01: Home Network Hardening

Type: Lab write-up  
Channel: GitHub portfolio  
Status: ready_for_review  
Slug: home-lab-01-network-hardening

Author: Cory Franek  
Recorded: November 7, 2026, updated November 8, 2026 (America/Phoenix)

## Overview

My home network mixed personal computers, cameras, streaming devices, and smart-home hardware on the same trusted LAN. I wanted to reduce that exposure and cut down on advertising and tracking traffic while keeping everyday use working.

I inventoried the network, updated the router, started a guest network, and installed AdGuard Home on a Mac mini. A few devices are on that guest network. This is a first pass. The rest of this note covers what I configured, what I checked on November 8, and what is still open.

## Goals

- Find out what was connected and flag devices that need attention.
- Reduce easy paths into the router and trusted devices.
- Give IoT devices a separate wireless network.
- Block known advertising, tracking, and malicious domains at DNS.
- Test normal browsing and blocked domains before calling the setup usable.

## Environment

- Netgear Nighthawk RAX30 consumer router.
- Home LAN, written here only as `<home-lan>`. The prefix stays off this page.
- 30 inventory entries, including the router: one recorder, six cameras, 14 other IoT devices, seven trusted devices, and one unidentified device.
- Cameras: five wired Hanwha Wisenet cameras and one Aosu camera; an Alarm.com NVR records video.
- IoT includes four speakers the router listed with Sonos names, a Chromecast, two Apple TVs, an onn Roku TV, a Samsung TV/device, Hubspace, Roomba, Nintendo, and two thermostats. The Sonos model numbers are unverified. The router names are the only source I used.
- On November 8 the router client list showed five devices on the guest network: the Chromecast, the Samsung TV/device, both thermostats, and the living-room Apple TV. Some of the remaining streaming devices, speakers, and IoT were still on the main network. I did not match every inventory row to a network.
- Trusted devices include iPhones, iPads, a MacBook Pro, and the Mac mini used as the DNS host.
- AdGuard Home v0.107.80 runs natively on the Mac mini as a service.

The [sanitized inventory](docs/inventory-sanitized.csv) uses category labels instead of hostnames. It contains no device addresses, MAC addresses, serial numbers, SSIDs, passwords, owner names, or room locations. The living-room Apple TV is named in this write-up because that is how it appeared on the router list. The inventory has two Apple TV rows and no room field, so neither row is marked as that guest device. Target zones are plans. Guest membership is recorded only on rows whose notes say the router list showed that device on the guest network.

## What I did

### 1. Discovery and inventory

I started with the router's client list, then used a ping/ARP sweep from the Mac mini to cross-check devices visible on the local network. I grouped the entries into trusted devices, cameras, IoT, core infrastructure, and an unknown device.

This was asset discovery. Sleeping devices, isolated clients, and devices that don't answer probes can be missed. The router's connection labels are observations from that list.

### 2. Router hardening

I updated the RAX30 firmware, changed the administrator and Wi-Fi passwords, and turned UPnP off. I confirmed those three items on the router.

WPS is unverified, including the PIN and whether WPS is disabled.

### 3. Guest network

I created a guest wireless network using WPA2-PSK with AES so devices that need WPA2 could join. Guest-to-LAN access and guest-to-guest access were turned off through the router's combined isolation setting.

The guest network exists. On November 8 the router list put five devices on it: Chromecast, Samsung TV/device, two thermostats, and the living-room Apple TV. That list is membership, which is separate from a test of the isolation boundary. I have not checked casting, speaker discovery, or local control from those devices. Wired cameras stay on the trusted LAN because a guest Wi-Fi network does not move a wired port. Some streaming devices, speakers, and other IoT hardware were still on the main network.

### 4. DNS filtering

I installed AdGuard Home v0.107.80 on the Mac mini. I set a DHCP reservation so the router keeps assigning that Mac the same address, and I confirmed the reservation is there. The address itself is omitted; call it `<adguard-home>`.

On the router, AdGuard Home is the primary DNS server and Quad9 is the secondary. The router can send lookups to Quad9 while AdGuard Home is up. Any lookup that takes the Quad9 path skips filtering. This is an anytime bypass, separate from outage failover.

Inside AdGuard Home I saved and tested this upstream configuration at 17:19 America/Phoenix on November 8:

- Main upstreams, both over HTTPS: Quad9 at `https://dns.quad9.net/dns-query`, and Cloudflare at `https://cloudflare-dns.com/dns-query`.
- Fallback upstream: Quad9 over HTTPS.
- Bootstrap resolvers: Quad9's published IP addresses. The addresses are left out of this page.
- The router is an upstream only for private reverse DNS. General lookups go to Quad9 and Cloudflare. I did not build a general forwarding loop through the router.

Other resolver settings from that configuration:

- EDNS Client Subnet is disabled.
- DNSSEC is enabled. AdGuard Home asks the upstream resolvers to validate answers. Local signature validation on the Mac mini is outside this configuration.
- IPv6 answer suppression is off, so AAAA answers are allowed to come back.
- Plain DNS is enabled for clients on the LAN.
- I did not set up an inbound HTTPS or DoH listener.

The dashboard showed traffic on the encrypted upstreams. Plain DNS is still enabled for LAN clients, and there is no inbound DoH or HTTPS listener, so that evidence covers the upstream leg.

I enabled these blocklists:

- AdGuard DNS filter
- HaGeZi Pro
- OISD Big
- HaGeZi Apple Tracker
- HaGeZi Threat Intelligence Feeds

The displayed list counts totaled about 2.1 million rules at setup time. Lists overlap, so that figure is a rule count on those lists, and it is separate from the query snapshot later on this page.

I also applied these custom rules:

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

These are the rules written down for this lab. Removing the broad `samsungcloudsolution.com` rule is a proposal I have not carried out, so the rule is still in this list. A block that wide can affect ordinary Samsung device services. The November 8 query checks cover `logs.roku.com` on the paths in the next section. The other names in this block were outside those checks.

## What I checked on November 8

I ran read-only lookups two ways: through the router, and directly to AdGuard Home. The network stayed as it was. Command output and packet logs stay out of this repo.

| Name | IPv4 and AAAA, through the router | IPv4 and AAAA, direct to AdGuard Home |
| --- | --- | --- |
| `apple.com` | Resolved | Resolved |
| `doubleclick.net` | Blocked | Blocked |
| `logs.roku.com` | Blocked | Blocked |

The table is three names on two paths, with IPv4 and AAAA checked on each path. That is the query evidence I can point to. Guest clients, and every device I did not query from, are still an open question.

### Dashboard snapshot

At 17:07 America/Phoenix the AdGuard Home dashboard showed 45,293 total queries and 15,267 blocked, about 34% on this refresh. The same reading included 9,536 lookups for one Sentry ingest hostname. The hostname stays off this page. This snapshot is twelve minutes before the 17:19 upstream save, so I am using it as a statistics reading. The upstream settings stand on the configuration I saved and tested at 17:19.

I read the figures with these limits:

- Different refreshes of this screen did not share one exact total. I am keeping this copy as an approximate snapshot, separate from an exact total for every list.
- The Sentry lookups are retries against a diagnostics hostname. I could not tell which app sent them, and I am treating the traffic as diagnostics.
- The malware filter counter read 0. That counter leaves the question of malware open.
- The adult-content counter read 2. I am keeping that as a counter, with no inference about an intentional visit.
- The safe-search counter read 954.

The router proxies DNS, so AdGuard Home records the router as the client. Router attribution was 99.96% of queries. The log hides which device asked.

## Lessons and limits

**The DNS host needs a stable address.** After a firmware reboot, DHCP gave the Mac mini a different address and the original setup link stopped working. The reservation is in place now. If I change that Mac's interface, I will need to check the reservation again.

**Secondary DNS can skip the filter at any time.** Quad9 on the router is there so name resolution has somewhere to go. Clients and the router can use it while AdGuard Home is healthy. Lookups that take that path are unfiltered. I have not pulled the Mac mini offline to watch the failover behavior. A second filtering resolver would close this gap. That second resolver is not built.

**Guest membership and guest DNS are different questions.** Five devices were on the guest list. The Mac mini is on the trusted LAN, and the read-only checks came from the router path and from a direct query to AdGuard Home. Isolated guests may be unable to reach `<adguard-home>` directly. A guest that uses the router as its resolver would show up in the log as the router. I still have to test that from a guest client, with guest-to-LAN access left closed.

**The log hides the originating client.** With 99.96% of queries attributed to the router, I cannot use this dashboard to say which device, or which app, produced the Sentry retries or any other name.

**A DNS blocklist is one control.** Devices can use a hardcoded resolver, their own encrypted DNS, a VPN, or a direct connection. Tracking that shares a name with useful content can remain. Account activity and device identifiers are outside this filter. The custom list includes vendor telemetry names; the November 8 checks show a block for `doubleclick.net` and `logs.roku.com` on the two paths above.

**Full segmentation needs other hardware.** The design I want is a firewall that routes between VLANs, a managed switch, and an access point that understands those VLANs. None of that is installed. Netgear's Nighthawk article on VLAN tagging describes a bridge for an ISP VLAN tag group, aimed at IPTV, where tagged traffic can skip the router's NAT ([VLAN tagging setup for a Nighthawk router](https://kb.netgear.com/25724/VLAN-tagging-setup-for-Nighthawk-router)). That feature is the ISP/IPTV bridge. It is a poor fit for separate home LAN segments. Netgear lists the guest network among the features AP mode disables ([router mode and AP mode](https://kb.netgear.com/000061927/What-is-the-difference-between-router-mode-and-AP-mode)). Putting this RAX30 in AP mode would remove the guest separation this lab uses.

**An encrypted overlay is a different project.** Onclave describes TrustedPlatform as an overlay that gives devices a cryptographic identity and encrypts their traffic ([Onclave TrustedPlatform](https://onclavenetworks.com/trustedplatform/)). That overlay and a VLAN design are different projects. The public TrustedPlatform page does not document an RAX30 install, and I have not installed it.

## Related write-ups

Two later notes sit next to this one. I am not re-checking them here, and I am not copying their results in as findings I verified again on November 8.

- [Home Lab 02: New-Device Watcher](https://github.com/m4573rk4i/home-lab-02-new-device-watcher) describes a nightly script on the Mac mini that compares the main LAN with a list of known devices. That write-up says the script sees the main network and misses the guest network.
- [Home Lab 03: Network Vulnerability Scan](https://github.com/m4573rk4i/home-lab-03-vulnerability-scan) is a separate scan write-up from this LAN. This page does not adopt that repo's fix list as work I re-checked.

## Security+ mapping

I mapped the work to CompTIA Security+ SY0-701 domains as a study aid. This is not a certification claim.

| Domain | Work in this lab |
| --- | --- |
| 1. General Security Concepts | Defense in depth, a guest network with isolation enabled in the router and still untested from a client, and the availability tradeoff of a second resolver |
| 2. Threats, Vulnerabilities, and Mitigations | Firmware update, UPnP turned off, and exposure that guest Wi-Fi does not remove from wired cameras |
| 3. Security Architecture | A partial trusted/guest split, DNS filtering with a single filtering host, and a VLAN design that is still only a plan |
| 4. Security Operations | Asset inventory, configuration changes, DNS logging, blocklist choices, and the narrow November 8 checks |
| 5. Security Program Management and Oversight | Writing down residual risk, keeping the inventory sanitized, and ordering the work that is still open |

## Next steps

1. Move the remaining wireless IoT that should leave the main network, then test local control, casting, guest isolation, and DNS from a guest client.
2. In my private notes, match the living-room Apple TV to one inventory row. Leave room labels out of the public inventory.
3. Identify the Sonos hardware from the devices themselves before deciding whether to replace any of them.
4. Move the wired cameras and recorder off the trusted LAN. Firmware and credentials for those devices are separate work, and this page does not claim that work is done.
5. Identify the unknown device, and pin down the Samsung TV/device beyond the router label.
6. Watch for resolvers that skip AdGuard Home: the Quad9 secondary on the router, client encrypted DNS, and IPv6 DNS handed out by the router. Apple's AAAA answer resolved in the November 8 checks, with IPv6 suppression off, so AAAA filtering still depends on the lookup reaching AdGuard Home.
7. Test a Mac mini outage, keep sleep from taking DNS down, and consider Ethernet plus a second filtering resolver.
8. Build the VLAN path on purpose: a firewall with inter-VLAN routing, a managed switch, and a VLAN-aware access point. Leave the RAX30 VLAN/bridge feature and AP mode out of that design for the reasons above. Leave Onclave out unless a supported install path shows up.
9. Look at Suricata for network detection and Wazuh for endpoint monitoring after segmentation exists. Neither is deployed.

## Notes for review

This write-up uses the November 7 setup record and the November 8 router list, resolver settings, read-only query checks, and dashboard snapshot. Raw command output, screenshots, and query logs are not in this repository.

Before publication I kept these out: the LAN prefix, device and resolver addresses, MAC addresses, SSIDs, serial numbers, the household public address, family usernames, and the Sentry ingest hostname. The inventory file follows the same rule. Room wording for the living-room Apple TV stays in this note because that is the router-list label, and it stays out of the inventory rows.

Claims that still need a fresh check, if this page is updated again: WPS state, exact Sonos models, removal of the `samsungcloudsolution.com` rule, guest-client DNS, isolation behavior, which devices remain on the main network, and what the router does if the Mac mini is offline. I did not run new live tests while editing this page, and I did not change the home network from this edit.
