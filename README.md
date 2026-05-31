# ExpressRoute Monitoring & DR — A Practical Field Guide

![Azure](https://img.shields.io/badge/Azure-ExpressRoute-0078D4)
![Topic](https://img.shields.io/badge/Topic-Monitoring%20%26%20DR-blue)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

Detailed, field-tested guidance for monitoring **ExpressRoute circuits + gateways** and designing for **high availability and disaster recovery** — so you catch problems before users do and don't get surprised by Microsoft maintenance, provider rate limiting, route-table saturation, or a single peering location going dark.

## Table of Contents

- [Why this matters](#why-this-matters)
- [1. Monitoring foundation — metrics, logs, and alerts](#1-monitoring-foundation--metrics-logs-and-alerts)
  - [Key metrics to alert on](#key-metrics-to-alert-on)
  - [Microsoft maintenance & the AS-PATH prepend gotcha](#microsoft-maintenance--the-as-path-prepend-gotcha)
  - [Provider rate limiting](#provider-rate-limiting)
  - [CLI / PowerShell — pull the current state](#cli--powershell--pull-the-current-state)
- [2. Traffic visibility — Traffic Collector, Network Insights, Connection Monitor](#2-traffic-visibility--traffic-collector-network-insights-connection-monitor)
- [3. Route limits — circuit, gateway, and the 1k outbound cap](#3-route-limits--circuit-gateway-and-the-1k-outbound-cap)
- [4. High availability + DR design](#4-high-availability--dr-design)
  - [BFD — sub-second failover](#bfd--sub-second-failover)
  - [Always run both BGP peers (MSEE1 + MSEE2)](#always-run-both-bgp-peers-msee1--msee2)
  - [Two distinct peering locations](#two-distinct-peering-locations)
  - [Bow-tie — one VNet, two circuits](#bow-tie--one-vnet-two-circuits)
- [5. Other best practices](#5-other-best-practices)
- [Validation commands cheat sheet](#validation-commands-cheat-sheet)
- [References](#references)
- [License](#license)

---

## Why this matters

ExpressRoute is the production circuit for most enterprise Azure workloads — file shares, AD, Kubernetes pulls, database replication, backup, you name it. When it has a bad day **without proper monitoring**, you find out from a P1 ticket rather than from a dashboard, and the time-to-recovery balloons.

Three classes of problems consistently bite teams that under-invest in ExR monitoring:

1. **Silent partial failure** — one of the two MSEE BGP peers goes down. The circuit looks "up" because the other peer is still forwarding, but you've lost half your redundancy and don't know it.
2. **Bandwidth surprise** — a backup job, replication burst, or a new chatty workload pushes the circuit toward its bandwidth ceiling. Provider rate-limiting kicks in, applications start dropping, and nobody knows why.
3. **Maintenance failover that doesn't fail over** — Microsoft prepends the MSEE under maintenance, but the customer has **local preference** configured on-prem, which beats AS-PATH in BGP best-path selection. Traffic stays pinned to the MSEE being worked on. Outage.

Everything in this guide is aimed at avoiding those three scenarios.

---

## 1. Monitoring foundation — metrics, logs, and alerts

Send the ExpressRoute **circuit** and **virtual network gateway** diagnostic data to a Log Analytics workspace, and put alerts on the metrics below via Azure Monitor.

### Key metrics to alert on

| Metric (scope) | Recommended threshold | Why |
| --- | --- | --- |
| `BgpAvailability` (circuit, **per peer**) | `< 100%` for `> 5m` → critical | Single-peer drop = lost redundancy |
| `ArpAvailability` (circuit, **per peer**) | `< 100%` for `> 5m` → critical | L2 health to the MSEE |
| `BitsInPerSecond` / `BitsOutPerSecond` (circuit) | `> 80%` of circuit BW → warning; `> 95%` → critical | Avoid rate-limit-induced drops |
| `BitsInPerSecond` / `BitsOutPerSecond` (ER GW) | `> 80%` of GW SKU → warning | GW data-plane saturation (if FastPath not in use) |
| `CountOfRoutesAdvertisedToPeer` (ER GW) | `> 850` (of the 1,000 outbound cap) → warning | Headroom before the GW→MSEE hard cap |
| `CountOfRoutesLearnedFromPeer` (ER GW) | `> 80%` of SKU limit (4k Std / 9.5k with ErGwScale) → warning | Hub route-table saturation |
| `FastPathRoutesCount` (ER GW, if FastPath) | trend | Validate FastPath programming |
| Service Health — ExpressRoute maintenance | every notification | So you can correlate spikes / failovers |
| Resource Health — Circuit / GW state | any non-`Available` | Microsoft-detected platform issue |

**Wire each alert into an Action Group** (email, SMS, Teams webhook, ITSM connector) so it actually wakes someone up.

> **Split BGP/ARP alerts per peer.** The defaults often roll them up across both MSEEs — that masks single-peer failures. In the alert rule, use a dimension/split on **PeeringType** (`AzurePrivatePeering`) **and** **Peer** (`Primary`/`Secondary`) so you get a row per peer.

Docs:
- [Custom route alerts](https://learn.microsoft.com/azure/expressroute/how-to-custom-route-alert)
- [Maintenance alerts](https://learn.microsoft.com/azure/expressroute/maintenance-alerts)

### Microsoft maintenance & the AS-PATH prepend gotcha

When Microsoft does platform maintenance on an MSEE, the affected MSEE **prepends `12076` x8** to its advertisements so on-prem BGP makes the *other* MSEE the best path and traffic drains gracefully. This relies on the **AS-PATH length** tiebreaker.

> ⚠️ **If you have `local-preference` set on the on-prem side that pins traffic to a specific MSEE, local-pref WINS** in the BGP best-path algorithm — it comes *before* AS-PATH length. AS-PATH prepending will **not** drain traffic, and you'll take a hit during maintenance.

**What to do:**
- Don't set local-pref to deliberately pin to one MSEE under steady state.
- If you must steer traffic between MSEEs, do it with **MED** or **AS-PATH prepending on the on-prem side** (both lose to MS's drain prepend).
- Subscribe to **Service Health → ExpressRoute** maintenance notifications and review your BGP policy before each maintenance window.

Docs: [Maintenance alerts](https://learn.microsoft.com/azure/expressroute/maintenance-alerts)

### Provider rate limiting

ExpressRoute provider circuits enforce **per-circuit bandwidth at the MSEE**. If you burst above the purchased rate, the provider rate-limits the excess — packets drop, retransmits surge, and apps slow down. This is *not* visible as a BGP/ARP problem.

- Alert on `BitsInPerSecond` / `BitsOutPerSecond` early (80% / 95%).
- Look at **percentiles**, not just averages — a 1-minute average can hide micro-bursts.
- Cross-reference with **ExpressRoute Traffic Collector** (next section) to find the workload causing the burst.

Docs: [Understanding provider rate limiting](https://learn.microsoft.com/azure/expressroute/provider-rate-limit)

### CLI / PowerShell — pull the current state

**Azure CLI**

```bash
# Circuit overview
az network express-route show -g <rg> -n <circuit> -o table

# BGP/ARP tables (per device path: primary vs secondary MSEE)
az network express-route list-route-tables -g <rg> -n <circuit> \
  --peering-name AzurePrivatePeering --path primary -o table
az network express-route list-route-tables -g <rg> -n <circuit> \
  --peering-name AzurePrivatePeering --path secondary -o table
az network express-route list-route-tables-summary -g <rg> -n <circuit> \
  --peering-name AzurePrivatePeering --path primary
az network express-route list-arp-tables -g <rg> -n <circuit> \
  --peering-name AzurePrivatePeering --path primary

# Gateway-side BGP
az network vnet-gateway list-bgp-peer-status   -g <rg> -n <er-gw>
az network vnet-gateway list-learned-routes    -g <rg> -n <er-gw>
az network vnet-gateway list-advertised-routes -g <rg> -n <er-gw> --peer <peer-ip>

# Live metric (single sample)
az monitor metrics list --resource <circuit-id> --metric BgpAvailability,ArpAvailability \
  --interval PT1M
```

**Azure PowerShell**

```powershell
Get-AzExpressRouteCircuit -ResourceGroupName <rg> -Name <circuit>
Get-AzExpressRouteCircuitRouteTable        -ResourceGroupName <rg> -ExpressRouteCircuitName <circuit> -PeeringType AzurePrivatePeering -DevicePath Primary
Get-AzExpressRouteCircuitRouteTableSummary -ResourceGroupName <rg> -ExpressRouteCircuitName <circuit> -PeeringType AzurePrivatePeering -DevicePath Primary
Get-AzExpressRouteCircuitARPTable          -ResourceGroupName <rg> -ExpressRouteCircuitName <circuit> -PeeringType AzurePrivatePeering -DevicePath Primary
Get-AzVirtualNetworkGatewayBgpPeerStatus   -ResourceGroupName <rg> -VirtualNetworkGatewayName <er-gw>
Get-AzVirtualNetworkGatewayLearnedRoute    -ResourceGroupName <rg> -VirtualNetworkGatewayName <er-gw>
Get-AzVirtualNetworkGatewayAdvertisedRoute -ResourceGroupName <rg> -VirtualNetworkGatewayName <er-gw> -Peer <peer-ip>
```

For day-to-day KQL queries against the diagnostic logs (BGP route changes, flow-level top talkers, etc.), see the companion repo: [adtork/ARG-Kusto-Queries](https://github.com/adtork/ARG-Kusto-Queries) — `queries/logs/expressroute-traffic-collector-*.kql`.

---

## 2. Traffic visibility — Traffic Collector, Network Insights, Connection Monitor

Metrics tell you *how much* and *whether it's up*. To answer **"why is bandwidth high?"** and **"is the path actually working end-to-end?"**, layer in:

### ExpressRoute Traffic Collector (ERTC)

Sends **IPFIX flow records** from the MSEE into Log Analytics. Lets you ask:

- Who are my top talkers right now? (5-tuple, bytes, packets)
- Which Azure destinations are pulling the most bytes from on-prem?
- Are there long-lived elephant flows that need offload?
- What's the protocol mix (TCP/UDP/SMB/HTTPS/etc.)?

Stand it up on every production circuit. Sample KQL is in [adtork/ARG-Kusto-Queries](https://github.com/adtork/ARG-Kusto-Queries/tree/main/queries/logs):
- `expressroute-traffic-collector-top-flows.kql`
- `expressroute-top-destinations-by-bytes.kql`

Docs: [Configure Traffic Collector](https://learn.microsoft.com/azure/expressroute/how-to-configure-traffic-collector)

### ExpressRoute Network Insights

A pre-built **Azure Monitor workbook** that gives you topology, metrics, and BGP/ARP state in one pane per circuit. Zero config — just open it from the circuit blade.

Docs: [Network Insights](https://learn.microsoft.com/azure/expressroute/expressroute-network-insights)

### Connection Monitor (Network Watcher)

Active synthetic probes from an on-prem agent → Azure endpoints (or vice-versa). Measures **loss, latency, jitter** continuously and alerts when SLOs slip.

- Deploy agents on the on-prem head-end LAN segments that actually use ER.
- Test against multiple Azure destinations (PaaS PE, IaaS VM, storage PE).
- Use this to **prove the path is healthy** during a maintenance window, not just that the circuit metrics say "up".

Docs: [Configure Connection Monitor for ExpressRoute](https://learn.microsoft.com/azure/expressroute/how-to-configure-connection-monitor)

---

## 3. Route limits — circuit, gateway, and the 1k outbound cap

You can monitor every metric in the world and still get bitten by a quiet route-table saturation. The hard caps to track:

| Where | Limit | Metric / how to check |
| --- | --- | --- |
| **On-prem → MSEE** (advertised by on-prem, accepted by MS) | **4,000 IPv4** (Std SKU) / **10,000 IPv4** (Premium) | Routes table summary on the circuit |
| **ER GW → MSEE** (advertised by Azure outbound to on-prem) | **1,000 IPv4** — hard cap | `CountOfRoutesAdvertisedToPeer` on the ER GW |
| **MSEE → ER GW** (learned by the ER GW) | **4,000** default / **9,500** with `ErGwScale` | `CountOfRoutesLearnedFromPeer` |
| **vWAN hub route table** | **10,000** routes total per hub | See companion repo |

> The **1,000 IPv4 outbound from the ER GW to MSEE** is the cap most teams overlook. Every VNet spoke address space injected into the hub consumes a slot. With 500 spokes on a hub, you're already at 500 — leaving little room for hub address spaces, additional connections, etc.

For the full contention map across ER, S2S BGP, SD-WAN NVA, and VNet peering — plus a mitigation playbook (summarization, route-maps, disjoint/LPM) — see the companion article: **[adtork/vwan-routing-limits](https://github.com/adtork/vwan-routing-limits)**.

---

## 4. High availability + DR design

Monitoring tells you what broke. **Design** decides whether the break causes an outage.

### BFD — sub-second failover

ExpressRoute supports **Bidirectional Forwarding Detection** on private peering. Standard BGP hold timer is **180s** — that's 3 minutes of black-holed traffic on a peer failure. BFD drops detection to **sub-second**.

- Enable on both on-prem and Azure sides of every private peering.
- Especially critical if you have **active/active circuits** where you depend on fast convergence.

Docs: [Configure BFD on ExpressRoute](https://learn.microsoft.com/azure/expressroute/expressroute-bfd)

### Always run both BGP peers (MSEE1 + MSEE2)

Every ExpressRoute circuit has **two MSEEs by design** (primary + secondary, each in a separate Microsoft Enterprise Edge router). Make sure **both BGP sessions are up at all times**:

- Both on-prem CEs (or both sub-interfaces on a single CE if you only have one) must peer to both MSEEs.
- Both peers should accept and advertise the **same prefix set** — true active/active.
- Alert on `BgpAvailability < 100%` **per peer** (see the [metrics table](#key-metrics-to-alert-on)).

Running only one BGP session "to keep it simple" is the #1 cause of single-peer outages becoming customer-facing outages.

### Two distinct peering locations

A single peering location (e.g., one carrier hotel in one metro) is a **single facility failure domain** — power, cooling, the carrier's gear, the building. Real HA means:

- Two circuits in **two physically diverse peering locations** (different metros where possible, or at minimum different buildings with diverse fiber).
- Each circuit terminating into the same Azure region (or paired regions) via its own ER GW or vWAN hub.
- Both circuits active by default; on-prem BGP ECMPs across them (or you steer with weight / local-pref / AS-PATH as needed).

Docs: [Designing for HA with ExpressRoute](https://learn.microsoft.com/azure/expressroute/designing-for-high-availability-with-expressroute) · [Designing for DR](https://learn.microsoft.com/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering)

### Bow-tie — one VNet, two circuits

For VNets that **must survive a peering-location failure**, connect the **same VNet (or vHub) to both circuits**. Then if peering POP #1 goes dark, the VNet is still reachable via circuit #2.

- **Classic VNet:** both circuits' connection objects point at the same ER GW. Azure ECMPs by default across both circuits.
- **vWAN:** ECMP across multiple circuits to the **same hub** requires a **no-op (dummy) route-map** on the connections to actually program ECMP. See [vWAN route-maps](https://learn.microsoft.com/azure/virtual-wan/route-maps-about).
- To **prefer one circuit** over the other (active / standby): set connection **weight** higher on the preferred connection, set **local-pref** outbound from on-prem, or **AS-PATH prepend** the less-preferred path (shortest AS-PATH wins).

---

## 5. Other best practices

- **Use Premium SKU** if you need: > 4,000 prefixes from MSEE, > 10 VNet links per circuit, or **Global Reach** (cross-region on-prem traffic via Microsoft backbone).
- **Enable `ErGwScale`** on the ER GW if you need to learn > 4,000 routes (raises to 9,500). Plan **before** you hit the wall.
- **Enable FastPath** on UltraPerformance / ErGw3AZ / vWAN ER GW so inbound flows bypass the GW data plane and you get full circuit bandwidth. Requires supported VM SKUs in the destination VNet.
- **Right-size the GW from day 1.** You can scale **up** non-disruptively but **cannot scale down** without delete + recreate. Same rule for circuit bandwidth — upgrades are non-disruptive, downgrades are a re-create.
- **Diagnostic settings everywhere.** Send `PeeringRouteLog` and metrics to Log Analytics — both the circuit and the ER GW. Without logs you have no history for post-mortems.
- **Encrypt if required without falling back to IPsec-over-ER.** Use **MACsec on ExpressRoute Direct** (10G / 100G ports you own) — see [MACsec for ER Direct](https://learn.microsoft.com/azure/expressroute/expressroute-howto-macsec). Or push encryption to the app layer (TLS / mTLS). Avoid IPsec-over-ER unless absolutely mandated — it caps you at ~10 Gbps aggregate and blocks FastPath. (See companion: [ipsec-over-er-to-er-only](https://github.com/adtork/ipsec-over-er-to-er-only).)
- **Plan + test DR drills.** Schedule a quarterly drill where you fail one circuit out of service (drop the BGP peer on the on-prem side) and validate traffic shifts cleanly to the other circuit. **An untested failover is not a failover.**
- **Capacity headroom.** Provision for the steady-state burst, not the steady-state average. Backup windows, monthly closes, and AD replication can easily double normal load.
- **Watch Azure Advisor** for ER recommendations (route limit warnings, BGP-only peerings missing redundancy, etc.).
- **Tag and document** every circuit + GW with: owner, peering location, provider, service key, contract end date, and the team's runbook URL. When you're 20 minutes into an outage you don't want to be hunting for the service key.
- **Service Health alerts** on the ExpressRoute resource type — Microsoft posts planned maintenance and incidents here first.

---

## Validation commands cheat sheet

### On-prem (Cisco IOS / IOS-XE)

```text
! BGP neighbor + per-peer state
show ip bgp vpnv4 all summary
show ip bgp vpnv4 all neighbors <msee-peer-ip>

! BFD sessions
show bfd neighbors

! Routes we're sending Azure
show ip bgp vpnv4 all neighbors <msee-peer-ip> advertised-routes
! Routes Azure is sending us
show ip bgp vpnv4 all neighbors <msee-peer-ip> routes

! Detect MS maintenance drain — look for AS 12076 prepended ~8x
show ip bgp vpnv4 all <prefix> | include AS_PATH
```

### Azure CLI — alert rules

```bash
# Per-peer BGP availability alert (primary)
az monitor metrics alert create \
  -g <rg> -n "ER-<circuit>-BGP-Primary-Down" \
  --scopes <circuit-id> \
  --condition "avg BgpAvailability < 100 where PeeringType includes AzurePrivatePeering and Peer includes Primary" \
  --window-size 5m --evaluation-frequency 1m \
  --severity 1 --action <action-group-id>

# Bandwidth ceiling alert (95% of a 10 Gbps circuit = 9.5e9 bits/s)
az monitor metrics alert create \
  -g <rg> -n "ER-<circuit>-Egress-95pct" \
  --scopes <circuit-id> \
  --condition "avg BitsOutPerSecond > 9500000000" \
  --window-size 5m --severity 2 --action <action-group-id>

# GW outbound route count headroom (warn at 850 of the 1k cap)
az monitor metrics alert create \
  -g <rg> -n "ER-GW-<gw>-OutboundRoutes-850" \
  --scopes <er-gw-id> \
  --condition "avg CountOfRoutesAdvertisedToPeer > 850" \
  --window-size 15m --severity 2 --action <action-group-id>
```

---

## References

- [ExpressRoute monitoring, metrics, and alerts](https://learn.microsoft.com/azure/expressroute/expressroute-monitoring-metrics-alerts)
- [Custom route alerts for ExpressRoute](https://learn.microsoft.com/azure/expressroute/how-to-custom-route-alert)
- [Maintenance alerts](https://learn.microsoft.com/azure/expressroute/maintenance-alerts)
- [Provider rate limiting](https://learn.microsoft.com/azure/expressroute/provider-rate-limit)
- [Configure ExpressRoute Traffic Collector](https://learn.microsoft.com/azure/expressroute/how-to-configure-traffic-collector)
- [ExpressRoute Network Insights](https://learn.microsoft.com/azure/expressroute/expressroute-network-insights)
- [Configure Connection Monitor for ExpressRoute](https://learn.microsoft.com/azure/expressroute/how-to-configure-connection-monitor)
- [Configure BFD on ExpressRoute](https://learn.microsoft.com/azure/expressroute/expressroute-bfd)
- [Designing for high availability with ExpressRoute](https://learn.microsoft.com/azure/expressroute/designing-for-high-availability-with-expressroute)
- [Designing for disaster recovery (ER private peering)](https://learn.microsoft.com/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering)
- [ExpressRoute FastPath](https://learn.microsoft.com/azure/expressroute/about-fastpath)
- [ExpressRoute Direct + MACsec](https://learn.microsoft.com/azure/expressroute/expressroute-howto-macsec)
- [ExpressRoute limits](https://learn.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-expressroute-limits)

### Companion repos

- **[adtork/vwan-routing-limits](https://github.com/adtork/vwan-routing-limits)** — route limits across ER, VPN, SD-WAN, and VNet peering + mitigation playbook
- **[adtork/ipsec-over-er-to-er-only](https://github.com/adtork/ipsec-over-er-to-er-only)** — migrating off IPsec-over-ER to ER-only + FastPath
- **[adtork/ARG-Kusto-Queries](https://github.com/adtork/ARG-Kusto-Queries)** — KQL queries for ARG inventory + Log Analytics troubleshooting (ER, AzFW, VPN, NSG, AppGW, PLS)

---

## License

This project is licensed under the MIT License. See [LICENSE](./LICENSE) for details.
