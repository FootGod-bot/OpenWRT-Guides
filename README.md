# OpenWrt: Secure Guest Wi-Fi Network & Selective LAN Access

**Purpose:** Create an isolated guest wireless network that can access
the internet but cannot access the router's administration services, the
main LAN, or other guest clients. Then create narrowly scoped exceptions
for approved devices and services.

> **Security model:** Default deny, explicit authorization, minimum
> necessary access.

This guide uses modern OpenWrt concepts, including firewall4/nftables
and DSA networking. LuCI labels can vary slightly by OpenWrt version and
device.

------------------------------------------------------------------------

## Table of contents

1.  [Network design](#1-network-design)
2.  [Back up your configuration](#2-back-up-your-configuration)
3.  [Create the guest interface](#3-create-the-guest-interface)
4.  [Configure DHCP and IPv6](#4-configure-dhcp-and-ipv6)
5.  [Create the guest firewall zone](#5-create-the-guest-firewall-zone)
6.  [Allow DHCP and DNS to the
    router](#6-allow-dhcp-and-dns-to-the-router)
7.  [Create the guest wireless
    network](#7-create-the-guest-wireless-network)
8.  [Configure authorized-device
    pass-through](#8-configure-authorized-device-pass-through)
9.  [Manage multiple authorized devices and
    services](#9-manage-multiple-authorized-devices-and-services)
10. [Optional: Guest VLAN for wired
    devices](#10-optional-guest-vlan-for-wired-devices)
11. [Test the configuration](#11-test-the-configuration)
12. [Troubleshooting](#12-troubleshooting)
13. [Final security checklist](#13-final-security-checklist)

------------------------------------------------------------------------

## 1. Network design

Use separate IP subnets for trusted and guest devices. The guest
wireless network must attach to its own logical interface, not to the
existing LAN bridge.

### Example topology

``` text
                         INTERNET
                            |
                         OpenWrt
                      /           \
                 Main LAN       Guest LAN
               192.168.1.0/24 192.168.50.0/24
                    |               |
             Trusted devices    Guest Wi-Fi
             Servers, PCs       Visitors, guests
                    |               |
                    |         Default-deny policy
                    |               |
                    +----< approved, specific
                           device/port rules only
```

### Addressing plan

  Setting                 Main LAN                      Guest LAN
  ----------------------- ----------------------------- ------------------------
  Network                 `192.168.1.0/24`              `192.168.50.0/24`
  Router address          `192.168.1.1`                 `192.168.50.1`
  DHCP                    Enabled                       Enabled
  Internet                Allowed                       Allowed
  Router administration   Allowed for trusted devices   Blocked
  Access to main LAN      Normal trusted policy         Blocked by default
  Guest-to-guest          Not applicable                Blocked at Wi-Fi layer

The guest network will have its own interface, DHCP service, firewall
zone, and SSID.

### Firewall policy overview

  -----------------------------------------------------------------------
  Traffic                             Policy
  ----------------------------------- -----------------------------------
  Guest → WAN                         Allow

  Guest → router                      Reject, except DHCP and DNS

  Guest → main LAN                    Reject by default

  Guest → IoT or other internal zones Reject by default

  Guest client → other guest wireless Block using AP/client isolation
  client                              

  Authorized guest device → approved  Allow only through a specific rule
  LAN service                         

  Return traffic for an allowed       Allowed by connection tracking
  connection                          
  -----------------------------------------------------------------------

**Do not add general `guest → lan` forwarding.** Individual traffic
rules will provide the exceptions.

------------------------------------------------------------------------

## 2. Back up your configuration

Before changing networking:

1.  Open LuCI.
2.  Navigate to **System → Backup / Flash Firmware**.
3.  Select **Generate archive**.
4.  Download and safely store the backup.
5.  Make sure you have a recovery method, preferably wired access, in
    case a wireless or VLAN change disconnects you.

------------------------------------------------------------------------

## 3. Create the guest interface

1.  Navigate to **Network → Interfaces**.
2.  Select **Add new interface**.
3.  Use these settings:

  Field          Value
  -------------- ----------------------------------------------
  Name           `guest`
  Protocol       Static address
  Device         Leave unassigned for a wireless-only network
  IPv4 address   `192.168.50.1`
  IPv4 netmask   `255.255.255.0`
  IPv4 gateway   Leave blank
  Custom DNS     Leave blank initially

4.  Select **Create interface**.

Do not attach the guest interface to the existing main LAN bridge. A
wireless-only guest network can attach its wireless access point
directly to this logical interface.

------------------------------------------------------------------------

## 4. Configure DHCP and IPv6

### DHCP

Edit the `guest` interface and open the **DHCP Server** tab. Enable DHCP
and configure:

  Setting      Value
  ------------ -------
  Start        `100`
  Limit        `100`
  Lease time   `12h`

The resulting DHCP range is:

-   First address: `192.168.50.100`
-   Last address: `192.168.50.199`
-   Router: `192.168.50.1`

### IPv6

An IPv4-only policy is not sufficient if guest clients can use IPv6 to
reach internal networks. Choose one of these approaches:

-   **Simpler approach:** Do not advertise or provide IPv6 on the guest
    network, and ensure the guest interface is not providing an
    alternate IPv6 path.
-   **Advanced approach:** Configure equivalent IPv6 firewall
    restrictions and explicit exceptions for IPv6 services.

If using the simpler IPv4-only approach, disable guest-interface Router
Advertisements and DHCPv6, and do not enable an NDP proxy for the guest
network. Do not disable IPv6 globally on the router.

Save the interface settings.

------------------------------------------------------------------------

## 5. Create the guest firewall zone

1.  Navigate to **Network → Firewall**.
2.  Open **General Settings**.
3.  Add a new zone with these settings:

  Setting            Value
  ------------------ ----------
  Name               `guest`
  Covered networks   `guest`
  Input              Reject
  Output             Accept
  Forward            Reject
  Masquerading       Disabled
  MSS clamping       Disabled

### What the policies mean

-   **Input: Reject** --- blocks guest-originated traffic to the router
    itself, except for explicitly allowed traffic rules.
-   **Output: Accept** --- permits traffic originating from the router.
-   **Forward: Reject** --- blocks guest traffic being routed to other
    networks unless a specific exception permits it.

### Zone forwarding

Add this forwarding:

``` text
guest → wan
```

Do **not** add:

``` text
guest → lan
```

Do not add guest forwarding to IoT or other trusted internal zones
either. Save the configuration.

------------------------------------------------------------------------

## 6. Allow DHCP and DNS to the router

The guest zone's Input policy is Reject, so explicitly allow only the
services guests need from the router.

Navigate to **Network → Firewall → Traffic Rules**.

### Rule A: Guest DHCP

Create a traffic rule:

  Field              Value
  ------------------ --------------------
  Name               `Allow-Guest-DHCP`
  Protocol           UDP
  Source zone        `guest`
  Source port        Any
  Destination        Device / input
  Destination port   `67`
  Action             Accept

DHCP clients use UDP source port 68 and contact the server on UDP
destination port 67. The rule should match the server destination port.

### Rule B: Guest DNS

Create another rule:

  Field              Value
  ------------------ -------------------
  Name               `Allow-Guest-DNS`
  Protocol           TCP + UDP
  Source zone        `guest`
  Source port        Any
  Destination        Device / input
  Destination port   `53`
  Action             Accept

DNS uses port **53**, not port 33. Both TCP and UDP should be permitted.

Do not change the guest zone's Input policy to Accept just to make DHCP
or DNS work. Keep the default-deny policy and add these narrow
exceptions.

------------------------------------------------------------------------

## 7. Create the guest wireless network

1.  Navigate to **Network → Wireless**.
2.  Select **Add** on the radio you want to use.
3.  Configure:

  Setting           Value
  ----------------- -------------------------------------
  Mode              Access Point
  ESSID             `BIGWAKE-GUEST`
  Network           `guest`
  Security          WPA2-PSK or WPA2/WPA3
  Encryption        AES/CCMP
  Password          Use a strong, unique guest password
  Isolate clients   Enabled

Enable **Isolate Clients** (also called AP isolation or client
isolation, depending on the driver and LuCI version). This helps prevent
wireless clients on the same access point from communicating directly.

### Add the guest SSID to another radio

If you want the guest SSID on both 2.4 GHz and 5 GHz:

1.  Add a guest access point on the first radio.
2.  Attach it to the `guest` network.
3.  Add another guest access point on the second radio.
4.  Attach it to the same `guest` network.
5.  Optionally use the same SSID and password on both radios.

You do not need a separate IP interface for each radio.

A wireless-only guest network does not require a switch VLAN just to
isolate the SSID. If you later add wired guest ports, see the optional
VLAN section.

Save and apply.

------------------------------------------------------------------------

## 8. Configure authorized-device pass-through

The guest network remains blocked from the main LAN by default. To let
one approved device reach one approved service, create a narrow traffic
rule.

### Example network

  Device                    Address
  ------------------------- ------------------
  Authorized guest laptop   `192.168.50.110`
  Main LAN media server     `192.168.1.50`
  Example media service     TCP `8096`

The desired permission is:

``` text
192.168.50.110 → 192.168.1.50 TCP/8096
```

All other guest-to-LAN traffic remains blocked.

### 8.1 Reserve an IP address for the authorized device

1.  Navigate to **Network → DHCP and DNS**.
2.  Find **Static Leases**.
3.  Add a lease for the authorized device.
4.  Use a descriptive hostname, such as `authorized-laptop`.
5.  Enter the device's Wi-Fi MAC address.
6.  Assign `192.168.50.110`.
7.  Save and reconnect the device to guest Wi-Fi.
8.  Verify that it receives the reserved address.

**MAC-address note:** Modern phones and computers may use a private or
randomized MAC address for each Wi-Fi network. Use the MAC address that
the device actually presents on this guest SSID. If the address changes,
the reservation and rule may no longer match.

A DHCP reservation is convenient for policy matching, but it is not
strong authentication. A device capable of spoofing the authorized MAC
and IP may impersonate it. For stronger identity-based access, use an
authenticated VPN or another device-authentication mechanism.

### 8.2 Create the firewall exception

Navigate to **Network → Firewall → Traffic Rules → Add**.

Create this rule:

  Field                 Value
  --------------------- ----------------------
  Name                  `GUEST-LAPTOP-MEDIA`
  Protocol              TCP
  Source zone           `guest`
  Source address        `192.168.50.110`
  Destination zone      `lan`
  Destination address   `192.168.1.50`
  Destination port      `8096`
  IP family             IPv4
  Action                Accept
  Enabled               Yes

Save and apply.

This rule permits only the matching source, destination, protocol, and
destination port. It does not enable general guest-to-LAN forwarding.

### 8.3 Add more services only when needed

Create a separate rule for each additional service.

  Service                 Protocol     Port Guidance
  ----------------------- ---------- ------ ------------------------
  Jellyfin                TCP          8096 Allow if needed
  HTTPS web application   TCP           443 Allow if needed
  SSH                     TCP            22 Only when necessary
  SMB file sharing        TCP           445 Avoid unless necessary
  HTTP web application    TCP            80 Only when necessary

Do not open all ports because one application is not working. Identify
the application's actual network requirements first.

### 8.4 Allow access to another internal network

If the router has a separate IoT or server zone, create another narrow
rule:

  Field                 Example
  --------------------- -----------------------------------------
  Source zone           `guest`
  Source address        `192.168.50.110`
  Destination zone      The actual destination zone, e.g. `iot`
  Destination address   The specific device IP
  Protocol              Required protocol only
  Destination port      Required port only
  Action                Accept

Do not add general `guest → iot` forwarding.

### 8.5 Return traffic

OpenWrt tracks connections. When the authorized laptop initiates an
allowed connection, the server's reply traffic is normally allowed as
part of that established connection.

You generally do not need a second rule for ordinary response traffic. A
server initiating a new connection back to the guest device is a
different flow and requires separate consideration.

------------------------------------------------------------------------

## 9. Manage multiple authorized devices and services

Keep a record of each exception so it is clear which device can access
which service.

### Example access-control table

  Source device   Destination          Protocol     Port Permission
  --------------- -------------------- ---------- ------ ------------
  Guest laptop    Media server         TCP          8096 Allow
  Guest laptop    Web server           TCP           443 Allow
  Guest phone     Media server         TCP          8096 Allow
  Guest phone     Main LAN computers   Any           Any Deny
  Unknown guest   Main LAN             Any           Any Deny
  Unknown guest   IoT LAN              Any           Any Deny

### Recommended rule naming

Use consistent names:

-   `GUEST-LAPTOP-MEDIA`
-   `GUEST-LAPTOP-WEBSERVER`
-   `GUEST-PHONE-MEDIA`
-   `GUEST-ADMIN-SSH`

### Remember destination-device firewalls

The destination server may have its own firewall. For example, a Debian
media server may block connections from `192.168.50.0/24`.

If necessary, add a narrow server-side rule allowing the guest subnet to
reach only the required application port. Do not disable the server
firewall.

------------------------------------------------------------------------

## 10. Optional: Guest VLAN for wired devices

A wireless-only guest network can be isolated without changing the
existing switch VLAN configuration.

If you want physical Ethernet ports for guest devices, use a separate
VLAN. On modern OpenWrt DSA devices, VLANs are configured through bridge
VLAN filtering rather than legacy `swconfig`.

Example plan:

    VLAN ID Network    Subnet
  --------- ---------- --------------------------
          1 Main LAN   `192.168.1.0/24`
         20 IoT        Your existing IoT subnet
         50 Guest      `192.168.50.0/24`

The guest VLAN needs its own:

-   VLAN interface
-   IP subnet
-   DHCP service
-   Firewall zone
-   Wireless or wired port membership, as applicable

**Warning:** Incorrect bridge VLAN tagging or membership can disconnect
your LAN and lock you out of LuCI. Do not modify the existing main LAN
VLAN just to create a wireless-only guest SSID. Make VLAN changes only
after confirming the router's DSA port layout and having a recovery
path.

------------------------------------------------------------------------

## 11. Test the configuration

Connect a phone or laptop to `BIGWAKE-GUEST`.

### Test 1: Check the assigned IP address

Expected values:

  Setting       Expected value
  ------------- ------------------------------------
  IP address    `192.168.50.100`--`192.168.50.199`
  Gateway       `192.168.50.1`
  DNS           `192.168.50.1`
  Subnet mask   `255.255.255.0`

If the device receives an address from the main LAN, check which logical
network the wireless access point is attached to.

### Test 2: Check internet and DNS

Open:

-   <https://google.com>
-   <https://cloudflare.com>

Then run:

``` bash
nslookup google.com
```

The websites should load and DNS should resolve.

### Test 3: Check router administration isolation

Try opening:

``` text
http://192.168.50.1
http://192.168.1.1
```

Also test SSH:

``` bash
ssh root@192.168.50.1
```

Router administration should be blocked from guest clients. DHCP and DNS
should continue working.

### Test 4: Check main LAN isolation

Try:

``` bash
ping 192.168.1.209
```

Then test a known service:

``` bash
curl http://192.168.1.209:80
```

These should fail unless that exact connection has been authorized.

A failed ping alone does not prove isolation because a destination
device may block ICMP. Test actual services too.

### Test 5: Check guest client isolation

Connect two devices to the guest Wi-Fi and try to communicate between
them. They should not be able to communicate directly when AP/client
isolation is supported and enabled.

### Test 6: Check authorized pass-through

From the authorized laptop:

``` bash
curl http://192.168.1.50:8096
```

This should reach the example media service.

Try an unapproved port:

``` bash
curl http://192.168.1.50:80
```

It should be blocked unless separately authorized.

Finally, repeat the media-server test from an unauthorized guest device.
It should be blocked.

### Expected results

  Test                                     Expected result
  ---------------------------------------- --------------------
  Internet access                          Allowed
  DHCP                                     Allowed
  DNS                                      Allowed
  Guest → router management                Blocked
  Guest → main LAN                         Blocked by default
  Guest → IoT LAN                          Blocked by default
  Guest → other wireless clients           Blocked
  Authorized device → approved service     Allowed
  Authorized device → unapproved port      Blocked
  Unauthorized device → approved service   Blocked

------------------------------------------------------------------------

## 12. Troubleshooting

  -----------------------------------------------------------------------
  Problem                 Possible cause          What to check
  ----------------------- ----------------------- -----------------------
  Guest Wi-Fi has no      Missing guest-to-WAN    Guest zone forwarding
  internet                forwarding              

  Guest receives no IP    DHCP disabled or DHCP   Guest DHCP settings and
  address                 rule missing            UDP destination port 67

  DNS does not work       DNS input rule missing  TCP/UDP destination
                          or incorrect            port 53

  Guest can access LuCI   Input policy or another Guest zone and all
                          rule permits access     applicable traffic
                                                  rules

  Guest can access main   Broad forwarding or     Remove guest-to-LAN
  LAN                     overly broad allow rule forwarding and inspect
                                                  exceptions

  Guest can access IoT    Broad guest-to-IoT      Remove broad forwarding
                          forwarding              and inspect exceptions

  Authorized device       Wrong source IP,        DHCP reservation and
  cannot reach server     destination IP, zone,   traffic rule
                          protocol, or port       

  One application works   Only one service port   Identify and authorize
  but another does not    was permitted           the additional required
                                                  port

  IPv6 bypasses isolation IPv6 firewall or        Guest IPv6
                          advertisements are not  configuration and
                          restricted              firewall rules
  -----------------------------------------------------------------------

Useful diagnostic commands over SSH:

Show UCI firewall configuration:

``` bash
uci show firewall
```

Inspect the active nftables ruleset:

``` bash
nft list ruleset
```

These commands are useful for diagnosing firewall4 behavior. Review the
active rules before adding broad permissions to work around a problem.

------------------------------------------------------------------------

## 13. Final security checklist

-   [ ] Guest network uses a separate subnet (`192.168.50.0/24`).
-   [ ] Guest interface has its own DHCP server.
-   [ ] Guest IPv6 is disabled or separately protected.
-   [ ] Guest firewall zone has Input: Reject.
-   [ ] Guest firewall zone has Output: Accept.
-   [ ] Guest firewall zone has Forward: Reject.
-   [ ] Guest-to-WAN forwarding is enabled.
-   [ ] Guest-to-LAN forwarding is not enabled.
-   [ ] Guest-to-IoT forwarding is not enabled.
-   [ ] Only DHCP and DNS are allowed to the router.
-   [ ] Guest wireless client isolation is enabled.
-   [ ] Guest Wi-Fi uses WPA2 or WPA3 security.
-   [ ] Authorized devices have reserved IP addresses.
-   [ ] Pass-through rules specify source device, destination device,
    protocol, and port.
-   [ ] Destination devices have appropriate local firewall rules.
-   [ ] Internet, isolation, and authorized-access tests have passed.

------------------------------------------------------------------------

## Official documentation

-   [OpenWrt Network
    Configuration](https://openwrt.org/docs/guide-user/network/network_configuration)
    --- interfaces, bridges, and network configuration.
-   [OpenWrt Firewall
    Configuration](https://openwrt.org/docs/guide-user/firewall/firewall_configuration)
    --- firewall zones, forwarding, and traffic rules.
-   [OpenWrt DHCP and DNS
    Configuration](https://openwrt.org/docs/guide-user/base-system/dhcp)
    --- DHCP leases and DNS.
-   [OpenWrt Guest Wi-Fi
    Guide](https://openwrt.org/docs/guide-user/network/wifi/guestwifi/guest-wlan)
    --- guest wireless networking.

------------------------------------------------------------------------

## Final result

The resulting network has three layers of protection:

1.  **Network isolation:** Guest devices cannot freely communicate with
    the router or trusted networks.
2.  **Client isolation:** Guest wireless clients cannot directly
    communicate with other guest wireless clients, where supported by
    the access point and driver.
3.  **Selective access:** Individual devices can reach only explicitly
    approved servers and ports.

The guest network remains untrusted. A device does not gain broad LAN
access just because it needs one application.

**Default deny. Explicit authorization. Minimum necessary access.**

