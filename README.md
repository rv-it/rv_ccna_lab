# CCNA Packet tracer lab

## OVERVIEW

![access layer](pictures\infra_all.png)

## Technologies used

- OSPF multi-area
- HSRP
- VLAN segmentation
- EtherChannel (LACP)
- STP (RPVST+)
- Router on a Stick
- Spine–Leaf data center
- Dual ISP connectivity
- NAT

## Topology description

- AREA 0: OSPF backbone
- AREA 1: Traditional campus for users (IT, Marketing, Accounting)
- AREA 2: Spine–leaf data center with servers
- Dual ISP connectivity for Internet access

## AREA1

It’s a campus design with access and Core/Distribution switches and a router-on-a-stick architecture.

### Acces layer

It's the layer to connect end hosts.

![access layer](pictures/access_layer.png)

**sw03 configuration**:

![access layer](pictures/sw03_cli.png)

**EtherChannel**

EtherChannel is an aggregation of two or more interfaces.
It avoids a single link failure and increases bandwidth.
Several protocols exist (LACP, PAgP...). In this lab, LACP is used.  
For exemple to create Port-channel1:
```bash
enable
configure terminal
interface range fastEthernet 0/10-11
channel-group 1 mode active
```
It will be syncronise with the interfaces on the others side.  
For information and debugging: `show etherchannel summary`.  
Interface Port-channel1 was created with interfaces FastEthernet 0/10 and 0/11 and Port-channel2 with interfaces FastEthernet 0/12 and 0/13.

**VLAN**

Vlan segment the broadcast domain and enable the logical grouping of devices that belong to different Layer 2 devices.    
In this lab: IT, Marketing and Accounting are divided into three logical group. This allow us to manage each group independently.    
By default, VLAN 1 is the native VLAN, and all devices connected to the switch are in it and can communicate. It's a good pratice to change change native VLAN on trunk ports to avoid unwanted connections.  
Access mode permit to affect a VLAN to an interface (and the end user connected) and Trunk mode permit to pass VLANs between network devices.  
For information and debugging: `show vlan brief`.

![access layer](pictures/sh_vl_br.png)

Vlan333:

![access layer](pictures/vlan333.png)

Vlan333 is for management of network devices (access switches and distribution/core switches), only IT Vlan can access it.  
Even on a Layer 2 switch, you can create a VLAN interface, assign an IP address, and define a default gateway for remote management.  
Exemple:
```bash
enable
configure terminal
vlan333
name management
exit
interface vlan 333
ip address 192.168.33.3 255.255.255.0
exit
ip address 192.168.33.3 255.255.255.0
```

**Spanning-tree**

Spanning Tree prevents broadcast storms in the network by blocking ports that could create loops.  
Several protocols exist, RPVST+ (Cisco standard) is used in this lab because it allows one spanning-tree instance per VLAN and provides fast convergence.  
In sw03 configuration:
```bash
interface FastEthernet0/7
  spanning-tree portfast
  spanning-tree bpduguard enable
```
Switches send BPDUs to manage the spanning tree and decide which ports should be blocked. While spanning tree is converging, interfaces remain in a non-forwarding state.  
Interfaces that are not connected to another switch do not need to participate in spanning tree. PortFast disables spanning tree on these interfaces and reduces convergence time.  
However, if someone connects a switch to a PortFast interface, a loop could be created and the root bridge could change (explained in the **Collapsed Distribution and Core layer** section). To prevent this, BPDU Guard shuts down an interface that receives BPDUs, avoiding an unwanted switch to join the spanning tree.  
For information and debugging: `show spanning-tree`


### Collapsed Distribution and Core layer

It's an aggregation point for the acces layer wich provide redundancy and scalability, you can connect access switches according your needs (ex: connect a new office, floor). Normally, no end hosts are connected to this layer.

![access layer](pictures/distri_core_layer.png)

