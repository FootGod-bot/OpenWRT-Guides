OpenWrt: Secure Guest Wi-Fi Network & Selective LAN Access

Security-focused configuration guide
 Modern OpenWrt / LuCI

Objective: Create an isolated guest wireless network that can access the internet but cannot access your main network, router administration interfaces, or other guest devices. Then configure specific exceptions that allow authorized devices to communicate with selected devices on your main LAN using only explicitly permitted ports.

This guide is based on the tutorial you provided, but expands it with modern OpenWrt networking, firewall configuration, device authorization, and selective traffic forwarding.
1. Network design
Before configuring anything, it's important to understand how the networks will be separated.

We'll use the following example configuration:
Network topology
Internet

External network access

OpenWrt Router

Firewall, routing, DHCP and DNS


Main LAN
Trusted
192.168.1.0/24
BIGWAKE

- Computers
- Servers
- Home devices
- Network storage


Guest LAN
Restricted
192.168.50.0/24
BIGWAKE-GUEST

- Guest phones
- Visitors
- Temporary devices
- Authorized exceptions



Selective access
Only explicitly authorized connections can reach specified LAN devices and ports.



Network configuration
Setting	Main LAN	Guest LAN
Network	192.168.1.0/24	192.168.50.0/24
Router IP	192.168.1.1	192.168.50.1
DHCP	Enabled	Enabled
Internet	Allowed	Allowed
Router administration	Allowed	Blocked
Main LAN access	Allowed	Blocked by default
Guest client communication	N/A	Blocked

The guest network will have its own IP subnet, DHCP server, firewall zone, and wireless SSID.

Important: We are not putting guest devices into the main LAN. Instead, we are allowing narrowly defined connections through the router's firewall. This preserves network separation.
2. How the firewall will work
OpenWrt uses firewall zones to control communication between different networks.

We'll create a dedicated guest firewall zone.

The zone will have these settings:
Firewall setting	Value	Purpose
Input	REJECT	Blocks access to the router
Output	ACCEPT	Allows router-originated traffic
Forward	REJECT	Blocks guest-to-other-network traffic
Masquerading	No	Not needed on the guest zone
MSS clamping	No	Not needed specifically here

The forwarding configuration will be:
Source	Destination	Permission
Guest	WAN	Allow
Guest	LAN	Deny by default
Guest	Guest	Deny client-to-client
LAN	Guest	Deny new connections by default

We will then add individual firewall exceptions for authorized devices.
The most important security principle

The guest network should have no general forwarding permission to the LAN.

Instead of allowing all guest devices to access the main network and trying to block individual devices, we deny everything first and create specific exceptions afterward.

This is known as a default-deny approach.

3. Back up your existing configuration
Before making network changes, create a backup.
1. Open your OpenWrt LuCI interface.
2. Navigate to System → Backup / Flash Firmware.
3. Find the Backup section.
4. Select Generate archive.
5. Download the configuration backup.
Keep this file somewhere safe.

Also, make sure you have a way to access the router locally if you accidentally break your wireless configuration.
4. Create the guest network interface
We first need to create a separate network interface for guest devices.
1. Navigate to Network → Interfaces.
2. Select Add new interface.
3. Enter the following settings:
New interface
Interface nameguest

New interface



ProtocolStatic address

DeviceLeave unassigned for now

A wireless interface will be attached later. Do not attach the guest network to the existing main LAN bridge.


IPv4 configuration
Field	Value
IPv4 address	192.168.50.1
IPv4 netmask	255.255.255.0
IPv4 gateway	Leave blank
Custom DNS	Leave blank


4. Select Create interface.
Why use a separate subnet?
The main LAN uses 192.168.1.0/24, while the guest network uses 192.168.50.0/24.

This allows OpenWrt to identify which network a device belongs to and apply different firewall policies.

A guest device receiving 192.168.50.120 will not be considered part of the main LAN.
5. Configure the guest DHCP server
DHCP automatically assigns IP addresses to devices joining the guest network.
1. Open the guest interface's Edit page.
2. Navigate to the DHCP Server tab.
3. Enable the DHCP server if it isn't already enabled.
4. Configure the address pool:
Setting	Value
Start	100
Limit	100
Lease time	12h

