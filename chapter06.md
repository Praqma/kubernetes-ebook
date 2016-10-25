# Kubernetes Control nodes

In this chapter we setup Kubernetes master / controller nodes. We setup HA for these nodes in the next chapter.

## The Kubernetes components that make up the control plane include the following components:

* Kubernetes API Server
* Kubernetes Scheduler
* Kubernetes Controller Manager

## Each component is being run on the same machines for the following reasons:

* The Scheduler and Controller Manager are tightly coupled with the API Server
* Only one Scheduler and Controller Manager can be active at a given time, but it's ok to run multiple at the same time. Each component will elect a leader via the API Server.
* Running multiple copies of each component is required for H/A
* Running each component next to the API Server eases configuration.


Setup TLS certificates in each controller node:

```
sudo mkdir -p /var/lib/kubernetes

sudo mv ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/
```

Download and install the Kubernetes controller binaries:

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kube-apiserver
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kube-controller-manager
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kube-scheduler
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kubectl
```

```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl

sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/bin/
```


## Kubernetes API Server
### Setup Authentication and Authorization
#### Authentication

Token based authentication will be used to limit access to the Kubernetes API. The authentication token is used by the following components:

* kubelet (client)
* kubectl (client)
* Kubernetes API Server (server)

The other components, mainly the scheduler and controller manager, access the Kubernetes API server locally over the insecure API port which does not require authentication. The insecure port is only enabled for local access.

Download the example token file:
```
wget https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/token.csv
``` 

Review the example token file and replace the default token.
``` 
[root@controller1 ~]# cat token.csv 
chAng3m3,admin,admin
chAng3m3,scheduler,scheduler
chAng3m3,kubelet,kubelet
[root@controller1 ~]# 
```

Move the token file into the Kubernetes configuration directory so it can be read by the Kubernetes API server.
``` 
sudo mv token.csv /var/lib/kubernetes/
```


#### Authorization

Attribute-Based Access Control (ABAC) will be used to authorize access to the Kubernetes API. In this lab ABAC will be setup using the Kubernetes policy file backend as documented in the Kubernetes authorization guide.

Download the example authorization policy file:

```
wget https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/authorization-policy.jsonl
```

Review the example authorization policy file. No changes are required.
```
[root@controller1 ~]# cat authorization-policy.jsonl
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"*", "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"admin", "namespace": "*", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"scheduler", "namespace": "*", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"kubelet", "namespace": "*", "resource": "*", "apiGroup": "*"}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:serviceaccounts", "namespace": "*", "resource": "*", "apiGroup": "*", "nonResourcePath": "*"}}
[root@controller1 ~]# 
```

Move the authorization policy file into the Kubernetes configuration directory so it can be read by the Kubernetes API server.
```
sudo mv authorization-policy.jsonl /var/lib/kubernetes/
``` 

## Create the systemd unit file

We need the IP address of each controller node, when we create the systemd file. We will setup a variable INTERNAL_IP with the IP address of each VM.

```
[root@controller1 ~]# export INTERNAL_IP='10.240.0.21'
```

```
[root@controller2 ~]# export INTERNAL_IP='10.240.0.22'
```


Create the systemd unit file:
```
cat > kube-apiserver.service <<"EOF"
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota \
  --advertise-address=INTERNAL_IP \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=ABAC \
  --authorization-policy-file=/var/lib/kubernetes/authorization-policy.jsonl \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --insecure-bind-address=0.0.0.0 \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --etcd-servers=https://10.240.0.11:2379,https://10.240.0.12:2379 \
  --service-account-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --token-auth-file=/var/lib/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
``` 

```
sed -i s/INTERNAL_IP/$INTERNAL_IP/g kube-apiserver.service
sudo mv kube-apiserver.service /etc/systemd/system/
``` 

``` 
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver
sudo systemctl start kube-apiserver

