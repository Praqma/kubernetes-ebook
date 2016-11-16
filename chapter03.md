# Infrastructure Setup

* Here we provision our machines and setup networking.

Note that I am doing this provisioning on my work computer, which is Fedora 23 64 bit, and I will use the built-in Libvirt/KVM for virtualization. You can use any other virtualization software, or you can use real hardware!

First, setting up the new infrastructure network in KVM.

If you use Libvirt then you probably know that it sets up a virtual network `192.168.124.0/24` by default. But, we'd prefer to follow Kelsey's guide, so my infrastructure network is going to be `10.240.0.0/24` and I will just create a new virtual network (10.240.0.0/24) on my work computer.

## Setup new virtual network in KVM:

Start Virtual Machine Manager and go to "Edit"->"Connection Details"->"Virtual Networks". Then follow the steps shown below to create a new virtual network and name it **Kubernetes**. Note that this is a NAT network connected to any/all physical devices on my computer, so it will work whether I am connected to a wired or a wireless network.

![images/libvirt-new-virtual-network-1.png](images/libvirt-new-virtual-network-1.png)
![images/libvirt-new-virtual-network-2.png](images/libvirt-new-virtual-network-2.png)
![images/libvirt-new-virtual-network-3.png](images/libvirt-new-virtual-network-3.png)
![images/libvirt-new-virtual-network-4.png](images/libvirt-new-virtual-network-4.png)
![images/libvirt-new-virtual-network-5.png](images/libvirt-new-virtual-network-5.png)
![images/libvirt-new-virtual-network-6.png](images/libvirt-new-virtual-network-6.png)

The wizard will create an internal DNS setup (automatically) for example.com.

Now that we have the network out of the way let's decide on the size of these virtual machines and what IPs will be assigned to them. At the time of VM creation we will attach them (VMs) to this new virtual network.


## IP addresses and VM provisioning:

Here are the IP addresses of the VMs we are about to create:

* etcd1		10.240.0.11/24
* etcd2		10.240.0.12/24
* etcd3		10.240.0.13/24
* controller1	10.240.0.21/24
* controller2	10.240.0.22/24
* worker1	10.240.0.31/24
* worker2	10.240.0.32/24
* lb1		10.240.0.41/24
* lb2		10.240.0.42/24

**Notes:**
* There will be (additional) floating IP/VIP for controllers which will be: `10.240.0.20` 
* There will be (additional) floating IP/VIP for load balancers which will be: `10.240.0.40` 
* If you decide to use HAProxy to provide HA for controller nodes then you can use the the load balancer's VIP (for port 6443) instead of having a dedicated (floating/V) IP for the control plane.


**More Notes:** 
* Kelsey's Kubernetes guide (the one this book uses as a reference) starts the node numbering from 0. We will start them from 1 for ease of understanding.
* The FQDN of each host is `*hostname*.example.com` 
* The nodes have only one user, **root** ; with a password: **redhat** .
* I used Libvirt's GUI interface (virt-manager) to create these VMs, but you can automate this by using CLI commands.


## Screenshots from actual installation

![images/libvirt-new-vm-01.png](images/libvirt-new-vm-01.png)
![images/libvirt-new-vm-02.png](images/libvirt-new-vm-02.png)
![images/libvirt-new-vm-03.png](images/libvirt-new-vm-03.png)
![images/libvirt-new-vm-04.png](images/libvirt-new-vm-04.png)
![images/libvirt-new-vm-05.png](images/libvirt-new-vm-05.png)
![images/libvirt-new-vm-06.png](images/libvirt-new-vm-06.png)
![images/libvirt-new-vm-07.png](images/libvirt-new-vm-07.png)
![images/libvirt-new-vm-08.png](images/libvirt-new-vm-08.png)
![images/libvirt-new-vm-09.png](images/libvirt-new-vm-09.png)

**Notes:** 
* One of the installation screens shows OS as Fedora 22 (Step 5 of 5), but it is actually Fedora 24. Libvirt is not updated yet to recognize Fedora 24 ISO images.
* The last screenshot is from the installation of the second etcd node (etcd2). (todo: may be we can get a new screenshot?)


## Actual resource utilization from a running Kubernetes cluster:
To give you an idea of how much RAM and other resources are actually used by each type of node we have provided some details from the nodes of a similar Kubernetes cluster. It should help you size your VMs accordingly. Though for production setups you definitely want more resources for each component. 


### etcd:
Looks like etcd uses very little RAM! (about 88 MB!). I already gave this VM the minimum of 512 MB of RAM.

