# Cisco Packet Tracer Lab: HQ and Branch Office Network

This lab simulates a small company with a **Headquarters (HQ)** and a **Branch Office**. Both offices have their own LAN, switch, router, and wireless access point. The routers connect through an ISP, and users at both sites can communicate with each other and access the internet.

## Objectives

- Build a two-site network in Cisco Packet Tracer
- Configure routers, switches, and access points
- Use private IP addressing for internal LANs
- Use public IP addressing for ISP/WAN links
- Configure static routes between HQ and Branch
- Configure NAT/PAT for internet access
- Configure DHCP for wired and wireless clients
- Configure VLANs and switch management
- Test and troubleshoot connectivity

## Network Topology

```text
HQ LAN                       ISP WAN                      Branch LAN
PCs / AP                     ISP Router                   PCs / AP
   |                            |                            |
SW-HQ --- R-HQ --- ISP --- R-BRANCH --- SW-BRANCH
```

### Devices Used

| Device | Name | Purpose |
|---|---|---|
| Router | `R-HQ` | HQ LAN gateway and WAN connection |
| Router | `R-BRANCH` | Branch LAN gateway and WAN connection |
| Router | `ISP` | Simulated ISP |
| Switch | `SW-HQ` | HQ wired and wireless access |
| Switch | `SW-BRANCH` | Branch wired and wireless access |
| Access Point | `AP-HQ` | HQ wireless network |
| Access Point | `AP-BRANCH` | Branch wireless network |
| PCs / Laptops | Various | Wired and wireless clients |

## IP Addressing Plan

| Location / Link | Network | Usable Range | Gateway |
|---|---:|---:|---:|
| HQ LAN | `192.168.10.0/24` | `192.168.10.1-254` | `192.168.10.1` |
| Branch LAN | `192.168.20.0/24` | `192.168.20.1-254` | `192.168.20.1` |
| HQ-to-ISP WAN | `203.0.113.0/30` | `203.0.113.1-2` | N/A |
| Branch-to-ISP WAN | `198.51.100.0/30` | `198.51.100.1-2` | N/A |
| HQ switch management | `192.168.10.0/24` | `192.168.10.2` | `192.168.10.1` |
| Branch switch management | `192.168.20.0/24` | `192.168.20.2` | `192.168.20.1` |

### Key Concepts

- **Public IP address:** Routable address used on the ISP-facing WAN link.
- **Private IP address:** Internal address used within a company LAN.
- **Default gateway:** The router interface that devices use to reach other networks.
- **Subnet mask:** Defines which portion of an IP address identifies the network.
- **Static route:** A manually configured route telling a router how to reach a remote network.
- **NAT/PAT:** Translates private internal addresses to a public address for internet access.
- **VLAN:** A logical Layer 2 network segment on a switch.
- **Trunk:** A link that carries traffic for multiple VLANs.
- **SSID:** The public name of a wireless network.
- **DHCP:** Automatically assigns IP address, subnet mask, gateway, and DNS settings.

## Cabling and Connections

Use **copper straight-through** cables for:

- PC to switch
- Access point to switch
- Router to switch

Use **copper cross-over** cables for:

- Router to router

### Recommended Connections

| From | Port | To | Port |
|---|---|---|---|
| `PC-HQ1` | FastEthernet0 | `SW-HQ` | `Fa0/1` |
| `PC-HQ2` | FastEthernet0 | `SW-HQ` | `Fa0/2` |
| `AP-HQ` | Port 0 | `SW-HQ` | `Fa0/3` |
| `SW-HQ` | `Gig0/1` | `R-HQ` | `Gig0/0` |
| `R-HQ` | `Gig0/1` | `ISP` | `Gig0/0` |
| `ISP` | `Gig0/1` | `R-BRANCH` | `Gig0/1` |
| `R-BRANCH` | `Gig0/0` | `SW-BRANCH` | `Gig0/1` |
| `PC-BRANCH1` | FastEthernet0 | `SW-BRANCH` | `Fa0/1` |
| `PC-BRANCH2` | FastEthernet0 | `SW-BRANCH` | `Fa0/2` |
| `AP-BRANCH` | Port 0 | `SW-BRANCH` | `Fa0/3` |

## Configure the ISP Router

The ISP router only needs its WAN-facing interfaces configured. It does not need routes to the private LANs because NAT hides them behind the public WAN IP addresses.

