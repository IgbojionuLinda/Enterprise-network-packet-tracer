# Enterprise Network Design and Implementation with Cisco Packet Tracer

## Project Overview

This project simulates an enterprise network with a Headquarters (HQ), a Branch office, and an Internet Service Provider (ISP). 
It was built in Cisco Packet Tracer to showcase enterprise LAN design, VLAN segmentation, inter-VLAN routing, DHCP, WAN connectivity, dynamic routing, 
BGP, route redistribution, and NAT/PAT.

The network contains three Cisco 2911 routers, two Cisco 2960-24TT switches, and 24 end-user PCs. Each office has four departments, with three PCs per department.

## Network Topology
The diagram below illustrates the enterprise network connecting the headquarters, branch office, and simulated ISP.
![Enterprise Network Topology]
(topology.png)

> **Project status:** DHCP, inter-VLAN routing, the HQ–Branch WAN link, OSPF, the eBGP neighbor relationship, and end-to-end ping from a Branch PC to the simulated ISP loopback were tested successfully. NAT/PAT commands were configured, and NAT translation table was successful after the test ping.

## Objectives

- Build a two-site enterprise network with HQ and Branch locations.
- Segment departments using VLANs.
- Configure router-on-a-stick inter-VLAN routing.
- Assign IP addresses automatically using DHCP.
- Connect HQ and Branch over a point-to-point WAN link.
- Use OSPF for internal routing.
- Connect the company network to a simulated ISP using eBGP.
- Advertise company department networks to the ISP.
- Redistribute the ISP test route into OSPF so the Branch can learn it.
- Configure NAT/PAT and investigate translation behaviour.
- Verify the network with Cisco IOS `show` commands, ping tests, and Packet Tracer Simulation Mode.

## Devices and Roles

| Device | Model | Role |
|---|---|---|
| R1 | Cisco 2911 | HQ router, inter-VLAN gateways, company WAN, eBGP peer, NAT/PAT configuration |
| R2 | Cisco 2911 | Branch router, inter-VLAN gateways, company WAN, OSPF peer |
| R3 | Cisco 2911 | Simulated ISP router and ISP test destination |
| SW1 | Cisco 2960-24TT | HQ access switch |
| SW2 | Cisco 2960-24TT | Branch access switch |
| 24 PCs | Packet Tracer end devices | four departments at each site, three PCs per departments |

## Logical Topology and Cabling

- SW1 GigabitEthernet0/1 connects to R1 GigabitEthernet0/0. This is the HQ VLAN trunk.
- SW2 GigabitEthernet0/1 connects to R2 GigabitEthernet0/0. This is the Branch VLAN trunk.
- R1 GigabitEthernet0/1 connects to R2 GigabitEthernet0/1. This is the company WAN link.
- R1 GigabitEthernet0/2 connects to R3 GigabitEthernet0/0. This is the ISP-facing link.
- R3 Loopback0 represents a stable ISP test destination.

## IP Addressing Plan

### HQ VLANs

| Department | VLAN | Network | Default gateway |
|---|---:|---|---|
| HR | 10 | 10.1.10.0/24 | 10.1.10.1 |
| SALES | 20 | 10.1.20.0/24 | 10.1.20.1 |
| FINANCE | 30 | 10.1.30.0/24 | 10.1.30.1 |
| NOC | 40 | 10.1.40.0/24 | 10.1.40.1 |

### Branch VLANs

| Department | VLAN | Network | Default gateway |
|---|---:|---|---|
| HR | 110 | 10.2.10.0/24 | 10.2.10.1 |
| SALES | 120 | 10.2.20.0/24 | 10.2.20.1 |
| FINANCE | 130 | 10.2.30.0/24 | 10.2.30.1 |
| NOC | 140 | 10.2.40.0/24 | 10.2.40.1 |

DHCP excludes `.1` through `.20` in each department subnet. DHCP clients can receive addresses beginning at `.21`. The subnet mask is `255.255.255.0`.

### WAN and ISP addressing

