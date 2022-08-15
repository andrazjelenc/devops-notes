# Access Point
I am using Mikrotik cAP RBcAP2nD to broadcast my home wifi and a separate wifi for guests. I want that my access points acts only as access point and leave all L3 services to the router.

## Day 0: Initial access
We can not use serial console for accessing this access point, so we need to use different approach. When we first power it on, the default configuration is loaded. It contains one unprotected wifi interface, so we can connect to it and access it using WinBox.

Because we will modify L3 and L2 settings we need to access the device via RoMON that is independent MAC layer peer discovery and data forwarding network.

On your PC create new file called `romon.rsc` with following content:
```
{
:delay 15s
/tool romon set enabled=yes
}
```
Now upload this file to your access point flash storage. Use WinBox, click on Files and select Upload button.

Now open new terminal and execute following command. It will remove all existing configuration and execute newly uploaded file after reset. It will keep RoMON on, but remove all other configuration.
```
/system reset-configuration no-defaults=yes skip-backup=yes run-after-reset=flash/romon.rsc
```

Wait few minutes for access point to reboot and now you can access it via router using RoMON. The RoMON must be enabled on the router too!

## Day 1: Configuration
```
# Access point AP1

##############################################
# Naming
##############################################

/system identity set name=AP1


##############################################
# VLAN Overview
##############################################

# 10 = TRUSTED
# 20 = GUEST
# 99 = MGMT


##############################################
# WIFI
##############################################

# JelencAP
/interface wireless security-profiles add name=JelencAP authentication-types=wpa2-psk mode=dynamic-keys wpa2-pre-shared-key=password1
/interface wireless set [ find default-name=wlan1 ] ssid=JelencAP mode=ap-bridge security-profile=JelencAP band=2ghz-b/g/n channel-width=20/40mhz-XX country=slovenia disabled=no distance=indoors frequency=auto installation=indoor wireless-protocol=802.11 wps-mode=disabled

# JelencAP-Guest
/interface wireless security-profiles add name=JelencAP-Guest authentication-types=wpa2-psk mode=dynamic-keys wpa2-pre-shared-key=password2
/interface wireless add name=wlan2 ssid=JelencAP-Guest disabled=no keepalive-frames=disabled mac-address=1A:FD:74:4A:60:5C master-interface=wlan1 multicast-buffering=disabled security-profile=JelencAP-Guest  wds-cost-range=0 wds-default-cost=0 wps-mode=disabled


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

add bridge=BR1 interface=wlan1 pvid=10
add bridge=BR1 interface=wlan2 pvid=20


# Egress (handled automatically)

##############################################
# Trunk Ports
##############################################

# Ingress
/interface bridge port

add bridge=BR1 interface=ether1

# Egress

/interface bridge vlan

# On MGMT VLAN we need L3 services, so include the bridge itself as tagged interface
add bridge=BR1 tagged=ether1 vlan-ids=10
add bridge=BR1 tagged=ether1 vlan-ids=20
add bridge=BR1 tagged=ether1,BR1 vlan-ids=99


##############################################
# IP Addressing
##############################################

/interface vlan add interface=BR1 name=MGMT_VLAN_99 vlan-id=99
/ip address add address=10.0.99.2/24 interface=MGMT_VLAN_99 network=10.0.99.0

/ip route add distance=1 gateway=10.0.99.1

##############################################
# VLAN Security
##############################################

/interface bridge port

# Access ports: Allow only ingress packets without tags
set bridge=BR1 frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes [find interface=wlan1]
set bridge=BR1 frame-types=admit-only-untagged-and-priority-tagged ingress-filtering=yes [find interface=wlan2]

# Trunk ports: Allow only ingress packets with tags
set bridge=BR1 ingress-filtering=yes frame-types=admit-only-vlan-tagged [find interface=ether1]

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
```

## Day 3: Wrap it up
Finally, create new admin account with password and try to login using SSH to 10.0.99.2 IP address.
```
/user add name=netadmin password=netadmin group=full
```
If everything works, disable RoMON and remove default admin account.
```
/tool romon set enabled=no
/user remove admin
```
