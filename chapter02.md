# Chapter 2: Infrastructure design and provisioning

* Write about what type of hardware is needed. If not physical hardware, then what size of VMs are needed. etc.
* Discuss what type of network technologies are we going to use. Such as flannel or CIDR, etc.
* This will be a relatively short chapter.

In chapter 1 we explored what Kubernetes is, what pods are and so on. So far so good, right? But, now that we know all that what we really want to do is create our own Kubernetes cluster. How do we design it? How do we set it up? 

This chapter will discuss how to design our infrastructure. This book is about Kubernetes on Bare Metal hardware, but what if you don't have the necessary amount of hardware? The most obvious solution is to use virtual machines! Of-course! And maybe we can use cloud providers too. However, we have to remember, every cloud provider has its own way of networking i.e. its own way of managing how machines talk to each other, and with the rest of the world. So, we decided to use simple VMs running on a simple (but very powerful) hypervisor - Libvirt/KVM. You can also use VirtualBox or VMware, or HyperV too. These VMs will be just as good as Bare Metal. We're using no fancy networks, firewalls, permission groups, or other access controls. It is just a hypervisor running on our work computer with a single virtual network. This makes it very easy to learn and experiment with Kubernetes. It is still possible to run this setup on AWS, GCE, or any other cloud provider of your choice though. 

## How many VMs and what size?

As we know from [chapter 1](chapter01.md), a Kubernetes cluster has three main components. These three components are:

* Etcd 
* Control plane (comprising of: API Server, Scheduler, Controller Manager) 
* Worker(s) 

For experimentation and learning a simple three node Kubernetes cluster is good enough. It is technically possible to have all these components on a single machine - a minimum of three nodes are recommended for smooth functionality and clarity. So, in a simple three node Kubernetes cluster you will have an etcd node holding all the cluster meta data, a controller node acting as a control plane talking to the etcd node, and one worker node which runs the pods. But, what if the worker node dies? This is the very reason we need to have at least two worker nodes in a Kubernetes cluster. If one node dies the Kubernetes scheduler can re-schedule the pods to the surviving node. That is effectively the main point of the Kubernetes project, otherwise a plain Docker setup could do pretty much the same thing! 

## What about reaching the services on the pods from outside the cluster?
To make sure that the incoming traffic can reach the pods we need an additional component called a load balancer (more about this later), or we can use NodePort (more about this too - I promise!). When you buy a camera an important component is usually missing - the **tripod** :). If you wan't to take great, professional looking pictures this is an additional purchase you'll need to make when you're buying a camera. Similarly, you need to setup/add a load balancer when you setup a Kubernetes cluster because, like the tripod, it is a component that doesn't come with the Kubernetes software (at the moment).


## Do we need high availability?
Absolutely! Especially if you want your cluster for more than just learning! 

Even if you use a dedicated node for each component there would still be no high availability to safeguard against failure of "non-worker" components of the cluster, such as etcd node and controller node. If our pods survive a failing node the cluster itself cannot survive if the controller node fails or if the etcd node fails. So, we need to have high availability for these components too. 

To have high availability for non-worker components we need to use some sort of (external) clustering solution. What we do is setup three nodes for etcd which can form its own individual/independent cluster. Then we choose two nodes for the controller instead of just one and use LVS/IPVS to make it a small cluster too. This makes is quite resilient to failures! We have a load balancer, but that will become a single point of failure in our cluster if we do not protect it against hardware failures. So, we also need to setup at least two nodes for the load balancer using LVS/IPVS HA technologies. We now have the required number of nodes as outlined below:


* 3 x Etcd nodes -		Each with: 0.5 GB RAM, 4 GB disk 
* 2 x Control plane nodes -	Each with: 0.5 GB RAM, 4 GB disk 
* 2 x Worker nodes - 		Each with: 1.5 GB RAM, 20 GB disk
* 2 x Load Balancer nodes -	Each with: 0.5 GB RAM, 4 GB disk

That makes a total of nine (9) nodes.


Though not abolsulutely necessary we can also use a shared storage solution to demonstrate the concept of network mounted volumes (todo - ????) . We *can* use the Load balancer nodes to setup NFS and make it Highly available.

