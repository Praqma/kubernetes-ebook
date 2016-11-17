# Chapter 5: Setup a etcd cluster

In this chapter we will setup up a three node etcd cluster. Kubernetes needs a data store and etcd works very well for that purpose. 
 
# Configure etcd nodes:

The reason for having dedicated etcd nodes, as explained by Kelsey:

All Kubernetes components are stateless, greatly simplifying the management of a Kubernetes cluster. All states are stored in etcd, which is a database and must receive special treatment. etcd is being run on a dedicated set of machines for the following reasons:

* The etcd lifecycle is not tied to Kubernetes. We should be able to upgrade etcd independently of Kubernetes.
* Scaling out etcd is different than scaling out the Kubernetes Control Plane.
* Prevent other applications from taking up resources (CPU, Memory, I/O) required by etcd.

First, move the certificates into place.

```
[root@etcd1 ~]# sudo mkdir -p /etc/etcd/
[root@etcd1 ~]# ls /etc/etcd/
[root@etcd1 ~]# sudo mv ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```



Next, install the necessary software on etcd nodes. Remember that the etcd version which comes with Fedora 24 is 2.2, whereas the latest version of etcd available from its github page is 3.0.7. So, we'll download and install that one.

Do the following steps on both nodes:
```
curl -L https://github.com/coreos/etcd/releases/download/v3.0.7/etcd-v3.0.7-linux-amd64.tar.gz -o etcd-v3.0.7-linux-amd64.tar.gz
tar xzvf etcd-v3.0.7-linux-amd64.tar.gz 
sudo cp etcd-v3.0.7-linux-amd64/etcd* /usr/bin/
sudo mkdir -p /var/lib/etcd
```

Create the etcd systemd unit file:

```
cat > etcd.service <<"EOF"
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/bin/etcd --name ETCD_NAME \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --initial-advertise-peer-urls https://INTERNAL_IP:2380 \
  --listen-peer-urls https://INTERNAL_IP:2380 \
  --listen-client-urls https://INTERNAL_IP:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://INTERNAL_IP:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster etcd1=https://10.240.0.11:2380,etcd2=https://10.240.0.12:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```



**Note:** Make sure to change the IP below to the one belonging to the etcd node you are configuring.
```
export INTERNAL_IP='10.240.0.11'
export ETCD_NAME=$(hostname -s)
sed -i s/INTERNAL_IP/$INTERNAL_IP/g etcd.service
sed -i s/ETCD_NAME/$ETCD_NAME/g etcd.service
sudo mv etcd.service /etc/systemd/system/
```

Start etcd:
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```



## Verify that etcd is running:
```
[root@etcd1 ~]# sudo systemctl status etcd --no-pager
● etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-09-09 11:12:05 CEST; 29s ago
     Docs: https://github.com/coreos
 Main PID: 1563 (etcd)
    Tasks: 6 (limit: 512)
   CGroup: /system.slice/etcd.service
           └─1563 /usr/bin/etcd --name etcd1 --cert-file=/etc/etcd/kubernetes.pem --key-file=/etc/etcd/kubernetes-key.pem --peer-cert-file=/e...

Sep 09 11:12:32 etcd1.example.com etcd[1563]: ffed16798470cab5 [logterm: 1, index: 2] sent vote request to 3a57933972cb5131 at term 20
Sep 09 11:12:33 etcd1.example.com etcd[1563]: ffed16798470cab5 is starting a new election at term 20
Sep 09 11:12:33 etcd1.example.com etcd[1563]: ffed16798470cab5 became candidate at term 21
Sep 09 11:12:33 etcd1.example.com etcd[1563]: ffed16798470cab5 received vote from ffed16798470cab5 at term 21
Sep 09 11:12:33 etcd1.example.com etcd[1563]: ffed16798470cab5 [logterm: 1, index: 2] sent vote request to 3a57933972cb5131 at term 21
Sep 09 11:12:34 etcd1.example.com etcd[1563]: publish error: etcdserver: request timed out
Sep 09 11:12:35 etcd1.example.com etcd[1563]: ffed16798470cab5 is starting a new election at term 21
Sep 09 11:12:35 etcd1.example.com etcd[1563]: ffed16798470cab5 became candidate at term 22
Sep 09 11:12:35 etcd1.example.com etcd[1563]: ffed16798470cab5 received vote from ffed16798470cab5 at term 22
Sep 09 11:12:35 etcd1.example.com etcd[1563]: ffed16798470cab5 [logterm: 1, index: 2] sent vote request to 3a57933972cb5131 at term 22
[root@etcd1 ~]# 


[root@etcd1 ~]# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 10.240.0.11:2379        0.0.0.0:*               LISTEN      1563/etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      1563/etcd           
tcp        0      0 10.240.0.11:2380        0.0.0.0:*               LISTEN      1563/etcd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      591/sshd            
tcp6       0      0 :::9090                 :::*                    LISTEN      1/systemd           
tcp6       0      0 :::22                   :::*                    LISTEN      591/sshd            
[root@etcd1 ~]# 