This gives guests addresses from:

192.168.50.100 through 192.168.50.199

The router's guest address remains 192.168.50.1.
IPv6 considerations
For this initial configuration, use IPv4-only guest networking unless you also configure equivalent IPv6 firewall restrictions.

Under the guest interface's IPv6 settings:
- Router Advertisement: Disabled
- DHCPv6 service: Disabled
- NDP proxy: Disabled
This prevents an unprotected IPv6 path from bypassing an IPv4-only isolation policy.

Do not disable IPv6 globally on your router. These settings are specifically for the guest network.

Save the interface configuration.
6. Create the guest firewall zone
Now we need to create the firewall zone that will enforce guest isolation.
1. Navigate to Network → Firewall.
2. Open the General Settings tab.
3. Find the Zones section.
4. Select Add.
Configure the new zone:
Setting	Value
Name	guest
Covered networks	guest
Input	Reject
Output	Accept
Forward	Reject
Masquerading	Disabled
MSS clamping	Disabled

Make sure the guest interface is assigned to this zone.
Understanding the three policies
Input: REJECT

Input controls traffic destined for the router itself.

This is what prevents guests from accessing router administration services, including:
- LuCI web interface
- SSH
- Router management services
- Other services listening on the router
We will create specific exceptions for DHCP and DNS.

Output: ACCEPT

Output controls traffic originating from the router.

This is normally left at ACCEPT.

Forward: REJECT

Forward controls traffic passing through the router to another network.

This blocks guest devices from accessing your main LAN by default.
Configure zone forwarding
Find the zone forwarding section.

Add:

guest → wan

Do not add:

guest → lan

This is extremely important.

The guest network needs permission to reach the internet, but it does not need general permission to reach your main network.

Save the configuration.
7. Allow DHCP and DNS through the firewall
At this point, the guest network is isolated, but guests still need two services to function properly:
- DHCP: Assigns IP addresses.
- DNS: Resolves domain names such as google.com.
We will allow only these services to communicate with the router.

Navigate to Network → Firewall → Traffic Rules.
Rule 1: Allow guest DHCP
Select Add and configure:
Field	Value
Name	Allow-Guest-DHCP
Protocol	UDP
Source zone	guest
Source port	Any
Destination zone	Device (input)
Destination port	67
Action	Accept

This allows guest devices to request an IP address from the router.
Rule 2: Allow guest DNS
Create another traffic rule.
Field	Value
Name	Allow-Guest-DNS
Protocol	TCP + UDP
Source zone	guest
Source port	Any
Destination zone	Device (input)
Destination port	53
Action	Accept

This allows guest devices to use the router's DNS service.
Expected result

Guests can communicate with the router for DHCP and DNS, but other router services remain blocked.
Important corrections to the original video:
- DNS uses port 53, not port 33.
- DHCP requests use UDP destination port 67.
- DHCP clients normally use source port 68.
- DNS should permit both TCP and UDP.
- Do not change the entire guest zone's Input policy to ACCEPT just to make DHCP and DNS work.
These exceptions are much safer than opening the router to the entire guest network.
8. Configure the guest wireless network
Now we need to create a wireless access point attached to the guest interface.
1. Navigate to Network → Wireless.
2. Find the radio you want to use.
3. Select Add or Add wireless network.
Configure the access point:
Wireless configuration
Setting	Value
Mode	Access Point
ESSID	BIGWAKE-GUEST
Network	guest
Wireless security	WPA2-PSK or WPA2/WPA3
Encryption	AES/CCMP
Key	Your own strong password
Isolate clients	Enabled


Wireless client isolation
Enable Isolate Clients (sometimes called AP isolation).

This prevents wireless clients connected to the same guest access point from communicating directly with one another.

For example:
- Guest phone A cannot communicate with guest phone B.
- Guest laptop A cannot directly access guest laptop B.
This is an additional layer of protection.

