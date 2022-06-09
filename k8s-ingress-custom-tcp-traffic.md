# K8s Ingress Custom TCP traffic

We usually expose K8s applications via Ingress. This way we place multiple customer facing endpoints behind one public IP. On AWS EKS we would have one Elastic Load Balancer (ELB) that would serve as a gateway to enter our cluster. We would then point multiple DNS domain names to point to the ELB public IP and configure Ingress to point different DNS domain names to different internal services.

But when talking about non-HTTP(S) traffic we hit a blocker, because Ingress Controller does not know how to deal it. So we need to take care of non-HTTP(S) traffic before it comes to the Ingress Controller App like Nginx. But we still want to use single entry point into the cluster.

Content:
- [Day 0: Deployment](#day-0-prepare-cluster)
- [Day 1: Deploy Application and Ingress](#day-1-deploy-application-and-ingress)
- [Day 2: Results](#day-2-results)



## Day 0: Prepare cluster
[Back to top](#k8s-ingress-custom-tcp-traffic)

We will first install Ingress Controller to our cluster. When installing we will specify that we want to redirect extra port 9000 to the inside service `default/example-svc` to port 9000.

Let's create `values.yaml` file that will use in helm install
```
tcp:
  9000: "default/example-svc:9000"
```

We know install nginx ingress on our cluster.
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f ingress-values.yaml
```

This will create one LoadBalancer Service that will expose ports 80, 443 and 9000:
- Nginx is listening on ports 80 and 443
- Traffic on port 9000 is forwarded to service default/example-svc to port 9000

## Day 1: Deploy Application and Ingress
[Back to top](#k8s-ingress-custom-tcp-traffic)

Install our application on the cluster. Create manifest file `app.yaml`.
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
  replicas: 1
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
            - containerPort: 9000
              name: custom-tcp
---
apiVersion: v1
kind: Service
metadata:
  name: example-svc
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
    - name: custom-tcp
      port: 9000
      protocol: TCP
      targetPort: 9000
  selector:
    app: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
spec:
  defaultBackend:
    service:
      name: example-svc
      port:
        number: 80
  ingressClassName: nginx
```

Apply the manifest to the cluster.
```
$ kubectl apply -f app.yaml
```

## Day 2: Results
[Back to top](#k8s-ingress-custom-tcp-traffic)

So right now should be able to access default nginx index page via ingress. And we can also start netcat listener inside nginx container on port 9000 and we should be able to connect to it using telnet using public IP or any DNS domain name that points to this public IP and port 9000.

So if we assume that Ingress Load Balancer External IP is a.b.c.d. 
We can then point different DNS domain to this IP:
```
a.b.c.d domain.lan
a.b.c.d demo.lan
...
```

So we can normal access HTTP(S) services via Ingress
```
http://a.b.c.d --> default backend
http://domain.lab --> domain.lab rule or default backend
http://demo.lan --> demo.lan rule or default backend
...
```

And we can access the service on 9000 port using different ways:
```
telnet a.b.c.d 9000 --> default/example-svc:9000
telnet domain.lan 9000 --> default/example-svc:9000
telnet demo.lan 9000 --> default/example-svc:9000
...
```

Just be aware that with this custom TCP traffic we hit SAME backend service every time regardless which DNS name we use.