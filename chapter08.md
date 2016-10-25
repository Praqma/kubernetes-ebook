# Chapter 8: Setup Kubernetes Worker Nodes


* Ease of deployment and configuration
* Avoid mixing arbitrary workloads with critical cluster components. We are building machine with just enough resources so we don't have to worry about wasting resources.

## Provision the Kubernetes Worker Nodes

Run the following commands on all worker nodes.

Move the TLS certificates in place
```
sudo mkdir -p /var/lib/kubernetes
sudo mv ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/
``` 

## Install Docker

Kubernetes should be compatible with the Docker 1.9.x - 1.11.x:

```
wget https://get.docker.com/builds/Linux/x86_64/docker-1.11.2.tgz

tar -xf docker-1.11.2.tgz

sudo cp docker/docker* /usr/bin/
```

Create the Docker systemd unit file:
```
sudo sh -c 'echo "[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/bin/docker daemon \
  --iptables=false \
  --ip-masq=false \
  --host=unix:///var/run/docker.sock \
  --log-level=error \
  --storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/docker.service'
```

```
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl start docker

sudo docker version
```


```
[root@worker1 ~]# sudo docker version
Client:
 Version:      1.11.2
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   b9f10c9
 Built:        Wed Jun  1 21:20:08 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.11.2
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   b9f10c9
 Built:        Wed Jun  1 21:20:08 2016
 OS/Arch:      linux/amd64
[root@worker1 ~]# 
```

## Setup kubelet on worker nodes:

The Kubernetes kubelet no longer relies on docker networking for pods! The Kubelet can now use CNI - the Container Network Interface to manage machine level networking requirements.

Download and install CNI plugins

```
sudo mkdir -p /opt/cni

wget https://storage.googleapis.com/kubernetes-release/network-plugins/cni-c864f0e1ea73719b8f4582402b0847064f9883b0.tar.gz

sudo tar -xvf cni-c864f0e1ea73719b8f4582402b0847064f9883b0.tar.gz -C /opt/cni
```
**Note:** Kelsey's guide does not mention this, but the kubernetes binaries look for plugin binaries in /opt/plugin-name/bin/, and then in other paths if nothing is found over there.


Download and install the Kubernetes worker binaries:
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kubectl
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kube-proxy
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kubelet
```

```
chmod +x kubectl kube-proxy kubelet

sudo mv kubectl kube-proxy kubelet /usr/bin/

sudo mkdir -p /var/lib/kubelet/
```


Create kubeconfig file:
```
sudo sh -c 'echo "apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://10.240.0.21:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet
  name: kubelet
current-context: kubelet
users:
- name: kubelet
  user:
    token: chAng3m3" > /var/lib/kubelet/kubeconfig'
```
**Note:** Notice that `server` is specified as 10.240.0.21, which is the IP address of the first controller. We can use the virtual IP of the controllers (which is 10.240.0.20) , but we have not actually configured a load balancer with this IP address yet. So we are just using the IP address of one of the controller nodes. Remember, Kelsey's guide uses the IP address of 10.240.0.20 , but that is the IP address of controller0 in his guide, not the  VIP of controller nodes.


Create the kubelet systemd unit file:
```
sudo sh -c 'echo "[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \
  --allow-privileged=true \
  --api-servers=https://10.240.0.21:6443,https://10.240.0.22:6443 \
  --cloud-provider= \
  --cluster-dns=10.32.0.10 \
  --cluster-domain=cluster.local \
  --configure-cbr0=true \
  --container-runtime=docker \
  --docker=unix:///var/run/docker.sock \
  --network-plugin=kubenet \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --reconcile-cidr=true \
  --serialize-image-pulls=false \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/kubelet.service'
```
**Note:** Notice `--configure-cbr0=true` , this enables the container bridge, which from the pool 10.200.0.0/16, and can be any of 10.200.x.0/24 network. Also notice that this service requires the docker service to be up before it starts.


Start the kubelet service and check that it is running:
```
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet

sudo systemctl status kubelet --no-pager
```


```
[root@worker1 ~]# sudo systemctl status kubelet --no-pager
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 11:38:03 CEST; 1s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1954 (kubelet)
    Tasks: 11 (limit: 512)
   CGroup: /system.slice/kubelet.service
           ├─1954 /usr/bin/kubelet --allow-privileged=true --api-servers=https://10.240.0.21:6443,https://10.240.0.22:6443 --cloud-provider= ...
           └─2002 journalctl -k -f

Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.018143    1954 kubelet.go:1197] Attempting to register node worker1....ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.019360    1954 kubelet.go:1200] Unable to register worker1.example.c...refused
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.220851    1954 kubelet.go:2924] Recording NodeHasSufficientDisk even...ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.221101    1954 kubelet.go:2924] Recording NodeHasSufficientMemory ev...ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.221281    1954 kubelet.go:1197] Attempting to register node worker1....ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.222266    1954 kubelet.go:1200] Unable to register worker1.example.c...refused
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.623784    1954 kubelet.go:2924] Recording NodeHasSufficientDisk even...ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.624059    1954 kubelet.go:2924] Recording NodeHasSufficientMemory ev...ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.624328    1954 kubelet.go:1197] Attempting to register node worker1....ple.com
Sep 14 11:38:04 worker1.example.com kubelet[1954]: I0914 11:38:04.625329    1954 kubelet.go:1200] Unable to register worker1.example.c...refused
Hint: Some lines were ellipsized, use -l to show in full.
[root@worker1 ~]#


[root@worker2 ~]# sudo systemctl status kubelet --no-pager
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 11:38:08 CEST; 920ms ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 1999 (kubelet)
    Tasks: 10 (limit: 512)
   CGroup: /system.slice/kubelet.service
           ├─1999 /usr/bin/kubelet --allow-privileged=true --api-servers=https://10.240.0.21:6443,https://10.240.0.22:6443 --cloud-provider= ...
           └─2029 journalctl -k -f

Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.218332    1999 manager.go:281] Starting recovery of all containers
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.254250    1999 manager.go:286] Recovery completed
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.359776    1999 kubelet.go:2924] Recording NodeHasSufficientDisk even...ple.com
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.359800    1999 kubelet.go:2924] Recording NodeHasSufficientMemory ev...ple.com
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.360031    1999 kubelet.go:1197] Attempting to register node worker2....ple.com
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.363621    1999 kubelet.go:1200] Unable to register worker2.example.c...refused
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.565044    1999 kubelet.go:2924] Recording NodeHasSufficientDisk even...ple.com
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.566188    1999 kubelet.go:2924] Recording NodeHasSufficientMemory ev...ple.com
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.566323    1999 kubelet.go:1197] Attempting to register node worker2....ple.com
Sep 14 11:38:09 worker2.example.com kubelet[1999]: I0914 11:38:09.568444    1999 kubelet.go:1200] Unable to register worker2.example.c...refused
Hint: Some lines were ellipsized, use -l to show in full.
[root@worker2 ~]# 
```

## kube-proxy
Kube-proxy sets up IPTables rules on the nodes so containers can find services.

Create systemd unit file for kube-proxy:
```
sudo sh -c 'echo "[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-proxy \
  --master=https://10.240.0.21:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --proxy-mode=iptables \
  --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/kube-proxy.service'
```
**Note:** We have used the IP address of the first controller in the systemd file above. Later, we can change it to use the VIP of the controller nodes.





```
sudo systemctl daemon-reload
sudo systemctl enable kube-proxy
sudo systemctl start kube-proxy

sudo systemctl status kube-proxy --no-pager
```


```
[root@worker1 ~]# sudo systemctl status kube-proxy --no-pager
● kube-proxy.service - Kubernetes Kube Proxy
   Loaded: loaded (/etc/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 12:02:35 CEST; 635ms ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 2373 (kube-proxy)
    Tasks: 4 (limit: 512)
   CGroup: /system.slice/kube-proxy.service
           └─2373 /usr/bin/kube-proxy --master=https://10.240.0.21:6443 --kubeconfig=/var/lib/kubelet/kubeconfig --proxy-mode=iptables --v=2

Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: I0914 12:02:35.508769    2373 server.go:202] Using iptables Proxier.
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: W0914 12:02:35.509552    2373 server.go:416] Failed to retrieve node info: Get ht...efused
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: W0914 12:02:35.509608    2373 proxier.go:227] invalid nodeIP, initialize kube-pro...nodeIP
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: I0914 12:02:35.509618    2373 server.go:214] Tearing down userspace rules.
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: I0914 12:02:35.521907    2373 conntrack.go:40] Setting nf_conntrack_max to 32768
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: I0914 12:02:35.522205    2373 conntrack.go:57] Setting conntrack hashsize to 8192
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: I0914 12:02:35.522521    2373 conntrack.go:62] Setting nf_conntrack_tcp_timeout_e... 86400
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: E0914 12:02:35.523511    2373 event.go:207] Unable to write event: 'Post https://...eping)
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: E0914 12:02:35.523709    2373 reflector.go:205] pkg/proxy/config/api.go:33: Faile...efused
Sep 14 12:02:35 worker1.example.com kube-proxy[2373]: E0914 12:02:35.523947    2373 reflector.go:205] pkg/proxy/config/api.go:30: Faile...efused
Hint: Some lines were ellipsized, use -l to show in full.
[root@worker1 ~]# 


