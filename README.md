# DevOps Technical Test - Part 2

Introduction
This repository contains the solution for the Part 2 of the DevOps Technical Test. The goal of this test is to deploy a simple API consisting of three stateless Spring Boot microservices into a cloud environment. The microservices communicate with each other via HTTP, and the api-gateway microservice should be publicly accessible with SSL enabled.


## Solution Overview

### Technology Stack

- Language/Framework: GKE and Terraform
- Cloud Provider: Google Kubernetes Engine (GKE)
- Infrastructure as Code (IaC): Terraform
- SSL Certificate: Let's Encrypt via cert-manager

### Deployment Steps

Follow the steps below to set up and test the solution:

### 1. Deploy GKE Cluster with Terraform

- Clone the repository:

  ```bash
  git clone <repository_url>
  cd <repository_directory>
  `````

- Apply the terraform code, taking into account that you have the project data correctly set in the terraform.tfvars file:

````bash
# Copyright (c) HashiCorp, Inc.
# SPDX-License-Identifier: MPL-2.0

project_id = "XXXXXXXX"
region     = "us-central1"
````


- Then apply the Terraform code:

````bash
terraform apply

and we should have a output like this:

Plan: 4 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + kubernetes_cluster_host = (known after apply)
  + kubernetes_cluster_name = "grand-fx-xxxx-gke"
  + project_id              = "grand-fx-xxx"
  + region                  = "us-central1"

````

### 2. Check the cluster

- We will check that the cluster is properly deploy

```bash
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-grand-fx-308506--grand-fx-308506--a8e385f0-6xb4   Ready    <none>   25m   v1.27.3-gke.100
gke-grand-fx-308506--grand-fx-308506--a8e385f0-jknc   Ready    <none>   25m   v1.27.3-gke.100
```

### 3. Deploy the microservices

- We will deploy the microservices that make up the API. For it we have the yaml files that define the application, the service to expose and the public ingress to be able to connect.
    *** You can check the different files in the app-deployment folder.

```bash
kubectl get apply -f *.yaml # we have the apply for each file `kubectl get apply -f customer-service.yaml...etc`
```

- We will explian how we have deployed each service:

    custormer service: We deployed this service like a deployment to have control over the replicas. The we added the Service like ClusterIP to expose the service in the k8s cluster.
    order service: Same that the customer service, the difference was that it need the conection throught customer service:
        ```value: http://customer-service``
    api-gateway: We created a deployment to delivery the app, Then we created a Service like Loadbalancer to expose the api outside kubernetes.
        ** The step we can make of differents forms (using the api of the cloud and call to create a loadbalancer with de application...etc.)


- Check services:
```bash
NAME                                               READY   STATUS    RESTARTS   AGE
pod/api-gateway-deployment-7b8b577cf6-df4zg        1/1     Running   0          6m
pod/api-gateway-deployment-7b8b577cf6-dk8ht        1/1     Running   0          6m
pod/api-gateway-deployment-7b8b577cf6-g6xgn        1/1     Running   0          12m
pod/api-gateway-deployment-7b8b577cf6-nm4jn        1/1     Running   0          6m
pod/customer-service-deployment-56c58db7d9-clrhg   1/1     Running   0          6m53s
pod/customer-service-deployment-56c58db7d9-v8fdd   1/1     Running   0          22m
pod/name-resolution-test                           1/1     Running   0          17m
pod/order-service-deployment-5465b4d967-25q8p      1/1     Running   0          6m26s
pod/order-service-deployment-5465b4d967-8rsz4      1/1     Running   0          6m26s
pod/order-service-deployment-5465b4d967-b47l8      1/1     Running   0          18m