```cisco
enable
configure terminal
hostname ISP

interface GigabitEthernet0/0
 description Link-to-HQ
 ip address 203.0.113.1 255.255.255.252
 no shutdown
exit

interface GigabitEthernet0/1
 description Link-to-Branch
 ip address 198.51.100.1 255.255.255.252
 no shutdown
exit

end
copy running-config startup-config
```

`no shutdown` enables a Cisco router interface because interfaces are administratively disabled by default.

## Configure the HQ Router

### Basic Security

```cisco
enable
configure terminal
hostname R-HQ

enable secret Cisco123!
line console 0
 password Cisco123!
 login
exit

service password-encryption
```

- `enable secret` protects privileged EXEC mode.
- The console password protects direct console access.
- `service password-encryption` encrypts plaintext passwords in the configuration.

### LAN and WAN Interfaces

```cisco
interface GigabitEthernet0/0
 description HQ-LAN
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface GigabitEthernet0/1
 description WAN-to-ISP
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shutdown
exit
```

- `Gig0/0` is the HQ LAN default gateway.
- `Gig0/1` is the public ISP connection.
- `ip nat inside` identifies the private LAN side.
- `ip nat outside` identifies the public internet side.

### Default Route

```cisco
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

This sends all traffic without a more specific route to the ISP.

### Route to Branch LAN

```cisco
ip route 192.168.20.0 255.255.255.0 198.51.100.2
```

This tells `R-HQ` how to reach the Branch private LAN through `R-BRANCH`.

### NAT/PAT Configuration

```cisco
access-list 10 permit 192.168.10.0 0.0.0.255
ip nat inside source list 10 interface GigabitEthernet0/1 overload
```

- Access list `10` identifies HQ private addresses allowed to use NAT.
- `overload` enables PAT so many devices can share the router public WAN IP.
- PAT uses different port numbers to track each internal device connection.

### DHCP for HQ

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.10

ip dhcp pool HQ-POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
exit
```

The excluded range prevents DHCP from assigning addresses already used by the router, switch, or other fixed devices.

### Save and Verify

```cisco
end
copy running-config startup-config

show ip interface brief
show ip route
show ip nat translations
```

`show ip route` should show a default route marked with `S*`.

## Configure the Branch Router

### Basic Security

```cisco
enable
configure terminal
hostname R-BRANCH

enable secret Cisco123!
line console 0
 password Cisco123!
 login
exit

service password-encryption
```

### LAN and WAN Interfaces

```cisco
interface GigabitEthernet0/0
 description Branch-LAN
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface GigabitEthernet0/1
 description WAN-to-ISP
 ip address 198.51.100.2 255.255.255.252
 ip nat outside
 no shutdown
exit
```

### Routing

```cisco
ip route 0.0.0.0 0.0.0.0 198.51.100.1
ip route 192.168.10.0 255.255.255.0 203.0.113.2
```

- The first command is the default route to the ISP.
- The second route allows `R-BRANCH` to reach the HQ private LAN.

### NAT/PAT Configuration

```cisco
access-list 10 permit 192.168.20.0 0.0.0.255
ip nat inside source list 10 interface GigabitEthernet0/1 overload
```

### DHCP for Branch

```cisco
ip dhcp excluded-address 192.168.20.1 192.168.20.10

ip dhcp pool BRANCH-POOL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
exit
```

### Save and Verify

```cisco
end
copy running-config startup-config

show ip interface brief
show ip route
show ip nat translations
```

## Configure the HQ Switch

A Layer 2 switch needs an IP address only for management. Normal PC traffic is forwarded at Layer 2.

```cisco
enable
configure terminal
hostname SW-HQ

vlan 10
 name HQ-USERS
exit

interface range FastEthernet0/1 - 20
 switchport mode access
 switchport access vlan 10
exit

interface GigabitEthernet0/1
 switchport mode trunk
exit

interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 no shutdown
exit

ip default-gateway 192.168.10.1

end
copy running-config startup-config
```

### Explanation

- VLAN 10 groups HQ user ports.
- `switchport mode access` configures a port for an end device.
- `switchport access vlan 10` assigns the port to VLAN 10.
- `Gig0/1` is configured as a trunk because it connects to the router.
- `interface vlan 10` creates the switch management interface.
- `ip default-gateway` allows the switch to communicate outside its local subnet.

### Verification

```cisco
show vlan brief
show interfaces trunk
show ip interface brief
```

## Configure the Branch Switch

