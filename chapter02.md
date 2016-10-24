# Chapter 2: Infrastructure design and provisioning

* Write about what type of hardware is needed. If not physical hardware, then what size of VMs are needed. etc.
* Discuss what type of network technologies are we going to use. Such as flannel or CIDR, etc.
* This will be a relatively short chapter.


So, from chapter 1, we have built our knowledge-base about what Kubernetes is, what are pods, etc. Naturally now the question arises, how can we have our own Kubernetes cluster? How to design it? How to set it up? In this chapter we have answers to these questions and more!


This chapter will discuss how we design our infrastructure. We know that the book is about Kubernetes on Bare Metal hardware. But what if one does not have the necessary amount of hardware? The most natural option which comes to mind is to use virtual machines! Of-course! And cloud providers come to mind also. However each cloud provider has their own way of networking .i.e. their own way of how machines talk to each other and with the rest of the world. So we decided to use simple VMs running on a simple (but very powerful) hypervisor - Libvirt/KVM. You are welcome to use VirtualBox or VMware, or HyperV too. These VMs will be as good as bare-metal. There are no fancy networks, or firewalls, or permission groups or other access controls. It is just a hypervisor running on our very own work computer, with a single virtual network, and that is about it. This makes is very easy to learn and experiment with Kubernetes. It is still possible to run this setup on AWS or GCE or any other cloud provider of your choice though. 


## How many VMs and what size?

As we know from [chapter 1](chapter01.md), Kubernetes cluster has three main components. These three components are:

* Etcd 
* Control plane (comprising of: API Server, Scheduler, Controller Manager) 
* Worker(s) 

It is technically possible to have all these components on a single machine - and brave souls have done that; a minimum of three nodes are recommended for smooth functionality - and clarity. So in a simple three node Kubernetes cluster, you will have an etcd node holding all the cluster meta data, a controller node acting as a control plane talking to the etcd node, and one worker node, which runs the pods. What if the worker node dies? Well, that very reason we need to have at least two worker nodes in a Kubernetes cluster, so if one node dies, Kubernetes scheduler can re-schedule the pods to the surviving node. That is actually the main point behind the Kubernetes project. Otherwise plain Docker setup can pretty much do the same thing! For experimentation and learning, a simple three node Kubernetes cluster is good enough.

## Do we need high availability?
Absolutely! - if you want your cluster for more than just learning! 

So, even if you use a dedicated node for each component, there would still be no high availability against failure of "non-worker" components of the cluster, such as etcd node and controller node. So even our pods may survive a failing node, the cluster itself cannot survive if the controller node fails or if the etcd node fails. So we need to have high availability for these components too. 

## What about reaching the services on the pods from outside the cluster?
To make sure that the incoming traffic can reach the pods, we need an additional component called a load balancer (more about this later - we promise!), or just use NodePort (more about this soon!). Just like when you buy a camera, an important component of the camera is missing, which is called a **tripod** :). You need to additionally buy this missing component when you buy a camera. Similarly you need to setup/add a load balancer when you setup a Kubernetes cluster, because - as you guessed it, it is a component which is missing from the Kubernetes software (at the moment).

To have high availability for non-worker components, we need to use some sort of (external) clustering solution. What we do is that we setup three nodes for etcd, which can form it's own individual/independent cluster. Then we choose to use two nodes for controller instead of just one and use LVS/IPVS to make it a small cluster too. This makes is quite resilient to failures! We have a load balancer, but that will become a single point of failure in our cluster, if we do not protect it against hardware failures. So we need to setup at least two nodes for load balancer, using LVS/IPVS HA technologies. We now have the number of required nodes which looks like the following:


* 3 x Etcd nodes -		Each with: 0.5 GB RAM, 4 GB disk 
* 2 x Control plane nodes -	Each with: 0.5 GB RAM, 4 GB disk 
* 2 x Worker nodes - 		Each with: 1.5 GB RAM, 20 GB disk
* 2 x Load Balancer nodes -	Each with: 0.5 GB RAM, 4 GB disk

That makes a total of nine (9) nodes.


Though not abolsulutely necessary, we also need a shared storage solution to demonstrate the concept of network mounted volumes (todo - ????) . We *can* use the Load balancer nodes to setup NFS and make it Highly available.

