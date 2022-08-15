# Design
In this document there is a design of my home network. I currently have one Mikrotik router called Router and one Access Point called AP1. The AP1 acts only as an access point and it broadcasts two SSIDs one for home usage and one for guests.

## VLANs

- Name: MGMT_VLAN_99  
  VLAN ID: 99  
  Subnet: 10.0.99.0/24

- Name: TRUSTED_VLAN_10  
  VLAN ID: 10  
  Subnet: 10.0.10.0/24

- Name: GUEST_VLAN_20  
  VLAN ID: 20  
  Subnet: 10.0.20.0/24

## Interfaces

- Router:
    - sfp1: Disabled
    - ether1: Untagged (Uplink to ISP router)
    - ether2: Disabled
    - ether3: Disabled
    - ether4: Disabled
    - ether5: Disabled
    - ether6: Access port to MGMT_VLAN_99
    - ether7: Access port to TRUSTED_VLAN_10
    - ether8: Access port to TRUSTED_VLAN_10
    - ether9: Access port to GUEST_VLAN_20
    - ether10: Trunk port for AP1 (pasive PoE)

- AP1:
    - ether1: Trunk port to Router (pasive PoE)
    - wlan1: Access port to TRUSTED_VLAN_10 (SSID: JelencAP)
    - wlan2: Access port to GUEST_VLAN_20 (SSID: JelencAP-Guest)


## IP Addresses

- Router:
    - ether1: 192.168.1.3/24
    - MGMT_VLAN_99: 10.0.99.1/24
    - TRUSTED_VLAN_10: 10.0.10.1/24
    - GUEST_VLAN_20: 10.0.20.1/24

- AP1:
    - MGMT_VLAN_99: 10.0.99.2/24