```cisco
enable
configure terminal
hostname SW-BRANCH

vlan 20
 name BRANCH-USERS
exit

interface range FastEthernet0/1 - 20
 switchport mode access
 switchport access vlan 20
exit

interface GigabitEthernet0/1
 switchport mode trunk
exit

interface vlan 20
 ip address 192.168.20.2 255.255.255.0
 no shutdown
exit

ip default-gateway 192.168.20.1

end
copy running-config startup-config
```

### Verification

```cisco
show vlan brief
show interfaces trunk
```

## Configure the Access Points

### HQ Access Point

1. Connect `AP-HQ` to a `SW-HQ` access port.
2. Click `AP-HQ`.
3. Open the **Config** tab.
4. Select **Port 1**.
5. Configure:

| Setting | Value |
|---|---|
| SSID | `HQ-WiFi` |
| Security | `WPA2-PSK` |
| Passphrase | `HQ-WiFi-123!` |
| Encryption | AES |

6. Save the configuration.

### Branch Access Point

1. Connect `AP-BRANCH` to a `SW-BRANCH` access port.
2. Click `AP-BRANCH`.
3. Open the **Config** tab.
4. Select **Port 1**.
5. Configure:

| Setting | Value |
|---|---|
| SSID | `Branch-WiFi` |
| Security | `WPA2-PSK` |
| Passphrase | `Branch-WiFi-123!` |
| Encryption | AES |

6. Save the configuration.

## Configure the PCs

### HQ PCs

1. Open `PC-HQ1` or `PC-HQ2`.
2. Go to **Desktop** > **IP Configuration**.
3. Select **DHCP**.
4. Confirm the PC receives:

| Setting | Expected Value |
|---|---|
| IP address | `192.168.10.x` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `192.168.10.1` |
| DNS server | `8.8.8.8` |

### Branch PCs

1. Open `PC-BRANCH1` or `PC-BRANCH2`.
2. Go to **Desktop** > **IP Configuration**.
3. Select **DHCP**.
4. Confirm the PC receives:

| Setting | Expected Value |
|---|---|
| IP address | `192.168.20.x` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `192.168.20.1` |
| DNS server | `8.8.8.8` |

## Testing Checklist

Run these commands from a PC command prompt.

### HQ PC Tests

```text
ping 192.168.10.1
ping 192.168.10.2
ping 203.0.113.2
ping 192.168.20.1
ping 192.168.20.10
```

### Branch PC Tests

```text
ping 192.168.20.1
ping 192.168.20.2
ping 198.51.100.2
ping 192.168.10.1
ping 192.168.10.10
```

### Trace Route

From an HQ PC:

```text
tracert 192.168.20.10
```

You should see the traffic pass through the HQ router, across the WAN, and into the Branch LAN.

## Expected Results

| Test | Expected Result | Meaning |
|---|---|---|
| PC to local gateway | Success | PC can reach its local router |
| PC to local switch | Success | Local VLAN and switch management work |
| PC to remote router WAN IP | Success | WAN connection and routing work |
| HQ PC to Branch PC | Success | Inter-site routing works |
| Wireless client to wired PC | Success | AP, VLAN, DHCP, and routing work |
| Internet access test | Success | Default route and NAT/PAT work |

## Troubleshooting Guide

| Problem | Likely Cause | Fix |
|---|---|---|
| PC receives `169.254.x.x` | DHCP failure | Check DHCP pool, excluded addresses, and switch port VLAN |
| PC cannot ping its gateway | Wrong gateway, bad cable, or shutdown interface | Verify PC gateway, cabling, and `show ip interface brief` |
| HQ cannot ping Branch PC | Missing or incorrect static route | Verify static routes on both routers |
| Internet access fails | Missing default route or NAT | Verify default route and NAT/PAT configuration |
| Wireless client gets no IP | AP connected to wrong VLAN or DHCP issue | Confirm AP switch port and DHCP pool |
| Switch cannot be pinged remotely | Missing management IP or default gateway | Configure `interface vlan` and `ip default-gateway` |

### Useful Verification Commands

```cisco
show running-config
show ip interface brief
show ip route
show ip nat translations
show vlan brief
show interfaces trunk
show ip dhcp binding
```

## Learning Outcomes

After completing this lab, you should understand:

- How HQ and branch offices communicate through an ISP
- The difference between public and private IP addressing
- How to configure router interfaces and default gateways
- How static routes enable communication between remote LANs
- How NAT/PAT allows private devices to access the internet
- How DHCP assigns IP settings automatically
- How VLANs and access ports work on Cisco switches
- How to configure a basic wireless access point
- How to test and troubleshoot a small business network
