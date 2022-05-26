# Docker Network

After you install Docker on your machine and run one of two containers, you can find some changes to your host network were made. The plan is to go step by step and try to understand why and what happens when you run a container. This is not a beginner guide to Docker. You may want to get familiar with Linux Container Networking first.

Content:
- [Day 0: Bridge and veth interfaces](#day-0-bridge-and-veth-interfaces)
- [Day 1: Network namespaces](#day-1-network-namespaces)
- [Day 2: Custom networks](#day-3-custom-networks)
- [Day 3: Internal custom networks](#day-3-internal-custom-networks)
- [Day 4: Exposing ports to the outside](#day-4-exposing-ports-to-the-outside)
- [Day 5: DNS Resolution in default network](#day-5-dns-resolution-in-default-network)
- [Day 6: DNS Resolution in custom network](#day-6-dns-resolution-in-custom-network)


We will be using custom made image that will already have installed some of the tools for inspecting and debugging network like ping or netstat.
```
# docker build -t ubuntu2 -f - . <<EOF
FROM ubuntu:latest
RUN apt update && apt install -y iproute2 net-tools iputils-ping dnsutils netcat telnet
EOF
```

## Day 0: Bridge and veth interfaces
[Back to top](#docker-network)

Just after you install and run your Docker engine, you can find default bridge called bridge0 was created with 172.17.0.1 IP address. And as you can see, no interfaces are attached to the bridge right now.

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
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:a6:e8:a3:10 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:a6ff:fee8:a310/64 scope link
       valid_lft forever preferred_lft forever

# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242a6e8a310       no
```


Let's run simple container in the background and check output again.
```
# docker run --rm -d --name c1 ubuntu2 sleep 100000
```

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
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:a6:e8:a3:10 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:a6ff:fee8:a310/64 scope link
       valid_lft forever preferred_lft forever
7: veth18ddf77@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether a2:51:e2:37:b4:8e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::a051:e2ff:fe37:b48e/64 scope link
       valid_lft forever preferred_lft forever

# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242a6e8a310       no              veth18ddf77
```

Now we have one interface more, called veth18ddf77 in our case. This is one end of veth tunnel. Second end is inside our newly created container. Because each container create its own network namespace we cannot see it here. 

```
##################
# host           #
#     docker0    #
#        |       #
#   veth18ddf77  #
#        |       #
# ############## #
# #    eth0    # #
# #            # #
# # container  # #
# ############## #
#                #
##################      
```


We need to peek inside container. We will see that the interface has IP address from the same subnet as our bridge and also that default route points to our bridge.

```
# docker exec -it c1 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

# docker exec -it c1 ip route
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```


We can also check that we can already ping to the internet.
```
# docker exec -it c1 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=55 time=13.7 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=55 time=13.8 ms
```

Let's now stop and remove our container.
```
# docker rm -f c1
```

## Day 1: Network namespaces
[Back to top](#docker-network)

Before we run any container we have only our main network namespace.

```
# lsns --type=net
        NS TYPE NPROCS PID USER COMMAND
4026531956 net     194   1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
```

We now create one simple container and check namespaces again.
```
# docker run --rm -d --name c1 ubuntu2 sleep 100000
# lsns --type=net
        NS TYPE NPROCS   PID USER COMMAND
4026531956 net     202     1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532443 net       1  1827 root sleep 100000
```

As we can see our container is started in new network namespace. Even if we start container that is not connected to any network it will be still in its own network namespace (with no veth interface tunnel).

```
# docker run --rm -d --name c2 --net none ubuntu2 sleep 200000
# lsns --type=net
        NS TYPE NPROCS   PID USER COMMAND
4026531956 net     198     1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532443 net       1  1827 root sleep 100000
4026532525 net       1  1889 root sleep 200000
```

But when we run container with `--net host` then the container is started in our main network namespace and no new namespace is created. In this case all interfaces that are seen from the host will also be seen from inside container.

```
# docker run --rm -d --name c3 --net host ubuntu2 sleep 300000
# lsns --type=net
        NS TYPE NPROCS   PID USER COMMAND
4026531956 net     167     1 root /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532443 net       1  1827 root sleep 100000
4026532525 net       1  1889 root sleep 200000
```
---

Note: If we check namespaces using `ip netns` tool we won't see it. This is because Docker is not creating the required symlinks (the tool works only with namespace symlinks that are in `/var/run/netns`). We could fix that manually or use `nsenter` tool instead, but we will just stick to the `docker exec` command.

---

Let's now delete existing containers.
```
# docker rm -f $(docker ps -aq)
```

## Day 2: Custom networks
[Back to top](#docker-network)

We can inspect current Docker networks. These are created by default.
```
# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
91b14ba728c5   bridge    bridge    local
c435d16ac0fe   host      host      local
200d34470131   none      null      local
```

We create a new bridge network and see that new bridge interface is indeed created. Each bridge is in a different subnet. By default bridge interface gets random name like `br-c71c083c059d`. We can specify our own name with `-o "com.docker.network.bridge.name"="net1-bridge"` parameter.
```
# docker network create -o "com.docker.network.bridge.name"="net1-bridge" net1

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
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:8f:6a:a0:0c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:8fff:fe6a:a00c/64 scope link
       valid_lft forever preferred_lft forever
30: net1-bridge: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:39:aa:9b:ff brd ff:ff:ff:ff:ff:ff
    inet 172.26.0.1/16 brd 172.26.255.255 scope global net1-bridge
       valid_lft forever preferred_lft forever
```

When we start container we can only specify one network for it to be in, but we can later attach more networks. This will create more veth interface pairs and attach them to the corresponding bridges on the host. But the default route will still point to the first bridge.
```
# docker run --rm -d --name c1 --net net1 ubuntu2 sleep 100000

# docker network connect bridge c1 

# docker exec -it c1 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
31: eth0@if32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:1a:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.26.0.2/16 brd 172.26.255.255 scope global eth0
       valid_lft forever preferred_lft forever
33: eth1@if34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth1
       valid_lft forever preferred_lft forever

# docker exec -it c1 ip route
default via 172.17.0.1 dev eth1
172.17.0.0/16 dev eth1 proto kernel scope link src 172.17.0.2
172.26.0.0/16 dev eth0 proto kernel scope link src 172.26.0.2
```

```
#################################
# host                          #
#     docker0     net1-bridge   #
#        |            |         #
#  veth09b67512   vethb5e70002  #
#        \            /         #
#       ################        #
#       # eth0    eth1 #        #
#       #              #        #
#       # container    #        #
#       ################        #
#                               #
#################################      
```

If we check iptables we can see that for each new bridge network Docker automatically creates NAT. It adds MASQUERADE rule in POSTROUTING chain of NAT table. This means that the traffic to the outside world can return to the container.

```
# iptables -L -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      !net1-bridge  172.26.0.0/16        0.0.0.0/0
    6   373 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  net1-bridge *       0.0.0.0/0            0.0.0.0/0
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
```

And it also inserts rules inside FILTER table that allows all established incoming traffic to and all outgoing traffic from container to the outside world, but not between different docker networks.
```
# iptables -L -nv
Chain INPUT (policy ACCEPT 156 packets, 11184 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      net1-bridge  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      net1-bridge  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  net1-bridge !net1-bridge  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  net1-bridge net1-bridge  0.0.0.0/0            0.0.0.0/0
 6003   36M ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
 5109  274K ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 90 packets, 18112 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  net1-bridge !net1-bridge  0.0.0.0/0            0.0.0.0/0
 5109  274K DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
20834   59M RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  *      net1-bridge  0.0.0.0/0            0.0.0.0/0
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
 9419  506K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination
20839   59M RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

So if we for example create two containers, each in its own network, we by default cannot establish connection between them.

```
# docker run --rm -d --name c1 --net bridge ubuntu2 sleep 100000
# docker run --rm -d --name c2 --net net1 ubuntu2 sleep 100000

# C1_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c1)
# C2_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c2)

# docker exec -it c1 ping $C2_IP
PING 172.26.0.2 (172.26.0.2) 56(84) bytes of data.
^C
--- 172.26.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms

# docker exec -it c2 ping $C1_IP
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
^C
--- 172.17.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms
```

We can fix this behavior with custom iptables rules that we put inside DOCKER-USER chain of FILTER table. If we allow all established traffic, we can than accept for example ICMP traffic from network net1 to network bridge.
```
# iptables -I DOCKER-USER 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# iptables -I DOCKER-USER 2 -p icmp -i net1-bridge -o docker0 -j ACCEPT
```

We can now ping one way only!
```
# docker exec -it c2 ping $C1_IP
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=63 time=0.215 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=63 time=0.071 ms

# docker exec -it c1 ping $C2_IP
PING 172.26.0.2 (172.26.0.2) 56(84) bytes of data.
^C
--- 172.26.0.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1000ms
```

## Day 3: Internal custom networks
[Back to top](#docker-network)

When we create a custom bridge network, we can specify that this network should be internal. This means traffic should neither enter nor leave this network.

```
# docker network create -o "com.docker.network.bridge.name"="internal-bridge" --internal internal
```
Inside iptables we can see that rules that drop all traffic from or to our newly create bridge network. We can override this inside DOCKER-USER chain that is hit before this DOCKER-ISOLATION-STAGE-1 chain.
```
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  *      internal-bridge !172.27.0.0/16        0.0.0.0/0
    0     0 DROP       all  --  internal-bridge *       0.0.0.0/0           !172.27.0.0/16
```

And there is also no MASQUERADE rule created in NAT table for our internal network.

```
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    3   252 MASQUERADE  all  --  *      !net1-bridge  172.26.0.0/16        0.0.0.0/0
    6   373 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0

```

## Day 4: Exposing ports to the outside
[Back to top](#docker-network)

When we run container with parameter `-p 8080:80` we expose port 80 inside container as a port 8080 on our host. We can specify on which interface we would like to expose port by specifying IP of the interface before port. We could limit exposure of the port to our host only with exposing port to loopback interface with `-p 127.0.0.1:8080:80`.

```
# docker run --rm -d --name c2 --net net1 -p 8080:80 ubuntu2 sleep 100000
# docker ps
CONTAINER ID   IMAGE     COMMAND          CREATED         STATUS        PORTS                                   NAMES
12f067cb2156   ubuntu2   "sleep 100000"   2 seconds ago   Up 1 second   0.0.0.0:8080->80/tcp, :::8080->80/tcp   c2
```
This will add one rule to the NAT table to create port forwarding from host's 8080 to the port 80 inside container (last line at DOCKER chain).
```
# iptables -L -nv -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    52 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    3   252 MASQUERADE  all  --  *      !net1-bridge  172.26.0.0/16        0.0.0.0/0
    6   373 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
    0     0 MASQUERADE  tcp  --  *      *       172.26.0.2           172.26.0.2           tcp dpt:80

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  net1-bridge *       0.0.0.0/0            0.0.0.0/0
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
    0     0 DNAT       tcp  --  !net1-bridge *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.26.0.2:80
```
And there is also rule inside FILTER table that allows forward traffic to net1-bridge to port 80 (because we hit PREROUTING NAT table before, so destination port 8080 is already changed to 80, and destination ip is from our "public" ip changed to ip of container).

```
# iptables -L -nv
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   168 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    2   168 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  internal-bridge internal-bridge  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      net1-bridge  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      net1-bridge  0.0.0.0/0            0.0.0.0/0
    2   168 ACCEPT     all  --  net1-bridge !net1-bridge  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  net1-bridge net1-bridge  0.0.0.0/0            0.0.0.0/0
 6003   36M ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
 5109  274K ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     tcp  --  !net1-bridge net1-bridge  0.0.0.0/0            172.26.0.2           tcp dpt:80
```

---

NOTE: If we try to expose port while container is in "internal" network, nothing will happen!

---

If you expose port with `-p 8080:8080` then the port is on host tied to all interfaces including loopback, ens192, bridge0 and net1-bridge. This means that the container from different Docker network can access the exposed service via bridge IP address. We can limit this by `-p 127.0.0.1:8080:8080` since containers cannot access loopback interface of the host. If we limit to ens192 interface, we did nothing as the Docker networks are routable to ens192 network. So the container can access the service via IP of ens192 interface. For more you need to tune DOCKER-USER chain in FILTER table of iptables.



## Day 5: DNS resolution in default network
[Back to top](#docker-network)

When we run a container without `--net` flag or with `--net bridge` flag, the container will be put in the default bridge network with bridge name docker0.

If your host have some DNS server specified, that this will also get inherited into the container.
```
# docker exec -it c1 cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 1.1.1.1
```

If your host does not have specified DNS server the container will use Google DNS servers.
```
# docker exec -it c1 cat /etc/resolv.conf
# Generated by NetworkManager

nameserver 8.8.8.8
nameserver 8.8.4.4
```

You can override this with `--dns` flag and specify custom DNS server. Independently you can specify some custom entries to `/etc/hosts` with `--add-host` flag.
```
# docker run -d --name c2 --dns 192.168.10.10 --add-host host2:10.10.10.10 ubuntu2 sleep 1000000

# docker exec -it c2 cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
10.10.10.10     host2
172.17.0.3      b22bb258dfe5

# docker exec -it c2 cat /etc/resolv.conf
nameserver 192.168.10.10
```

## Day 6: DNS Resolution in custom network
[Back to top](#docker-network)

Let's run a container in a user-defined network and inspect the `/etc/resolv.conf`.
```
# docker run --rm -it --net net1 ubuntu2 cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
```

Instead of host's DNS settings we see that DNS server is set to be `127.0.0.11`. 
The docker daemon implements an embedded DNS server. It provides service discovery by container name or net-alias property. So we can resolve IP address of containers using its name.

```
# docker run -d -it --net net1 --name c2 ubuntu2 sleep 10000
# docker run -d -it --net net1 --name c1 ubuntu2 sleep 10000

# docker exec -it c1 ping c2
PING c2 (172.26.0.2) 56(84) bytes of data.
64 bytes from c2.net1 (172.26.0.2): icmp_seq=1 ttl=64 time=0.055 ms
64 bytes from c2.net1 (172.26.0.2): icmp_seq=2 ttl=64 time=0.059 ms
```

If address cannot be resolved inside docker network the embedded DNS server will use host's DNS server to resolve the address. And if host's DNS server is not set, Docker will use Google DNS servers. You can still override this with `--dns` flag and set you custom DNS server, but the change would not be visible inside container.

## Day 7: Demo usage of Docker networks (Simple)
[Back to top](#docker-network)

We can now combine all knowledge we gathered and setup the scenario with simple web application and database, that is hidden in internal bridge network.

```
#####################################################
# host                                              #
#     outside-bridge    inside-bridge (internal)    #
#          |            /       \                   #
#     veth09b67512 vetheebbab0b  veth511faf7a       #
#          |          /           \                 #
#     ###################       ################### #
#     #  eth0      eth1 #       #  eth0           # #
#     #                 #       #                 # #
#     # application     #       # database        # #
#     ###################       ################### #
#####################################################
```

Let's create bridges and attach containers to them. Container "application" exposes port 8080 to the outside, via this port the users would access our application. Communication between application and database container goes via inside-bridge network and therefore we don't need to expose any ports!
```
# docker network create -o "com.docker.network.bridge.name"="outside-bridge" outside-bridge
# docker network create --internal -o "com.docker.network.bridge.name"="inside-bridge" inside-bridge

# docker run -d --name application --net outside-bridge -p 8080:8080 ubuntu2 sleep 10000
# docker network connect inside-bridge application

# docker run -d --name database --net inside-bridge ubuntu2 sleep 10000
```

We now check that application can access database by name of the container.
```
# docker exec -it application ping database
PING database (172.29.0.3) 56(84) bytes of data.
64 bytes from database.inside-bridge (172.29.0.3): icmp_seq=1 ttl=64 time=0.099 ms
64 bytes from database.inside-bridge (172.29.0.3): icmp_seq=2 ttl=64 time=0.072 ms
```

## Day 8: Demo usage of Docker networks (Advanced)
Let's keep previous design of application and database. If we want to create more restricted access between them, we must separate them into different networks and then configure iptables to allow just specific traffic between them. To avoid routing troubles we put each container in only one network.

Because of different networks, DNS resolving won't work, we will use static IPs and static entries in hosts file.

```
# docker network create \
  -o "com.docker.network.bridge.name"="outside-bridge" \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  outside-bridge

# docker network create \
  --internal \
  -o "com.docker.network.bridge.name"="internal-bridge" \
  --subnet=192.168.20.0/24 \
  --gateway=192.168.20.1 \
  internal-bridge

# docker run \
  -d \
  --name application \
  --net outside-bridge \
  --ip 192.168.10.2 \
  --add-host database:192.168.20.2 \
  -p 8080:8080 \
  ubuntu2 \
  sleep 10000

# docker run \
  -d \
  --name database \
  --net internal-bridge \
  --ip 192.168.20.2 \
  ubuntu2 \
  sleep 10000
```

So now we have the following situation. Application exposes port 8080 via outside-bridge and also has internet access via this network. It will also access database using this network.
```
#################################################
# host                                          #
# outside-bridge <- routing -> internal-bridge  #
#      |                            |           #
# veth09b67512                 veth511faf7a     #
#      |                            |           #
# ###################       ################### #
# #  eth0           #       #  eth0           # #
# #                 #       #                 # #
# # application     #       # database        # #
# ###################       ################### #
#################################################
```
We now need to configure iptables to allow specific traffic between outside-bridge and internal-bridge. So the application will be able to access database on port 5432/tcp.


```
# iptables -I DOCKER-USER 1 -i internal-bridge -o outside-bridge -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# iptables -I DOCKER-USER 2 -i outside-bridge -o internal-bridge -p tcp --dport 5432 -s 192.168.10.2 -d 192.168.20.2 -j ACCEPT
```

If we now start netcat listener acting as a database service, we can connect to it from application container using database container name, because of `--add-host` flag we used when we started application container.

```
# docker exec -it database nc -l -v -n -p 5432
Listening on 0.0.0.0 5432
Connection received on 192.168.20.1 41734
```

```
docker exec -it application telnet database 5432
Trying 192.168.20.2...
Connected to database.
Escape character is '^]'.
```

## Day 9: Conclusion

Docker simplifies dealing with container networks, but we still need to understand how everything works under the hood. Unless we can make invalid assumptions and compromise the whole application.

When creating custom bridge networks do set `com.docker.network.bridge.name` parameter, it will make debugging much easier!