| Link / interface | IP address | Mask |
|---|---|---|
| R1 G0/1 — company WAN | 10.255.12.1 | 255.255.255.252 |
| R2 G0/1 — company WAN | 10.255.12.2 | 255.255.255.252 |
| R1 G0/2 — ISP link | 203.0.113.1 | 255.255.255.252 |
| R3 G0/0 — ISP link | 203.0.113.2 | 255.255.255.252 |
| R3 Loopback0 — ISP test destination | 198.51.100.1 | 255.255.255.255 |

The addresses `203.0.113.0/24` and `198.51.100.0/24` are documentation ranges. They are used here for a lab simulation, not as real public internet addresses.

## VLAN and Switch Port Plan

### SW1 — HQ

| Ports | Department | VLAN |
|---|---|---:|
| Fa0/1–Fa0/4 | HQ HR PCs | 10 |
| Fa0/5–Fa0/8 | HQ SALES PCs | 20 |
| Fa0/9–Fa0/12 | HQ SALES PCs | 30 |
| Fa0/13–Fa0/16 | HQ NOC PCs | 40 |
| Gi0/1 | Trunk to R1 G0/0 | 10, 20, 30, 40 |

### SW2 — Branch

| Ports | Department | VLAN |
|---|---|---:|
| Fa0/1–Fa0/4 | Branch HR PCs | 110 |
| Fa0/5–Fa0/8 | Branch SALES PCs | 120 |
| Fa0/9–Fa0/12 | Branch FINANCE PCs | 130 |
| Fa0/13–Fa0/16 | Branch NOC PCs | 140 |
| Gi0/1 | Trunk to R2 G0/0 | 110, 120, 130, 140 |

---

# Configuration Commands and Explanations

The following sections document the configuration commands used in the lab. Commands are grouped by device and stage. 
Commands used only for verification are listed separately.

## Stage 1 — Physical topology and cabling

The devices and links were placed in Cisco Packet Tracer and connected according to the logical topology above. No CLI commands were required for the physical placement or cabling.

## Stage 2A — Configure SW1 (HQ switch)

```text
enable
configure terminal
hostname SW1
no ip domain-lookup
vlan 10
 name HR
exit
vlan 20
 name SALES
exit
vlan 30
 name FINANCE
exit
vlan 40
 name NOC
exit
interface range fastEthernet 0/1-3
 description HQ-HR-PCs
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit
interface range fastEthernet 0/4-6
 description HQ-SALES-PCs
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
exit
interface range fastEthernet 0/7-9
 description HQ-FINANCE-PCs
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
exit
interface range fastEthernet 0/10-12
 description HQ-NOC-PCs
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
exit
interface gigabitEthernet 0/1
 description TRUNK-TO-R1-G0/0
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 no shutdown
exit
end
copy running-config startup-config
```

### SW1 command summary

| Command / command group | Purpose |
|---|---|
| `enable` | Enters privileged EXEC mode. |
| `configure terminal` | Enters global configuration mode. |
| `hostname SW1` | Sets the switch hostname to SW1. |
| `no ip domain-lookup` | Stops the switch from attempting DNS lookup when a command is mistyped. |
| `vlan 10` / `vlan 20` / `vlan 30` / `vlan 40` | Creates the four HQ VLANs. |
| `name HR` / `name SALES` / `name FINANCE` / `name NOC` | Gives each VLAN a readable name. |
| `interface range fastEthernet ...` | Selects a group of access ports together. |
| `description ...` | Documents the purpose of the selected ports. |
| `switchport mode access` | Configures the ports as access ports rather than trunks. |
| `switchport access vlan ...` | Assigns the ports to the correct department VLAN. |
| `spanning-tree portfast` | Allows an end-device access port to enter forwarding more quickly. Use on end-device ports, not switch-to-switch links. |
| `interface gigabitEthernet 0/1` | Selects the uplink to R1. |
| `switchport mode trunk` | Configures the uplink to carry tagged traffic for multiple VLANs. |
| `switchport trunk allowed vlan 10,20,30,40` | Limits the trunk to the four HQ VLANs. |
| `no shutdown` | Administratively enables the interface. |
| `exit` | Leaves the current configuration submode. |
| `end` | Returns to privileged EXEC mode. |
| `copy running-config startup-config` | Saves the active configuration so it persists after a reload. |

