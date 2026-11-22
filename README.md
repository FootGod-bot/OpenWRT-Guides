# OpenWrt Secure Guest Network & Selective Device Access

**Technical Configuration Guide**

**Platform:** OpenWrt (LuCI / Firewall4 / DSA)  
**Target hardware:** NETGEAR Orbi RBR750  
**Configuration type:** VLAN-based network segmentation  
**Primary objective:** Provide guest devices with unrestricted internet access while preventing unauthorized access to the trusted home network, router administration, and other guest devices.

---

## 1. Project Overview

This guide describes how to configure an isolated guest network on an OpenWrt router.

The guest network will use a separate VLAN, IP subnet, DHCP configuration, wireless SSID, and firewall zone.

By default, guest devices will only be permitted to access the internet and the minimum router services required for network connectivity.

Access to devices on the primary LAN will be denied unless an administrator explicitly creates an authorization rule.

Authorization rules will support restrictions based on:

- Source network.
- Source IP address.
- Destination IP address.
- Destination port.
- Network protocol (TCP, UDP, or both).

This allows an administrator to provide access to individual services without exposing an entire device or network.

### 1.1 Primary Requirements

The completed configuration must provide:

1. A dedicated guest VLAN and wireless network.
2. Automatic IP assignment through DHCP.
3. Internet access for guest devices.
4. Default-deny access to the primary LAN.
5. No access to router administration services.
6. Isolation between guest wireless clients.
7. Selective access to authorized devices.
8. Port-level access restrictions.
9. IPv4 and IPv6 security.
10. A configuration that can be maintained through LuCI.

### 1.2 Network Architecture

The network will consist of two primary security zones.

| Network | Purpose | Subnet |
|---|---|---|
| Main LAN | Trusted devices and infrastructure | 192.168.1.0/24 |
| Guest LAN | Visitors and external-network testing | 192.168.50.0/24 |

The existing primary LAN will remain unchanged.

The guest network will use VLAN ID 50.

### 1.3 Network Diagram

Internet  
↓  
OpenWrt Router  
├── Main LAN (Trusted)  
│   ├── Desktop / Sunshine  
│   ├── Jellyfin Server  
│   ├── NAS  
│   ├── Home Assistant  
│   └── Other trusted devices  
│  
└── Guest VLAN 50 (Untrusted)  
    ├── Guest smartphones  
    ├── Guest laptops  
    ├── Moonlight testing devices  
    └── Other temporary devices  

The guest network will have internet access but will not be permitted to access the main LAN by default.

Individual exceptions will be created through firewall traffic rules.

---

# 2. Network Configuration

## 2.1 Guest Network Settings

The following configuration will be used for the guest network.

| Setting | Value |
|---|---|
| Network name | guest |
| VLAN ID | 50 |
| Bridge VLAN device | Hardware/configuration dependent |
| Protocol | Static IPv4 |
| Router address | 192.168.50.1 |
| Subnet mask | 255.255.255.0 |
| DHCP | Enabled |
| DHCP start | 100 |
| DHCP limit | 100 |
| DHCP lease time | 12h |

The DHCP allocation range will be:

192.168.50.100 – 192.168.50.199

The primary LAN must retain its existing configuration.

## 2.2 Back Up the Router

Before modifying the network:

1. Connect to the router through Ethernet if possible.
2. Open LuCI.
3. Navigate to **System → Backup / Flash Firmware**.
4. Create and download a configuration backup.
5. Keep the existing primary Wi-Fi network available.

VLAN and bridge changes can interrupt connectivity. A wired connection and a configuration backup provide a recovery path.

## 2.3 Inspect Existing Network Devices

Navigate to:

**Network → Devices**

Identify:

- The existing LAN bridge.
- Whether VLAN filtering is enabled.
- Existing bridge VLAN entries.
- Physical LAN port assignments.

The RBR750 SNAPSHOT configuration must be inspected before modifying bridge membership.

Do not replace the existing LAN bridge or remove existing VLAN entries.

## 2.4 Create VLAN 50

Navigate to the existing bridge's VLAN configuration.

Create a new VLAN entry:

| Setting | Value |
|---|---|
| VLAN ID | 50 |
| Bridge | Existing LAN bridge |
| Router bridge membership | Required |
| Physical LAN ports | Excluded unless specifically needed |

The router must be able to terminate the VLAN interface.

The exact DSA bridge membership configuration depends on the installed OpenWrt build and hardware port layout.