[root@worker2 ~]# sudo systemctl status kube-proxy --no-pager
● kube-proxy.service - Kubernetes Kube Proxy
   Loaded: loaded (/etc/systemd/system/kube-proxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 12:02:46 CEST; 1s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 2385 (kube-proxy)
    Tasks: 4 (limit: 512)
   CGroup: /system.slice/kube-proxy.service
           └─2385 /usr/bin/kube-proxy --master=https://10.240.0.21:6443 --kubeconfig=/var/lib/kubelet/kubeconfig --proxy-mode=iptables --v=2

Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: W0914 12:02:46.660676    2385 proxier.go:227] invalid nodeIP, initialize kube-pro...nodeIP
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: I0914 12:02:46.660690    2385 server.go:214] Tearing down userspace rules.
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: I0914 12:02:46.670904    2385 conntrack.go:40] Setting nf_conntrack_max to 32768
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: I0914 12:02:46.671630    2385 conntrack.go:57] Setting conntrack hashsize to 8192
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: I0914 12:02:46.671687    2385 conntrack.go:62] Setting nf_conntrack_tcp_timeout_e... 86400
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: E0914 12:02:46.673067    2385 event.go:207] Unable to write event: 'Post https://...eping)
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: E0914 12:02:46.673266    2385 reflector.go:205] pkg/proxy/config/api.go:33: Faile...efused
Sep 14 12:02:46 worker2.example.com kube-proxy[2385]: E0914 12:02:46.673514    2385 reflector.go:205] pkg/proxy/config/api.go:30: Faile...efused
Sep 14 12:02:47 worker2.example.com kube-proxy[2385]: E0914 12:02:47.674206    2385 reflector.go:205] pkg/proxy/config/api.go:33: Faile...efused
Sep 14 12:02:47 worker2.example.com kube-proxy[2385]: E0914 12:02:47.674254    2385 reflector.go:205] pkg/proxy/config/api.go:30: Faile...efused
Hint: Some lines were ellipsized, use -l to show in full.
[root@worker2 ~]# 
```


At this point, you should be able to see the nodes as **Ready**.

```
[root@controller1 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   


[root@controller1 ~]# kubectl get nodes
NAME                  STATUS    AGE
worker1.example.com   Ready     47s
worker2.example.com   Ready     41s
[root@controller1 ~]# 
```

**Note:** Sometimes the nodes do not show up as Ready in the output of `kubectl get nodes` command. It is ok to reboot the worker nodes. 


**Note:** Worker node configuration is complete at this point.

------
## Some notes on CIDR/CNI IP address showing/not-showing on the worker nodes:

(to do) Add a step to make sure that the worker nodes have got the CIDR IP address. Right now, in my setup, I do not see CIDR addresses assigned to my worker nodes, even though they show up as Ready . 


( **UPDATE:** I recently found out that the CIDR network assigned to each worker node shows up in the output of `kubectl describe node <NodeName>` command. This is very handy! )

```
[root@worker1 ~]# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:4a:68:e4:2f  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.240.0.31  netmask 255.255.255.0  broadcast 10.240.0.255
        inet6 fe80::5054:ff:fe03:a650  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:03:a6:50  txqueuelen 1000  (Ethernet)
        RX packets 2028  bytes 649017 (633.8 KiB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 1689  bytes 262384 (256.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 20  bytes 1592 (1.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 20  bytes 1592 (1.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@worker1 ~]#
```
**Note:** Where is the IP address from CIDR?

Here is a hint of the underlying problem (which, actually is not a problem):
```
[root@worker1 ~]# systemctl status kubelet -l
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 13:16:13 CEST; 9min ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 4744 (kubelet)
    Tasks: 11 (limit: 512)
   CGroup: /system.slice/kubelet.service
           ├─4744 /usr/bin/kubelet --allow-privileged=true --api-servers=https://10.240.0.21:6443,https://10.240.0.22:6443 --cloud-provider= --c
           └─4781 journalctl -k -f

4744 kubelet.go:2510] skipping pod synchronization - [Kubenet does not have netConfig. This is most likely due to lack of PodCIDR]
4744 kubelet.go:2510] skipping pod synchronization - [Kubenet does not have netConfig. This is most likely due to lack of PodCIDR]
4744 kubelet.go:2510] skipping pod synchronization - [Kubenet does not have netConfig. This is most likely due to lack of PodCIDR]
4744 kubelet.go:2924] Recording NodeReady event message for node worker1.example.com
```

On the controller I see that the kube-controller-manager has some details:

```
[root@controller2 ~]# systemctl status kube-controller-manager.service -l
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 13:07:10 CEST; 25min ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 550 (kube-controller)
    Tasks: 5 (limit: 512)
   CGroup: /system.slice/kube-controller-manager.service
           └─550 /usr/bin/kube-controller-manager --allocate-node-cidrs=true --cluster-cidr=10.200.0.0/16 --cluster-name=kubernetes --leader-ele