## Stage 2B — Configure SW2 (Branch switch)

```text
enable
configure terminal
hostname SW2
no ip domain-lookup
vlan 110
 name HR
exit
vlan 120
 name SALES
exit
vlan 130
 name FINANCE
exit
vlan 140
 name NOC
exit
interface range fastEthernet 0/1-3
 description BR-HR-PCs
 switchport mode access
 switchport access vlan 110
 spanning-tree portfast
exit
interface range fastEthernet 0/4-6
 description BR-SALES-PCs
 switchport mode access
 switchport access vlan 120
 spanning-tree portfast
exit
interface range fastEthernet 0/7-9
 description BR-FINANCE-PCs
 switchport mode access
 switchport access vlan 130
 spanning-tree portfast
exit
interface range fastEthernet 0/10-12
 description BR-NOC-PCs
 switchport mode access
 switchport access vlan 140
 spanning-tree portfast
exit
interface gigabitEthernet 0/1
 description TRUNK-TO-R2-G0/0
 switchport mode trunk
 switchport trunk allowed vlan 110,120,130,140
 no shutdown
exit
end
copy running-config startup-config
```

### SW2 command summary

These commands have the same functions as the SW1 commands, but I useD Branch VLAN IDs `110`, `120`, `130`, and `140`, Branch port descriptions, and the trunk to R2. 
The VLAN IDs differ between sites to make the site and department segmentation clear.

## Stage 3A — Configure R1 router-on-a-stick gateways for HQ

```text
enable
configure terminal
hostname R1
no ip domain-lookup
interface gigabitEthernet 0/0
 description TRUNK-TO-SW1-G0/1
 no shutdown
exit
interface gigabitEthernet 0/0.10
 description HQ-HR-GATEWAY
 encapsulation dot1Q 10
 ip address 10.1.10.1 255.255.255.0
exit
interface gigabitEthernet 0/0.20
 description HQ-SALES-GATEWAY
 encapsulation dot1Q 20
 ip address 10.1.20.1 255.255.255.0
exit
interface gigabitEthernet 0/0.30
 description HQ-FINANCE-GATEWAY
 encapsulation dot1Q 30
 ip address 10.1.30.1 255.255.255.0
exit
interface gigabitEthernet 0/0.40
 description HQ-NOC-GATEWAY
 encapsulation dot1Q 40
 ip address 10.1.40.1 255.255.255.0
exit
end
copy running-config startup-config
```

### R1 Stage 3 command summary

| Command / command group | Purpose |
|---|---|
| `hostname R1` | Names the HQ router R1. |
| `interface gigabitEthernet 0/0` | Selects the physical interface connected to SW1. |
| `interface gigabitEthernet 0/0.10` and similar | Creates logical subinterfaces, one for each HQ VLAN. |
| `encapsulation dot1Q 10` and similar | Associates each subinterface with its 802.1Q VLAN tag. |
| `ip address ...` | Assigns the default gateway address for that department subnet. |
| `no shutdown` on G0/0 | Enables the physical trunk interface. |

This is called **router-on-a-stick**: one physical router interface carries multiple VLANs over a trunk, while separate subinterfaces route traffic between VLANs.

## Stage 3B — Configure R2 router-on-a-stick gateways for Branch

```text
enable
configure terminal
hostname R2
no ip domain-lookup
interface gigabitEthernet 0/0
 description TRUNK-TO-SW2-G0/1
 no shutdown
exit
interface gigabitEthernet 0/0.110
 description BR-HR-GATEWAY
 encapsulation dot1Q 110
 ip address 10.2.10.1 255.255.255.0
exit
interface gigabitEthernet 0/0.120
 description BR-SALES-GATEWAY
 encapsulation dot1Q 120
 ip address 10.2.20.1 255.255.255.0
exit
interface gigabitEthernet 0/0.130
 description BR-FINANCE-GATEWAY
 encapsulation dot1Q 130
 ip address 10.2.30.1 255.255.255.0
exit
interface gigabitEthernet 0/0.140
 description BR-NOC-GATEWAY
 encapsulation dot1Q 140
 ip address 10.2.40.1 255.255.255.0
exit
end
copy running-config startup-config
```

