# How do I install NodeLocal DNSCache on my Amazon EKS cluster?

I want to install NodeLocal DNSCache on my Amazon Elastic Kubernetes Service (Amazon EKS) cluster.

### Short description:
This is a step by step guide about NodeLocal DNSCache installation on an EKS cluster.<br>
*You need to use a Linux shell to run the commands below*
*NodeLocal DNSCache uses CoreDNS. Do not delete CoreDNS, NodeLocal DNSCache will not work without it*

### Resolution:
#### Installation.

Delete the previous NodeLocal DNSCache installation if it is necessary:
```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

<br>Download the original NodeLocal DNSCache template:
```
wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

<br>Modify the default parameters in the template:
```
PILLAR__DNS__SERVER=$(kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP})
```
```
sed -i "s/__PILLAR__LOCAL__DNS__/169.254.20.10/g; s/__PILLAR__DNS__DOMAIN__/cluster.local/g; s/__PILLAR__DNS__SERVER__/$PILLAR__DNS__SERVER/g" nodelocaldns.yaml
```
The `__PILLAR__CLUSTER__DNS__` and `__PILLAR__UPSTREAM__SERVERS__` parameters does not need to be replaced in the template.

<br>Apply the template:<br>
```
kubectl apply -f nodelocaldns.yaml
```

#### <br>NodeLocal DNSCache is installed, now let's check it.
Check if NodeLocal DNSCache pods are running:
```
kubectl get pods -n kube-system | grep node-local-dns
```

<br>Run a pod and check if DNS resolution works:
```
kubectl run -i --tty busybox --image=busybox -- sh
```
```
/ # nslookup -type=a amazon.com
Server:         172.20.0.10
Address:        172.20.0.10:53

Non-authoritative answer:
Name:   amazon.com
Address: 176.32.103.205
Name:   amazon.com
Address: 54.239.28.85
Name:   amazon.com
Address: 205.251.242.103

/ # nslookup -type=a kubernetes.default.svc.cluster.local
Server:         172.20.0.10
Address:        172.20.0.10:53

Name:   kubernetes.default.svc.cluster.local
Address: 172.20.0.1

/ # exit
```
```
kubectl delete pod busybox
```

#### <br>DNS query workflow with NodeLocal DNSCache.

* Hit cache cases:
  * `Pod → NodeLocal DNSCache`

* Miss cache cases:
  * Local (cluster.local) domain requests: `Pod → NodeLocal DNSCache → CoreDNS`

  * External (Internet) domain requests: `Pod → NodeLocal DNSCache → VPC DNS Resolver`