The firewall zone provides network-level isolation, while wireless client isolation handles communication between clients on the wireless access point itself.
Configure additional radios
If you want the guest network available on both 2.4 GHz and 5 GHz:
1. Create a guest access point on the first radio.
2. Set its network to guest.
3. Create another guest access point on the second radio.
4. Set its network to the same guest interface.
5. Use the same SSID and security credentials if you want a unified wireless network.
You do not need to create a separate IP interface for each radio.

Note for DSA routers: A wireless-only guest network does not require you to create a physical switch VLAN just to separate the SSID. The wireless interface can attach directly to its own logical network interface. If you later want dedicated guest Ethernet ports, you can configure a separate bridge VLAN.

Save and apply.
9. Configure authorized device pass-through
Important security feature
This is where we go beyond the original tutorial.

Normally, the guest network cannot access anything on your main LAN.

However, there may be situations where you want a specific guest device to access a specific service on your main network.

For example:
- A guest laptop needs access to a media server.
- A guest phone needs access to a particular application server.
- An authorized computer needs access to a specific service on another LAN.
We can accomplish this using individual firewall rules.
9.1 Understand how selective access works
Imagine the following network:
Guest network

192.168.50.0/24
Authorized laptop
192.168.50.110

Other guest devices
Blocked


Specific firewall exception
TCP port 8096 only

Main LAN

192.168.1.0/24
Media server
192.168.1.50

Only the specifically permitted service is accessible.


In this example:
- 192.168.50.110 is the authorized guest laptop.
- 192.168.1.50 is the main LAN media server.
- TCP port 8096 is the only permitted destination port.
Other guest devices cannot access that server through this rule.

The authorized laptop also cannot access other ports on that server.
9.2 Give the authorized guest device a reserved IP address
First, we need to make sure the authorized device always receives the same IP address.
1. Navigate to Network → DHCP and DNS.
2. Find the Static Leases section.
3. Select Add.
4. Enter the device's MAC address.
5. Assign it an IP address.
Example:
Setting	Value
Hostname	authorized-laptop
MAC address	Device's actual Wi-Fi MAC
IPv4 address	192.168.50.110

Save the lease.

Reconnect the device to the guest Wi-Fi and verify that it receives the reserved address.

Important: Use the device's private/randomized Wi-Fi MAC consistently. If it changes its MAC address, the reservation and firewall rule may stop matching.

A reserved IP is an identification mechanism for firewall policy, not strong authentication. Someone who can spoof that IP or MAC may be able to impersonate the authorized device. For stronger identity-based access, use a dedicated authenticated VPN or another device-authentication mechanism.
9.3 Create the selective firewall rule
Navigate to:

Network → Firewall → Traffic Rules → Add

We will create a rule allowing the authorized laptop to access the media server.
Example: Allow media server access
Restricted access
General settings
Field	Value
Name	Guest-Laptop-Media
Protocol	TCP
Source zone	guest
Source address	192.168.50.110
Destination zone	lan
Destination address	192.168.1.50
Destination port	8096
Action	Accept


Advanced settings
Field	Value
IP family	IPv4
Enabled	Yes


Save and apply.

You have now created a narrowly scoped exception.

The firewall still blocks general guest-to-LAN forwarding. This rule allows only matching connections.
9.4 Allow additional services
If the authorized laptop needs another service, create another rule for that service.

For example:
Service	Protocol	Port	Recommendation
Jellyfin	TCP	8096	Allow if needed
HTTPS web application	TCP	443	Allow if needed
SSH	TCP	22	Only when necessary
SMB file sharing	TCP	445	Avoid unless necessary
HTTP web application	TCP	80	Only when necessary

Never open all ports simply because one application isn't working.

Also remember that an application may require more than one port or protocol. Only add additional rules after identifying the actual requirements.
9.5 Allow a device to access more than one LAN
If your router has multiple trusted network zones, you can repeat this process for each destination.

For example:
Source	Destination	Permission
Authorized guest laptop	Main LAN server	TCP 8096
Authorized guest laptop	IoT server	TCP 443
Other guest devices	Main LAN	Blocked
Other guest devices	IoT LAN	Blocked