Do not apply generic legacy `swconfig` instructions to a DSA-based installation.

## 2.5 Create the Guest Interface

Navigate to:

**Network → Interfaces → Add new interface**

Configure:

| Setting | Value |
|---|---|
| Name | guest |
| Protocol | Static address |
| Device | Guest VLAN interface |
| IPv4 address | 192.168.50.1 |
| Netmask | 255.255.255.0 |

Save the interface.

## 2.6 Configure DHCP

Open the guest interface's DHCP Server settings.

Enable DHCP and configure:

| Setting | Value |
|---|---|
| DHCP enabled | Yes |
| Start | 100 |
| Limit | 100 |
| Lease time | 12h |

Guests will automatically receive an IP address, gateway, and DNS configuration.

---

# 3. Guest Firewall Configuration

The firewall is the primary security boundary between the guest network and the trusted network.

The guest network will use a default-deny policy.

## 3.1 Create the Guest Firewall Zone

Navigate to:

**Network → Firewall → General Settings**

Create a firewall zone named `guest`.

Configure:

| Setting | Value |
|---|---|
| Zone name | guest |
| Covered networks | guest |
| Input | REJECT |
| Output | ACCEPT |
| Forward | REJECT |
| Masquerading | Disabled |
| MSS clamping | Disabled |

Masquerading will be handled by the WAN zone.

## 3.2 Configure Inter-Zone Forwarding

Configure the following forwarding permissions:

| Source | Destination | Permission |
|---|---|---|
| guest | wan | ALLOW |
| guest | lan | DENY |
| lan | guest | DENY |

Guest devices must be able to reach the WAN to access the internet.

They must not be able to route into the trusted LAN.

Do not create a general guest-to-LAN forwarding rule.

## 3.3 Router Access Restrictions

The guest zone must not have general access to the router itself.

The following services must remain inaccessible:

| Service | Port | Permission |
|---|---:|---|
| LuCI HTTP | 80 | DENY |
| LuCI HTTPS | 443 | DENY |
| SSH | 22 | DENY |
| Other administration services | Various | DENY |

However, DHCP and DNS must remain available if they are provided by the router.

## 3.4 Allow DHCP

Navigate to:

**Network → Firewall → Traffic Rules → Add**

Create:

**Name:** Guest-DHCP

| Setting | Value |
|---|---|
| Source zone | guest |
| Destination | Device (input) |
| Protocol | UDP |
| Destination port | 67 |
| Action | ACCEPT |

This permits guest devices to obtain DHCP leases.

## 3.5 Allow DNS

Create another traffic rule:

**Name:** Guest-DNS

| Setting | Value |
|---|---|
| Source zone | guest |
| Destination | Device (input) |
| Protocol | TCP and UDP |
| Destination port | 53 |
| Action | ACCEPT |

This permits guest devices to resolve domain names through the router.

These permissions must not grant access to router administration.

If DNS is hosted on another device, adjust the configuration accordingly.

---

# 4. Guest Wireless Network

Navigate to:

**Network → Wireless**

Create a new wireless access point on the desired radio.

Configure:

| Setting | Value |
|---|---|
| SSID | BIGWAKE-GUEST |
| Mode | Access Point |
| Network | guest |
| Encryption | WPA2-PSK or WPA2/WPA3 |
| Password | Separate guest password |
| Client isolation | Enabled |

If desired, repeat the process on the other radio so guests can connect through both 2.4 GHz and 5 GHz.

Do not modify the existing BIGWAKE or BIGWAKE-IOT networks.

## 4.1 Wireless Client Isolation

Enable wireless client isolation on the guest access point.

This prevents wireless clients connected to that access point from communicating directly with one another.

Wireless client isolation is separate from firewall isolation.

If wired guest devices are also added to the guest VLAN, additional layer-2 isolation may be required to prevent those devices from communicating with other devices on the same VLAN.

---

# 5. Authorized Device Access

This section describes how to selectively allow guest devices to access specific services on the trusted LAN.

The default policy remains DENY.

Each authorization must explicitly specify the destination device and permitted service.

## 5.1 Device IP Reservations

Before creating firewall rules, assign static DHCP reservations to devices that will receive authorized access.

Navigate to:

**Network → DHCP and DNS → Static Leases**

Example device inventory:

| Device | Reserved IP |
|---|---|
| Printer | 192.168.1.50 |
| Jellyfin | 192.168.1.60 |
| Desktop | 192.168.1.70 |
| NAS | 192.168.1.80 |

Use the actual MAC addresses of the devices.

Reservations ensure that firewall rules continue targeting the intended devices.

## 5.2 Creating an Authorization Rule

Navigate to:

**Network → Firewall → Traffic Rules → Add**

Every authorization rule should include:

- A descriptive name.
- Source zone.
- Optional source IP restriction.
- Destination zone.
- Destination IP.
- Protocol.
- Destination port.
- ACCEPT action.

The general rule structure is:

Guest Network → Authorized Device → Authorized Port

All other traffic remains blocked.

## 5.3 Example: Printer Authorization

Assume the printer is located at:

192.168.1.50

Create:

**Name:** Guest-Allow-Printer-IPP

| Setting | Value |
|---|---|
| Source zone | guest |
| Source address | Blank or specific guest IP |
| Destination zone | lan |
| Destination address | 192.168.1.50 |
| Protocol | TCP |
| Destination port | 631 |
| Action | ACCEPT |

TCP 631 is commonly used for Internet Printing Protocol (IPP).

Other possible printer ports include:

| Port | Protocol | Purpose |
|---|---|---|
| 631 | TCP | IPP |
| 9100 | TCP | RAW printing |
| 515 | TCP | LPD printing |

Only enable the ports required by the printer.

## 5.4 Example: Jellyfin Authorization

Assume Jellyfin runs at:

192.168.1.60

Create:

**Name:** Guest-Allow-Jellyfin

| Setting | Value |
|---|---|
| Source zone | guest |
| Destination zone | lan |
| Destination address | 192.168.1.60 |
| Protocol | TCP |
| Destination port | 8096 |
| Action | ACCEPT |

This allows guest devices to connect to Jellyfin's HTTP service.

It does not automatically authorize access to other services running on the same server.

## 5.5 Example: Restrict Access to One Guest Device

Guest DHCP reservations can also be used to limit access to a particular guest device.

Assume the laptop has:

192.168.50.100

Modify the Jellyfin authorization rule:

| Setting | Value |
|---|---|
| Source zone | guest |
| Source address | 192.168.50.100 |
| Destination zone | lan |
| Destination address | 192.168.1.60 |
| Protocol | TCP |
| Destination port | 8096 |
| Action | ACCEPT |

Now only the designated guest IP can access Jellyfin through that exception.

A DHCP reservation should be used for the laptop so its address remains consistent.

---

# 6. Port-Level Access Control

An individual device may run several services.

Authorization should be based on the specific service rather than allowing access to the entire device.

Example:

| Destination device | Port | Protocol | Permission |
|---|---:|---|---|
| Printer | 631 | TCP | ALLOW |
| Printer | 9100 | TCP | DENY |
| Jellyfin | 8096 | TCP | ALLOW |
| Jellyfin | 22 | TCP | DENY |
| NAS | 445 | TCP | DENY |
| NAS | 22 | TCP | DENY |

Only explicitly authorized ports should be reachable.

Multiple rules can target the same device.

For example, if a printer requires both IPP and RAW printing, create separate rules for TCP 631 and TCP 9100.

Do not create broad rules that allow every port on an authorized device.

## 6.1 TCP and UDP

Always select the correct transport protocol.

TCP and UDP are separate protocols.

A rule allowing TCP 8096 does not allow UDP 8096.

If an application requires both, configure both explicitly.

## 6.2 Stateful Firewall Behavior

OpenWrt's firewall tracks established connections.

When a guest device initiates an authorized connection to a main-LAN device, the destination device can send replies through the existing connection.

A separate broad LAN-to-guest forwarding rule is not necessary.

This preserves the isolation policy while allowing legitimate responses.

---

# 7. Advanced Device Authorization Management

For a larger home network, firewall sets can be used to simplify authorization.

Instead of maintaining many individual firewall rules, an administrator can maintain IP address and port sets that the firewall references.

Example authorization inventory:

| Device | Address | Authorized services |
|---|---|---|
| Printer | 192.168.1.50 | TCP 631 |
| Jellyfin | 192.168.1.60 | TCP 8096 |
| Desktop | 192.168.1.70 | Individually selected ports |
| NAS | 192.168.1.80 | None |

This makes it easier to expand the authorization system without creating unnecessary broad firewall permissions.

Firewall sets are an optional enhancement. Start with individual LuCI rules before implementing them.

