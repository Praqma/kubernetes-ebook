# Chapter 11: Setup Load Balancer nodes - with Praqma Load Balancer

* Setup load balancer nodes
* Setup HA for the nodes.
* Deploy praqma load balancer and show how it works , etc.


This chapter deploys the praqma load balancer on the load balancer nodes. First we do it with NodePort, and then we do it with our load balancer.

How kubctl will access the cluster from outside, will be covered in next chapter.


# Smoke test - with NodePort

Kelsey likes to do this smoke test at this point, using the **NodePort** method. We can do this now, but what we are really interested in, is to be able to access the services using IP addresses and not using fancy ports. 

First, we do it the node port way. 

To begin with, we needs some pods running a web server. We already have two pods running centos-multitool which also contains (and runs) apache web server.

```
[root@controller1 ~]# kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP           NODE
centos-multitool-3822887632-6qbrh   1/1       Running   1          1d        10.200.1.4   worker2.example.com
centos-multitool-3822887632-jeyhb   1/1       Running   0          1d        10.200.0.2   worker1.example.com
[root@controller1 ~]# 
```
The deployment behind these pods is centos-multitool.

```
[root@controller1 ~]# kubectl get deployments -o wide
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
centos-multitool   2         2         2            2           1d
[root@controller1 ~]# 
```


If you don't have it running, or if you would like to run something else, such as a simple nginx web server, you can do that. Lets follow Kelsey's example:


```
[root@controller1 ~]# kubectl run nginx --image=nginx --port=80 --replicas=3
deployment "nginx" created
[root@controller1 ~]# 
```

```
[root@controller1 ~]# kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP           NODE
centos-multitool-3822887632-6qbrh   1/1       Running   1          1d        10.200.1.4   worker2.example.com
centos-multitool-3822887632-jeyhb   1/1       Running   0          1d        10.200.0.2   worker1.example.com
nginx-2032906785-a6pt5              1/1       Running   0          2m        10.200.0.4   worker1.example.com
nginx-2032906785-foq6g              1/1       Running   0          2m        10.200.1.6   worker2.example.com
nginx-2032906785-zbbkv              1/1       Running   0          2m        10.200.1.7   worker2.example.com
[root@controller1 ~]# 
```


Lets create a service out of this deployment:

```
[root@controller1 ~]# kubectl expose deployment nginx --type NodePort
service "nginx" exposed
[root@controller1 ~]# 
```
**Note:** At this point `--type=LoadBalancer` will not work because we did not configure a cloud provider when bootstrapping this cluster.


Extract the NodePort setup for this nginx service:
```
[root@controller1 ~]# NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')

[root@controller1 ~]# echo $NODE_PORT 
32133
[root@controller1 ~]# 
```

Lets try accessing this service using the port we have. We can use the IP address of any of the worker nodes to access this service using the NODE_PORT .

```
[root@controller1 ~]# curl http://10.240.0.31:32133
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@controller1 ~]# 
```

From the other worker node, this time using the worker node's DNS name:

```
[root@controller1 ~]# curl http://worker2.example.com:32133
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@controller1 ~]# 
```

So the node port method works!

Thanks to correct routing setup, I can also access the nginx web server directly by using the IP address of the pod directly from my controller node:

```
[root@controller1 ~]# curl 10.200.0.4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@controller1 ~]# 
```
----------

# Smoke test - Praqma Load Balancer:

(aka. The real deal!)

First we need to have the haproxy package installed on this VM. Also make sure that iptables service is disabled and SELINUX is also disabled. You need to install nmap as well; it is used by the script.

```
[root@lb ~]# yum -y install haproxy git nmap
```

If there are some pods already running in the cluster, then it is a good time to ping them to make sure that the load balancer is able to reach the pods. We started some pods in the above section, so we should be able to ping them. First we obtain endpoints of the nginx service from the controller node.

```
[root@controller1 ~]#kubectl get endpoints nginx
NAME      ENDPOINTS                                               AGE
nginx     10.200.0.4:80,10.200.0.5:80,10.200.0.6:80 + 7 more...   2d
[root@controller1 ~]# 
```

You can also use curl to get a list of IPs in json form and then filter it:

```
[root@controller1 ~]# curl -s http://localhost:8080/api/v1/namespaces/default/endpoints/nginx |  grep "ip"
          "ip": "10.200.0.4",
          "ip": "10.200.0.5",
          "ip": "10.200.0.6",
          "ip": "10.200.0.7",
          "ip": "10.200.0.8",
          "ip": "10.200.1.10",
          "ip": "10.200.1.6",
          "ip": "10.200.1.7",
          "ip": "10.200.1.8",
          "ip": "10.200.1.9",
[root@controller1 ~]# 
```