R2 uses the same router-on-a-stick design as R1, with Branch VLAN IDs and `10.2.x.0/24` subnets.

## Stage 4A — Configure DHCP on R1 (HQ)

```text
enable
configure terminal
ip dhcp excluded-address 10.1.10.1 10.1.10.20
ip dhcp excluded-address 10.1.20.1 10.1.20.20
ip dhcp excluded-address 10.1.30.1 10.1.30.20
ip dhcp excluded-address 10.1.40.1 10.1.40.20
ip dhcp pool HQ-HR
 network 10.1.10.0 255.255.255.0
 default-router 10.1.10.1
 dns-server 8.8.8.8
exit
ip dhcp pool HQ-SALES
 network 10.1.20.0 255.255.255.0
 default-router 10.1.20.1
 dns-server 8.8.8.8
exit
ip dhcp pool HQ-FINANCE
 network 10.1.30.0 255.255.255.0
 default-router 10.1.30.1
 dns-server 8.8.8.8
exit
ip dhcp pool HQ-NOC
 network 10.1.40.0 255.255.255.0
 default-router 10.1.40.1
 dns-server 8.8.8.8
exit
end
copy running-config startup-config
```

### DHCP command summary

| Command / command group | Purpose |
|---|---|
| `ip dhcp excluded-address first last` | Reserves addresses so DHCP does not lease them to clients. |
| `ip dhcp pool NAME` | Creates a named DHCP pool. |
| `network ... 255.255.255.0` | Defines the subnet served by the pool. |
| `default-router ...` | Supplies the default gateway to clients. |
| `dns-server 8.8.8.8` | Supplies a DNS server address to clients. This is only a configured DNS address; successful DNS service depends on reachability and service availability. |

## Stage 4B — Configure DHCP on R2 (Branch)

```text
enable
configure terminal
ip dhcp excluded-address 10.2.10.1 10.2.10.20
ip dhcp excluded-address 10.2.20.1 10.2.20.20
ip dhcp excluded-address 10.2.30.1 10.2.30.20
ip dhcp excluded-address 10.2.40.1 10.2.40.20
ip dhcp pool BR-HR
 network 10.2.10.0 255.255.255.0
 default-router 10.2.10.1
 dns-server 8.8.8.8
exit
ip dhcp pool BR-SALES
 network 10.2.20.0 255.255.255.0
 default-router 10.2.20.1
 dns-server 8.8.8.8
exit
ip dhcp pool BR-FINANCE
 network 10.2.30.0 255.255.255.0
 default-router 10.2.30.1
 dns-server 8.8.8.8
exit
ip dhcp pool BR-NOC
 network 10.2.40.0 255.255.255.0
 default-router 10.2.40.1
 dns-server 8.8.8.8
exit
end
copy running-config startup-config
```

### DHCP validation performed

On R1 and R2:

```text
show ip dhcp pool
show running-config | section dhcp
show ip dhcp binding
```

On each test PC, `Desktop > IP Configuration > DHCP` was selected. The PCs received addresses from the expected department subnet and used the matching gateway. 
Local gateway pings succeeded.

## Stage 5 — Configure the HQ–Branch WAN link

### R1

```text
enable
configure terminal
interface gigabitEthernet 0/1
 description WAN-TO-R2-G0/1
 ip address 10.255.12.1 255.255.255.252
 no shutdown
exit
end
copy running-config startup-config
```

### R2

```text
enable
configure terminal
interface gigabitEthernet 0/1
 description WAN-TO-R1-G0/1
 ip address 10.255.12.2 255.255.255.252
 no shutdown
exit
end
copy running-config startup-config
```

### WAN validation

On R1 and R2:

```text
show ip interface brief
```

From R1:

```text
ping 10.255.12.2
```

From R2:

```text
ping 10.255.12.1
```

Both WAN interfaces were reported up/up and the WAN pings succeeded.

## Stage 6A — Configure OSPF on R1

