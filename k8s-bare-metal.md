# K8s Bare Metal
We are all familiar with K8s services that are provided by big cloud platforms. We have managed K8s services on AWS, Google Clouds, Azure and so on. But sometimes we want to have our own bare metal cluster, running inside our own datacenter or even on our own computer.

When we plan on going bare metal we must know that some K8s features will not work out of the box:
- Persistent volumes
- Load Balancer service
- Ingress

For example when you create Load balancer service on AWS cluster, the external Load Balancer on AWS will be created with a public IP address. On bare metal cluster you will just get stuck with `pending` state. So we need to take extra steps to solve these three issues.

Our K8S cluster will contain 3 VMs with CentOS 7:
- demo-k8s-1 (10.156.30.70)
- demo-k8s-2 (10.156.30.71)
- demo-k8s-3 (10.156.30.72)

The first VM will be our master node that will run control plane. The second and the third will be our worker nodes.

Content:
- [Day 1: Prepare VMs](#day-1-prepare-vms)
- [Day 2: Containerd and K8S](#day-2-containerd-and-k8s)
- [Day 3: Create cluster](#day-3-create-cluster)
- [Day 4: Flannel for pod networks](#day-4-flannel-for-pod-networks)
- [Day 5: MetalLB for Load Balancing](#day-5-metallb-for-load-balancing)
- [Day 6: Deploy demo app](#day-6-deploy-demo-app)
- [Day 7: Ingress](#day-7-ingress)
- [Day 8: LB in front of Ingress](#day-8-lb-in-front-of-ingress)

## Day 1: Prepare VMs
[Back to top](#k8s-bare-metal)

Do this steps on all three VMs.

### Networking and hostnames 
Configure basic networking, assign static IPs to all three hosts and set appropriate hostnames. Then edit `/etc/hosts` and add the following lines
```
10.156.30.70 demo-k8s-1
10.156.30.71 demo-k8s-2
10.156.30.72 demo-k8s-3
```

We should now be able to ping from one host to another using hostnames.

### Disabling swap
Since K8S does not like swap, we need to disable it. We do that with the following command.
```
# swapoff -a
```
To make this permanent we need to find our swap partition inside file `/etc/fstab` and comment it out using # sign. 

### Disabling SELinux
To allow containers to access hosts' file systems we need to disable SELinux.
```
# setenforce 0
```
To make this permanent modify `/etc/sysconfig/selinux`. Set SELINUX setting from enforcing to permissive.

### Disabling firewalld
We will be using iptables directly, so just stop and disable firewalld service
```
# systemctl stop firewalld
# systemctl disable firewalld
```

### Configure extra modules

We need to enable two extra modules br_netfilter and overlay. 
We first load them and made them permanent.
```
# modprobe br_netfilter
# modprobe overlay
# echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf
# echo "overlay" > /etc/modules-load.d/overlay.conf
```
Then create file `/etc/sysctl.d/kubernetes.conf` and add the following lines. This will force bridged traffic to go via host's iptables and enable routing between bridges.
```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```
Apply changes without reboot.
```
sysctl --system
```
## Day 2: Containerd and K8S
[Back to top](#k8s-bare-metal)

Do this steps on all three VMs.

We need container runtime on each node. K8S will use it to spawn containers. We will be using containerd runtime.

We first setup the repository for docker packages
```
# yum install -y yum-utils
# yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
We now install containerd package
```
# yum install containerd.io
```

Let us create default containerd config file.
```
# containerd config default > /etc/containerd/config.toml
```
Now edit this config and change `SystemdCgroup` to `true`.
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

We can now start and enable containerd service.
```
# systemctl enable containerd
# systemctl start containerd
```

We add official K8s repos and then install all the tools. Create file `/etc/yum.repos.d/kubernetes.repo` and add following content:
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

---

NOTE: GPG check is turned off! I have some problems with validation.

---

We can now install tools we need with the following command.
```
# yum install -y kubeadm kubelet kubectl
```
What are these tools?
- kubeadm is a tool to administer cluster (create, join, reset...)
- kubelet is a tool that do all the actual work behind the scenes
- kubectl is a tool for operating cluster (deploy pods, create services...)

We first need to start kubelet.
```
# systemctl start kubelet
# systemctl enable kubelet
```

## Day 3: Create cluster
[Back to top](#k8s-bare-metal)

On our master node we now create new cluster. We will add `--pod-network-cidr` parameter that we will later need when installing Flannel for handling pod network.
```
# kubeadm init --pod-network-cidr=10.244.0.0/16
```

Our newly created kubeconfig is located at `/etc/kubernetes/admin.conf`. We must copy it to the home folder of our user, as kubectl expects the config to be there. We can alternatively move kubeconfig file to another host and access our cluster from different host.

```
# mkdir -p $HOME/.kube
# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
```

At the end of our setup output we should get printed out `kubeadm join` command with some random tokens. We need to copy and run this command on every other host to join them to the cluster.

At the end we should be able to access our cluster with kubectl and see all our hosts.
```
# kubectl get nodes
NAME         STATUS     ROLES           AGE    VERSION
demo-k8s-1   NotReady   control-plane   34m    v1.24.1
demo-k8s-2   NotReady   <none>          22m    v1.24.1
demo-k8s-3   NotReady   <none>          107s   v1.24.1
```

## Day 4: Flannel for pod networks
[Back to top](#k8s-bare-metal)

Just like K8s need containerd to handle creating containers, it also needs Flannel or somebody else like Calico to handle networking.

We first download flannel kubernetes manifest
```
wget https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

Now edit the file and change version from 1.18 to 1.17. I had issues with 1.18 version. You also need to make sure that "pod network cidr" that you used at `kubeadm init` matches the one that is specified inside manifest.

Apply manifest to your cluster
```
# kubectl apply -f kube-flannel.yml
```
And after some time you should see that all nodes are ready and all pods are running.
```
# kubectl get nodes
NAME         STATUS   ROLES           AGE     VERSION
demo-k8s-1   Ready    control-plane   3m19s   v1.24.1
demo-k8s-2   Ready    <none>          2m47s   v1.24.1
demo-k8s-3   Ready    <none>          2m29s   v1.24.1

# kubectl get pods -A
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   coredns-6d4b75cb6d-8mp7v               1/1     Running   0          3m5s
kube-system   coredns-6d4b75cb6d-vq5hv               1/1     Running   0          3m5s
kube-system   etcd-demo-k8s-1                        1/1     Running   1          3m20s
kube-system   kube-apiserver-demo-k8s-1              1/1     Running   1          3m19s
kube-system   kube-controller-manager-demo-k8s-1     1/1     Running   0          3m19s
kube-system   kube-flannel-ds-652kp                  1/1     Running   0          90s
kube-system   kube-flannel-ds-gmf44                  1/1     Running   0          90s
kube-system   kube-flannel-ds-rkbc4                  1/1     Running   0          90s
kube-system   kube-proxy-5498z                       1/1     Running   0          3m5s
kube-system   kube-proxy-h5jq5                       1/1     Running   0          2m50s
kube-system   kube-proxy-rrxf5                       1/1     Running   0          2m32s
kube-system   kube-scheduler-demo-k8s-1              1/1     Running   1          3m19s
```

# Day 5: MetalLB for Load Balancing
[Back to top](#k8s-bare-metal)

If we now go and deploy some Load Balancer service we will see it will hang in `pending` state. We need to deploy Load Balancer first.

We can deploy MetalLB directly from manifests on the Internet.
```
# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
```

The only thing we need to create manually is a config map with IP range that will we available for LB to assign to services. Create file metallb.yml with content:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.156.30.76-10.156.30.79
```
And apply it with 
```
# kubectl apply -f metallb.yml
```

## Day 6: Deploy demo app
[Back to top](#k8s-bare-metal)

We will deploy Tea Store application from https://github.com/DescartesResearch/TeaStore. Let us download manifest.
```
# wget https://raw.githubusercontent.com/DescartesResearch/TeaStore/master/examples/kubernetes/teastore-clusterip.yaml
```

We can now apply it with kubectl.
```
# kubectl apply -f teastore-clusterip.yaml
```
After some time the pods should be all running.
```
# kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
teastore-auth-7f7c655fdf-7gb5l          1/1     Running   0          56s
teastore-db-777bfff4cc-lkp9g            1/1     Running   0          57s
teastore-image-5b9df7f8d7-4vjfg         1/1     Running   0          56s
teastore-persistence-74ddb6fd8-4g594    1/1     Running   0          56s
teastore-recommender-66db979fc9-rxkgl   1/1     Running   0          56s
teastore-registry-55d668c99d-vhcv2      1/1     Running   0          56s
teastore-webui-6b66f7ffb7-4wfnr         1/1     Running   0          56s
```
If we inspect services we see one service of type NodePort. This means we can access the service from the outside world using IP address of one of the nodes and this mapped service port.
```
# kubectl get svc
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          40m
teastore-auth          ClusterIP   10.111.202.188   <none>        8080/TCP         77s
teastore-db            ClusterIP   10.109.239.39    <none>        3306/TCP         77s
teastore-image         ClusterIP   10.97.220.170    <none>        8080/TCP         77s
teastore-persistence   ClusterIP   10.98.65.132     <none>        8080/TCP         77s
teastore-recommender   ClusterIP   10.111.114.10    <none>        8080/TCP         77s
teastore-registry      ClusterIP   10.110.30.81     <none>        8080/TCP         77s
teastore-webui         NodePort    10.97.171.173    <none>        8080:30080/TCP   77s
```

So if we open one of the following links, we should see our homepage:
- http://10.156.30.70:30080/
- http://10.156.30.71:30080/
- http://10.156.30.72:30080/

We can modify this teastore-webui service to use Load Balancer instead. We open manifest file and change service type from NodePort to LoadBalancer. We can also delete nodePort property.
```
---
apiVersion: v1
kind: Service
metadata:
  name: teastore-webui
  labels:
    app: teastore
    run: teastore-webui
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      protocol: TCP
  selector:
    run: teastore-webui
```

If we now reapply manifest and inspect the services again, we see that teastore-webui service has now external IP from our LB range.
```
# kubectl get svc
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)          AGE
kubernetes             ClusterIP      10.96.0.1        <none>         443/TCP          45m
teastore-auth          ClusterIP      10.111.202.188   <none>         8080/TCP         6m24s
teastore-db            ClusterIP      10.109.239.39    <none>         3306/TCP         6m24s
teastore-image         ClusterIP      10.97.220.170    <none>         8080/TCP         6m24s
teastore-persistence   ClusterIP      10.98.65.132     <none>         8080/TCP         6m24s
teastore-recommender   ClusterIP      10.111.114.10    <none>         8080/TCP         6m24s
teastore-registry      ClusterIP      10.110.30.81     <none>         8080/TCP         6m24s
teastore-webui         LoadBalancer   10.97.171.173    10.156.30.76   8080:30080/TCP   6m24s
```
So now we can access our app using this external IP and service port 8080:
- http://10.156.30.76:8080

The downside of this approach is that we burn one IP for each external service.

## Day 7: Ingress
[Back to top](#k8s-bare-metal)

If we used Load Balancer service do to L2 load balancing, we use Ingress to do L7 load balancing. It can be realized on many different ways. With Nginx or HAProxy for example. We will go with Nginx.

We first need to deploy ingress controller.
```
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/baremetal/deploy.yaml
```

Now we can create some ingress rules. We create new file ingress.yml. We specify that everything that comes in and is meant for teastore.lan domain gets redirected to teastore-webui service to port 8080.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-teastore
spec:
  rules:
  - host: teastore.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: teastore-webui
            port:
              number: 8080
  ingressClassName: nginx
```

Let's apply this manifest and see that our ingress is indeed created.
```
# kubectl get ingress
NAME               CLASS   HOSTS          ADDRESS   PORTS   AGE
ingress-teastore   nginx   teastore.lan             80      5s
```

Ingress controller created NodePort service so we can access it using one of the node IP address and mapped port.
```
# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.98.249.10     <none>        80:30047/TCP,443:30562/TCP   2m57s
ingress-nginx-controller-admission   ClusterIP   10.108.124.251   <none>        443/TCP                      2m57s
```

If we open one of the following links:
- http://10.156.30.70:30047/
- http://10.156.30.71:30047/
- http://10.156.30.72:30047/

We should get 404 Not Found page as we need to hit ingress controller with Host header set to teastore.lan. We can do this with configuring our DNS server or manually adding line to our hosts file that points teastore.lan to 10.156.30.71 for example.

We could also fix this by setting default backend service in our ingress. So if there is no match for domain we hit this default service.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-teastore
spec:
  defaultBackend:
    service:
      name: teastore-webui
      port:
        number: 8080
  rules:
  - host: teastore.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: teastore-webui
            port:
              number: 8080
  ingressClassName: nginx
```

If we now try again we should see our teastore app again even if we use IP address instead of domain name. The problem we still have is that we need to know IP of one of our cluster nodes and also to use this weird high number port. We will fix this by placing Load Balancer in front of Ingress controller service. 

## Day 8: LB in front of Ingress
[Back to top](#k8s-bare-metal)

We basically have two options:
- Use external LB outside of cluster
- Use internal MetalLB

If we go with the first option we just created single point of failure and we need to deal with resistance and high available on our own. But why do that if we already have K8s to help us with that. So we will use MetalLB that is running inside our cluster to do load balancing in front of our ingress controller.

We just need to edit ingress-nginx-controller service and change service type from NodePort to LoadBalancer. We can also specify the IP we want to get from MetalLB using loadBalancerIP property.

```
# kubectl edit svc -n ingress-nginx ingress-nginx-controller
```
```
...
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.98.249.10
  clusterIPs:
  - 10.98.249.10
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  loadBalancerIP: 10.156.30.79
  ports:
  - appProtocol: http
    name: http
    nodePort: 30047
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    nodePort: 30562
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  sessionAffinity: None
  type: LoadBalancer
```

If we now inspect our services we see that our nginx ingress is now listening on IP we specified.
```
# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.98.249.10     10.156.30.79   80:30047/TCP,443:30562/TCP   151m
ingress-nginx-controller-admission   ClusterIP      10.108.124.251   <none>         443/TCP                      151m
```

So now we set DNS server (or our local hosts file) to point domain teastore.lan to 10.156.30.79 and we are good to go!