**Note:** jq can be used to parse json output! (More on this later. to do)
```
$ sudo yum -y install jq
```


We will use two ips from two different networks and see if we can ping them from our load balancer. If we are successful, it means our routing is setup correctly.

```
[root@lb ~]# ping -c 1 10.200.0.4 
PING 10.200.0.4 (10.200.0.4) 56(84) bytes of data.
64 bytes from 10.200.0.4: icmp_seq=1 ttl=63 time=0.960 ms

--- 10.200.0.4 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.960/0.960/0.960/0.000 ms


[root@lb ~]# ping -c 1 10.200.1.6
PING 10.200.1.6 (10.200.1.6) 56(84) bytes of data.
64 bytes from 10.200.1.6: icmp_seq=1 ttl=63 time=1.46 ms

--- 10.200.1.6 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.463/1.463/1.463/0.000 ms
[root@lb ~]# 
```

Clearly, we are able to ping pods from our load balancer. Good!

Create a combined certificate and then move certificates to /var/lib/kubernetes/. 
```
mkdir /var/lib/kubernetes/
cat /root/kubernetes.pem /root/kubernetes-key.pem > /root/kubernetes-combined.pem
mv /root/*.pem /var/lib/kubernetes/
```

Next, we need the load balancer script and config files. You can clone the entire LearnKubernetes repository somewhere on the load balancer's file system. (You need to have git on load balancer machine!)

```
[root@lb ~]# git clone https://github.com/Praqma/LearnKubernetes.git
[root@lb ~]# cd LearnKubernetes/kamran/LoadBalancer-Files/
[root@lb LoadBalancer-Files]# 
```


Next, we need to copy the loadbalancer.conf to /opt/ .

```
[root@lb LoadBalancer-Files]#  cp loadbalancer.conf /opt/
```

And copy loadbalancer.sh to /usr/local/bin/ :
```
[root@lb LoadBalancer-Files]# cp loadbalancer.sh.cidr /usr/local/bin/loadbalancer.sh
```


Now, edit the loadbalancer.conf file and adjust it as following:
```
[root@lb LoadBalancer-Files]# vi /opt/loadbalancer.conf 
 # This file contains the necessary information for loadbalancer script to work properly.
 # This IP / interface will never be shutdown.
 LB_PRIMARY_IP=10.240.0.200
 # LB_DATABASE=/opt/LoadBalancer.sqlite.db
 LB_LOG_FILE=/var/log/loadbalancer.log
 # IP Address of the Kubernetes master node.
 MASTER_IP=10.240.0.21
 # The user on master node, which is allowed to run the kubectl commands. This user needs to have the public RSA key from the root 
 #   user at load  balancer in it's authorized keys file. 
 MASTER_USER=root
 PRODUCTION_HAPROXY_CONFIG=/etc/haproxy/haproxy.cfg
``` 


Time to generate RSA key pair for user root at loadbalancer VM. Then, we will copy the public key of the RSA keypair to the authorized_keys file of root user on the controller nodes.

```
[root@lb LoadBalancer-Files]# ssh-keygen -t rsa -N ''
```

```
[root@lb LoadBalancer-Files]# ssh-copy-id root@controller1
The authenticity of host 'controller1 (10.240.0.21)' can't be established.
ECDSA key fingerprint is 84:5d:ae:17:17:07:06:46:b6:7d:69:2f:32:25:50:d0.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@controller1's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@controller1'"
and check to make sure that only the key(s) you wanted were added.

[root@lb LoadBalancer-Files]#
```

Verify that the passwordless login works from load balancer to the controller:

```
[root@lb LoadBalancer-Files]# ssh root@controller1 hostname
controller1.example.com
[root@lb LoadBalancer-Files]#
```


Now, make sure that the service you are interested in has an external IP specified in Kubernetes. If it does not exist, delete the service and recreate it.
```
[root@controller1 ~]# kubectl delete service nginx
service "nginx" deleted

[root@controller1 ~]# kubectl expose deployment nginx --external-ip=10.240.0.2
service "nginx" exposed
[root@controller1 ~]# 
```

Then, run the loadbalancer.sh program. First, in show mode and then in create mode.