```text
enable
configure terminal
router ospf 10
 router-id 1.1.1.1
 network 10.1.10.0 0.0.0.255 area 0
 network 10.1.20.0 0.0.0.255 area 0
 network 10.1.30.0 0.0.0.255 area 0
 network 10.1.40.0 0.0.0.255 area 0
 network 10.255.12.0 0.0.0.3 area 0
 passive-interface gigabitEthernet 0/0.10
 passive-interface gigabitEthernet 0/0.20
 passive-interface gigabitEthernet 0/0.30
 passive-interface gigabitEthernet 0/0.40
end
copy running-config startup-config
```

## Stage 6B — Configure OSPF on R2

```text
enable
configure terminal
router ospf 10
 router-id 2.2.2.2
 network 10.2.10.0 0.0.0.255 area 0
 network 10.2.20.0 0.0.0.255 area 0
 network 10.2.30.0 0.0.0.255 area 0
 network 10.2.40.0 0.0.0.255 area 0
 network 10.255.12.0 0.0.0.3 area 0
 passive-interface gigabitEthernet 0/0.110
 passive-interface gigabitEthernet 0/0.120
 passive-interface gigabitEthernet 0/0.130
 passive-interface gigabitEthernet 0/0.140
end
copy running-config startup-config
```

### OSPF command summary

| Command / command group | Purpose |
|---|---|
| `router ospf 10` | Enables OSPF process 10. The process ID is locally significant. |
| `router-id ...` | Sets a stable OSPF router ID. |
| `network ... wildcard area 0` | Enables OSPF on matching interfaces and advertises the connected networks in area 0. |
| `passive-interface ...` | Advertises the department subnet but prevents OSPF neighbor formation on that end-user-facing subinterface. |
| `show ip ospf neighbor` | Verifies OSPF adjacency. The expected neighbor state is FULL. |
| `show ip route` | Displays the routing table. OSPF-learned routes are marked `O`. |

### OSPF validation performed

On R1 and R2:

```text
show ip ospf neighbor
show ip route
```

The OSPF neighbor relationship reached FULL. Routes for the remote site's department networks appeared in the routing table. A PC at HQ successfully pinged the Branch HR gateway `10.2.10.1`, and a Branch PC successfully pinged the HQ HR gateway `10.1.10.1`.

## Stage 7 — Configure the ISP-facing link and ISP loopback

### R1 — ISP-facing interface

```text
enable
configure terminal
interface gigabitEthernet 0/2
 description WAN-TO-ISP-R3-G0/0
 ip address 203.0.113.1 255.255.255.252
 no shutdown
exit
end
copy running-config startup-config
```

### R3 — ISP router

```text
enable
configure terminal
hostname R3
no ip domain-lookup
interface gigabitEthernet 0/0
 description WAN-TO-HQ-R1-G0/2
 ip address 203.0.113.2 255.255.255.252
 no shutdown
exit
interface loopback 0
 description ISP-TEST-DESTINATION
 ip address 198.51.100.1 255.255.255.255
exit
end
copy running-config startup-config
```

### ISP link validation

On R1 and R3:

```text
show ip interface brief
```

From R1:

```text
ping 203.0.113.2
```

From R3:

```text
ping 203.0.113.1
```

Both direct-link pings succeeded. R3 Loopback0 was used as a simulated ISP destination.

## Stage 8A — Configure eBGP on R1 (company AS 65010)

```text
enable
configure terminal
router bgp 65010
 bgp log-neighbor-changes
 neighbor 203.0.113.2 remote-as 65000
exit
end
copy running-config startup-config
```

## Stage 8B — Configure eBGP on R3 (ISP AS 65000)

```text
enable
configure terminal
router bgp 65000
 bgp log-neighbor-changes
 neighbor 203.0.113.1 remote-as 65010
 network 198.51.100.1 mask 255.255.255.255
exit
end
copy running-config startup-config
```

### BGP command summary