[root@etcd1 ~]# etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
cluster may be unhealthy: failed to list members
Error:  client: etcd cluster is unavailable or misconfigured
error #0: client: endpoint http://127.0.0.1:2379 exceeded header timeout
error #1: dial tcp 127.0.0.1:4001: getsockopt: connection refused

[root@etcd1 ~]# 
```

**Note:** When there is only one node the etcd cluster will show up as unavailable or misconfigured.


## Verify:

After also executing all the steps on etcd2 I have the following status of services on it:
```
[root@etcd2 ~]# systemctl status etcd
● etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2016-09-09 11:26:15 CEST; 5s ago
     Docs: https://github.com/coreos
 Main PID: 2210 (etcd)
    Tasks: 7 (limit: 512)
   CGroup: /system.slice/etcd.service
           └─2210 /usr/bin/etcd --name etcd2 --cert-file=/etc/etcd/kubernetes.pem --key-file=/etc/etcd/kubernetes-key.pem --peer-cert-file=/etc/

Sep 09 11:26:16 etcd2.example.com etcd[2210]: 3a57933972cb5131 [logterm: 1, index: 2, vote: 0] voted for ffed16798470cab5 [logterm: 1, index: 2]
Sep 09 11:26:16 etcd2.example.com etcd[2210]: raft.node: 3a57933972cb5131 elected leader ffed16798470cab5 at term 587
Sep 09 11:26:16 etcd2.example.com etcd[2210]: published {Name:etcd2 ClientURLs:[https://10.240.0.12:2379]} to cluster cdeaba18114f0e16
Sep 09 11:26:16 etcd2.example.com etcd[2210]: ready to serve client requests
Sep 09 11:26:16 etcd2.example.com etcd[2210]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
Sep 09 11:26:16 etcd2.example.com etcd[2210]: forgot to set Type=notify in systemd service file?
Sep 09 11:26:16 etcd2.example.com etcd[2210]: ready to serve client requests
Sep 09 11:26:16 etcd2.example.com etcd[2210]: serving client requests on 10.240.0.12:2379
Sep 09 11:26:16 etcd2.example.com etcd[2210]: set the initial cluster version to 3.0
Sep 09 11:26:16 etcd2.example.com etcd[2210]: enabled capabilities for version 3.0
lines 1-19/19 (END)
```

```
[root@etcd2 ~]# netstat -antlp 
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 10.240.0.12:2379        0.0.0.0:*               LISTEN      2210/etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      2210/etcd           
tcp        0      0 10.240.0.12:2380        0.0.0.0:*               LISTEN      2210/etcd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      592/sshd            
tcp        0      0 127.0.0.1:40780         127.0.0.1:2379          ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2379        10.240.0.12:35998       ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40780         ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:34986       10.240.0.11:2380        ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:35998       10.240.0.12:2379        ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2379        10.240.0.12:36002       ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:40784         127.0.0.1:2379          ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2379        10.240.0.12:35996       ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2379        10.240.0.12:35994       ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:36002       10.240.0.12:2379        ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40788         ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:36004       10.240.0.12:2379        ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:35994       10.240.0.12:2379        ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40782         ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2380        10.240.0.11:37048       ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2380        10.240.0.11:37050       ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2380        10.240.0.11:37046       ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:40782         127.0.0.1:2379          ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:35996       10.240.0.12:2379        ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2380        10.240.0.11:37076       ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:40786         127.0.0.1:2379          ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40790         ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:34988       10.240.0.11:2380        ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2379        10.240.0.12:36000       ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:40788         127.0.0.1:2379          ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40784         ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:22          10.240.0.1:51040        ESTABLISHED 1796/sshd: root [pr 
tcp        0      0 10.240.0.12:35014       10.240.0.11:2380        ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:2379          127.0.0.1:40786         ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:36000       10.240.0.12:2379        ESTABLISHED 2210/etcd           
tcp        0      0 127.0.0.1:40790         127.0.0.1:2379          ESTABLISHED 2210/etcd           
tcp        0      0 10.240.0.12:2379        10.240.0.12:36004       ESTABLISHED 2210/etcd           
tcp6       0      0 :::9090                 :::*                    LISTEN      1/systemd           
tcp6       0      0 :::22                   :::*                    LISTEN      592/sshd            
[root@etcd2 ~]#
```


```
[root@etcd2 ~]# etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
member 3a57933972cb5131 is healthy: got healthy result from https://10.240.0.12:2379
member ffed16798470cab5 is healthy: got healthy result from https://10.240.0.11:2379
cluster is healthy
[root@etcd2 ~]# 
```

```
[root@etcd1 ~]# etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
member 3a57933972cb5131 is healthy: got healthy result from https://10.240.0.12:2379
member ffed16798470cab5 is healthy: got healthy result from https://10.240.0.11:2379
cluster is healthy
[root@etcd1 ~]# 
```


**Note:** (to do) I noticed that when one etcd node (out of total two) was switched off the worker nodes started having problems:

```
Sep 19 11:21:58 worker1.example.com kubelet[27240]: E0919 11:21:58.974948   27240 kubelet.go:2913] Error updating node status, will retry: client: etcd cluster is unavailable or misconfigured
```