Note: Core and Distribution layers can be separated by adding two additional core switches between the Distribution and Core switches and the HSRP routers.
This would allow to add more distribution and access layers (for example, to connect a new building to our HSRP routers).
However, with the OSPF backbone (Area 0), you can also create a new area (for example, Area 3) for a new infrastructure.

**Spanning-tree**

![access layer](pictures/spanning_tree_root_rio.png)

Spanning Tree elects a root bridge, which is the reference point used to calculate loop-free paths in the Layer 2 topology.  
By default, the switch with the lowest bridge ID becomes the root bridge. The bridge ID is composed of the priority value and the MAC address. However, this switch may not be the most appropriate one for optimal traffic flow.  
You can manually define the root bridge as primary and secondary. If the primary root bridge fails, the secondary will take over.  
It is a best practice to synchronise the root bridge with the active HSRP router.  
In the capture above, sw01 is the primary root bridge for VLAN 10 and 30, and secondary for VLAN 20 and 333.  
The switch with the lowest priority becomes the root bridge. If the priorities are the same, the one with the lowest MAC address becomes the root bridge.      
You can define the primary and secondary root bridge with:
```bash
enable
configure terminal
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
```
It is a best practice to synchronise the root bridge with the active HSRP routers (explain in the **HSRP** section).

![access layer](pictures/hsrp_root_bridge.png)


### Distribution Routers (rtr01 and rtr02)

![access layer](pictures/hsrp_isp_rtr.png)

rtr01 configuration:

```bash
interface Loopback0
 ip address 172.31.255.1 255.255.255.255
!
interface FastEthernet0/0
 ip address 10.1.0.1 255.255.255.252
 duplex auto
 speed auto
!
interface FastEthernet0/1
 ip address 10.1.1.1 255.255.255.252
 ip nat inside
 duplex auto
 speed auto
!
interface FastEthernet1/0
 no ip address
 duplex auto
 speed auto
!
interface FastEthernet1/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 10.2.110.1
 ip nat inside
 ip access-group vlan_10 in
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt
!
interface FastEthernet1/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 10.2.110.1
 ip nat inside
 standby 20 ip 192.168.20.254
!
interface FastEthernet1/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 10.2.110.1
 ip nat inside
 standby 30 ip 192.168.30.254
 standby 30 priority 110
 standby 30 preempt
!
interface FastEthernet1/0.33
 encapsulation dot1Q 333
 ip address 192.168.33.1 255.255.255.0
 standby 33 ip 192.168.33.254
!
interface FastEthernet1/1
 ip address 100.50.25.1 255.255.255.252
 ip nat outside
 duplex auto
 speed auto
!
router ospf 1
 log-adjacency-changes
 passive-interface FastEthernet1/1
 passive-interface Loopback0
 passive-interface FastEthernet1/0.10
 passive-interface FastEthernet1/0.20
 passive-interface FastEthernet1/0.30
 passive-interface FastEthernet1/0.33
 auto-cost reference-bandwidth 100000
 network 192.168.0.0 0.0.255.255 area 1
 network 172.31.255.1 0.0.0.0 area 1
 network 10.1.0.0 0.0.255.255 area 1
 network 200.100.50.0 0.0.0.3 area 1
 default-information originate
!
ip nat inside source list nat_vlan interface FastEthernet1/1 overload
ip nat inside source static tcp 10.2.120.1 443 100.50.25.1 443 
ip classless
ip route 0.0.0.0 0.0.0.0 100.50.25.2 
ip route 0.0.0.0 0.0.0.0 10.1.0.2 5
!
ip flow-export version 9
!
!
ip access-list extended vlan_10
 deny ip 192.168.10.0 0.0.0.255 192.168.33.0 0.0.0.255
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit tcp 192.168.10.0 0.0.0.255 host 10.2.120.1 eq 443
 permit udp 192.168.10.0 0.0.0.255 host 10.2.110.2 eq domain
 permit icmp 192.168.10.0 0.0.0.255 host 10.2.110.1 echo
 permit icmp 192.168.10.0 0.0.0.255 host 10.2.110.2 echo
 permit icmp 192.168.10.0 0.0.0.255 host 10.2.120.1 echo
 deny ip 192.168.10.0 0.0.0.255 host 10.2.120.1
 deny ip 192.168.10.0 0.0.0.255 host 10.2.110.1
 deny ip 192.168.10.0 0.0.0.255 host 10.2.110.2
 permit icmp 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255 echo-reply
 permit tcp 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255 established
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
ip access-list standard nat_vlan
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.25
``

**Router-on-a-Stick**

Router-on-a-Stick allows you to route multiple VLANs using a single physical interface (trunk link).   
Each VLAN is configured on a subinterface, and Each VLAN subinterface acts as the default gateway for that VLAN.

Steps:

- Create a subinterface for each VLAN.
- Associate the VLAN to the subinterface with dot1Q.
- Assign an IP address.

Exemple:

```bash
enable
configure terminal
interface fastEthernet 1/0.10
encapsulation dot1Q 20
ip address 192.168.10.1 255.255.255.0
```

**HSRP**

HSRP (Hot Standby Router Protocol) allows multiple routers to share a single virtual default gateway for hosts in the same VLAN.
One router is active, another is standby, Hosts use a virtual IP as their default gateway.  
If the active router fails, the standby router takes over automatically.  

Example configuration on rtr01 (you have the mirror configuration on rtr02 to load balance VLAN's):

Active VLAN 10 (default priority is 100)
```bash
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt
```
Stanby VLAN 20 

`standby 20 ip 192.168.20.254`

- priority decides which router becomes active.
- preempt allows a router with higher priority to take over if it goes down and come back online.
- The standby group number (here 1) must match on all routers in the group.

**OSPF**

OSPF is a dynamic routing protocol, it converges quickly, supports load balancing, and is suitable to medium and large networks.
OSPF sends hello packets to exchange with neighbor routers and create a link-state database.  
Each router has a router ID. It is the highest Loopback address and if we have no loopback, the highest physical interface address is used, and we can also set it manually.  
It's a good practice to use a loopback address:  

```bash
interface Loopback0
 ip address 172.31.255.1 255.255.255.255