| Command | Purpose |
|---|---|
| `router bgp 65010` | Starts BGP in the company's AS. |
| `router bgp 65000` | Starts BGP in the ISP's AS. |
| `bgp log-neighbor-changes` | Logs BGP neighbor state changes. |
| `neighbor IP remote-as ASN` | Defines a BGP neighbor and its remote AS number. |
| `network 198.51.100.1 mask 255.255.255.255` | Advertises the ISP loopback prefix if an exact matching route exists in R3's routing table. |
| `show ip bgp summary` | Checks BGP neighbor state and received-prefix count. |
| `show ip route bgp` | Displays BGP-learned routes. |

**AS** means **Autonomous System**. It is a set of IP networks and routers managed under one administrative authority and routing policy. 
In this lab, AS 65010 is the company and AS 65000 is the ISP. eBGP exchanges routes between these separate ASes; OSPF is used for internal company routing.

### BGP validation performed

The BGP neighbor session between R1 and R3 was confirmed as established. The R1 ping to `198.51.100.1` was used to test reachability to the ISP loopback after routing was configured.

## Stage 9A — Advertise company department networks through BGP on R1

```text
enable
configure terminal
router bgp 65010
 network 10.1.10.0 mask 255.255.255.0
 network 10.1.20.0 mask 255.255.255.0
 network 10.1.30.0 mask 255.255.255.0
 network 10.1.40.0 mask 255.255.255.0
 network 10.2.10.0 mask 255.255.255.0
 network 10.2.20.0 mask 255.255.255.0
 network 10.2.30.0 mask 255.255.255.0
 network 10.2.40.0 mask 255.255.255.0
end
copy running-config startup-config
```

BGP `network` statements advertise a prefix only when the router has an exact matching route in its routing table. 
The HQ subnets are connected routes, and the Branch subnets are learned through OSPF.

On R3, the following command was used to inspect BGP routes:

```text
show ip bgp
```

## Stage 9B — Redistribute BGP into OSPF on R1

A prefix-list/route-map filtering attempt was rejected by the Packet Tracer IOS implementation. The accepted lab configuration used the following broader redistribution command:

```text
enable
configure terminal
router ospf 10
 redistribute bgp 65010 subnets
end
copy running-config startup-config

### Route and end-to-end validation

On R1:

```text
show ip route 198.51.100.1
show ip bgp
```

On R2:

```text
show ip route 198.51.100.1
```

On R3:

```text
show ip route
```

From a Branch PC:

```text
ping 198.51.100.1
```

The Branch PC ping to `198.51.100.1` succeeded after route redistribution. This confirmed end-to-end IP reachability to the simulated ISP loopback.

## Stage 10A — Configure NAT inside/outside roles on R1

```text
enable
configure terminal
interface gigabitEthernet 0/0.10
 ip nat inside
exit
interface gigabitEthernet 0/0.20
 ip nat inside
exit
interface gigabitEthernet 0/0.30
 ip nat inside
exit
interface gigabitEthernet 0/0.40
 ip nat inside
exit
interface gigabitEthernet 0/1
 ip nat inside
exit
interface gigabitEthernet 0/2
 ip nat outside