**show mode:**
```
[root@lb LoadBalancer-Files]# loadbalancer.sh show

Beginning execution of main program - in show mode...

Showing status of service: haproxy
----------------------------------
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; disabled; vendor preset: disabled)
   Active: inactive (dead)


Starting Sanity checks ...

Checking if kubernetes master 10.240.0.21 is reachable over SSH ...Yes!
Success connecting to Kubernetes master 10.240.0.21 on port 22.

Running command 'uptime' as user root on Kubernetes Master 10.240.0.21.

 13:58:44 up  2:42,  1 user,  load average: 0.00, 0.00, 0.00

Running command 'kubectl get cs' as user root on Kubernetes Master 10.240.0.21.

NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   

Sanity checks completed successfully!

Following services were found with external IPs - on Kubernetes master ...
====================================================================================================
default       nginx        10.32.0.230   <nodes>       80/TCP          3d

Here are Top 10 IPs from the available pool:
--------------------------------------------
10.240.0.2
10.240.0.3
10.240.0.4
10.240.0.5
10.240.0.6
10.240.0.7
10.240.0.8
10.240.0.9
10.240.0.10
10.240.0.13


oooooooooooooooooooo Show load balancer configuration and status. - Operation completed. oooooooooooooooooooo
Logs are in: /var/log/loadbalancer.log

TODO:
-----
* - Use [root@loadbalancer ~]# curl -k -s -u vagrant:vagrant  https://10.245.1.2/api/v1/namespaces/default/endpoints/apache | grep ip
    The above is better to use instead of getting endpoints from kubectl, because kubectl only shows 2-3 endpoints and says +XX more...
* - Create multiple listen sections depending on the ports of a service. such as 80, 443 for web servers. This may be tricky. Or there can be two bind commands in one listen directive/section.
* - Use local kubectl instead of SSHing into Master

[root@lb LoadBalancer-Files]#
```



**create mode:**
```
[root@lb LoadBalancer-Files]# loadbalancer.sh create

Beginning execution of main program - in create mode...

Acquiring program lock with PID: 27196 , in lock file: /var/lock/loadbalancer

Starting Sanity checks ...

Checking if kubernetes master 10.240.0.21 is reachable over SSH ...Yes!
Success connecting to Kubernetes master 10.240.0.21 on port 22.

Running command 'uptime' as user root on Kubernetes Master 10.240.0.21.

 14:04:56 up  2:48,  1 user,  load average: 0.00, 0.01, 0.00

Running command 'kubectl get cs' as user root on Kubernetes Master 10.240.0.21.

NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   

Sanity checks completed successfully!

Following services were found with external IPs - on Kubernetes master ...
====================================================================================================
default       nginx        10.32.0.237   10.240.0.2    80/TCP          42s
-----> Creating HA proxy section: default-nginx-80
listen default-nginx-80
        bind 10.240.0.2:80
        server pod-1 10.200.0.4:80 check
        server pod-2 10.200.0.5:80 check
        server pod-3 10.200.0.6:80 check
        server pod-4 10.200.0.7:80 check
        server pod-5 10.200.0.8:80 check
        server pod-6 10.200.1.10:80 check
        server pod-7 10.200.1.6:80 check
        server pod-8 10.200.1.7:80 check
        server pod-9 10.200.1.8:80 check
        server pod-10 10.200.1.9:80 check

Comparing generated (haproxy) config with running config ...

20c20
<         bind 10.240.0.2:80
---
>         bind <nodes>:80

The generated and running (haproxy) config files differ. Replacing the running haproxy file with the newly generated one, and reloading haproxy service ...

Checking/managing HA Proxy service ...
HA Proxy process was not running on this system. Starting the service ... Successful.

Aligning IP addresses on eth0...
Adding IP address 10.240.0.2 to the interface eth0.

Here is the final status of the network interface eth0 :
---------------------------------------------------------------------------------------
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:36:27:7d brd ff:ff:ff:ff:ff:ff
    inet 10.240.0.200/24 brd 10.240.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.240.0.2/24 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe36:277d/64 scope link 
       valid_lft forever preferred_lft forever
---------------------------------------------------------------------------------------

Releasing progarm lock: /var/lock/loadbalancer

oooooooooooooooooooo Create haproxy configuration. - Operation completed. oooooooooooooooooooo
Logs are in: /var/log/loadbalancer.log

TODO:
-----
* - Use [root@loadbalancer ~]# curl -k -s -u vagrant:vagrant  https://10.245.1.2/api/v1/namespaces/default/endpoints/apache | grep ip
    The above is better to use instead of getting endpoints from kubectl, because kubectl only shows 2-3 endpoints and says +XX more...
* - Create multiple listen sections depending on the ports of a service. such as 80, 443 for web servers. This may be tricky. Or there can be two bind commands in one listen directive/section.
* - Use local kubectl instead of SSHing into Master

[root@lb LoadBalancer-Files]# 
```