For the second rule, choose the actual IoT firewall zone and destination IP address.

Do not create a general guest-to-IoT or guest-to-LAN forwarding rule.

One important distinction: This is routed access between networks, not literally placing the authorized device on multiple LANs. It keeps the network separation intact.
9.6 Understand return traffic
You do not normally need a second firewall rule to allow the server's response.

OpenWrt tracks connections. Once an authorized connection is accepted, the associated return traffic is allowed automatically.

For example:
1. The authorized laptop connects to 192.168.1.50:8096.
2. OpenWrt checks the firewall rule.
3. The connection is accepted.
4. The server responds.
5. OpenWrt recognizes the response as part of the established connection.
This does not automatically allow the server to initiate a separate new connection to the guest laptop.

If an application needs reverse connections, you must evaluate those separately.
10. Understand how to manage authorized devices
As you add more devices, it is important to keep track of exactly what each one can access.

I recommend using a simple access-control table.
Device	Destination	Port	Protocol	Permission
Guest laptop	Media server	8096	TCP	Allow
Guest laptop	Web server	443	TCP	Allow
Guest phone	Media server	8096	TCP	Allow
Guest phone	Main LAN computers	All	All	Deny
Unknown guest	Main LAN	All	All	Deny
Unknown guest	IoT LAN	All	All	Deny

This makes it much easier to understand which devices have access to your network.
Recommended naming convention
Use consistent names for firewall rules.

Examples:
- GUEST-LAPTOP-MEDIA
- GUEST-LAPTOP-WEBSERVER
- GUEST-PHONE-MEDIA
- GUEST-ADMIN-SSH
This makes managing a larger collection of rules much easier.
Important: Device firewall settings
OpenWrt is not the only firewall that matters.

The destination device may also have its own firewall.

For example, if your media server runs Debian, its firewall may block connections from 192.168.50.0/24.

You may need to permit the guest subnet on the specific application port at the server itself.

Do not disable the server's firewall to solve this problem. Add an equally narrow exception there.

11. Optional: Add a separate guest VLAN for wired devices
Your current design can isolate wireless guest clients without changing your existing LAN switch configuration.

However, if you eventually want to connect physical guest Ethernet devices, you can use a dedicated VLAN.

Since your router uses modern DSA networking, this is configured using bridge VLAN filtering rather than the older swconfig method.

An example VLAN plan:
VLAN ID	Network	Subnet
1	Main LAN	192.168.1.0/24
20	IoT	Existing IoT subnet
50	Guest	192.168.50.0/24

The guest VLAN should have:
- Its own VLAN interface.
- Its own IP subnet.
- Its own DHCP service.
- Its own firewall zone.
- No general access to the main LAN.
You would assign the guest VLAN to whichever physical ports should serve guest devices.
DSA configuration warning

Be careful when editing bridge VLAN settings on an existing router. Incorrect VLAN membership or tagging can disconnect your LAN, including the computer you're using to administer OpenWrt.

Do not modify the existing main LAN VLAN just to create a wireless-only guest network.

12. Test the guest network
Before considering the configuration finished, we need to verify that isolation actually works.

Connect a phone or laptop to BIGWAKE-GUEST.
Test 1: Verify the IP address
Open the device's network settings.

You should see something similar to:
Setting	Expected value
IP address	192.168.50.100–199
Gateway	192.168.50.1
DNS	192.168.50.1
Subnet mask	255.255.255.0

If the device receives an address from your main LAN, the wireless network is probably attached to the wrong interface.
Test 2: Verify internet access
Open a browser and visit:
- https://google.com
- https://cloudflare.com
Both should load.

You can also test DNS using:
nslookup google.com


Test 3: Verify router administration is blocked
Try opening:
http://192.168.50.1


Then try:
http://192.168.1.1


Also test SSH:
ssh root@192.168.50.1


These connections should be rejected or time out.

DHCP and DNS should still work.
Test 4: Verify main LAN isolation
Try accessing a known main LAN device.

For example:
ping 192.168.1.209


Then test a known service:
curl http://192.168.1.209:80