. . .
13:10:52.772513     550 nodecontroller.go:534] NodeController is entering network segmentation mode.
13:10:52.772630     550 event.go:216] Event(api.ObjectReference{Kind:"Node", Namespace:"", Name:"worker2.example.com", UID:"worker2.example.
13:10:57.775051     550 nodecontroller.go:534] NodeController is entering network segmentation mode.
13:11:02.777334     550 nodecontroller.go:534] NodeController is entering network segmentation mode.
13:11:07.781592     550 nodecontroller.go:534] NodeController is entering network segmentation mode.
13:11:12.784489     550 nodecontroller.go:534] NodeController is entering network segmentation mode.
13:11:17.787018     550 nodecontroller.go:539] NodeController exited network segmentation mode.
13:17:36.729147     550 request.go:347] Field selector: v1 - serviceaccounts - metadata.name - default: need to check if this is versioned correctly.
13:25:32.730591     550 request.go:347] Field selector: v1 - serviceaccounts - metadata.name - default: need to check if this is versioned correctly.
```


Also look at this issue: [https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/58](https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/58)


**Note:** The above issue explains/shows that the cbr0 network only gets created on pods when firt pod is created and is placed on the worker node. This also means that we cannot update routing table on our router until we know which network exists on which node?!


This means, at this point, we shall create a test pod, and see if worker node gets a cbr0 IP address . We will also use this information at a later step, when we add routes to the routing table on our router.

------


# Create a test pod:

Login to controller1 and run a test pod. Many people like to run nginx, which runs the ngin webserver, but does not have any tools for network troubleshooting. There is a centos based multitool I created, which runs apache and has many network troubleshooting tools built into it. It is available at dockerhub kamranazeem/centos-multitool . 

```
[root@controller1 ~]# kubectl run centos-multitool --image=kamranazeem/centos-multitool
deployment "centos-multitool" created
[root@controller1 ~]# 