---

# 8. IPv6 Security

IPv6 must follow the same isolation policy as IPv4.

An IPv4-only firewall configuration is not sufficient if guest devices receive IPv6 addresses and can communicate with the trusted network through IPv6.

Two approaches are available.

**Option A: Disable guest IPv6**

Disable IPv6 assignment and advertisements on the guest network if IPv6 guest connectivity is not required.

**Option B: Enable isolated guest IPv6**

Configure a separate guest IPv6 prefix and apply equivalent firewall restrictions.

If IPv6 is enabled:

- Guest-to-WAN forwarding may be permitted.
- Guest-to-LAN forwarding must remain blocked.
- Router administration must remain inaccessible.
- Device exceptions must be explicitly configured for IPv6 where needed.

Do not assume IPv4 restrictions automatically protect IPv6.

---

# 9. Printer Discovery and mDNS

Some printers advertise themselves using multicast DNS (mDNS/Bonjour).

This discovery traffic does not normally cross VLAN boundaries.

As a result, a printer can be reachable by IP address while remaining invisible to automatic printer discovery.

Possible solutions:

1. Add the printer manually using its reserved IP address.
2. Configure an mDNS reflector such as Avahi.
3. Restrict reflected discovery traffic to the necessary services and networks.

Manual IP configuration is the recommended starting point because it avoids introducing unnecessary cross-network discovery traffic.

---

# 10. Security Verification

After configuration, test the guest network from an actual guest device.

## 10.1 Connectivity Tests

| Test | Expected result |
|---|---|
| Obtain DHCP lease | PASS |
| Resolve DNS | PASS |
| Browse internet | PASS |
| Access router LuCI | BLOCKED |
| Access router SSH | BLOCKED |
| Access main desktop | BLOCKED |
| Access NAS | BLOCKED |
| Access authorized printer | PASS |
| Access authorized Jellyfin service | PASS |
| Access unauthorized ports | BLOCKED |
| Communicate with other guest Wi-Fi clients | BLOCKED |

## 10.2 Moonlight Testing

The guest network can also be used to simulate an external network for Tailscale testing.

1. Connect the Moonlight laptop to BIGWAKE-GUEST.
2. Keep the Sunshine desktop connected to BIGWAKE.
3. Confirm the laptop cannot reach the desktop through its private LAN address.
4. Connect to the desktop using its Tailscale address.
5. Check Tailscale's connection status.

A direct Tailscale connection may still be established through NAT traversal. This does not mean the guest VLAN isolation failed.

For a guaranteed test involving an external relay, use Tailscale's relay behavior or an actual external network such as a mobile hotspot.

## 10.3 Additional Security Tests

Verify that:

- Guest devices cannot reach other private LAN addresses.
- Guest devices cannot access router management.
- Guest devices cannot communicate with each other.
- Unauthorized TCP and UDP ports remain inaccessible.
- IPv6 does not bypass the isolation policy.
- Authorized services work only from permitted guest devices.

---

# 11. Maintenance and Administration

The guest network should be maintained using the following practices:

1. Use unique passwords for guest Wi-Fi.
2. Keep the OpenWrt firmware updated.
3. Review authorization rules periodically.
4. Remove unused firewall exceptions.
5. Use DHCP reservations for authorized devices.
6. Avoid broad port ranges whenever possible.
7. Avoid exposing router administration to guest devices.
8. Maintain configuration backups before major changes.

New authorization rules should always be tested individually.

---

# 12. Final Security Policy

The completed network should follow these principles:

**Default deny:** Guest devices cannot access the trusted LAN unless explicitly authorized.

**Least privilege:** Authorized devices are accessible only through specifically permitted ports and protocols.

**Network separation:** Guest devices cannot directly communicate with trusted devices or other wireless guests.

**Internet availability:** Guest devices retain normal internet connectivity.

**Router protection:** Only essential network services such as DHCP and DNS are accessible on the router. Administration interfaces remain inaccessible.

**Selective authorization:** Access is granted to individual devices, individual services, and optionally individual guest IP addresses.

The result is a segmented guest network that provides internet connectivity while maintaining tight control over access to the rest of the home network.

---

**Implementation note:** This guide describes the intended architecture and LuCI configuration. Exact VLAN bridge membership and interface device names must be verified against the installed RBR750 OpenWrt SNAPSHOT build before applying changes. Do not blindly replace an existing DSA bridge configuration.
