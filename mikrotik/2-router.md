# Router
I am using Mikrotik RB2011UiAS-RM as a router.
## Initial access
Just use serial console and remove existing config with following command.
```
/system reset-configuration no-defaults=yes skip-backup=yes
```

## Configuration
```
# Router

##############################################
# Naming
##############################################

/system identity set name=Router


##############################################
# VLAN Overview
##############################################

# 10 = TRUSTED
# 20 = GUEST
# 99 = MGMT


##############################################
# Bridge
##############################################

# Vlan mode off while we configure
/interface bridge add name=BR1 protocol-mode=none vlan-filtering=no


##############################################
# Access Ports
##############################################

# Ingress

/interface bridge port

add bridge=BR1 interface=ether6 pvid=99
add bridge=BR1 interface=ether7 pvid=10
add bridge=BR1 interface=ether8 pvid=10
add bridge=BR1 interface=ether9 pvid=20

# Egress (handled automatically)

##############################################
# Trunk Ports
##############################################

# Ingress
/interface bridge port

add bridge=BR1 interface=ether10

# Egress

/interface bridge vlan

# All VLANs need L3 services, so include the bridge itself as tagged interface
add bridge=BR1 tagged=BR1,ether10 vlan-ids=10
add bridge=BR1 tagged=BR1,ether10 vlan-ids=20
add bridge=BR1 tagged=BR1,ether10 vlan-ids=99


##############################################
# IP Addressing
##############################################

/interface vlan
add interface=BR1 name=GUEST_VLAN_20 vlan-id=20
add interface=BR1 name=MGMT_VLAN_99 vlan-id=99
add interface=BR1 name=TRUSTED_VLAN_10 vlan-id=10

/ip address
add address=10.0.99.1/24 interface=MGMT_VLAN_99 network=10.0.99.0
add address=10.0.10.1/24 interface=TRUSTED_VLAN_10 network=10.0.10.0
add address=10.0.20.1/24 interface=GUEST_VLAN_20 network=10.0.20.0

# WAN
/ip address add interface=ether1 address=192.168.1.3/24 network=192.168.1.0
/ip route add distance=1 gateway=192.168.1.1

/ip dns set servers=1.1.1.1,8.8.8.8


##############################################
# IP Services
##############################################

# MGMT_VLAN_99 DHCP Server
/ip pool add name=dhcp_pool0 ranges=10.0.99.100-10.0.99.254
/ip dhcp-server add address-pool=dhcp_pool0 disabled=no interface=MGMT_VLAN_99 name=dhcp1
/ip dhcp-server network add address=10.0.99.0/24 gateway=10.0.99.1 dns-server=1.1.1.1,8.8.8.8

# TRUSTED_VLAN_10 DHCP Server
/ip pool add name=dhcp_pool1 ranges=10.0.10.100-10.0.10.254
/ip dhcp-server add address-pool=dhcp_pool1 disabled=no interface=TRUSTED_VLAN_10 name=dhcp2
/ip dhcp-server network add address=10.0.10.0/24 gateway=10.0.10.1 dns-server=1.1.1.1,8.8.8.8

# GUEST_VLAN_20 DHCP Server
/ip pool add name=dhcp_pool2 ranges=10.0.20.100-10.0.20.254
/ip dhcp-server add address-pool=dhcp_pool2 disabled=no interface=GUEST_VLAN_20 name=dhcp3
/ip dhcp-server network add address=10.0.20.0/24 gateway=10.0.20.1 dns-server=1.1.1.1,8.8.8.8


##############################################
# VLAN Security
##############################################

/interface bridge port

# Access ports: Allow only ingress packets without tags
set bridge=BR1 frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes [find interface=ether6]
set bridge=BR1 frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes [find interface=ether7]
set bridge=BR1 frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes [find interface=ether8]
set bridge=BR1 frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes [find interface=ether9]

# Trunk ports: Allow only ingress packets with tags
set bridge=BR1 ingress-filtering=yes frame-types=admit-only-vlan-tagged [find interface=ether10]

##############################################
# Mgmt settings
##############################################

/interface list add name=MGMT
/interface list member add interface=MGMT_VLAN_99 list=MGMT
/ip neighbor discovery-settings set discover-interface-list=MGMT
/tool mac-server mac-winbox set allowed-interface-list=MGMT
/tool mac-server set allowed-interface-list=MGMT

/ip service set ssh address=10.0.99.0/24
/ip service set winbox address=10.0.99.0/24

/ip service set telnet disabled=yes
/ip service set ftp disabled=yes
/ip service set www disabled=yes
/ip service set api disabled=yes
/ip service set api-ssl disabled=yes

#######################################
# Turn on VLAN mode
#######################################

/interface bridge set BR1 vlan-filtering=yes

#######################################
# Firewall
#######################################

/interface list add name=WAN
/interface list member add interface=ether1 list=WAN

/ip firewall filter

add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input in-interface-list=MGMT action=accept
add chain=input action=drop

add chain=forward connection-state=established,related action=fasttrack-connection 
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward connection-state=new connection-nat-state=!dstnat in-interface=ether1 action=drop
add chain=forward connection-state=new in-interface=TRUSTED_VLAN_10 out-interface-list=WAN action=accept
add chain=forward connection-state=new in-interface=GUEST_VLAN_20 out-interface-list=WAN action=accept
add chain=forward action=drop

/ip firewall nat add chain=srcnat action=masquerade out-interface-list=WAN

#######################################
# Other
#######################################

/lcd set enabled=no
/tool bandwidth-server set enabled=no

# Disable interfaces that are not in use
/interface 
set disabled=yes [find name=ether2]
set disabled=yes [find name=ether3]
set disabled=yes [find name=ether4]
set disabled=yes [find name=ether5]
```

## Day 3: Wrap it up
Finally, create new admin account with password and try to login using SSH to 10.0.99.1 IP address.
```
/user add name=netadmin password=netadmin group=full
```
If everything works, disable RoMON and remove default admin account.
```
/tool romon set enabled=no
/user remove admin
```