**Note about NodePort:**
In Kubernetes, `NodePort` is way to expose a service to the outside world - when you specify it in the `kubectl expose <service> --type=NodePort` (todo - confirm / verify). What this does is that it allows you to expose a service on the worker node, just like you can do the port mapping on a simple docker host. This makes the service available through all the worker nodes, because node port works it's magic by setting up certain iptables rules on all worker nodes. So no matter which port you land on (from outside the cluster), the service responds to you. However, to do this, you need to have the external DNS pointing to the worker nodes. e.g. www.example.com can have two A type addresses in DNS pointing to the IP addresses of two worker nodes. In case a node fails, DNS will do it's own DNS round robin. This may work, but DNS round robin does not work well with sticky sessions. Also, managing services through ports is kind of difficult to manage / keep track of. Plus it is totally uncool! (todo: editor can remove this if unhappy :(  ).


## The choice of HA technology for etcd, controllers and load balancers:

We selected three nodes for the etcd service. First, because it stores the meta data about our cluster, which is most important. Secondly etcd nodes can form their own individual/independent cluster. To satisfy the needs of a quorum, we decided to use three nodes for etcd cluster. This means failure of one node does not affect the etcd operation. (todo: rewrite:) In order for etcd cluster to be healthy, it needs to have quorum. i.e. alive nodes being more than half of the size of the total number of nodes of etcd cluter. So having a total of two nodes is not helpful. When one fails, the etcd custer will instantly become unhealthy. This is what we do not want to happen in a production cluster.

We know that all Kubernetes components watch the API server, running on controller node. So making sure that control plane remains available at all times. Unfortunately, controller nodes are not cluster aware. So we need to provide some sort of HA for them. This high availability can be provided in two ways. 

* By setting up a proxy/load balancer (using HAProxy on our load balancer). This would mean that the load balancer has to be the first component to come alive when the cluster boots up. So cluster has dependency on the load balancer nodes. This is easier to setup but has dependency problem.
* By setting up IPVS/LVS (using heartbeat, etc) on the controller nodes themselves, and only have a floating IP/VIP on the two nodes. This way we do not have to wait for the load balancer to come up before the rest of the cluster. We then use this *Controller VIP* in the worker nodes to point to controller node. This involves installing and configuring some HA software on controller nodes, but does not have dependency problems.

We believe the second method holds more value, so we go for it. Although just for completeness sake, we will also show a method to do it through HAProxy.

(todo: Load balancers use: (using IPVS/LVS + Heartbeat))


**todo:** In case we use IPVS, would we need additional STONITH device to prevent split brain syndrome? 


## DNS names:
It is understood that all nodes in this cluster will have some hostname assigned to them. It is important to have consistent hostnames, and if there is a DNS server in your infrastructure, then it is also important what are the reverse lookup names of these nodes. This information is  critical at the time when you will generate SSL certificates. 

The dns domainname we will use in this setup is `example.com` . Each node will have a hostname in the form of `*hostname*.example.com` . 


## Operating System:
We are using Fedora 24 64 bit server edition - on all nodes (Download from [here](https://getfedora.org/en/server/download/) ). You can use a Linux distribution of your choice. 
For poeple wanting to use Fedora Atomic, we would like to issue a warning. Fedora Atomic is a collection of binaries (etcd, kubernetes) bundled together (in a read only  filesystem), and individual packages *cannot* be updated. There is no yum/dnf, etc. In this book we are using Kubernetes 1.3 (the latest version), which is still not part of Fedora Atomic 24, so we decided to use Fedora 24 server edition (in minimal configuration), and added the packages we need directly from their official websites.

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


Before we start building VMs, we would like to hightlight few things about Kubernetes networking.

## Kubernetes Networking:
Kubernetes uses three different types of networks. They are:

* Infrastructure Network: The network your physical (or virtual) machines are connected to. Normally your production network, or a part of it.
* Service Network: The (completely) virtual (rather fictional) network, which is used to assign IP addresses to Kubernetes Services, which you will be creating. (A Service is a frontend to a RC or a Deployment). It must be noted that IP from this network are **never** assigned to any of the interfaces of any of the nodes/VMs, etc. These (Service IPs) are used behind the scenes by kube-proxy to create (weird) iptables rules on the worker nodes. 
* Pod Network: This is the network, which is used by the pods. However it is not a simple network either, depending on what kubernetes network solution you are employing. If you are using flannel, then this would be a large software defined overlay network, and each worker node will get a subnet of this network and configured for it's docker0 interface (in very simple words, there is a little more to it). If you are employing CIDR network, using CNI, then it would be a large network called **cluster-cidr** , with small subnets corresponding to your worker nodes. The routing table of the router handling your part of infrastructure network will need to be updated with routes to these small subnets. This proved to be a challenge on AWS VPC router, but this is piece of cake on a simple/generic router in your network. I will be doing it on my work computer, and setting up routes on Linux is a very simple task.

Kelsey used the following three networks in his guide, and we intend to use the same ones, so people are not confused in different IP schemes when they are following this book and at the same time checking his guide. So here are my three networks , which we will use in this book.

* Infrastructure network:     10.240.0.0/24 
* Service Network:            10.32.0.0/24 
* Pod Network (Cluster CIDR): 10.200.0.0/16 


# Infrastructure layout / network layout:

Building upon the information we have gathered so far, especially about the Kubernetes networking, we have designed our cluster to look like this: 

![images/Kubernetes-BareMetal-Cluster-setup.png](images/Kubernetes-BareMetal-Cluster-setup.png)


# Conclusion of chapter 2:
In this chapter, we designed our infrastructure. 