**Note about NodePort:**
In Kubernetes, `NodePort` is way to expose a service to the outside world - when you specify it in the `kubectl expose <service> --type=NodePort` (todo - confirm / verify). What this does is that it allows you to expose a service on the worker node, just like you can do the port mapping on a simple docker host. This makes the service available through all the worker nodes, because node port works it's magic by setting up certain iptables rules on all worker nodes. So no matter which port you land on (from outside the cluster), the service responds to you. However, to do this, you need to have the external DNS pointing to the worker nodes. e.g. www.example.com can have two A type addresses in DNS pointing to the IP addresses of two worker nodes. In case a node fails, DNS will do it's own DNS round robin. This may work, but DNS round robin does not work well with sticky sessions. Also, managing services through ports is kind of difficult to manage / keep track of. Plus it is totally uncool! (todo: editor can remove this if unhappy :(  ).


## The choice of HA technology for etcd, controllers and load balancers:

We selected three nodes for the etcd service. First, because it stores the meta data about our cluster, which is most important. Secondly etcd nodes can form their own individual/independent cluster. To satisfy the needs of a quorum, we decided to use three nodes for etcd cluster. This means failure of one node does not affect the etcd operation. (todo: rewrite:) In order for etcd cluster to be healthy, it needs to have quorum. i.e. alive nodes being more than half of the size of the total number of nodes of etcd cluter. So having a total of two nodes is not helpful. When one fails, the etcd custer will instantly become unhealthy. This is what we do not want to happen in a production cluster.

We know that all Kubernetes components watch the API server, running on controller node. So making sure that control plane remains available at all times. Unfortunately, controller nodes are not cluster aware. So we need to provide some sort of HA for them. This high availability can be provided in two ways. 

* By setting up a proxy/load balancer (using HAProxy on our load balancer). This would mean that the load balancer has to be the first component to come alive when the cluster boots up. So cluster has dependency on the load balancer nodes. This is easier to setup but has dependency problem.
* By setting up IPVS/LVS (using heartbeat, etc) on the controller nodes themselves, and only have a floating IP/VIP on the two nodes. This way we do not have to wait for the load balancer to come up before the rest of the cluster. We then use this *Controller VIP* in the worker nodes to point to controller node. This involves installing and configuring some HA software on controller nodes, but does not have dependency problems.

We believe the second method holds more value, so we go for it. Although just for completeness sake, we will also show a method to do it through HAProxy.

(todo: Load balancers use: (using IPVS/LVS + Heartbeat))
(todo: In case we use IPVS, would we need additional STONITH device to prevent split brain syndrome?)


So now we know all about HA! Lets talk about Kubernetes networking as well!


## Kubernetes Networking:
A Kubernetes cluster uses three different types of networks.

todo: Needs review


There are three main networks / IP ranges involved in a Kubernetes cluster setup. They are:
* Infrastructure network 
* Pod Network (Overlay or CNI) (We are using CNI network in this document.)
* Service Network (aka. Cluster Network)

Out of above three, the first two (Infrastructure and Pod networks) are straight forward. The *Infrastructure network* is where the actual physical (or virtual) machines are connected. The *Pod network* can be either an overlay network or a CNI based network. Normally overlay networks are software defined networks (SDN). Flannel is an example of overlay network. Overlay network has this big network spread over several cluster nodes, and each node will have a subnet of that overlay network configured in it. Normally only worker nodes have overlay network configured, and to get this done, a special service such as flanneld (in case it is flannel you are using) runs on the nodes. Then you configure docker to use this network to create containers on. Normally overlay network is only configured on worker nodes, and is only accessible within the worker nodes. If you want to access pod IPs (belonging to this overlay network) from other cluster nodes, you will need to run the overlay network service such as flannel on that node. Only then the pods become accessible from that node.

In CNI networking, there is again a big (sort-of) overlay network. Each worker node obtains a subnet from the big (sort of overlay) network and configures a bridge (cbr0) using that subnet. Each worker node creates pods/containers in it's own subnet. These subnets *do not* talk to each other. To make them talk to each other we add routes of each subnet on the router (working as default gateway) connected to our cluster network. Now each request from any node in the cluster (or even from outside), can consult the routing table and reach the required pods. So there is no need of running an additional service, and the benfit is that the pod IPs (no matter which worker node the pods are on), are accessible from any node within the cluster. This is the network used and shown in the diagram at the beginning of this document. 


The third (Service network), is not actually a network. (It is, and it is not - at the same time; it is special / complicated). It is used when you decide to expose a kubernetes deployment as a service. That is the only time when this comes into play. When you expose a deployment as a service, Kubernetes assigns it a Cluster IP from the *service* IP range. This IP is then used by kube-proxy running on each worker node to write special IPTables rules so these services are accessible from the various pods running anywhere on the worker nodes. The Service IP or a Cluster IP is actually just an abstraction, and represents the pods at it's backend; which (the pods) actually belong to a *Deployment* (formerly known as *Replication Controller*).

## How does the networking look like in Kubernetes?

So when you setup a Kubernetes cluster, and start up your first pod, the pod is deployed on a worker node. The pod gets an IP from the pod network, which can be a type of overlay network, or it can be a type of CNI based network. Either way, when the pod gets an IP from the pod network, the worker nodes are the only *nodes* which can access that pod. Of-course if you run more pods, they (the pods) can definitely access each other directly. To access these nodes from outside the cluster, you still need a bit of work. You know that every pod is actually part of a *Deployment* object (formerly *RC* or *Replication Controller*). So you expose this deployment as a service, which automatically assigns it a *Cluster IP*. This Cluster IP is still not accessible from outside the cluster. This new *service* can now be referenced/accessed by the pods using a special name formatted as a DNS name. While exposing the deployment as a service, you also assign it an *External IP*, which normally belongs to the infrastructure network. Although this external IP still does no good directly. It is barely used as a label. If you try to access this external IP from anywhere in the cluster, or from anywhere on the infrastructure, it will not be accessible. 


* Infrastructure Network: The network your physical (or virtual) machines are connected to. Normally your production network, or a part of it.
* Service Network: The (completely) virtual (rather fictional) network, which is used to assign IP addresses to Kubernetes Services, which you will be creating. (A Service is a frontend to a RC or a Deployment). It must be noted that IP from this network are **never** assigned to any of the interfaces of any of the nodes/VMs, etc. These (Service IPs) are used behind the scenes by kube-proxy to create (weird) iptables rules on the worker nodes. 
* Pod Network: This is the network, which is used by the pods. However it is not a simple network either, depending on what kubernetes network solution you are employing. If you are using flannel, then this would be a large software defined overlay network, and each worker node will get a subnet of this network and configured for it's docker0 interface (in very simple words, there is a little more to it). If you are employing CIDR network, using CNI, then it would be a large network called **cluster-cidr** , with small subnets corresponding to your worker nodes. The routing table of the router handling your part of infrastructure network will need to be updated with routes to these small subnets. This proved to be a challenge on AWS VPC router, but this is piece of cake on a simple/generic router in your network. I will be doing it on my work computer, and setting up routes on Linux is a very simple task.

Kelsey used the following three networks in his guide, and we intend to use the same ones, so people are not confused in different IP schemes when they are following this book and at the same time checking his guide. Below are the three networks , which we will use in this book.

* Infrastructure network:     10.240.0.0/24 
* Service Network:            10.32.0.0/24 
* Pod Network (Cluster CIDR): 10.200.0.0/16 


# Infrastructure layout:

Building upon the information we have gathered so far, especially about the Kubernetes networking, we have designed our cluster to look like this: 

![images/Kubernetes-BareMetal-Cluster-setup.png](images/Kubernetes-BareMetal-Cluster-setup.png)


# Other software components of the cluster:

## DNS:
It is understood that all nodes in this cluster will have some hostname assigned to them. It is important to have consistent hostnames, and if there is a DNS server in your infrastructure, then it is also important what are the reverse lookup names of these nodes. This information is  critical at the time when you will generate SSL certificates. 

The dns domainname we will use in this setup is `example.com` . Each node will have a hostname in the form of `*hostname*.example.com` . If you do not have a DNS setup yet, it is good time to set it up now. As we mentioned this earlier, we are using Libvirt/KVM to provide VMs for our example setup. It would be interesting for you to know that libvirt has build in DNS service called `dnsmasq`. Setting up dnsmasq service is very simple. The dnsmasq service uses the `/etc/hosts` file for name resolution. So we will populate our `/etc/hosts` file with the necessary hostnames and corresponding IP addresses and restart the dnsmasq service. That way, any nodes (including our local work computer), which use dnsmasq, will resolve the example.com related hostnames correctly. 

## Operating System:
We are using Fedora 24 64 bit server edition - on all nodes (Download from [here](https://getfedora.org/en/server/download/) ). You can use a Linux distribution of your choice. 
For poeple wanting to use Fedora Atomic, we would like to issue a warning. Fedora Atomic, (great distribtion), is a collection of binaries (etcd, kubernetes) bundled together (in a read only  filesystem), and individual packages *cannot* be updated. There is no yum/dnf, etc. In this book we are using Kubernetes 1.3 (the latest version), which is still not part of Fedora Atomic 24, so we decided to use Fedora 24 server edition (in minimal configuration), and added the packages we need directly from their official websites.

## Supporting software needed for this setup:
* Kubernetes - 1.3.0 or later (Download latest from Kubernetes website)
* etcd - 2.2.5 or later (The one that comes with Fedora is good enough)
* Docker - 1.11.2 or later (Download latest from Docker website)
* CNI networking [https://github.com/containernetworking/cni](https://github.com/containernetworking/cni)
* Linux IPVS, heartbeat / pacemaker (todo) 


## Expectations

With the infrastructure choices made above, we have hope to have the following working on our Kubernetes cluster.

* 3 x etcd nodes (in H/A configuration) 
* 2 x Kubernetes controller nodes (in H/A configuration) 
* 2 x Kubernetes worker nodes
* SSL based communication between all Kubernetes components
* Internal Cluster DNS (SkyDNS) - as cluster addon
* Default Service accounts and Secrets
* Load Balancer (in H/A configuration)

# Summary:
In this chapter, we designed our infrastructure. Hurray!