NAME                          TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
service/api-gateway-service   LoadBalancer   10.111.245.67    34.28.46.240   80:30647/TCP   11m
service/customer-service      ClusterIP      10.111.254.47    <none>         80/TCP         13m
service/order-service         ClusterIP      10.111.250.190   <none>         80/TCP         13m

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api-gateway-deployment        4/4     4            4           12m
deployment.apps/customer-service-deployment   2/2     2            2           22m
deployment.apps/order-service-deployment      3/3     3            3           18m

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/api-gateway-deployment-7b8b577cf6        4         4         4       12m
replicaset.apps/customer-service-deployment-56c58db7d9   2         2         2       22m
replicaset.apps/order-service-deployment-5465b4d967      3         3         3       18m
```

- Add ssl certificate to the application:

For this we can install certmanager inside the cluster and add a Secret with the certificate generation.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

- Create certificate.

```bash
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: xxxxxxxx #  Replace this with your email address
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
        ingress:
          name: api-ingress
```

- Create Secret

```bash
apiVersion: v1
kind: Secret
metadata:
  name: web-ssl
type: kubernetes.io/tls
stringData:
  tls.key: ""
  tls.crt: ""
```

- Deploy the ingress call to the seret:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    # This tells Google Cloud to create an External Load Balancer to realize this Ingress
    kubernetes.io/ingress.class: gce
    # This enables HTTP connections from Internet clients
    kubernetes.io/ingress.allow-http: "true"
 #   # This tells Google Cloud to associate the External Load Balancer with the static IP which we created earlier
    kubernetes.io/ingress.global-static-ip-name: web-ip
    cert-manager.io/issuer: letsencrypt-staging
spec:
  tls:
    - secretName: web-ssl
      hosts:
        - DOMAIN.com
  defaultBackend:
    service:
      name: api-gateway-service
      port:
        number: 80
```


### 4. Check the application

- Finally we will check that the application works. To do this we will launch a curl on the domain name calling the path /order:

```bash
curl --insecure https://dispositivoandroid.com/order/       

## the result was:

{"id":1,"customerId":1,"amount":80.0}%
```

- Check the logs of the pods:



```bash
kubectl logs -l app=order-service


2023-10-29 10:51:29.443  INFO 1 --- [           main] c.n.o.OrderServiceApplication            : Starting OrderServiceApplication v0.0.1-SNAPSHOT using Java 11.0.10 on order-service-deployment-5465b4d967-b47l8 with PID 1 (/workspace/BOOT-INF/classes started by cnb in /workspace)
2023-10-29 10:51:29.455  INFO 1 --- [           main] c.n.o.OrderServiceApplication            : No active profile set, falling back to default profiles: default
2023-10-29 10:51:34.008  INFO 1 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port 8080
2023-10-29 10:51:34.036  INFO 1 --- [           main] c.n.o.OrderServiceApplication            : Started OrderServiceApplication in 5.657 seconds (JVM running for 6.767)
2023-10-29 11:02:09.160  INFO 1 --- [or-http-epoll-4] c.n.orderservice.api.OrderController     : Processing order: OrderDto{id=1, customerId=1, amount=100.0}
2023-10-29 11:02:09.736  INFO 1 --- [or-http-epoll-4] c.n.orderservice.api.OrderController     : Returning order: OrderDto{id=1, customerId=1, amount=80.0}
```





```bash

kubectl logs -l app=customer-service


2023-10-29 10:47:39.389  INFO 1 --- [           main] c.n.c.CustomerServiceApplication         : Started CustomerServiceApplication in 5.736 seconds (JVM running for 6.578)
2023-10-29 11:02:09.500  INFO 1 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-10-29 11:02:09.501  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-10-29 11:02:09.503  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
2023-10-29 11:02:09.572  INFO 1 --- [nio-8080-exec-1] c.n.c.api.CustomerController             : Getting customer with id = 1...
2023-10-29 11:02:09.576  INFO 1 --- [nio-8080-exec-1] c.n.c.api.CustomerController             : Returning customer: CustomerDto{id=1, discount=20}
2023-10-29 11:06:57.747  INFO 1 --- [nio-8080-exec-2] c.n.c.api.CustomerController             : Getting customer with id = 1...
2023-10-29 11:06:57.748  INFO 1 --- [nio-8080-exec-2] c.n.c.api.CustomerController             : Returning customer: CustomerDto{id=1, discount=20}
```