sudo systemctl status kube-apiserver --no-pager
``` 

Verify that kube-api server is listening on both controller nodes:

```
[root@controller1 ~]# sudo systemctl status kube-apiserver --no-pager
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-09-13 11:08:12 CEST; 17s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1464 (kube-apiserver)
    Tasks: 6 (limit: 512)
   CGroup: /system.slice/kube-apiserver.service
           └─1464 /usr/bin/kube-apiserver --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota...

Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: W0913 11:08:13.299066    1464 controller.go:307] Resetting endpoints for ...ion:""
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: [restful] 2016/09/13 11:08:13 log.go:30: [restful/swagger] listing is ava...erapi/
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: [restful] 2016/09/13 11:08:13 log.go:30: [restful/swagger] https://10.240...er-ui/
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.439571    1464 genericapiserver.go:690] Serving securely o...0:6443
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.439745    1464 genericapiserver.go:734] Serving insecurely...0:8080
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.940647    1464 handlers.go:165] GET /api/v1/serviceaccount...56140]
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.944980    1464 handlers.go:165] GET /api/v1/secrets?fieldS...56136]
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.947133    1464 handlers.go:165] GET /api/v1/resourcequotas...56138]
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.950795    1464 handlers.go:165] GET /api/v1/namespaces?res...56142]
Sep 13 11:08:13 controller1.example.com kube-apiserver[1464]: I0913 11:08:13.966576    1464 handlers.go:165] GET /api/v1/limitranges?re...56142]
Hint: Some lines were ellipsized, use -l to show in full.
[root@controller1 ~]#

[root@controller2 ~]# sudo systemctl status kube-apiserver --no-pager
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-09-13 11:08:16 CEST; 1min 16s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1488 (kube-apiserver)
    Tasks: 5 (limit: 512)
   CGroup: /system.slice/kube-apiserver.service
           └─1488 /usr/bin/kube-apiserver --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota...

Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: W0913 11:08:17.165892    1488 controller.go:342] Resetting endpoints for ...ion:""
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: [restful] 2016/09/13 11:08:17 log.go:30: [restful/swagger] listing is ava...erapi/
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: [restful] 2016/09/13 11:08:17 log.go:30: [restful/swagger] https://10.240...er-ui/
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.244260    1488 genericapiserver.go:690] Serving securely o...0:6443
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.244275    1488 genericapiserver.go:734] Serving insecurely...0:8080
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.757433    1488 handlers.go:165] GET /api/v1/resourcequotas...35132]
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.759790    1488 handlers.go:165] GET /api/v1/secrets?fieldS...35126]
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.761101    1488 handlers.go:165] GET /api/v1/serviceaccount...35128]
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.763786    1488 handlers.go:165] GET /api/v1/limitranges?re...35130]
Sep 13 11:08:17 controller2.example.com kube-apiserver[1488]: I0913 11:08:17.768911    1488 handlers.go:165] GET /api/v1/namespaces?res...35124]
Hint: Some lines were ellipsized, use -l to show in full.
[root@controller2 ~]#
```

## Kubernetes Controller Manager

```
cat > kube-controller-manager.service <<"EOF"
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-controller-manager \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --leader-elect=true \
  --master=http://INTERNAL_IP:8080 \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
``` 

```
sed -i s/INTERNAL_IP/$INTERNAL_IP/g kube-controller-manager.service
sudo mv kube-controller-manager.service /etc/systemd/system/
``` 

```
sudo systemctl daemon-reload
sudo systemctl enable kube-controller-manager
sudo systemctl start kube-controller-manager

sudo systemctl status kube-controller-manager --no-pager
``` 

Verify that kube-api-server is running on both nodes:

```
[root@controller1 ~]# sudo systemctl status kube-controller-manager --no-pager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-09-13 11:12:13 CEST; 13s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1531 (kube-controller)
    Tasks: 5 (limit: 512)
   CGroup: /system.slice/kube-controller-manager.service
           └─1531 /usr/bin/kube-controller-manager --allocate-node-cidrs=true --cluster-cidr=10.200.0.0/16 --cluster-name=kubernetes --leader...

Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.485918    1531 pet_set.go:144] Starting petset controller
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.561887    1531 plugins.go:340] Loaded volume plugi...-ebs"
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.562103    1531 plugins.go:340] Loaded volume plugi...e-pd"
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.562227    1531 plugins.go:340] Loaded volume plugi...nder"
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.570878    1531 attach_detach_controller.go:191] St...oller
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: E0913 11:12:23.583095    1531 util.go:45] Metric for serviceaccou...tered
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: W0913 11:12:23.595468    1531 request.go:347] Field selector: v1 ...ctly.
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.619022    1531 endpoints_controller.go:322] Waitin...netes
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: W0913 11:12:23.649898    1531 request.go:347] Field selector: v1 ...ctly.
Sep 13 11:12:23 controller1.example.com kube-controller-manager[1531]: I0913 11:12:23.737340    1531 endpoints_controller.go:322] Waitin...netes
Hint: Some lines were ellipsized, use -l to show in full.
[root@controller1 ~]# 


[root@controller2 ~]# sudo systemctl enable kube-controller-manager
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service to /etc/systemd/system/kube-controller-manager.service.

