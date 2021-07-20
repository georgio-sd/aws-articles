# Why some pods cannot resolve a specific domain name on my Amazon EKS cluster?

I noticed that some pods cannot resolve `xyz.xyz` domain name on my Amazon Elastic Kubernetes Service (Amazon EKS) cluster. 
Other domain names are OK, the issue is only with `xyz.xyz`. When I resolve this domain on the cluster nodes, it resolves with no issue.

### Short description:
By default, CoreDNS follows the client protocol when talking to the upstream servers. If a client sent DNS request over UDP without the EDNS0 option, 
and the upstream server sent a truncated response, CoreDNS doesn't retry the request over TCP. It also doesn't set the TC flag in the 
response sent to the client. As a result, client ends up with a potentially incomplete DNS response, and potentially fail to resolve 
the hostname permanently.

### Resolution:
Let's reproduce this scenario and then fix it.
<br>Create a busybox pod:
```
kubectl run busybox --image=busybox --command sleep 8h
```

<br>and login to it:
```
kubectl exec busybox -it -- sh
```

<br>Then attempt to resolve for example this domain name:
```
/ # nslookup -type=a aerserv-bc-us-east.bidswitch.net
Server:         172.20.0.10
Address:        172.20.0.10:53

Non-authoritative answer:
aerserv-bc-us-east.bidswitch.net        canonical name = bidcast-bcserver-gce-sc.bidswitch.net
bidcast-bcserver-gce-sc.bidswitch.net   canonical name = bidcast-bcserver-gce-sc-multifo.bidswitch.net
```
```
/ # wget aerserv-bc-us-east.bidswitch.net
wget: bad address 'aerserv-bc-us-east.bidswitch.net'
```
*** As you can see both commands failed. You can also try to resolve your `xyz.xyz` domain.

#### <br>To fix it, we need to allow CoreDNS to make requests over TCP when a UDP request failed.
Modify CoreDNS config map:
```
kubectl edit cm coredns -n kube-system
```

<br>and replace this snippet:
```
...
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
...
```

<br>with this one:
```
...
        prometheus :9153
        forward . /etc/resolv.conf {
          prefer_udp
        }
        cache 30
...
```

<br>Wait for a couple of minutes for the CoreDNS pods to reread the new config map and check if this resolved the issue:
```
kubectl exec busybox -it -- sh
```
```
/ # nslookup -type=a aerserv-bc-us-east.bidswitch.net
Server:         172.20.0.10
Address:        172.20.0.10:53

aerserv-bc-us-east.bidswitch.net        canonical name = bidcast-bcserver-gce-sc.bidswitch.net
bidcast-bcserver-gce-sc.bidswitch.net   canonical name = bidcast-bcserver-gce-sc-multifo.bidswitch.net
Name:   bidcast-bcserver-gce-sc-multifo.bidswitch.net
Address: 35.211.159.137
Name:   bidcast-bcserver-gce-sc-multifo.bidswitch.net
Address: 35.207.29.9
Name:   bidcast-bcserver-gce-sc-multifo.bidswitch.net
...
Name:   bidcast-bcserver-gce-sc-multifo.bidswitch.net
Address: 35.211.37.223
Name:   bidcast-bcserver-gce-sc-multifo.bidswitch.net
Address: 35.211.131.202
Name:   bidcast-bcserver-gce-sc-multifo.bidswitch.net
Address: 35.211.123.196
```
*** No errors anymore
