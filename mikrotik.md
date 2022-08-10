# Mikrotik

## Day 0: Serial console
1. Get a correct serial cable. I am using [USB-A to RJ45 cable with FT232RL chip](https://krajnik.si/proizvod/ugreen-usb-a-na-rj45-kabel-mrezna-kartica-1-5m/).
2. On your PC install drivers for your cable. Read manuals that comes with the cable.
3. Connect console cable to your PC.
4. Identify serial port by opening "Computer management" and then "Device Manager". Your port should be specified under Ports (COM & LPT). Let's assume it is on COM3.
5. Open Putty and under Connection->Serial set:
    - Serial line: COM3
    - Speed: 115200
    - Data bits: 8
    - Stop bits: 1
    - Parity: None
    - Flow control: None
    
    Navigate to Session tab, select Serial and click on Open button.
6. Press ENTER.
7. Login using your credentials.

## Day 1: Basic Configuration

We will use serial console for removing existing configuration and setting initial IP addresses. First remove existing configuration with following command.
```
/system reset-configuration no-defaults=yes skip-backup=yes
```

Now login using username `admin` and blank password.

We will configure the following scenario. Interface ether1 will be used for WAN with IP address 192.168.1.3/24 and default gateway 192.168.1.1. Interfaces ether2 and ether3 will be used for LAN. We will join them inside bridge0. The bridge0 IP address will be 10.0.0.1/24 and there will also be DHCP server running inside bridge0.

```
   WAN        LAN
    |          |     
##########################
# ether1    bridge0      #
#            /   \       #
#        ether2 ether3   #
#                        #
##########################
```

Let's start with establishing upstream connection. Assign IP address to the WAN interface, set default gateway and DNS servers.

```
/ip address add address=192.168.1.3/24 interface=ether1
/ip route add gateway=192.168.1.1
/ip dns set servers=8.8.8.8
```

We should be now able to ping to the outside world.
```
[admin@RouterOS] > /ping google.com
  SEQ HOST                                     SIZE TTL TIME  STATUS
    0 142.250.203.110                            56 118 12ms
    1 142.250.203.110                            56 118 12ms
    2 142.250.203.110                            56 118 12ms
    sent=3 received=3 packet-loss=0% min-rtt=12ms avg-rtt=12ms max-rtt=12ms
```

Now create LAN bridge and assign interfaces to it.
```
/interface bridge add name=bridge0
/interface bridge port add interface=ether2 bridge=bridge0
/interface bridge port add interface=ether3 bridge=bridge0
/ip address add address=10.0.0.1/24 interface=bridge0
```

For creating DHCP server just run wizard with `/ip dhcp-server setup`.
Alternatively you can create DHCP server without wizard.
```
/ip pool add name=dhcp_pool0 ranges=10.0.0.100-10.0.0.254
/ip dhcp-server add address-pool=dhcp_pool0 disabled=no interface=bridge0 name=dhcp1
/ip dhcp-server network add address=10.0.0.0/24 dns-server=8.8.8.8 gateway=10.0.0.1
```

We can now try to ping outside world from LAN side, but it won't work. The reason is that we still need to configure NAT between WAN and LAN interfaces.
```
[admin@RouterOS] /ping src-address=10.0.0.1 google.com
  SEQ HOST                                     SIZE TTL TIME  STATUS
    0 172.217.168.78                                          timeout
    1 172.217.168.78                                          timeout
    2 172.217.168.78                                          timeout
    sent=3 received=0 packet-loss=100%
```

We have to add masquerade rule to traffic leaving WAN interface.
```
 /ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

Let's try pinging outside world again and it should work.
```
[admin@RouterOS] /ping src-address=10.0.0.1 google.com
  SEQ HOST                                     SIZE TTL TIME  STATUS                                                                                         
    0 172.217.168.78                             56 117 13ms
    1 172.217.168.78                             56 117 13ms
    2 172.217.168.78                             56 117 13ms
    sent=3 received=3 packet-loss=0% min-rtt=13ms avg-rtt=13ms max-rtt=13ms
```

If we now connect our PC to ether2 or ether3 interfaces we should get IP address from DHCP server and we should be able to access outside world.

## Day 2: Secure router

Add new password protected user with admin rights and remove default `admin` account. Before removing `admin` account test if your newly created account works! If you are connected to ether2 port, you should be able to use SSH to 10.0.0.1 IP Address.

```
/user add name=netadmin password=netadmin group=full
/user remove admin
```

And if you are not using any legacy device, enable strong SSH crypto.
```
/ip ssh set strong-crypto=yes
```

Mikrotik is also running mac server, that allows everybody to connect to it with just mac address. If you show running configuration with command `export` you will not see it! You must run `export verbose` to really get everything.

We will limit this feature to our LAN bridge only! We will also limit feature called neighbor discovery to our LAN bridge0.
```
/interface list add name=list_bridge0
/interface list member add list=list_bridge0 interface=bridge0
/tool mac-server set allowed-interface-list=list_bridge0
/tool mac-server mac-winbox set allowed-interface-list=list_bridge0
/ip neighbor discovery-settings set discover-interface-list=list_bridge0
```

Next we configure firewall to limit access to our router. We only allow SSH, WinBox and ICMP traffic from LAN.
```
/ip firewall filter add chain=input connection-state=established,related action=accept
/ip firewall filter add chain=input connection-state=invalid action=drop
/ip firewall filter add chain=input in-interface=bridge0 protocol=tcp port=22 action=accept
/ip firewall filter add chain=input in-interface=bridge0 protocol=tcp port=8291 action=accept comment="Allow WinBox"
/ip firewall filter add chain=input in-interface=bridge0 protocol=icmp action=accept
/ip firewall filter add chain=input action=drop;
```
If you want to also allow ICMP traffic from WAN you need to insert the rule before last drop everything rule. Find rule number with print command.
```
[netadmin@RouterOS] /ip firewall filter print
Flags: X - disabled, I - invalid, D - dynamic
 0    chain=input action=accept connection-state=established,related

 1    chain=input action=drop connection-state=invalid

 2    chain=input action=accept protocol=tcp in-interface=bridge0 port=22

 3    ;;; Allow WinBox
      chain=input action=accept protocol=tcp in-interface=bridge0 port=8291

 4    chain=input action=accept protocol=icmp in-interface=bridge0

 5    chain=input action=drop

```

Now we know we want to insert our rule before rule 5.
```
/ip firewall filter add chain=input in-interface=ether1 protocol=icmp action=accept place-before=5
```

Now we just need to disable some administrative services that we do not need. We can check what is running with print command.
```
[netadmin@RouterOS] /ip service print
Flags: X - disabled, I - invalid
 #   NAME          PORT       ADDRESS          CERTIFICATE
 0   telnet          23
 1   ftp             21
 2   www             80
 3   ssh             22
 4 XI www-ssl       443                               none
 5   api           8728
 6   winbox        8291
 7   api-ssl       8729                               none
```
We will disable everything except ssh and WinBox. We will also limit services to our bridge0 network.
```
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set ssh address=10.0.0.0/24
/ip service set winbox address=10.0.0.0/24
```

There is also bandwidth server running on our router. Disable it!
```
/tool bandwidth-server set enabled=no
```

Did you notice that your router may have fancy LCD display on it :) Disable it!
```
/lcd set enabled=no
```

Finally we want to shutdown interfaces that are not in use.
```
[netadmin@RouterOS] /interface print
Flags: D - dynamic, X - disabled, R - running, S - slave
 #     NAME                                TYPE       ACTUAL-MTU L2MTU
 0  R  ether1                              ether            1500  1598
 1  RS ether2                              ether            1500  1598
 2   S ether3                              ether            1500  1598
 3     ether4                              ether            1500  1598
 4     ether5                              ether            1500  1598
 5     ether6                              ether            1500  1598
 6     ether7                              ether            1500  1598
 7     ether8                              ether            1500  1598
 8     ether9                              ether            1500  1598
 9     ether10                             ether            1500  1598
10     sfp1                                ether            1500  1598
11  R  bridge0                             bridge           1500  1598
```

Let's disable all interfaces from 3 to 10.
```
/interface set 3 disabled=yes
/interface set 4 disabled=yes
/interface set 5 disabled=yes
/interface set 6 disabled=yes
/interface set 7 disabled=yes
/interface set 8 disabled=yes
/interface set 9 disabled=yes
/interface set 10 disabled=yes
```

## Day 2: Secure clients

We will protect clients that are hidden behind our NAT. Just two notes:
- Rule with action=fasttrack-connection allows bypassing the firewall and reducing CPU usage.
- Last rule block access from WAN to LAN even if somebody guess private IP address.

```
/ip firewall filter 
  add chain=forward action=fasttrack-connection connection-state=established,related;
  add chain=forward action=accept connection-state=established,related;
  add chain=forward action=drop connection-state=invalid;
  add chain=forward action=drop connection-state=new connection-nat-state=!dstnat in-interface=ether1;
```