```
[root@etcd1 ~]# ps aux | grep etcd
root       660  0.2  9.2 10569580 46508 ?      Ssl  Sep14  16:31 /usr/bin/etcd --name etcd1 --cert-file=/etc/etcd/kubernetes.pem --key-file=/etc/etcd/kubernetes-key.pem --peer-cert-file=/etc/etcd/kubernetes.pem --peer-key-file=/etc/etcd/kubernetes-key.pem --trusted-ca-file=/etc/etcd/ca.pem --peer-trusted-ca-file=/etc/etcd/ca.pem --initial-advertise-peer-urls https://10.240.0.11:2380 --listen-peer-urls https://10.240.0.11:2380 --listen-client-urls https://10.240.0.11:2379,http://127.0.0.1:2379 --advertise-client-urls https://10.240.0.11:2379 --initial-cluster-token etcd-cluster-0 --initial-cluster etcd1=https://10.240.0.11:2380,etcd2=https://10.240.0.12:2380 --initial-cluster-state new --data-dir=/var/lib/etcd
[root@etcd1 ~]#


[root@etcd1 ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:            488          88         122           0         278         359
Swap:           511           7         504
[root@etcd1 ~]# 
```

### Controller (aka master):
Looks like the controller nodes use only 167 MB RAM and can function properly while running on 512 MB RAM. Of course, this may become insufficient as your cluster becomes larger and you create more pods. (todo: not tested though!)
```
[root@controller1 ~]# ps aux | grep kube
root      8251  0.6 11.4 147236 116540 ?       Ssl  09:12   0:42 /usr/bin/kube-apiserver --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota --advertise-address=10.240.0.21 --allow-privileged=true --apiserver-count=2 --authorization-mode=ABAC --authorization-policy-file=/var/lib/kubernetes/authorization-policy.jsonl --bind-address=0.0.0.0 --enable-swagger-ui=true --etcd-cafile=/var/lib/kubernetes/ca.pem --insecure-bind-address=0.0.0.0 --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem --etcd-servers=https://10.240.0.11:2379,https://10.240.0.12:2379 --service-account-key-file=/var/lib/kubernetes/kubernetes-key.pem --service-cluster-ip-range=10.32.0.0/24 --service-node-port-range=30000-32767 --tls-cert-file=/var/lib/kubernetes/kubernetes.pem --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem --token-auth-file=/var/lib/kubernetes/token.csv --v=2

root      8292  0.2  5.1  80756 51988 ?        Ssl  09:12   0:15 /usr/bin/kube-controller-manager --allocate-node-cidrs=true --cluster-cidr=10.200.0.0/16 --cluster-name=kubernetes --leader-elect=true --master=http://10.240.0.21:8080 --root-ca-file=/var/lib/kubernetes/ca.pem --service-account-private-key-file=/var/lib/kubernetes/kubernetes-key.pem --service-cluster-ip-range=10.32.0.0/24 --v=2

root      8321  0.0  2.9  46844 29844 ?        Ssl  09:12   0:04 /usr/bin/kube-scheduler --leader-elect=true --master=http://10.240.0.21:8080 --v=2
[root@controller1 ~]#


[root@controller1 ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:            992         167          99           0         726         644
Swap:           511           0         511
[root@controller1 ~]# 

```


### Worker:
The worker nodes need the most amount of RAM because they run your containers. Even though we see only 168 MB of utilization this capacity will be used up as soon as there are a few pods running on the cluster. Worker nodes will be the beefiest nodes of your cluster.

```
[root@worker1 ~]# ps aux | grep kube
root     13743  0.0  1.8  43744 28200 ?        Ssl  Sep16   0:15 /kube-dns --domain=cluster.local --dns-port=10053
root     13942  0.0  0.4  14124  7320 ?        Ssl  Sep16   0:07 /exechealthz -cmd=nslookup kubernetes.default.svc.cluster.local 127.0.0.1 >/dev/null && nslookup kubernetes.default.svc.cluster.local 127.0.0.1:10053 >/dev/null -port=8080 -quiet
root     22925  0.0  0.0 117148   980 pts/0    S+   11:10   0:00 grep --color=auto kube
root     27240  0.5  4.0 401936 61372 ?        Ssl  09:14   0:36 /usr/bin/kubelet --allow-privileged=true --api-servers=https://10.240.0.21:6443,https://10.240.0.22:6443 --cloud-provider= --cluster-dns=10.32.0.10 --cluster-domain=cluster.local --configure-cbr0=true --container-runtime=docker --docker=unix:///var/run/docker.sock --network-plugin=kubenet --kubeconfig=/var/lib/kubelet/kubeconfig --reconcile-cidr=true --serialize-image-pulls=false --tls-cert-file=/var/lib/kubernetes/kubernetes.pem --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem --v=2
root     27314  0.7  1.8  41536 28072 ?        Ssl  09:14   0:50 /usr/bin/kube-proxy --master=https://10.240.0.21:6443 --kubeconfig=/var/lib/kubelet/kubeconfig --proxy-mode=iptables --v=2
[root@worker1 ~]# 


[root@worker1 ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           1496         168         139           0        1188        1104
Swap:          1023           0        1023
[root@worker1 ~]# 
```