rtr01#sh ip ospf 
 Routing Process "ospf 1" with ID 172.31.255.1
 ```

Example of configuration on rtr01:

```bash
router ospf 1
 log-adjacency-changes
 passive-interface FastEthernet1/1
 passive-interface Loopback0
 passive-interface FastEthernet1/0.10
 passive-interface FastEthernet1/0.20
 passive-interface FastEthernet1/0.30
 passive-interface FastEthernet1/0.33
 auto-cost reference-bandwidth 100000
 network 192.168.0.0 0.0.255.255 area 1
 network 172.31.255.1 0.0.0.0 area 1
 network 10.1.0.0 0.0.255.255 area 1
 network 200.100.50.0 0.0.0.3 area 1
 default-information originate
```

- Passive-interface allows a network to be announced without sending hello packets and creating adjacency with the other router.
It is useful for third-party routers like an ISP (we don’t want to share all our routing tables), loopbacks (not real interfaces, no neighbors, no need to send hello packets), and if possible every interface that does not connect to an OSPF router (example: VLAN subinterfaces).

- By default, the reference bandwidth is 100, so FastEthernet, GigabitEthernet, 10Gig… will be treated the same way when choosing the best path (cost = reference-bandwidth / interface bandwidth, and cost = 1 is the lowest). With auto-cost reference-bandwidth 100000, you solve this issue.

- Networks are announced with a wildcard mask (OSPF searches interfaces that match it) and an area. Small networks can use a single area.
- To announce a default gateway in OSPF, you have to use the command default-information originate.

Many commands can be used for debugging and information:
```bash
rtr01#sh ip ospf ?
  <1-65535>       Process ID number
  border-routers  Border and Boundary Router Information
  database        Database summary
  interface       Interface information
  neighbor        Neighbor list
  virtual-links   Virtual link information
