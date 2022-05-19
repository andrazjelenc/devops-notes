# Linux Container Network
The plan is to go through linux container networking step by step and see what happens behind the scenes when we use Docker.

Content:
- [Day 0: Bridge and veth interfaces](#day-0-bridge-and-veth-interfaces)
- [Day 1: Network namespaces](#day-1-network-namespaces)
- [Day 2: From container to the Internet](#day-2-from-container-to-the-internet)
- [Day 3: From the internet to the container](#day-3-from-the-internet-to-the-container)
- [Day 4: Limit access between the internet and container](#day-4-limit-access-between-the-internet-and-container)
- [Day 5: Limit access between container and the host](#day-5-limit-access-between-container-and-the-host)
- [Day 6: Limit access between containers](#day-6-limit-access-between-containers)
- [Day 7: Conclusion](#day-7-conclusion)

## Day 0: Bridge and veth interfaces 
[Back to top](#linux-container-network)

We can look at bridge as simple L2 switch. To easier work with them we install tool **brctl** from package **bridge-utils**.
```
# yum install bridge-utils
```

We can now inspect the status of bridges we currently have and see we have none.
```
# brctl show
bridge name     bridge id               STP enabled     interfaces
```

Let's create our first bridge and call it bridge0
```
# brctl addbr bridge0
# brctl show
bridge name     bridge id               STP enabled     interfaces
bridge0         8000.000000000000       no
```

Now we need to create some interfaces and attach them to the bridge. We will be using interfaces of type [**VETH**](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#veth). The VETH devices act as a local Ethernet tunnel. Each tunnel has two interfaces and packages that are send on one device and then immediately seen on the other device from this pair. Let's create two pairs of VETH interfaces.
```
# ip link add veth1-a type veth peer name veth1-b
# ip link add veth2-a type veth peer name veth2-b
```
If we now inspect current status we can see all our devices. Note that the bridge and veth devices are all in DOWN state.
```
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
    inet 10.156.30.11/24 brd 10.156.30.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::211:22ff:fe33:4455/64 scope link
       valid_lft forever preferred_lft forever
14: bridge0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 4e:9a:b2:55:cc:f6 brd ff:ff:ff:ff:ff:ff
15: veth1-b@veth1-a: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 9e:b0:4e:eb:b1:33 brd ff:ff:ff:ff:ff:ff
16: veth1-a@veth1-b: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether da:36:a9:f5:13:58 brd ff:ff:ff:ff:ff:ff
17: veth2-b@veth2-a: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 82:8c:f2:15:e8:c9 brd ff:ff:ff:ff:ff:ff
18: veth2-a@veth2-b: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 36:c5:8b:91:9f:b7 brd ff:ff:ff:ff:ff:ff
```

We will now add veth1-a and veth2-a to our bridge.
```
# brctl addif bridge0 veth1-a veth2-a
# brctl show
bridge name     bridge id               STP enabled     interfaces
bridge0         8000.36c58b919fb7       no              veth1-a
                                                        veth2-a
```

Bring all the devices up.
```
# ip link set up bridge0
# ip link set up veth1-a
# ip link set up veth1-b
# ip link set up veth2-a
# ip link set up veth2-b
```

We currently have situation like this
```
              bridge0
              /     \
          veth1-a  veth2-a 
            |         |
          veth1-b  veth2-b 
```

Let's now assign IP addresses from the same range to the devices bridge0, veth1-b and veth2-b.
```
# ip addr add 192.168.20.1/24 dev bridge0
# ip addr add 192.168.20.2/24 dev veth1-b
# ip addr add 192.168.20.3/24 dev veth2-b
```

We can now test if our bridge works correctly. Start tcpdump on bridge interface and let it run.
```
# tcpdump -nn -i bridge0
```
Open new terminal and try to ping from veth1-b to veth2-b.
```
# ping 192.168.20.3 -I veth1-b
```
```
# tcpdump -nn -i bridge0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on bridge0, link-type EN10MB (Ethernet), capture size 262144 bytes
08:30:18.017305 ARP, Request who-has 192.168.20.3 tell 192.168.20.2, length 28
08:30:19.019510 ARP, Request who-has 192.168.20.3 tell 192.168.20.2, length 28
08:30:20.021520 ARP, Request who-has 192.168.20.3 tell 192.168.20.2, length 28
```
We see that the traffic successfully reached our bridge interface! 

## Day 1: Network namespaces
[Back to top](#linux-container-network)

We are now going to segment our host network. We want to achieve something like that:
```
###################################
# host                            #
#              bridge0            #
#              /     \            #
#          veth1-a  veth2-a       #
#            |         |          #
#  ##############  ############## #
#  #    veth1-b #  # veth2-b    # #
#  #            #  #            # #
#  # container1 #  # container2 # #
#  ##############  ############## #
#                                 #
###################################      
```

We don't want veth1-b and veth2-b interface to be shown as we run `ip addr` on the host.

We will start with inspecting current network namespaces on our machine.
```
 # lsns --type=net
         NS TYPE NPROCS PID USER COMMAND
4026531956 net     167   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
```

We see that only our main network namespace is present. To segment the network we will now create two additional namespaces, ns1 for container1 and ns2 for container2. We will then move veth1-b to ns1 and veth2-b to ns2.
```
# ip netns add ns1
# ip netns add ns2
# ip link set veth1-b netns ns1
# ip link set veth2-b netns ns2
```

If we now check `ip addr` we see that veth1-b and veth2-b are no longer there.
```
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
    inet 10.156.30.11/24 brd 10.156.30.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::211:22ff:fe33:4455/64 scope link
       valid_lft forever preferred_lft forever
14: bridge0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 36:c5:8b:91:9f:b7 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::34c5:8bff:fe91:9fb7/64 scope link
       valid_lft forever preferred_lft forever
16: veth1-a@if15: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue master bridge0 state LOWERLAYERDOWN group default qlen 1000
    link/ether da:36:a9:f5:13:58 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::d836:a9ff:fef5:1358/64 scope link
       valid_lft forever preferred_lft forever
18: veth2-a@if17: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue master bridge0 state LOWERLAYERDOWN group default qlen 1000
    link/ether 36:c5:8b:91:9f:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::34c5:8bff:fe91:9fb7/64 scope link
       valid_lft forever preferred_lft forever
```

But if we check status of network namespaces with lsns, we won't see ns1 and ns2. This is because there isn't any process running inside these namespaces. But we can see them with `ip netns` command.
```
# ip netns
ns2 (id: 1)
ns1 (id: 0)
```

We can now enter our newly created namespaces. Let's enter ns1 with bash shell.
```
# ip netns exec ns1 bash
# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
15: veth1-b@if16: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 9e:b0:4e:eb:b1:33 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

As you can see in ns1 only veth1-b interface exists.
If we now open new terminal we can see that our namespace is now visible in lsns.
```
# lsns --type=net
        NS TYPE NPROCS   PID USER COMMAND
4026531956 net     169     1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532438 net       1 11106 root bash
```

We now need to configure interfaces in ns1 and ns2. We can modify them from host using `ip netns exec` command.
```
# ip netns exec ns1 ip addr add 192.168.20.2/24 dev veth1-b
# ip netns exec ns2 ip addr add 192.168.20.3/24 dev veth2-b
# ip netns exec ns1 ip link set up veth1-b
# ip netns exec ns2 ip link set up veth2-b
```

Also ensure that bridge0 has correct IP address.
```
# ip addr add 192.168.20.1/24 dev bridge0
# ip link set up bridge0
```

Now we can try our configuration. Start tcpdump on bridge0 and in another terminal try to ping from interface in ns1 to interface in ns2.
```
# ip netns exec ns1 ping 192.168.20.3
PING 192.168.20.3 (192.168.20.3) 56(84) bytes of data.
64 bytes from 192.168.20.3: icmp_seq=1 ttl=64 time=0.076 ms
64 bytes from 192.168.20.3: icmp_seq=2 ttl=64 time=0.071 ms
```
```
# tcpdump -nn -i bridge0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on bridge0, link-type EN10MB (Ethernet), capture size 262144 bytes
09:27:23.444325 IP 192.168.20.2 > 192.168.20.3: ICMP echo request, id 11161, seq 1, length 64
09:27:23.444366 IP 192.168.20.3 > 192.168.20.2: ICMP echo reply, id 11161, seq 1, length 64
09:27:24.443617 IP 192.168.20.2 > 192.168.20.3: ICMP echo request, id 11161, seq 2, length 64
09:27:24.443656 IP 192.168.20.3 > 192.168.20.2: ICMP echo reply, id 11161, seq 2, length 64
```
As you can see, the traffic can get from one namespace to another one. We can also ping from host into our namespaces and from our namespaces to our host using bridge0 IP address.

```
# ip netns exec ns1 ping 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.117 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=0.074 ms
```
```
# ping 192.168.20.2
PING 192.168.20.2 (192.168.20.2) 56(84) bytes of data.
64 bytes from 192.168.20.2: icmp_seq=1 ttl=64 time=0.048 ms
64 bytes from 192.168.20.2: icmp_seq=2 ttl=64 time=0.056 ms
```
## Day 2: From container to the Internet
[Back to top](#linux-container-network)

We can try to access internet from our namespace.
```
# ip netns exec ns1 ping 8.8.8.8
connect: Network is unreachable
```

We can also try to ping "public" interface of our host.
```
# ip netns exec ns1 ping 10.156.30.11
connect: Network is unreachable
```
Something is missing!

We first need to check routing inside our namespaces.
```
# ip netns exec ns1 ip route
192.168.20.0/24 dev veth1-b proto kernel scope link src 192.168.20.2
```

Let's add default gateway that point to our bridge0.
```
# ip netns exec ns1 ip route add default via 192.168.20.1 dev veth1-b
# ip netns exec ns2 ip route add default via 192.168.20.1 dev veth2-b
```
We also need to enable IPv4 forwarding on our host. Check if forwarding is already enabled
```
# sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
```
And if response is 0 (disabled), turn it on
```
# sysctl -w net.ipv4.ip_forward=1
```

Pinging our "public" interface is now working, but we still cannot successfully access the internet.
```
# ip netns exec ns1 ping 10.156.30.11
PING 10.156.30.11 (10.156.30.11) 56(84) bytes of data.
64 bytes from 10.156.30.11: icmp_seq=1 ttl=64 time=0.099 ms
64 bytes from 10.156.30.11: icmp_seq=2 ttl=64 time=0.063 ms
```

The problem is that package's source address is not replaced by IP address of our "public" interface and therefore the traffic does not find the path back to our host. We need to configure NAT, so the traffic from our host will always have ens192 interface IP address as source IP. We can achieve this using iptables. To simplify chains, we will first stop the firewalld, the wrapper around iptables.
```
# systemctl stop firewalld
```
If we now inspect the iptables' chains we should see empty chains.
```
# iptables -L -nv
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

# iptables -L -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```
Iptables consists of many tables, but for us only filter and nat tables matter.

We now add rule that will replace source IP of every package that will leave our host via ens192 with ens192 IP address. With this rule we will mask IP address from our internal network 192.168.20.0/24, so the traffic from outside will find the way back to our host.
```
# iptables -t nat -A POSTROUTING -o ens192 -j MASQUERADE
```

We can now try to ping internet again.
```
# ip netns exec ns1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=13.7 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=13.7 ms
```

## Day 3: From the internet to the container
[Back to top](#linux-container-network)

We sometimes want to run application that will listen on some port. If the application runs inside container, we need to configure iptables to map port on host to port inside container.

Let's say that we want to map host port 8080 to the port 80 inside ns1. This could be because we are running webserver like Nginx inside container. For our simulation we will use netcat that will listen on port 80. And we will use telnet to establish connection.

Run netcat listener
```
# ip netns exec ns1 nc -l -v -n -p 80
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
```

And now execute telnet command from host to establish connection.
```
# telnet 192.168.20.2 80
Trying 192.168.20.2...
Connected to 192.168.20.2.
Escape character is '^]'.
```
Connections is established! To close the connection press CTRL+Č and enter `quit`. Now start the netcat listener again.

It also works when we try to connect from container2.
```
]# ip netns exec ns2 telnet 192.168.20.2 80
Trying 192.168.20.2...
Connected to 192.168.20.2.
Escape character is '^]'.
```

So now we know that from host (and also from container2) we can access port 80 inside container1.

Now, we need to expose this port 80 from container1 to the outside world. Add new rule to our iptables that will redirect all incoming traffic on port 8080 to the container1 to port 80.
```
# iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.20.2:80
```

We will now run telnet from some other machine and target our "public" IP address and port 8080.
```
[user@outside ~]# telnet 10.156.30.11 8080
Trying 10.156.30.11...
Connected to 10.156.30.11.
Escape character is '^]'.
```

And connection is established! Our netcat listener shows connection information.
```
# ip netns exec ns1 nc -l -v -n -p 80
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
Ncat: Connection from 10.156.30.45.
Ncat: Connection from 10.156.30.45:49564.
```

## Day 4: Limit access between the internet and container
[Back to top](#linux-container-network)

When we created new namespace for our container, we kind of duplicated whole Linux network stack. This also means that inside namespace ns1 we have its own iptables rules. We can check that using `ip netns exec` command.

```
# ip netns exec ns1 iptables -L -nv -t nat
Chain PREROUTING (policy ACCEPT 6 packets, 384 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 6 packets, 384 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 20 packets, 1605 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 20 packets, 1605 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

We could set rules inside namespace and limit access here, but we are usually not doing that. This is because the process (if running as root) inside container can modify these iptables. So the plan is to restrict access using host main iptables rules.

If we inspect iptables on the host we see that all defaults are set to ACCEPT. And only two rules are in NAT table.
```
# iptables -L -nv
Chain INPUT (policy ACCEPT 3021 packets, 231K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 20 packets, 1445 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 1969 packets, 251K bytes)
 pkts bytes target     prot opt in     out     source               destination

# iptables -L -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    60 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.20.2:80

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination
    7   526 MASQUERADE  all  --  *      ens192  0.0.0.0/0            0.0.0.0/0
```
We will limit exposure with adding rules to FILTER table. This is also the default table, so we do not need to specify "-t" flag.
```
                  Incoming packet 
                         |
                         V
                   NAT PREROUTING           Application
                         |                      |
                         V                      V
                  For this host?             NAT OUTPUT
                   YES      NO                  |
                   /         \                  V
                  /           \             FILTER OUTPUT
                 V             V                |
            FILTER INPUT    FILTER FORWARD      |
                 |             |                |
                 V             V                |
            Application     NAT POSTROUTING <----
                               |
                               V
                            Outgoing packet
```

Our traffic from the internet to the container will come to ens192 interface. It will first hit NAT PREROUTING chain that will determine that the traffic is NOT for this host (due to our DNAT rule), then the FILTER FORWARD for limiting access and then NAT POSTROUTING chain. After that the traffic will be routed to bridge0 interface.

The traffic from container to the internet will come to bridge0 interface. And it will also first hit NAT PREROUTING that will determine that the traffix is NOT for this host (we are aiming to the internet), then FILTER FORWARD and then NAT POSTROUTING, where masquarede will happen. The traffic will then continue to interface ens192 and to the internet. 

So FORWARD chain of FILTER table will be hit on the way in and on the way out. We will first modify default policy from ACCEPT to DROP.
```
# iptables -P FORWARD DROP
```
We now add rule for accepting all established and related traffic.
```
# iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```
If we now try to ping from container to the internet, we are blocked.
```
# ip netns exec ns1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 3999ms
```
And if we check iptables, we see how many packets got blocked (see "policy DROP 7 packets" next to FORWARD chain).
```
# iptables -L -nv
Chain INPUT (policy ACCEPT 101 packets, 7576 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 7 packets, 588 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT 67 packets, 9560 bytes)
 pkts bytes target     prot opt in     out     source               destination
 ```

So to allow all traffic from container to the internet we just add the following rule. This will allow all traffic from our bridge0 interface to the ens192 "public" interface.
```
# iptables -A FORWARD -i bridge0 -o ens192 -j ACCEPT
```

But establishing connection from outside world will not work.
```
[user@outside ~]# telnet 10.156.30.11 8080
Trying 10.156.30.11...
```
---
NOTE:
Access from container2 to container1 works, since they are both in the same segment.

---

So right now, nobody from the internet can access our application inside container. But we can open specific port to the specific host.
```
# iptables -A FORWARD -i ens192 -o bridge0 -p tcp -d 192.168.20.2 --dport 80 -j ACCEPT
```

Now you can again establish connection from outside world using `telnet 10.156.30.11 8080`. Note that in iptables we specify port 80 as a destination port. This is because the traffic first hit PREROUTING chain of NAT table and only then FORWARD chain of FILTER.

So now we know how to filter from the internet to the container and from container to the internet.


## Day 5: Limit access between container and the host
[Back to top](#linux-container-network)

Let's run netcat listener on our host. We can then connect to it from container using bridge IP address or even IP address of ens192 interface.
```
# nc -l -v -n -p 80
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
Ncat: Connection from 192.168.20.2.
Ncat: Connection from 192.168.20.2:54174.
```
```
# ip netns exec ns1 telnet 192.168.20.1 80
Trying 192.168.20.1...
Connected to 192.168.20.1.
Escape character is '^]'.
```

The traffic from container will come to bridge0. It will pass NAT PREROUTING and because this traffic is for this host it will hit FILTER INPUT chain.

So to block all access from container to the host we can add one rule to the INPUT table.
```
iptables -A INPUT -i bridge0 -j DROP
```
With this rule we blocked access to the host.
```
# ip netns exec ns1 telnet 10.156.30.11 80
Trying 10.156.30.11...
```

We can do the same for limiting access from host to container. With a simple command we prevent packages from host application to access bridge0 interface and therefore containers.
```
# iptables -A OUTPUT -o bridge0 -j DROP
```

---
NOTE: In case when we want to allow communication only between containers that are tied to the same bridge, we can remove IP address from bridge itself. With that we lose all L3 functionality on our bridge (routing to other networks, firewalling...).
```
# ip addr delete 192.168.20.1/24 dev bridge0
```
We could add to the same network namespace two different veth interfaces (with two different subnets) that are tied to two different bridges without L3 functionality and then run routing between them inside this namespace instead on main network namespace.

---

## Day 6: Limit access between containers
[Back to top](#linux-container-network)

If we want to filter traffic between two containers we first need to put them into separate segments. This means we need to create new bridge. And then we will be able to filter traffic between bridge0 and brige1.
```
###################################
# host                            #
#       bridge0    bridge1        #
#          |          |           #
#       veth1-a    veth2-a        #
#          |          |           #
#  ##############  ############## #
#  #    veth1-b #  # veth2-b    # #
#  #            #  #            # #
#  # container1 #  # container2 # #
#  ##############  ############## #
#                                 #
###################################      
```

So we create new bridge named bridge1 and move veth2-a from bridge0 to bridge1.
```
# brctl addbr bridge1
# brctl delif bridge0 veth2-a
# brctl addif bridge1 veth2-a

# brctl show
bridge name     bridge id               STP enabled     interfaces
bridge0         8000.da36a9f51358       no              veth1-a
bridge1         8000.36c58b919fb7       no              veth2-a
```

We now configure basic IP settings on newly created bridge1 and modify veth2-b address to 192.168.30.3.
```
# ip link set up bridge1
# ip link set up veth2-a
# ip addr add 192.168.30.1/24 dev bridge1
# ip netns exec ns2 ip addr delete 192.168.20.3/24 dev veth2-b
# ip netns exec ns2 ip addr add 192.168.30.3/24 dev veth2-b
# ip netns exec ns2 ip link set up veth2-b
# ip netns exec ns2 ip route add default via 192.168.30.1 dev veth2-b
```

After that we can check setup inside the container2
```
# ip netns exec ns2 ip route
default via 192.168.30.1 dev veth2-b
192.168.30.0/24 dev veth2-b proto kernel scope link src 192.168.30.3

# ip netns exec ns2 ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
17: veth2-b@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 82:8c:f2:15:e8:c9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.30.3/24 scope global veth2-b
       valid_lft forever preferred_lft forever
    inet6 fe80::808c:f2ff:fe15:e8c9/64 scope link
       valid_lft forever preferred_lft forever
```

We can now allow all traffic from the container2 to the internet. Just like we did before.
```
# iptables -A FORWARD -i bridge1 -o ens192 -j ACCEPT
```

So now we can ping to the internet
```
# ip netns exec ns2 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=13.6 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=13.8 ms
```

We can also ping bridge0, but we cannot access container1
```
# ip netns exec ns2 ping 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.057 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=0.063 ms

[root@centos7-networking ~]# ip netns exec ns2 ping 192.168.20.2
PING 192.168.20.2 (192.168.20.2) 56(84) bytes of data.
^C
--- 192.168.20.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms
```

Let's assume we want to allow ICMP traffic from bridge1 to bridge0, but not other way around.
```
# iptables -A FORWARD -i bridge1 -o bridge0 -p icmp -j ACCEPT
```

We can check that it works only in one way:
```
# ip netns exec ns2 ping 192.168.20.2
PING 192.168.20.2 (192.168.20.2) 56(84) bytes of data.
64 bytes from 192.168.20.2: icmp_seq=1 ttl=63 time=0.138 ms
64 bytes from 192.168.20.2: icmp_seq=2 ttl=63 time=0.088 ms

[root@centos7-networking ~]# ip netns exec ns1 ping 192.168.20.3
PING 192.168.20.3 (192.168.20.3) 56(84) bytes of data.
^C
--- 192.168.20.3 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

Great, the final step is to allow port 80 to be accessible from container2, just do similar as above
```
# iptables -A FORWARD -i bridge1 -o bridge0 -p tcp --dport 80 -j ACCEPT
```
Because we did not specify the destination IP address we could access port 80 from every container that is connected to bridge0.

Final check using telnet:
```
# ip netns exec ns2 telnet 192.168.20.2 80
Trying 192.168.20.2...
Connected to 192.168.20.2.
Escape character is '^]'.
```
```
# ip netns exec ns1 nc -l -v -n -p 80
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
Ncat: Connection from 192.168.30.3.
Ncat: Connection from 192.168.30.3:35854.
```

## Day 7: Conclusion
[Back to top](#linux-container-network)

We should now have a better picture at how linux container network works. All this details are hidden from us when using Docker. We now understand that in order to access service from container that is connected to the same bridge we do not need to expose port or do anything else. The biggest mistake people do (in connection with container networks at least) is to expose everything to the outside, including private services (like Postgres) to the internet. This is in combination with weak default passwords a recipe for disaster!