exit
end
copy running-config startup-config
```

The HQ department subinterfaces and company WAN interface are marked as NAT inside interfaces. The ISP-facing interface is marked as NAT outside.

## Stage 10B — Configure NAT ACL and PAT on R1

```text
enable
configure terminal
access-list 1 permit 10.1.0.0 0.0.255.255
access-list 1 permit 10.2.0.0 0.0.255.255
ip nat inside source list 1 interface gigabitEthernet 0/2 overload
end
copy running-config startup-config
```

### NAT/PAT command summary

| Command | Purpose |
|---|---|
| `ip nat inside` | Marks an interface as the NAT inside side. |
| `ip nat outside` | Marks an interface as the NAT outside side. |
| `access-list 1 permit 10.1.0.0 0.0.255.255` | Matches source addresses in the HQ `10.1.0.0/16` range. |
| `access-list 1 permit 10.2.0.0 0.0.255.255` | Matches source addresses in the Branch `10.2.0.0/16` range. |
| `ip nat inside source list 1 interface gigabitEthernet 0/2 overload` | Configures PAT to use the IP address of R1 G0/2 for matched traffic, distinguishing sessions by transport-layer ports where applicable. |
| `show ip nat statistics` | Displays NAT interface roles and translation statistics. |
| `show ip nat translations` | Displays active or retained NAT translations. |
| `show access-lists` | Checks ACL entries and match counters. |

### NAT/PAT verification status

A Branch PC successfully pinged `198.51.100.1` after the NAT/PAT commands were entered.
# Verification Command Reference

| Device | Command | What it verifies |
|---|---|---|
| SW1 / SW2 | `show vlan brief` | VLANs and access-port assignments |
| SW1 / SW2 | `show interfaces trunk` | Trunk status and allowed VLANs |
| SW1 / SW2 | `show running-config` | Switch configuration |
| R1 / R2 / R3 | `show ip interface brief` | Interface IP addresses and interface status |
| R1 / R2 | `show ip dhcp pool` | DHCP pools and available addresses |
| R1 / R2 | `show ip dhcp binding` | DHCP leases issued to clients |
| R1 / R2 | `show ip ospf neighbor` | OSPF neighbor adjacency |
| R1 / R2 | `show ip route` | Connected and learned routes |
| R1 / R3 | `show ip bgp summary` | BGP neighbor state |
| R1 / R3 | `show ip bgp` | BGP routing information |
| R1 | `show ip route bgp` | Routes learned through BGP |
| R1 | `show ip nat statistics` | NAT configuration and statistics |
| R1 | `show ip nat translations` | NAT translation entries |
| R1 | `show access-lists` | ACL contents and match counters |
| Any router | `show running-config` | Active device configuration |
| Any router | `copy running-config startup-config` | Saves configuration persistently |

## Troubleshooting notes

- If a PC receives an address in the `169.254.0.0/16` range, it did not receive a normal DHCP lease. Check VLAN assignment, switch trunk, router subinterface, DHCP pool, and DHCP exclusions.
- If a subinterface is down, check the parent physical interface and switch trunk.
- If OSPF routes are missing, check `show ip ospf neighbor`, network statements, interface status, and passive-interface configuration.
- If BGP is not established, verify both neighbor addresses, remote AS numbers, and reachability between the peer IP addresses.
- If a Branch PC cannot reach the ISP loopback, inspect the route on R2, R1, and R3 in sequence.
- If NAT translations remain empty, do not infer that NAT works from ping alone. Check NAT interface roles, ACL matching, translation statistics, and the route/return path.

# Screenshots and Evidence

1. `01-full-topology.png` — Full network topology with device names and links.
2. `02-sw1-vlans.png` — SW1 VLAN and access-port verification.
3. `03-sw2-vlans.png` — SW2 VLAN and access-port verification.
4. `04-hq-router-subinterfaces.png` — R1 subinterfaces and gateway addresses.
5. `05-branch-router-subinterfaces.png` — R2 subinterfaces and gateway addresses.
6. `06-dhcp-hq.png` — HQ PC receiving a DHCP address and gateway.
7. `07-dhcp-branch.png` — Branch PC receiving a DHCP address and gateway.
8. `08-wan-ping.png` — Successful R1-to-R2 WAN ping.
9. `09-ospf-neighbor-routes.png` — OSPF FULL neighbor and learned routes.
10. `10-isp-loopback-ping.png` — Successful Branch PC ping to `198.51.100.1`.
11. `11-simulation-packet-path.png` — Packet Tracer Simulation Mode showing a packet travelling through the network.

## Skills demonstrated

- VLAN creation and port assignment
- Access-port and 802.1Q trunk configuration
- Router-on-a-stick inter-VLAN routing
- DHCP pool configuration and client validation
- IPv4 addressing and `/30` point-to-point links
- OSPF configuration and route verification
- eBGP neighbor configuration and route advertisement
- OSPF/BGP route redistribution
- NAT/PAT configuration and troubleshooting
- Cisco IOS verification commands
- Packet Tracer Simulation Mode and packet-path analysis
- Technical documentation and troubleshooting discipline

## Conclusion

This lab demonstrates how an enterprise network can be segmented into departmental VLANs, provide automatic IP configuration, route between sites using OSPF, 
and exchange routes with a simulated ISP using eBGP. It also documents the difference between successful IP reachability and verified NAT translation.