## Prepare the OS on each node:

To create a uniform way of accessing the cluster update the `/etc/hosts` file on your work computer as per the design of your cluster:

```
[kamran@kworkhorse ~]$ sudo vi /etc/hosts
127.0.0.1               localhost.localdomain localhost
10.240.0.11     etcd1.example.com       etcd1
10.240.0.12     etcd2.example.com       etcd2
10.240.0.13     etcd3.example.com       etcd3
10.240.0.20     controller.example.com 	controller 	controller-vip
10.240.0.21     controller1.example.com controller1
10.240.0.22     controller2.example.com controller2
10.240.0.31     worker1.example.com     worker1
10.240.0.32     worker2.example.com     worker2
10.240.0.40	lb.example.com		lb		lb-vip
10.240.0.41	lb1.example.com		lb1
10.240.0.42	lb2.example.com		lb2
```

We will copy this file to all the nodes in just a moment. First, we create RSA-2 keypair for SSH connections and copy our key to all the nodes. This means we can ssh into them without requiring a password.

If you do not have a RSA keypair you can generate it by using the following command on your work computer:

```
ssh-keygen -t rsa
```
**Note:** It is recommended that you have a passphrase assigned to your key. If it is stolen the key will be useless without a passphrase, so protecting the key in this way is a good idea.


Assuming you have generated a rsa keypair the command to copy the public part of the keypair to the nodes will be:

```
ssh-copy-id root@<NodeName|NodeIP>
```

Sample run:
```
[kamran@kworkhorse ~]$ ssh-copy-id root@etcd1
The authenticity of host 'etcd1 (10.240.0.11)' can't be established.
ECDSA key fingerprint is SHA256:FUMy5JNZnaLXhkW3Y0/WlXzQQrjU5IZ8LMOcgBTOiLU.
ECDSA key fingerprint is MD5:5e:9b:2d:ae:8e:16:7a:ee:ca:de:de:da:9a:04:19:8b.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 2 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@etcd1's password: 

Number of key(s) added: 2

Now try logging into the machine, with:   "ssh 'root@etcd1'"
and check to make sure that only the key(s) you wanted were added.

[kamran@kworkhorse ~]$ 
```
**Note:** You cannot run a loop for copying the keys because each time it will ask for confirmation of the RSA fingerprint of the target node, and also the root user password for the first time. So this is a manual step!


After setting up your keys in all the nodes you should be able to execute commands on the nodes using ssh:

```
[kamran@kworkhorse ~]$ ssh root@etcd1 uptime
 13:16:27 up  1:29,  1 user,  load average: 0.08, 0.03, 0.04
[kamran@kworkhorse ~]$ 
```


Now, copy the `/etc/hosts` file to all nodes:

```
for node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2} ; do scp /etc/hosts root@${node}:/etc/hosts ; done
``` 


After all VMs are created we update OS on them using `yum -y update`, disable firewalld service, and also disable SELINUX in `/etc/selinux/config` file and reboot all nodes for these changes to take effect. 



Disable firewall on all nodes:

Note: For some strange reason disabling `firewalld` service did not work. I had to actually remove the `firewalld` package from all of the nodes.
```
for node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2} ; do ssh root@${node} "yum -y remove firewalld" ; done
```


Disable SELINUX on all nodes:

```
for node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2} ; do ssh root@${node} "echo 'SELINUX=disabled' > /etc/selinux/config" ; done
```
**Note:** Setting the `/etc/selinux/config` file to contain only a single line saying `SELINUX=disabled` is enough to disable SELINUX at the next system boot.

OS update on all nodes, and reboot:
```
for node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2} ; do ssh root@${node} "yum -y update && reboot" ; done
```

After all nodes are rebooted verify that SELINUX is disabled:

```
for i in node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2} ; do ssh root@${i} "hostname; getenforce" ; done
```

Expected output from the above command:
```
[kamran@kworkhorse ~]$ for i in node in etcd{1,2,3} controller{1,2} worker{1,2} lb{1,2} ; do ssh root@${i} "hostname; getenforce"; done
etcd1.example.com
Disabled
etcd2.example.com
Disabled
etcd3.example.com
Disabled
controller1.example.com
Disabled
controller2.example.com
Disabled
worker1.example.com
Disabled
worker2.example.com
Disabled
lb1.example.com
Disabled
lb2.example.com
Disabled
[kamran@kworkhorse ~]$ 
```


# Conclusion:
In this chapter we provisioned a fresh network in Libvirt and provisioned our VMs. In the next chapter we are going to create SSL certificates which will be used by various components of the cluster. 

