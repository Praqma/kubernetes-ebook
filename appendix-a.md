# Appendix A - DNS  

I used KVM/Libvirt to create my VMs for this lab. Libvirt uses DNSMASQ, which uses /etc/hosts for DNS records, and forwards them to upstream DNS server if the host/ IP address is not found in /etc/hosts on the virtualization server. It is important that the hostnames are resolved to correct IP addresses. I noticed that even though I had the correct IP address / hosts mapping in /etc/hosts in my physical server (KVM host),the names of the hosts were not resolving correctly from the VMs.

First, here is the `/etc/hosts` file from my physical server:

```
[root@kworkhorse ~]# cat /etc/hosts
127.0.0.1		localhost.localdomain localhost
10.240.0.11	etcd1.example.com	etcd1
10.240.0.12	etcd2.example.com	etcd2
10.240.0.21	controller1.example.com	controller1
10.240.0.22	controller2.example.com	controller2
10.240.0.31	worker1.example.com	worker1
10.240.0.32	worker2.example.com	worker2
```


When I tried to resolve the names from a VM, it did not work:

```
 [root@worker1 ~]# dig worker1.example.com

 ;; QUESTION SECTION:
 ;worker1.example.com.		IN	A

 ;; ANSWER SECTION:
 worker1.example.com.	0	IN	A	52.59.239.224

 ;; Query time: 0 msec
 ;; SERVER: 10.240.0.1#53(10.240.0.1)
 ;; WHEN: Wed Sep 14 11:40:14 CEST 2016
 ;; MSG SIZE  rcvd: 64

 [root@worker1 ~]#
```


This means I should restart the dnsmasq service on the physical server:

```
[root@kworkhorse ~]# service dnsmasq stop
Redirecting to /bin/systemctl stop  dnsmasq.service
[root@kworkhorse ~]#
```

Then I start it again:
```
[root@kworkhorse ~]# service dnsmasq start
Redirecting to /bin/systemctl start  dnsmasq.service
[root@kworkhorse ~]# 
```

But it failed to start:
```
[root@kworkhorse ~]# service dnsmasq status
Redirecting to /bin/systemctl status  dnsmasq.service
● dnsmasq.service - DNS caching server.
   Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2016-09-14 11:43:12 CEST; 5s ago
  Process: 10029 ExecStart=/usr/sbin/dnsmasq -k (code=exited, status=2)
 Main PID: 10029 (code=exited, status=2)

Sep 14 11:43:12 kworkhorse systemd[1]: Started DNS caching server..
Sep 14 11:43:12 kworkhorse systemd[1]: Starting DNS caching server....
Sep 14 11:43:12 kworkhorse dnsmasq[10029]: dnsmasq: failed to create listening socket for port 53: Address already in use
Sep 14 11:43:12 kworkhorse dnsmasq[10029]: failed to create listening socket for port 53: Address already in use
Sep 14 11:43:12 kworkhorse dnsmasq[10029]: FAILED to start up
Sep 14 11:43:12 kworkhorse systemd[1]: dnsmasq.service: Main process exited, code=exited, status=2/INVALIDARGUMENT
Sep 14 11:43:12 kworkhorse systemd[1]: dnsmasq.service: Unit entered failed state.
Sep 14 11:43:12 kworkhorse systemd[1]: dnsmasq.service: Failed with result 'exit-code'.
[root@kworkhorse ~]# 
```

This is because all the DNSMASQ processes did not exit.
```
[root@kworkhorse ~]# netstat -ntlp 
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:43873         0.0.0.0:*               LISTEN      3573/chrome         
tcp        0      0 127.0.0.1:56133         0.0.0.0:*               LISTEN      24333/GoogleTalkPlu 
tcp        0      0 127.0.0.1:5900          0.0.0.0:*               LISTEN      8379/qemu-system-x8 
tcp        0      0 127.0.0.1:5901          0.0.0.0:*               LISTEN      9990/qemu-system-x8 
tcp        0      0 127.0.0.1:5902          0.0.0.0:*               LISTEN      11664/qemu-system-x 
tcp        0      0 127.0.0.1:5903          0.0.0.0:*               LISTEN      13021/qemu-system-x 
tcp        0      0 127.0.0.1:5904          0.0.0.0:*               LISTEN      14446/qemu-system-x 
tcp        0      0 127.0.0.1:5905          0.0.0.0:*               LISTEN      15613/qemu-system-x 
tcp        0      0 127.0.0.1:5939          0.0.0.0:*               LISTEN      1265/teamviewerd    
tcp        0      0 127.0.0.1:60117         0.0.0.0:*               LISTEN      24333/GoogleTalkPlu 
tcp        0      0 10.240.0.1:53           0.0.0.0:*               LISTEN      6410/dnsmasq        
tcp        0      0 172.16.0.1:53           0.0.0.0:*               LISTEN      1543/dnsmasq        
tcp        0      0 192.168.124.1:53        0.0.0.0:*               LISTEN      1442/dnsmasq        
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1240/sshd           
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      2479/cupsd          
tcp6       0      0 :::22                   :::*                    LISTEN      1240/sshd           
tcp6       0      0 ::1:631                 :::*                    LISTEN      2479/cupsd          
[root@kworkhorse ~]#
```


So I just `killall` all the dnsmasq processes on the physical server, and started the service again. Which resulted in correct name resolution on the nodes:

```
[root@kworkhorse ~]# killall dnsmasq
[root@kworkhorse ~]#
```


```
[root@kworkhorse ~]# service dnsmasq start
Redirecting to /bin/systemctl start  dnsmasq.service
```

```
[root@kworkhorse ~]# service dnsmasq status
Redirecting to /bin/systemctl status  dnsmasq.service
● dnsmasq.service - DNS caching server.
   Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2016-09-14 11:43:50 CEST; 2s ago
 Main PID: 10765 (dnsmasq)
   Memory: 600.0K
      CPU: 3ms
   CGroup: /system.slice/dnsmasq.service
           └─10765 /usr/sbin/dnsmasq -k

Sep 14 11:43:50 kworkhorse systemd[1]: Started DNS caching server..
Sep 14 11:43:50 kworkhorse systemd[1]: Starting DNS caching server....
Sep 14 11:43:50 kworkhorse dnsmasq[10765]: started, version 2.76 cachesize 150
Sep 14 11:43:50 kworkhorse dnsmasq[10765]: compile time options: IPv6 GNU-getopt DBus no-i18n IDN DHCP DHCPv6 no-Lua TFTP no-conntrac... inotify
Sep 14 11:43:50 kworkhorse dnsmasq[10765]: reading /etc/resolv.conf
Sep 14 11:43:50 kworkhorse dnsmasq[10765]: using nameserver 192.168.100.1#53
Sep 14 11:43:50 kworkhorse dnsmasq[10765]: using nameserver fe80::1%wlp2s0#53
Sep 14 11:43:50 kworkhorse dnsmasq[10765]: read /etc/hosts - 10 addresses
Hint: Some lines were ellipsized, use -l to show in full.
[root@kworkhorse ~]# 
```


Correct name resolution from the VM:
```
[root@worker1 ~]# dig worker1.example.com

 ;; QUESTION SECTION:
 ;worker1.example.com.		IN	A

 ;; ANSWER SECTION:
 worker1.example.com.	0	IN	A	10.240.0.31

 ;; Query time: 3 msec
 ;; SERVER: 10.240.0.1#53(10.240.0.1)
 ;; WHEN: Wed Sep 14 11:56:12 CEST 2016
 ;; MSG SIZE  rcvd: 64

 [root@worker1 ~]#
``` 