```

**ACL**

Acces control list allowes to manage witch network or host can acces to the interface, you can handle inbound and outbound traffic of the interface.  
Two type of ACL existe standard and extended
Standard handle only source nertwork or host
Extended permit to be more granular it handle protocol and destination network or host.
In this lab standard ACL is used for NAT, it permit VLAN 10, 20 and 30 to use the NAT (explain at **NAT** section)
Exemple rtr01:

```bash
ip access-list standard nat_vlan
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.255
 ```

Extended ACLs are used to limit traffic of VLAN 10 and 20, it's good practice to give acces to only what users needed, only VLAN 30 (IT) have full acces in this infra.
Detail of the Extended ACL vlan_10 (same ACL for vlan20):

```bash
ip access-list extended vlan_10
 deny ip 192.168.10.0 0.0.0.255 192.168.33.0 0.0.0.255                     -> deny access to vlan 33 (management)
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255                     -> deny acces to vlan 20 (marketing)
 permit tcp 192.168.10.0 0.0.0.255 host 10.2.120.1 eq 443                  -> permit to external_web_server (only HTTPS)
 permit udp 192.168.10.0 0.0.0.255 host 10.2.110.2 eq domain               -> permit access to DNS server (only port 53)
 permit icmp 192.168.10.0 0.0.0.255 host 10.2.110.1 echo                   -> permit ping on three server in AREA 2
 permit icmp 192.168.10.0 0.0.0.255 host 10.2.110.2 echo
 permit icmp 192.168.10.0 0.0.0.255 host 10.2.120.1 echo
 deny ip 192.168.10.0 0.0.0.255 host 10.2.120.1                            -> deny access on three server in AREA 2 (except what is above)
 deny ip 192.168.10.0 0.0.0.255 host 10.2.110.1
 deny ip 192.168.10.0 0.0.0.255 host 10.2.110.2
 permit icmp 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255 echo-reply      -> allow ping reply if the source is VLAN30 (IT)
 permit tcp 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255 established      -> allow TCP response if the source is VLAN (IT)
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255                     -> deny acces to VLAN 30
 permit ip 192.168.10.0 0.0.0.255 any                                      -> rest of the traffic for VLAN 10
```
the ACL must be aafected to the right interface in or out.
Exemple rtr01:

```bash
interface FastEthernet1/0.10
 ip access-group vlan_10 in
```

**NAT**

NAT allows a private IP address to be translated into a public IP address.
In this lab static NAT is used to map external-web-server to the rtr01 public interface only on the port 443 (HTTPS), this permit to external user to access the server via rtr1's public address.

![access layer](pictures/ext_user_ext_wab_srv.png)

Who need to define first who is nat inside and ouside and then create nat rule
Exemple (rtr01):

```bash
interface FastEthernet0/1
 ip address 10.1.1.1 255.255.255.252
 ip nat inside
 interface FastEthernet1/1
 ip address 100.50.25.1 255.255.255.252
 ip nat outside
 
 
ip nat inside source static tcp 10.2.120.1 443 100.50.25.1 443
```




Port Address Translation (PAT) is used by hosts on vlan 10,20 and 30 for acces to the wan. 
Private ip address are not routable on the WAN, with PAT you can map severals private address with a single public address.
Exemple (rtr01):

```ip nat inside source list nat_vlan interface FastEthernet1/1 overload```

We used the access-list standard nat_vlan for authorize VLAN 10,20 and 30 to used PAT on the FastEthernet1/1

With PAT PC5 can ping the wan:

![access layer](pictures/pc5_icmp_wan.png)

## AREA2

It's a Spine and leaf design.
DHCP helper
DNS

## AREA0


It's the BAckbone for the OSPF Routing.
Multiple OSPF AREA is composed of ABR routers have interfaces into two diffents area (backbone and one other), thay are usefull for summarize route to an area and decrease charge on the routing table and separate potential network devices or ospf issues into smaller area.
OSPF work with single AREA.
password for OSPF is recommended on production envirronment, without any new connected router couldbe do an adjencency with our ospf instance.

