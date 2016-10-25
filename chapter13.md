# Chapter 13: Deploying the cluster add-on: DNS (skydns)

DNS add-on is required for every Kubernetes cluster. ( I wonder why it is not part of core kubernetes!) . Without the DNS add-on the following things will not work:

* DNS based service discovery
* DNS lookups from containers running in pods

## Create kubedns service

```
kubectl create -f https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/services/kubedns.yaml
```

Verification
```
[root@controller1 ~]# kubectl get svc --namespace=kube-system
NAME       CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   10.32.0.10   <none>        53/UDP,53/TCP   2s
[root@controller1 ~]# 
```

## Create the kubedns deployment:
```
kubectl create -f https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/deployments/kubedns.yaml
```


## Verification & Validation

### Verification
```
kubectl --namespace=kube-system get pods
```

```
[root@controller1 ~]# kubectl --namespace=kube-system get pods -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
kube-dns-v19-965658604-1jq36   3/3       Running   2          33m       10.200.1.5   worker2.example.com
kube-dns-v19-965658604-oyws2   3/3       Running   0          33m       10.200.0.3   worker1.example.com
[root@controller1 ~]# 
```

(todo) I wonder why one pod had two restarts!


## Validation that DNS is actually working - using a pod:

```
[root@controller1 ~]# kubectl exec centos-multitool-3822887632-6qbrh -it -- bash 
```

First confirm that the pod has correct DNS setup in it's /etc/resolv.conf file:

```
[root@centos-multitool-3822887632-6qbrh /]# cat /etc/resolv.conf 
search default.svc.cluster.local svc.cluster.local cluster.local example.com
nameserver 10.32.0.10
options ndots:5
[root@centos-multitool-3822887632-6qbrh /]# 
```

Now check if it resolves the service name registered with kubernetes internal DNS:
```
[root@centos-multitool-3822887632-6qbrh /]# dig kubernetes.default.svc.cluster.local                   

; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7_2.3 <<>> kubernetes.default.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1090
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubernetes.default.svc.cluster.local. IN A

;; ANSWER SECTION:
kubernetes.default.svc.cluster.local. 22 IN A	10.32.0.1

;; Query time: 9 msec
;; SERVER: 10.32.0.10#53(10.32.0.10)
;; WHEN: Fri Sep 16 10:42:07 UTC 2016
;; MSG SIZE  rcvd: 81

[root@centos-multitool-3822887632-6qbrh /]# 
```
Great! So it is able to resolve the internal service names! Lets see if it can also resolve host names outside this cluster.

```
[root@centos-multitool-3822887632-6qbrh /]# dig yahoo.com

; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7_2.3 <<>> yahoo.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55948
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 6, ADDITIONAL: 11

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;yahoo.com.			IN	A

;; ANSWER SECTION:
yahoo.com.		23	IN	A	206.190.36.45
yahoo.com.		23	IN	A	98.139.183.24
yahoo.com.		23	IN	A	98.138.253.109

;; AUTHORITY SECTION:
yahoo.com.		10714	IN	NS	ns4.yahoo.com.
yahoo.com.		10714	IN	NS	ns3.yahoo.com.
yahoo.com.		10714	IN	NS	ns5.yahoo.com.
yahoo.com.		10714	IN	NS	ns2.yahoo.com.
yahoo.com.		10714	IN	NS	ns6.yahoo.com.
yahoo.com.		10714	IN	NS	ns1.yahoo.com.

;; ADDITIONAL SECTION:
ns5.yahoo.com.		258970	IN	A	119.160.247.124
ns4.yahoo.com.		327698	IN	A	98.138.11.157
ns2.yahoo.com.		333473	IN	A	68.142.255.16
ns1.yahoo.com.		290475	IN	A	68.180.131.16
ns6.yahoo.com.		10714	IN	A	121.101.144.139
ns3.yahoo.com.		325559	IN	A	203.84.221.53
ns2.yahoo.com.		32923	IN	AAAA	2001:4998:140::1002
ns1.yahoo.com.		8839	IN	AAAA	2001:4998:130::1001
ns6.yahoo.com.		158315	IN	AAAA	2406:2000:108:4::1006
ns3.yahoo.com.		6892	IN	AAAA	2406:8600:b8:fe03::1003

;; Query time: 23 msec
;; SERVER: 10.32.0.10#53(10.32.0.10)
;; WHEN: Fri Sep 16 11:12:12 UTC 2016
;; MSG SIZE  rcvd: 402

[root@centos-multitool-3822887632-6qbrh /]# 
``` 
It clearly does so!


-------
