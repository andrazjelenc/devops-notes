# Cisco CSR1000V
The plan is to show basic configuration of CSR1000V, the NAT between LAN and WAN scenario and steps to establish VPN site-to-site connection to the Umbrella.

We are working with CSR1000V with software version 16.09.

Content:
- [Day 0: Basics](#day-0-basics)
- [Day 1: Basic configuration of CSR](#day-1-basic-configuration-of-csr)
- [Day 2: NAT between LAN and WAN](#day-2-nat-between-lan-and-wan)
- [Day 3: DHCP Server for LAN](#day-3-dhcp-server-for-lan)
- [Day 4: Management VRF](#day-4-management-vrf)
- [Day 5: Site-to-Site VPN to Umbrella](#day-5-site-to-site-vpn-to-umbrella)
- [Day 6: Site-to-Site VPN to Umbrella (UNDO)](#day-6-site-to-site-vpn-to-umbrella-undo)
- [Day 7: Netconf](#day-7-netconf)


```


############         LAN
#          #    192.168.0.0/24    #############
# CSR1000V # -------------------- # Client VM #   
#          # .1 (Gi2)          .2 #############
############
     | .25 (Gi1)
     |
     |       WAN
     |  10.156.30.0/24
     |
     | .254
############
#    GW    #
############
    |
    |
    |
############                ############
# Internet # -------------- # Umbrella #
############                ############



```


## Day 0: Basics
[Back to top](#cisco-csr1000v)

We start with blank configuration, so we access the CSR via console connection. When we first connect we get to the unprivileged shell.
```
Router>
```
If we want to actually do something we need to elevate to privileged mode.
```
Router>enable
Router#
```
From there we can enter configure terminal 
```
Router#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#
```
When we will configure the device we will move around configure terminal.
```
CSR1000V(config)#interface GigabitEthernet1
CSR1000V(config-if)#
```
To go back just enter `exit` command and you will jump back.
```
CSR1000V(config-if)#exit
CSR1000V(config)#
```

And to go back to the privileged mode enter `end` command.
```
CSR1000V(config)#end
CSR1000V#
```

The modification you do will be saved in running-configuration, you can take a look with `show running-config` command.

```
CSR1000V#show running-config
Building configuration...

Current configuration : 2784 bytes
!
! Last configuration change at 13:28:55 UTC Wed Jul 13 2022 by admin
!
version 16.9
service timestamps debug datetime msec
service timestamps log datetime msec
platform qfp utilization monitor load 80
no platform punt-keepalive disable-kernel-core
platform console virtual
...
```
If we now reboot the device all the changes would be lost. So to prevent that we will save running-config into the statup-config.
```
CSR1000V#write memory
Building configuration...
[OK]
```
We can now start with configuring our device!

## Day 1: Basic configuration of CSR
[Back to top](#cisco-csr1000v)

We will start with setting basic settings:
- set hostname and domain (this is important for generating host key for SSH)
- add `admin` user with password `admin` that have full permissions
- set enable secret to also `admin`
- generate host key for SSH
- enable SSH

Open configure terminal and enter following commands:
```
hostname CSR1000V
ip domain-name lab.local

username admin privilege 15 password admin
enable secret admin

crypto key generate rsa modulus 2048

line vty 0 4
  transport input ssh
  login local
```

Next step is to configure basic networking:
- set ip address of GigabitEthernet1 to 10.156.30.25/24
- set default gateway to 10.156.30.254

Into configure terminal type:
```
interface GigabitEthernet 1
  ip address 10.156.30.25 255.255.255.0
  no shutdown

ip route 0.0.0.0 0.0.0.0 10.156.30.254
```

After that you should be able to ping to the Internet.
```
CSR1000V# ping 8.8.8.8
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 15/15/16 ms
```

And you should be able to SSH to the CSR using credentials `admin/admin` and IP address `10.156.30.25`.

If everything works as expected save running configuration into startup.
```
CSR1000V#write memory
Building configuration...
[OK]
```

## Day 2: NAT between LAN and WAN
[Back to top](#cisco-csr1000v)

We continue with configuring the inside interface GigabitEthernet 2.
```
interface GigabitEthernet 2
  ip address 192.168.0.1 255.255.255.0
  no shutdown
```
If we now check status, we should have both Gi1 and Gi2 up.
```
CSR1000V#show ip int brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.156.30.25    YES manual up                    up
GigabitEthernet2       192.168.0.1     YES manual up                    up
GigabitEthernet3       unassigned      YES unset  down                  down
GigabitEthernet4       unassigned      YES unset  down                  down
GigabitEthernet5       unassigned      YES unset  down                  down
GigabitEthernet6       unassigned      YES unset  down                  down
```

On a Client VM we now configure static ip address to 192.168.0.2/24 and default gateway address to 192.168.0.1. You should be able to ping the default gateway address. Same from the CSR you should be able to ping your Client VM.

```
CSR1000V#ping 192.168.0.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.0.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

Let's now configure NAT. Interface Gi1 is the outside and the Gi2 is in inside interface. We also create access list named `InsideSubnet`. As a source we specify whole inside subnet and as destination we place `any`, as we do not want to limit destinations.

In configure terminal type:
```
interface GigabitEthernet1
  ip nat outside

interface GigabitEthernet2
  ip nat inside

ip access-list extended InsideSubnet
  permit ip 192.168.0.0 0.0.0.255 any

ip nat inside source list InsideSubnet interface GigabitEthernet1 overload
```

Now go to Client VM and try to ping 8.8.8.8. It should work and on the CSR we should see translation. 
```
CSR1000V#show ip nat translations
Pro  Inside global         Inside local          Outside local         Outside global
icmp 10.156.30.25:13338    192.168.0.2:13338     8.8.8.8:13338         8.8.8.8:13338
Total number of translations: 1
```

## Day 3: DHCP Server for LAN
[Back to top](#cisco-csr1000v)

We want to provide DHCP service for our clients on LAN side. We can do that with adding the following configuration.
```
ip dhcp pool LAN
  network 192.168.0.0 255.255.255.0
  default-router 192.168.0.1
  dns-server 8.8.8.8
```

We also want to exclude some ip addresses from the dhcp range. These ip addresses could be later used for static assignments. Add the following line to the config:
```
ip dhcp exclude-address 192.168.0.1 192.168.0.10
```

# Day 4: Management VRF
[Back to top](#cisco-csr1000v)

We want to separate management traffic from data traffic. On switches we achieve this with VLANs and on routers we need VRFs to create different routing table.

We begin with defining the VRF and continue with attaching it to the management interface.

```
ip vrf MGMT
  rd 1:666
```
```
interface GigabitEthernet3
  ip vrf forwarding MGMT
  ip address 192.168.99.10 255.255.255.0
```

We could also add default gateway for this VRF with the following config line, but in case where we do not want to route anywhere we can skip it.
```
ip route vrf MGMT 0.0.0.0 0.0.0.0 192.168.99.1
```

# Day 5: Site-to-Site VPN to Umbrella
[Back to top](#cisco-csr1000v)

## Umbrella
We first need to configure VPN on Umbrella.

Go to Umbrella panel and navigate to Deployments > Network Tunnels. Click on Add button. When asked for Device type select Other. Then scroll down to Configure Tunnel ID and Passphrase section. Keep authentication method FQDN. As Tunnel ID enter some name (`csr1000v-dev` for example) and some random Passphrase. Then click Save.

After that you will get popup with information that you need to enter to the CSR.

```
Tunnel ID: csr1000v-dev@XXXXX-YYYYYY-umbrella.com
Passphrase: SuperSecurePassword111
```

## CSR
We will now configure VPN on CSR.

### IKEv2 Proposal
First create IKEv2 proposal. The selected parameters must be also supported by Umbrella. Check table on https://docs.umbrella.com/umbrella-user-guide/docs/supported-ipsec-parameters.
```
crypto ikev2 proposal umbrella-proposal
  encryption aes-cbc-256
  integrity sha256
  group 19 20
```

### IKEv2 Policy
Now create IKEv2 Policy. This will tie Proposal together with our WAN interface.
```
crypto ikev2 policy umbrella-pol
  proposal umbrella-proposal
  match address local 10.156.30.25
```

### IKEv2 Keyring
Now create IKEv2 Keyring. Specify password that you entered to Umbrella and select Umbrella DC IP address that is in your region. We will use `146.112.67.8`. See: https://docs.umbrella.com/umbrella-user-guide/docs/cisco-umbrella-data-centers.

```
crypto ikev2 keyring umbrella-kr
  peer umbrella
    address 146.112.67.8
    pre-shared-key SuperSecurePassword111
```

### IKEv2 Profile
Create new IKEv2 Profile. In this step enter your Tunnel ID. 
```
crypto ikev2 profile umbrella-ikev2-profile
  match identity remote address 146.112.67.8 255.255.255.255
  identity local email csr1000v-dev@XXXXX-YYYYYY-umbrella.com 
  authentication remote pre-share
  authentication local pre-share
  keyring local umbrella-kr
  dpd 10 2 periodic
```

### Ipsec transform set
Now create ipsec transform set. The selected parameters must be also supported by Umbrella. Check table on https://docs.umbrella.com/umbrella-user-guide/docs/supported-ipsec-parameters
```
crypto ipsec transform-set umbrella-tset esp-aes 256 esp-sha256-hmac
  mode tunnel
```

### Ipsec profile
Let us create new ipsec profile that will tie together IKEv2 profile and ipsec transform set.
```
crypto ipsec profile umbrella-ipsec-profile
  set transform-set umbrella-tset
  set ikev2-profile umbrella-ikev2-profile
```

### Tunnel Interface
At last we create Tunnel Interface. As tunnel source select your WAN interface. That is in our case GigabitEthernet1. Select Ipsec profile that you create in previous step.
```
interface Tunnel1
  ip unnumbered GigabitEthernet1
  tunnel source GigabitEthernet1
  tunnel mode ipsec ipv4
  tunnel destination 146.112.67.8
  tunnel protection ipsec profile umbrella-ipsec-profile
```

Let us check that the ipsec tunnel is established.

```
CSR1000V#show crypto ikev2 session
 IPv4 Crypto IKEv2 Session

Session-id:1, Status:UP-ACTIVE, IKE count:1, CHILD count:1

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         10.156.30.25/4500     146.112.67.8/4500     none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:20, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/828 sec
Child sa: local selector  0.0.0.0/0 - 255.255.255.255/65535
          remote selector 0.0.0.0/0 - 255.255.255.255/65535
          ESP spi in/out: 0x415E12CC/0xC1B99FD9

 IPv6 Crypto IKEv2 Session
```
```
CSR1000V#show crypto ipsec sa

interface: Tunnel1
    Crypto map tag: Tunnel1-head-0, local addr 10.156.30.25

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 146.112.67.8 port 4500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 25, #pkts encrypt: 25, #pkts digest: 25
    #pkts decaps: 22, #pkts decrypt: 22, #pkts verify: 22
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 10.156.30.25, remote crypto endpt.: 146.112.67.8
     plaintext mtu 1422, path mtu 1500, ip mtu 1500, ip mtu idb GigabitEthernet1
     current outbound spi: 0xC1B99FD9(3250167769)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0x415E12CC(1096684236)

```

### Redirect LAN traffic to Tunnel1
Now we just need to redirect traffic from LAN interface Gig2 to newly created Tunnel1.

We begin with creating route map that will take whole inside subnet and point traffic to Tunnel1 interface.
```
route-map umbrella-route-map permit 10
 match ip address InsideSubnet
 set interface Tunnel1
```
And now attach this route map to LAN interface GigabitEthernet2
```
interface GigabitEthernet2
  ip policy route-map umbrella-route-map
```

And now should all traffic from Client VM go to the internet via VPN to Umbrella. You can verify that by running `traceroute` command and see traffic path from Client VM to the destination in the Internet.

# Day 6: Site-to-Site VPN to Umbrella (UNDO)
[Back to top](#cisco-csr1000v)

To remove the VPN enter the following lines in the configure terminal:
```
interface GigabitEthernet2
  no ip policy route-map umbrella-route-map
  exit
no route-map umbrella-route-map permit 10
 
no interface Tunnel1
no crypto ipsec profile umbrella-ipsec-profile
no crypto ipsec transform-set umbrella-tset esp-aes 256 esp-sha256-hmac
no crypto ikev2 profile umbrella-ikev2-profile
no crypto ikev2 keyring umbrella-kr
no crypto ikev2 policy umbrella-pol
no crypto ikev2 proposal umbrella-proposal
```

Finally clear established IKEv2 SA with
```
CSR1000V#clear crypto ikev2 sa
```

# Day 7: Netconf
[Back to top](#cisco-csr1000v)

We can automate CSR using Python library `netmiko`. It connects to the device using SSH and then we send bunch CLI commands to establish state we want.

But easier way to automate network devices is with Netconf. It is more suitable for programmatic approach. For example use `ncclient` library for Python.

Enable netconf by entering the following line in the configure terminal:
```
netconf-yang
```

To test it try to SSH to it to port 830.
```
root@LAPTOP-LO0IHM65:~# ssh -p 830 admin@10.156.30.25 netconf
admin@10.156.30.25's password:

<?xml version="1.0" encoding="UTF-8"?>
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<capabilities>
<capability>urn:ietf:params:netconf:base:1.0</capability>
<capability>urn:ietf:params:netconf:base:1.1</capability>
<capability>urn:ietf:params:netconf:capability:writable-running:1.0</capability>
<capability>urn:ietf:params:netconf:capability:xpath:1.0</capability>
```
And that's it! For constructing YANG payload take a look at [Cisco YANG suite](https://developer.cisco.com/yangsuite/).