[root@controller1 ~]# kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP           NODE
centos-multitool-3822887632-6qbrh   1/1       Running   0          6m        10.200.1.2   worker2.example.com
[root@controller1 ~]# 
```

Check if the node got a cbr0 IP belonging to 10.200.x.0/24 , which in-turn will be a subnet of 10.200.0.0/16 .

```
[root@worker2 ~]# ifconfig
cbr0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 10.200.1.1  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::2c94:fff:fe9d:9cf6  prefixlen 64  scopeid 0x20<link>
        ether 16:89:74:67:7b:33  txqueuelen 1000  (Ethernet)
        RX packets 8  bytes 536 (536.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10  bytes 732 (732.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:bb:60:8d:d0  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.240.0.32  netmask 255.255.255.0  broadcast 10.240.0.255
        inet6 fe80::5054:ff:fe4c:f48a  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:4c:f4:8a  txqueuelen 1000  (Ethernet)
        RX packets 44371  bytes 132559205 (126.4 MiB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 37129  bytes 3515567 (3.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 20  bytes 1592 (1.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 20  bytes 1592 (1.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth5a59821e: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::1489:74ff:fe67:7b33  prefixlen 64  scopeid 0x20<link>
        ether 16:89:74:67:7b:33  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 648 (648.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17  bytes 1290 (1.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@worker2 ~]# 
```

Good!

Lets increase the number of replicas of this pod to two, which is the same as number of worker nodes. This will hopefully distribute the pods evenly on all workers.

```
[root@controller1 ~]# kubectl scale deployment centos-multitool --replicas=2
deployment "centos-multitool" scaled
[root@controller1 ~]#
```

Check the pods and the nodes they are put on:

```
[root@controller1 ~]# kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP           NODE
centos-multitool-3822887632-6qbrh   1/1       Running   0          16m       10.200.1.2   worker2.example.com
centos-multitool-3822887632-jeyhb   1/1       Running   0          9m        10.200.0.2   worker1.example.com
[root@controller1 ~]# 
```

Check the cbr0 interface on worker1 too:
```
[root@worker1 ~]# ifconfig
cbr0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 10.200.0.1  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::6cb1:ddff:fe78:4d2f  prefixlen 64  scopeid 0x20<link>
        ether 0a:79:9f:11:20:22  txqueuelen 1000  (Ethernet)
        RX packets 8  bytes 536 (536.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10  bytes 732 (732.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:fc:7a:23:24  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.240.0.31  netmask 255.255.255.0  broadcast 10.240.0.255
        inet6 fe80::5054:ff:fe03:a650  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:03:a6:50  txqueuelen 1000  (Ethernet)
        RX packets 32880  bytes 114219841 (108.9 MiB)
        RX errors 0  dropped 5  overruns 0  frame 0
        TX packets 28126  bytes 2708515 (2.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 18  bytes 1492 (1.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 18  bytes 1492 (1.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth06329870: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::879:9fff:fe11:2022  prefixlen 64  scopeid 0x20<link>
        ether 0a:79:9f:11:20:22  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 648 (648.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17  bytes 1290 (1.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@worker1 ~]#
```

Good! Lets find the IP addresses of the pods:

```
[root@controller1 ~]# kubectl exec centos-multitool-3822887632-6qbrh ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.200.1.2  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::acbc:28ff:feae:3397  prefixlen 64  scopeid 0x20<link>
        ether 0a:58:0a:c8:01:02  txqueuelen 0  (Ethernet)
        RX packets 17  bytes 1290 (1.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 648 (648.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@controller1 ~]#
```

```
[root@controller1 ~]# kubectl exec centos-multitool-3822887632-jeyhb ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.200.0.2  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::442d:6eff:fe18:f7e0  prefixlen 64  scopeid 0x20<link>
        ether 0a:58:0a:c8:00:02  txqueuelen 0  (Ethernet)
        RX packets 17  bytes 1290 (1.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 648 (648.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@controller1 ~]#
```


At this point, the pods will not be able to ping the pods on other nodes. It is because the routing is not setup on the router this cluster is connected to.

``` 
[root@controller1 ~]# kubectl exec centos-multitool-3822887632-6qbrh -it -- bash 
[root@centos-multitool-3822887632-6qbrh /]#

[root@centos-multitool-3822887632-6qbrh /]# ping -c 1 10.200.1.1 
PING 10.200.1.1 (10.200.1.1) 56(84) bytes of data.
64 bytes from 10.200.1.1: icmp_seq=1 ttl=64 time=0.062 ms

--- 10.200.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.062/0.062/0.062/0.000 ms


[root@centos-multitool-3822887632-6qbrh /]# ping -c 1 10.200.0.1 
PING 10.200.0.1 (10.200.0.1) 56(84) bytes of data.
^C
--- 10.200.0.1 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms


[root@centos-multitool-3822887632-6qbrh /]# ping -c 1 10.200.0.2 
PING 10.200.0.2 (10.200.0.2) 56(84) bytes of data.
^C
--- 10.200.0.2 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

[root@centos-multitool-3822887632-6qbrh /]# 
```

We will setup routing in the coming steps.



# ------

# Managing the Container Network Routes

Now that each worker node is online we need to add routes to make sure that Pods running on different machines can talk to each other. In this lab we are not going to provision any overlay networks and instead rely on Layer 3 networking. That means we need to add routes to our router. In GCP and AWS each network has a router that can be configured. Ours is a bare metal installation, which means we have to add routes to our local router. Since my setup is a VM based setup on KVM/Libvirt, the router in question here is actually my local work computer. 

So, we know from experience above (during worker node setup), that the cbr0 on a worker node does not get an IP address until first pod is scheduled on it. This means we are not sure which node will get which network segment (10.200.x.0/24) from the main CIDR network (10.200.0.0/16) . That means, either we do it manually, or we can create a script which does this investigation for us; and (ideally) updates router accordingly. 

Basically this information is available from the output of `kubectl describe node <nodename>` command. (If it was not, I would have to ssh into each worker node and try to see what IP address is assigned to cbr0 interface!) .

```
[root@controller1 ~]# kubectl get nodes
NAME                  STATUS    AGE
worker1.example.com   Ready     23h
worker2.example.com   Ready     23h
```

```
[root@controller1 ~]# kubectl describe node worker1
Name:			worker1.example.com
Labels:			beta.kubernetes.io/arch=amd64
			beta.kubernetes.io/os=linux
			kubernetes.io/hostname=worker1.example.com
Taints:			<none>
CreationTimestamp:	Wed, 14 Sep 2016 13:10:44 +0200
Phase:			
Conditions:
  Type			Status	LastHeartbeatTime			LastTransitionTime			Reason				Message
  ----			------	-----------------			------------------			------				-------
  OutOfDisk 		False 	Thu, 15 Sep 2016 12:55:18 +0200 	Thu, 15 Sep 2016 08:53:55 +0200 	KubeletHasSufficientDisk 	kubelet has sufficient disk space available
  MemoryPressure 	False 	Thu, 15 Sep 2016 12:55:18 +0200 	Wed, 14 Sep 2016 13:10:43 +0200 	KubeletHasSufficientMemory 	kubelet has sufficient memory available
  Ready 		True 	Thu, 15 Sep 2016 12:55:18 +0200 	Thu, 15 Sep 2016 08:59:17 +0200 	KubeletReady 			kubelet is posting ready status
Addresses:		10.240.0.31,10.240.0.31
Capacity:
 alpha.kubernetes.io/nvidia-gpu:	0
 cpu:					1
 memory:				1532864Ki
 pods:					110
Allocatable:
 alpha.kubernetes.io/nvidia-gpu:	0
 cpu:					1
 memory:				1532864Ki
 pods:					110
System Info:
 Machine ID:			87ac0ddf52aa40dcb138117283c65a10
 System UUID:			0947489A-D2E7-416F-AA1A-517900E2DCB5
 Boot ID:			dbc1ab43-183d-475a-886c-d445fa7b41b4
 Kernel Version:		4.6.7-300.fc24.x86_64
 OS Image:			Fedora 24 (Twenty Four)
 Operating System:		linux
 Architecture:			amd64
 Container Runtime Version:	docker://1.11.2
 Kubelet Version:		v1.3.6
 Kube-Proxy Version:		v1.3.6
PodCIDR:			10.200.0.0/24
ExternalID:			worker1.example.com
Non-terminated Pods:		(1 in total)
  Namespace			Name						CPU Requests	CPU Limits	Memory Requests	Memory Limits
  ---------			----						------------	----------	---------------	-------------
  default			centos-multitool-3822887632-jeyhb		0 (0%)		0 (0%)		0 (0%)		0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted. More info: http://releases.k8s.io/HEAD/docs/user-guide/compute-resources.md)
  CPU Requests	CPU Limits	Memory Requests	Memory Limits
  ------------	----------	---------------	-------------
  0 (0%)	0 (0%)		0 (0%)		0 (0%)
No events.
[root@controller1 ~]#
``` 

Extract the PodCIDR information from the output above:
```
[root@controller1 ~]# kubectl describe node worker1 | grep PodCIDR
PodCIDR:			10.200.0.0/24
[root@controller1 ~]# 
```


We also know this:
```
[root@controller1 ~]# kubectl get nodes -o name
node/worker1.example.com
node/worker2.example.com
[root@controller1 ~]# 
```


```
[root@controller1 ~]# kubectl get nodes -o name | sed 's/^.*\///'
worker1.example.com
worker2.example.com
[root@controller1 ~]#
```

Ok, so we know what to do!

```
[root@controller1 ~]# NODE_LIST=$(kubectl get nodes -o name | sed 's/^.*\///')

[root@controller1 ~]# for node in $NODE_LIST; do echo ${node}; kubectl describe node ${node} | grep PodCIDR; echo "------------------"; done

[root@controller1 ~]# kubectl describe node worker1.example.com | grep PodCIDR| tr -d '[[:space:]]' | cut -d ':' -f2
10.200.0.0/24
[root@controller1 ~]# 
```

We also need the network address of the worker node:
```
[root@controller1 ~]# kubectl describe node worker1.example.com | grep Addresses| tr -d '[[:space:]]' | cut -d ':' -f 2 | cut -d ',' -f 1
10.240.0.31
[root@controller1 ~]# 
```



```
[root@controller1 ~]# for node in $NODE_LIST; do echo ${node}; echo -n "Network: " ; kubectl describe node ${node} | grep PodCIDR| tr -d '[[:space:]]' | cut -d ':' -f2; echo -n "Reachable through: "; kubectl describe node ${node} | grep Addresses| tr -d '[[:space:]]' | cut -d ':' -f 2 | cut -d ',' -f 1; echo "--------------------------------"; done
worker1.example.com
Network: 10.200.0.0/24
Reachable through: 10.240.0.31
--------------------------------
worker2.example.com
Network: 10.200.1.0/24
Reachable through: 10.240.0.32
--------------------------------
[root@controller1 ~]# 
```


We can use this information to add routes to our network router, which is my work computer in our case.

```
[root@kworkhorse ~]# route add -net 10.200.0.0 netmask 255.255.255.0 gw 10.240.0.31 
[root@kworkhorse ~]# route add -net 10.200.1.0 netmask 255.255.255.0 gw 10.240.0.32 
```

( I will automate this by making a script out of the above manual steps).


Here is how my routing table looks like on my work computer:
```
[root@kworkhorse ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.100.1   0.0.0.0         UG    600    0        0 wlp2s0
10.200.0.0      10.240.0.31     255.255.255.0   UG    0      0        0 virbr2
10.200.1.0      10.240.0.32     255.255.255.0   UG    0      0        0 virbr2
10.240.0.0      0.0.0.0         255.255.255.0   U     0      0        0 virbr2
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 virbr3
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-8b79f8723f87
192.168.100.0   0.0.0.0         255.255.255.0   U     600    0        0 wlp2s0
192.168.124.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
[root@kworkhorse ~]# 
```

## Moment of truth!
Now one pod should be able to ping the other pod running on the other worker node:
```
[root@controller1 ~]# kubectl exec centos-multitool-3822887632-6qbrh -it -- bash 

[root@centos-multitool-3822887632-6qbrh /]# ping -c 1 10.200.1.1 
PING 10.200.1.1 (10.200.1.1) 56(84) bytes of data.
64 bytes from 10.200.1.1: icmp_seq=1 ttl=64 time=0.268 ms

--- 10.200.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.268/0.268/0.268/0.000 ms


[root@centos-multitool-3822887632-6qbrh /]# ping -c 1 10.200.0.1 
PING 10.200.0.1 (10.200.0.1) 56(84) bytes of data.
64 bytes from 10.200.0.1: icmp_seq=1 ttl=62 time=4.57 ms

--- 10.200.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 4.570/4.570/4.570/0.000 ms


[root@centos-multitool-3822887632-6qbrh /]# ping -c 1 10.200.0.2 
PING 10.200.0.2 (10.200.0.2) 56(84) bytes of data.
64 bytes from 10.200.0.2: icmp_seq=1 ttl=61 time=0.586 ms

--- 10.200.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.586/0.586/0.586/0.000 ms
[root@centos-multitool-3822887632-6qbrh /]# 
```

Great! It works!
Configuring the Kubernetes Client - Remote Access

This is step 6 in Kelseys guide.

This step is not entirely necessary, as we can just login directly on one of the controller nodes, and can still manage the cluster. 

This is a (To do)

## Download and Install kubectl on your local work computer
Linux
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.3.6/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin
``` 