After running the loadbalancer in the create mode, the resultant `/etc/haproxy/haproxy.conf` file looks like this:

```
[root@lb LoadBalancer-Files]# cat /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

listen default-nginx-80
        bind 10.240.0.2:80
        server pod-1 10.200.0.4:80 check
        server pod-2 10.200.0.5:80 check
        server pod-3 10.200.0.6:80 check
        server pod-4 10.200.0.7:80 check
        server pod-5 10.200.0.8:80 check
        server pod-6 10.200.1.10:80 check
        server pod-7 10.200.1.6:80 check
        server pod-8 10.200.1.7:80 check
        server pod-9 10.200.1.8:80 check
        server pod-10 10.200.1.9:80 check
[root@lb LoadBalancer-Files]#
```


You can also re-run the loadbalancer in the **show** mode, just to be sure:

```
[root@lb LoadBalancer-Files]# loadbalancer.sh show

Beginning execution of main program - in show mode...

Showing status of service: haproxy
----------------------------------
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; disabled; vendor preset: disabled)
   Active: inactive (dead)

Sep 19 14:00:13 lb.example.com haproxy-systemd-wrapper[27151]: haproxy-systemd-wrapper: executing /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
Sep 19 14:00:14 lb.example.com haproxy-systemd-wrapper[27151]: [ALERT] 262/140013 (27158) : parsing [/etc/haproxy/haproxy.cfg:20] : 'bind' : invalid address: '<nodes>' in '<nodes>:80'
Sep 19 14:00:14 lb.example.com haproxy-systemd-wrapper[27151]: [ALERT] 262/140013 (27158) : Error(s) found in configuration file : /etc/haproxy/haproxy.cfg
Sep 19 14:00:14 lb.example.com haproxy-systemd-wrapper[27151]: [ALERT] 262/140014 (27158) : Fatal errors found in configuration.
Sep 19 14:00:14 lb.example.com haproxy-systemd-wrapper[27151]: haproxy-systemd-wrapper: exit, haproxy RC=256
Sep 19 14:04:57 lb.example.com systemd[1]: Started HAProxy Load Balancer.
Sep 19 14:04:57 lb.example.com systemd[1]: Starting HAProxy Load Balancer...
Sep 19 14:04:57 lb.example.com haproxy-systemd-wrapper[27252]: haproxy-systemd-wrapper: executing /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
Sep 19 14:04:57 lb.example.com haproxy-systemd-wrapper[27252]: [ALERT] 262/140457 (27254) : Starting proxy default-nginx-80: cannot bind socket [10.240.0.2:80]
Sep 19 14:04:57 lb.example.com haproxy-systemd-wrapper[27252]: haproxy-systemd-wrapper: exit, haproxy RC=256


Starting Sanity checks ...

Checking if kubernetes master 10.240.0.21 is reachable over SSH ...Yes!
Success connecting to Kubernetes master 10.240.0.21 on port 22.

Running command 'uptime' as user root on Kubernetes Master 10.240.0.21.

 14:10:54 up  2:54,  1 user,  load average: 0.02, 0.03, 0.00

Running command 'kubectl get cs' as user root on Kubernetes Master 10.240.0.21.

NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   

Sanity checks completed successfully!

Following services were found with external IPs - on Kubernetes master ...
====================================================================================================
default       nginx        10.32.0.237   10.240.0.2    80/TCP          6m

Here are Top 10 IPs from the available pool:
--------------------------------------------
10.240.0.3
10.240.0.4
10.240.0.5
10.240.0.6
10.240.0.7
10.240.0.8
10.240.0.9
10.240.0.10
10.240.0.13
10.240.0.14


oooooooooooooooooooo Show load balancer configuration and status. - Operation completed. oooooooooooooooooooo
Logs are in: /var/log/loadbalancer.log

TODO:
-----
* - Use [root@loadbalancer ~]# curl -k -s -u vagrant:vagrant  https://10.245.1.2/api/v1/namespaces/default/endpoints/apache | grep ip
    The above is better to use instead of getting endpoints from kubectl, because kubectl only shows 2-3 endpoints and says +XX more...
* - Create multiple listen sections depending on the ports of a service. such as 80, 443 for web servers. This may be tricky. Or there can be two bind commands in one listen directive/section.
* - Use local kubectl instead of SSHing into Master

[root@lb LoadBalancer-Files]#
```


## Moment of truth:
Access the service from some other machine, such as my work computer:

```
[kamran@kworkhorse ~]$ curl http://10.240.0.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[kamran@kworkhorse ~]$ 
```

**It works!** Hurray!