[root@controller2 ~]# sudo systemctl start kube-controller-manager
[root@controller2 ~]# sudo systemctl status kube-controller-manager --no-pager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-09-13 11:12:18 CEST; 11s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1553 (kube-controller)
    Tasks: 4 (limit: 512)
   CGroup: /system.slice/kube-controller-manager.service
           └─1553 /usr/bin/kube-controller-manager --allocate-node-cidrs=true --cluster-cidr=10.200.0.0/16 --cluster-name=kubernetes --leader...

Sep 13 11:12:18 controller2.example.com systemd[1]: Started Kubernetes Controller Manager.
Sep 13 11:12:18 controller2.example.com kube-controller-manager[1553]: I0913 11:12:18.246979    1553 leaderelection.go:296] lock is held...pired
Sep 13 11:12:21 controller2.example.com kube-controller-manager[1553]: I0913 11:12:21.701152    1553 leaderelection.go:296] lock is held...pired
Sep 13 11:12:25 controller2.example.com kube-controller-manager[1553]: I0913 11:12:25.960509    1553 leaderelection.go:296] lock is held...pired
Sep 13 11:12:29 controller2.example.com kube-controller-manager[1553]: I0913 11:12:29.558337    1553 leaderelection.go:296] lock is held...pired
Hint: Some lines were ellipsized, use -l to show in full.
[root@controller2 ~]# 
```

## Kubernetes Scheduler

```
cat > kube-scheduler.service <<"EOF"
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-scheduler \
  --leader-elect=true \
  --master=http://INTERNAL_IP:8080 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
sed -i s/INTERNAL_IP/$INTERNAL_IP/g kube-scheduler.service
sudo mv kube-scheduler.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
sudo systemctl enable kube-scheduler
sudo systemctl start kube-scheduler

sudo systemctl status kube-scheduler --no-pager
```

Verify that kube-scheduler is running on both nodes:

```
[root@controller1 ~]# sudo systemctl status kube-scheduler --no-pager
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-09-13 11:16:19 CEST; 1s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1591 (kube-scheduler)
    Tasks: 4 (limit: 512)
   CGroup: /system.slice/kube-scheduler.service
           └─1591 /usr/bin/kube-scheduler --leader-elect=true --master=http://10.240.0.21:8080 --v=2

Sep 13 11:16:19 controller1.example.com systemd[1]: Started Kubernetes Scheduler.
Sep 13 11:16:19 controller1.example.com kube-scheduler[1591]: I0913 11:16:19.701363    1591 factory.go:255] Creating scheduler from alg...vider'
Sep 13 11:16:19 controller1.example.com kube-scheduler[1591]: I0913 11:16:19.701740    1591 factory.go:301] creating scheduler with fit predi...
Sep 13 11:16:19 controller1.example.com kube-scheduler[1591]: E0913 11:16:19.743682    1591 event.go:257] Could not construct reference to: '...
Sep 13 11:16:19 controller1.example.com kube-scheduler[1591]: I0913 11:16:19.744595    1591 leaderelection.go:215] sucessfully acquired...eduler
Hint: Some lines were ellipsized, use -l to show in full.
[root@controller1 ~]#



[root@controller2 ~]# sudo systemctl status kube-scheduler --no-pager
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-09-13 11:16:24 CEST; 1s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1613 (kube-scheduler)
    Tasks: 4 (limit: 512)
   CGroup: /system.slice/kube-scheduler.service
           └─1613 /usr/bin/kube-scheduler --leader-elect=true --master=http://10.240.0.22:8080 --v=2

Sep 13 11:16:24 controller2.example.com systemd[1]: Started Kubernetes Scheduler.
Sep 13 11:16:25 controller2.example.com kube-scheduler[1613]: I0913 11:16:25.111478    1613 factory.go:255] Creating scheduler from alg...vider'
Sep 13 11:16:25 controller2.example.com kube-scheduler[1613]: I0913 11:16:25.112652    1613 factory.go:301] creating scheduler with fit predi...
Sep 13 11:16:25 controller2.example.com kube-scheduler[1613]: I0913 11:16:25.163057    1613 leaderelection.go:296] lock is held by cont...xpired
Hint: Some lines were ellipsized, use -l to show in full.
[root@controller2 ~]# 
```


Verify using `kubectl get componentstatuses`:
```
[root@controller1 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
[root@controller1 ~]# 


[root@controller2 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
[root@controller2 ~]# 
```



# Setup Kubernetes frontend load balancer

This is not critical, and can be done later.

(TODO)(To do)

