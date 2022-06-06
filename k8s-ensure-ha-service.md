# K8s Ensure HA Service
We use K8s to ensure high availability (HA) of our services. But we need to be careful.

Content:
- [Day 0: Deployment](#day-0-deployment)
- [Day 1: Problem](#day-1-problem)
- [Day 2: Solution](#day-2-solution)
- [Day 3: Conclusion](#day-3-conclusion)


## Day 0: Deployment
[Back to top](#k8s-ensure-ha-service)

Let's assume we have three node cluster. First node is used for running control plane, and second two nodes for running our pods.
```
$ kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
demo-k8s-1   Ready    control-plane   3d19h   v1.24.1
demo-k8s-2   Ready    <none>          3d19h   v1.24.1
demo-k8s-3   Ready    <none>          3d19h   v1.24.1
```

We now deploy simple app that serves some static page. Let's create app.yaml manifest file with the following content.
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
              name: http-server
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: nginx
```
We now apply it with kubectl.
```
$ kubectl apply -f app.yaml
```
We get two pods and one service exposed to the outside.
```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
nginx-deployment-c6cf569f-b8mjw   1/1     Running   0          9m11s
nginx-deployment-c6cf569f-h8l4w   1/1     Running   0          73s
```

We can now open http://10.156.30.76 and see nginx default home page.


## Day 1: Problem
[Back to top](#k8s-ensure-ha-service)

Let's make a test. We will turn off network adapter on node demo-k8s-2.

If we try to open http://10.156.30.76 we get "This site can't be reached" error. But why, don't we have two pods and two nodes?!?

If we inspect pods more closely we see that both pods are living on demo-k8s-2 node.
```
$ kubectl get pods -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-c6cf569f-b8mjw   1/1     Running   0          13m     10.244.1.36   demo-k8s-2     <none>           <none>
nginx-deployment-c6cf569f-h8l4w   1/1     Running   0          5m38s   10.244.1.37   demo-k8s-2     <none>           <none>
```

And because we disconnected node demo-k8s-2 we cannot access any working pods on node demo-k8s-2. 

K8s detected that our node is down.
```
$ kubectl get node
NAME           STATUS     ROLES           AGE   VERSION
demo-k8s-1     Ready      control-plane   4d    v1.24.1
demo-k8s-2     NotReady   <none>          4d    v1.24.1
demo-k8s-3     Ready      <none>          4d    v1.24.1
```
But it will wait for 5 minutes before it will reschedule pods to another node. So we will have 5 minutes outbreak.
```
$ kubectl describe nodes demo-k8s-2
...
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
...
```

After that K8s creates two new pods on demo-k8s-3 node and our page is back online.
```
$ kubectl get pods -o wide
NAME                              READY   STATUS        RESTARTS   AGE    IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-c6cf569f-6kz2d   1/1     Running       0          108s   10.244.2.34   demo-k8s-3     <none>           <none>
nginx-deployment-c6cf569f-9t6hf   1/1     Running       0          108s   10.244.2.32   demo-k8s-3     <none>           <none>
nginx-deployment-c6cf569f-b8mjw   1/1     Terminating   0          19m    10.244.1.36   demo-k8s-2     <none>           <none>
nginx-deployment-c6cf569f-h8l4w   1/1     Terminating   0          11m    10.244.1.37   demo-k8s-2     <none>           <none>
```
And if we now connect node demo-k8s-2 back, the two pods on demo-k8s-2 will get terminated.
```
# kubectl get pods -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-c6cf569f-6kz2d   1/1     Running   0          8m44s   10.244.2.34   demo-k8s-3     <none>           <none>
nginx-deployment-c6cf569f-9t6hf   1/1     Running   0          8m44s   10.244.2.32   demo-k8s-3     <none>           <none>
```
Because we do not want this 5 minute interruption we must enforce that pods get deployed over more nodes. So we will not finished in situation where all 10 pods deployed on one single node.

## Day 2: Solution
[Back to top](#k8s-ensure-ha-service)

We need to add `topologySpreadConstraints` key to our K8s manifest and specify that we want linear distribution of pods on our nodes. So the manifest should look something like that. 
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: nginx
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
              name: http-server
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: nginx
```

The pods are now distributed over both nodes. And if one node gets down, the app will keep working since we still have one working pod on a different node.
```
$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-7f8657ff74-nqh9t   1/1     Running   0          15s   10.244.1.39   demo-k8s-2     <none>           <none>
nginx-deployment-7f8657ff74-w25h4   1/1     Running   0          15s   10.244.2.35   demo-k8s-3     <none>           <none>
```

And after 5 minutes, the K8s will try to move pod from demo-k8s-2. But not to the demo-k8s-3 node as this would violate linear distribution of pods and since we do not have any working worker node without nginx pod, the new pod will be stuck in pending state.

```
$ kubectl get pods -o wide
NAME                                READY   STATUS        RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-7f8657ff74-8gnpp   0/1     Pending       0          8s      <none>        <none>         <none>           <none>
nginx-deployment-7f8657ff74-nqh9t   1/1     Terminating   0          6m48s   10.244.1.39   demo-k8s-2     <none>           <none>
nginx-deployment-7f8657ff74-w25h4   1/1     Running       0          6m48s   10.244.2.35   demo-k8s-3     <none>           <none>
```

When demo-k8s-2 node gets back online, the old pod will be delete and new pod will be created on this same node. And service will be back in HA mode again.

```
$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP            NODE           NOMINATED NODE   READINESS GATES
nginx-deployment-7f8657ff74-8gnpp   1/1     Running   0          4m7s   10.244.1.40   demo-k8s-2     <none>           <none>
nginx-deployment-7f8657ff74-w25h4   1/1     Running   0          10m    10.244.2.35   demo-k8s-3     <none>           <none>

```

## Day 3: Conclusion
[Back to top](#k8s-ensure-ha-service)

When deploying some pods the K8s first filters nodes to see which ones are feasible to run the pod. In the next step it ranks feasible nodes and then deploys pod to the node with the highest score. Without of an appropriate `topologySpreadConstraints` setting it can happen that pods are deployed to a single node.