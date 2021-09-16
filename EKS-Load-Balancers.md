# How can I create and manage load balancers in an EKS cluster?


### Short description:
EKS provides two loadbalancer controllers, which can be used to create and manage AWS load balancers. This article is a good introduction into the topic

### Resolution:
In this example we are going to exepose nginx deployment using differet types of load balancers:
```
kubectl create deployment nginx --image=nginx
```
If you want to create a **Classic Load Balancer (CLB)** you have to use the Kubernetes in-tree (built-in) load balancer controller.  
To do that, you need to create a loadbalancer service; a CLB will be created by default.

For example this template will create an internet-facing CLB:
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
  type: LoadBalancer
```

If you want to create a **Network Load Balancer (NLB)** you have two options: 
- Kubernetes in-tree (built-in) load balancer controller.
- AWS Load Balancer Controller (has to be installed)

They both use loadbalancer services to create NLBs.
This template will create an internet-facing NLB using the Kubernetes in-tree load balancer controller.
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
  type: LoadBalancer
```

This template will create an internet-facing NLB using the AWS Load Balancer Controller.
```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
  type: LoadBalancer
```
*** `service.beta.kubernetes.io/aws-load-balancer-type: external` instructs Kubernetes to use AWS Load Balancer Controller.

If you want to create an **Application Load Balancer (ALB)** you have to use the AWS Load Balancer Controller and specify ALB parameters using an ingress object.
Before you create an ingress, you should expose the deployment using a nodeport service:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
  type: NodePort
```

Now, let's create an ALB using ingress:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
  labels:
    app: ingress
spec:
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: nginx
              port:
                number: 80
```