Neither should be accessible unless you have explicitly authorized that device and service.

A failed ping alone does not prove isolation because some devices block ICMP. Test actual services too.
Test 5: Verify guest client isolation
Connect two devices to the guest Wi-Fi.

Try accessing one device from the other.

They should not be able to communicate directly.
Test 6: Verify authorized pass-through
Use the authorized laptop from our earlier example.

Run:
curl http://192.168.1.50:8096


This should reach the media server.

Now try another port:
curl http://192.168.1.50:80


If port 80 has not been authorized, it should be blocked.

Finally, connect an unauthorized guest device and repeat the media server test.

It should be blocked.
Expected security test results
Test	Expected result
Internet access	Allowed
DHCP	Allowed
DNS	Allowed
Guest → Router management	Blocked
Guest → Main LAN	Blocked
Guest → IoT LAN	Blocked
Guest → Other guests	Blocked
Authorized device → Approved port	Allowed
Authorized device → Other ports	Blocked
Unauthorized device → Approved port	Blocked



13. Troubleshooting
Problem	Possible cause	Solution
Guest Wi-Fi has no internet	Missing guest-to-WAN forwarding	Check firewall zone forwarding
Guest doesn't receive an IP	DHCP not enabled or DHCP rule missing	Check guest DHCP settings and UDP 67 rule
DNS doesn't work	DNS input rule missing	Check TCP/UDP 53
Guest can access LuCI	Input policy or another firewall exception allows it	Inspect guest zone and traffic rules
Guest can access main LAN	Broad forwarding or an overly broad allow rule	Remove guest-to-LAN forwarding and inspect exceptions
Authorized device cannot access server	Incorrect IP, port, protocol or destination zone	Check the reservation and firewall rule
Authorized device works on one port but not another	Only one port was allowed	Add a separate rule if the second service is needed
IPv6 bypasses isolation	IPv6 firewall or RA configuration is incomplete	Disable guest IPv6 or configure equivalent restrictions

If something doesn't work, inspect the firewall configuration before adding broad permissions.

You can also check your current firewall configuration through SSH:
uci show firewall


And inspect active firewall rules:
nft list ruleset


These commands are useful for diagnosing problems with firewall4.
14. Final security checklist
Before putting the network into use, make sure everything below is configured.
Configuration progress
0/16



[ ] Guest network uses a separate subnet (192.168.50.0/24).
[ ] Guest interface has its own DHCP server.
[ ] Guest IPv6 is disabled or separately protected.
[ ] Guest firewall zone has Input: REJECT.
[ ] Guest firewall zone has Output: ACCEPT.
[ ] Guest firewall zone has Forward: REJECT.
[ ] Guest-to-WAN forwarding is enabled.
[ ] Guest-to-LAN forwarding is NOT enabled.
[ ] Guest-to-IoT forwarding is NOT enabled.
[ ] Only DHCP and DNS are allowed to the router.
[ ] Guest wireless client isolation is enabled.
[ ] Guest Wi-Fi uses WPA2 or WPA3 security.
[ ] Authorized devices have reserved IP addresses.
[ ] Pass-through rules specify source device, destination device, protocol and port.
[ ] Destination devices have appropriate local firewall rules.
[ ] Internet, isolation and authorized access tests have passed.
Reset checklist
15. Official documentation
These are useful references for understanding and maintaining the configuration:
- OpenWrt Network Configuration — Interfaces, bridges and DSA networking.
- OpenWrt Firewall Configuration — Firewall zones, forwarding and traffic rules.
- OpenWrt DHCP and DNS Configuration — DHCP leases and DNS configuration.

Final result
You now have a guest network with three distinct levels of protection:
1. Network isolation: Guest devices cannot freely communicate with your trusted networks or router.
2. Client isolation: Guest devices cannot directly communicate with other guest wireless clients.
3. Selective access: Individual devices can be granted access to specific servers and ports without opening the entire main network.
The most important part is that you never need to trust the entire guest network just because one device needs access to one service.

Default deny, explicit authorization, and minimum necessary access. That's the security model this configuration is designed around.
