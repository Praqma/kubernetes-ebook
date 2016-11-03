# Chapter 1: Kubernetes introduction

This chapter is meant to offer an introductory overview to Kubernetes. 

It is assumed that the reader already knows what containers are, what Docker is, and wants to take the next logical step by looking at Kubernetes. But, if your memory is a bit rusty a quick refresher is provided anyway!


## The golden era of virtual machines:
Before containers came along we all used virtual machines to setup different (guest) operating systems that ran on top of the main operating system of our computer. XEN, KVM, VMWare, VirtualBox, Microsoft Virtual PC and later Hyper-V are a few popular virtualization solutions. Virtual machines allowed us to slice the resources of our computer, mainly RAM, CPU, and hard drive, and give it to different virtual machines connected through a virtual network on a physical machine. This development came about because most of the time people were using barely 5-10% of the resources of a given physical machine. Virtual machines were cool, and saved a lot of money for a lot of people! All of the wasted hardware resources on each physical server were now being utilized by multiple virtual machines. There is something called a vitualization ratio which is the number of VMs running on a physical server. With virtualization people achieved 1:20 and sometimes more! 

The downside of virtualization was (still is) that each VM requires a full OS installation. Each VM running on a physical machine shares only the hardware resources and nothing else, so a full OS was the only solution. This can be advantageous as it allows for Windows and Linux VMs on the same physical hardware, but the full OS is what makes VM very bulky in a software sense. Therefore, if you have to run only a small web server on say, Apache, or Nginx, a full (albeit minimal) installation of Linux OS is required. 

In terms of automation VMs pose a significant challenge; fully fledged provisioning software and configuration managers which are required to automate the installation of multipe VMs on one or more physical machines are required.

But, people began to ask "why do I need a full blown OS just to run Apache?" and before long a solution was found that eliminated this requirement. Linux Kernel 2.6.24 introduced cgroups (Control Groups) and LXC - Linux Containers - was born and first released in August 2008. 

Instead of creating a fully-fledged virtual machine LXC provides operating system-level virtualization through a virtual environment that has its own process and network space. LXC relies on the Linux kernel cgroups functionality. It also relies on other kinds of namespace isolation functionality, such as network namespaces, which were developed and integrated into the mainline Linux kernel.

With LXC it became possible to run only the required piece of software in an isolated (sort of change-rooted) environment by having just enough supporting libraries in the container, and by sharing the Linux kernel from the host OS running on the physical machine. So, to run Apache, a full blown linux OS was no longer needed! 

In 2013 Docker seized the opportunity to create a very usable and user-friendly container (image) format around LXC and provided the necessary tooling to create and manage containers on Linux OS. This brought Docker into the limelight and made them so popular that many people mistakenly think that Docker is the software (or company) which invented containers. This is, of course, not true. Docker just made it super easy to use containers. 

The golden era of VMs came to an end, and the golden era of containers started. 

# Docker Containers

Best explained from Docker's own web pages:

>Docker containers wrap up a piece of software in a complete filesystem that contains everything it needs to run: code, runtime, system tools, system libraries – anything you can install on a server. This guarantees that it will always run the same, regardless of the environment it is running in.

Docker uses the resource isolation features of the Linux kernel, such as cgroups and kernel namespaces, and a union-capable file system such as aufs and others to allow independent "containers" to run within a single Linux instance, avoiding the overhead of starting and maintaining virtual machines. This helps Docker to provide an additional layer of abstraction and automation of operating-system-level virtualization on Linux.

So running an Apache web server on some (physical or virtual) Linux computer would be as simple as:

```
[kamran@kworkhorse kamran]$ docker run -p 80:80 -d httpd
Unable to find image 'httpd:latest' locally
latest: Pulling from library/httpd
43c265008fae: Pull complete 
2421250c862c: Pull complete 
f267bf8fc4ac: Pull complete 
48efff98b4ba: Pull complete 
acb686eb7ab7: Pull complete 
Digest: sha256:9b29c9ba465af556997e6924f18efc1bbe7af0dc1b3f11884010592e700ddb09
Status: Downloaded newer image for httpd:latest
14954a5c880c343d57e00cf270ca3f3212ef0e3b23635c1a5a40fe279548671f
[kamran@kworkhorse kamran]$ 
```

The -p 80:80 binds the container's port 80 to the host's port 80. To make sure that it is actually running on port 80 on the host we verify:

```
[kamran@kworkhorse kamran]$ docker ps
CONTAINER ID        IMAGE               COMMAND              CREATED             STATUS              PORTS                NAMES
14954a5c880c        httpd               "httpd-foreground"   5 minutes ago       Up 5 minutes        0.0.0.0:80->80/tcp   nauseous_nobel
[kamran@kworkhorse kamran]$ 
```

Accessing this web server from any other machine (e.g. localhost/itself)is now possible using normal methods, such as curl:

```
[kamran@kworkhorse kamran]$ curl localhost
<html><body><h1>It works!</h1></body></html>
[kamran@kworkhorse kamran]$ 
```


To show you that how tiny this Apache (httpd) image is, here is something to feast your eyes:
```
[kamran@kworkhorse kamran]$ docker images | grep httpd
httpd                                      latest              9a0bc463edaa        3 days ago          193.3 MB
[kamran@kworkhorse kamran]$
```
Just 193 MB! Sure it is much more than the Apache/HTTPD package itslef, but it is tiny compared to the full OS installation on any Linux distribtion and installing Apache on top of that. (Yes, we know Alpine is tiny OS itslef - so excluding Alpine!)


Ok so now you have an apache service running as a container! Great! Now what? Scale up to 50 apache instances? Well, can't really do that with VMs, we know that already! Scaling up with docker - well not straight forward exactly. What if the container dies for some reason? Any possibility to have it restarted? Again, with plain docker, that is not possible? 

So, do we need to create our own tooling to solve such problems, No! - because we have - Kubernetes!

# Kubernetes - The container orchestrator:

Kubernetes (Greek for "helmsman" or "pilot") - often referred simply as *k8s* - was founded and announced by Google in 2014. It is an open source container cluster manager, originally designed by Google, and is heavily based on Google's own *Borg* system. It is a platform for automating deployment, scaling, and operations of containers across a cluster of nodes. By design it is a collection of loosely coupled components, which help it to be flexible, scaleable and very manageable. This architecture helps it to meet a wide variety of workloads. Kubernetes uses Docker as it's container runtime engine, though it is possible to use other container runtime engines as well, such as *Rocket*. 

Kubernetes uses Pod as its fundamental unit for scheduling instead of Container. Whereas a pod can be composed of one or more containers guaranteed to be colocated on a single host. Instead of containers having an IP address - as in Docker, it is the pod which is assigned a (unique) IP address by the container runtime engine, and that IP is shared with all the containers inside that particular pod. Two same containers (exposing the same port number) can never be part of the same pod though. 

To elaborate it, we can take an example of a PHP based web-application, which uses a MySQL database backend. The web application is accessible on port 80 and the database is available on port 3306. When these services are containerized, they can be broken down in two containers - a web/PHP container and a db container. In plain docker setup, even involving docker-compose, these two would have been two containers running separately, each having a different IP address from the IP range belonging to docker0 bridge interface on the docker host. However in Kubernetes, these two could have been in the same pod, and they would actually share an IP address assigned to the pod by the same docker0 bridge! This helps to create applications in a tighter way. Continueing the same PHP/MySQL example, now we do not need to *expose* the mysql port at pod level. The web service can easily access the mysql application using **localhost** . So the only exposed port from this example pod would be port 80 used by the front end web application. 

It is arguably correct that combining the web application and a mysql database in a single pod is not a good idea and defeats the purpose of the availability of a service when one component goes down. That is why it is good idea to use separate pod for the web app and separate pod for db. This way we can independently scale up and down the web app, without worrying about scaling issues - such as - with mysql db.

Perhaps an example of multiple containers in a single pod can be a web service and a cache service such as memcache, etc. Since it is best to have a cache as close to the service as possible. Also, normally cache is stateless, (as is web service (normally)), this pod (containing these two containers), can be scaled up and down independently.

So if you are asking your self, "Why use Kubernetes instead of plain Docker or Docker-Compose?", the answer is simple. Kubernetes has all the tooling available to deal with containers, clustering and high availability, which a simple docker or docker-compose setup does not have. It is explained next.

One of the main design ideas / principles of Kubernetes is the concept of "declared state". When you want to run containers on Kubernetes cluster, you describe what you want to run. You declare that to Kubernetes. Kubernetes takes that as declared state for that container and ensures that the running state matches the declared state. i.e. Kubernetes runs the container as declared in the declared state. If, for example, you kill the pod yourself, or somehow the pod gets killed accidentally (for whatever reason), Kubernetes sees this as a mismatch between the declared state and the running state. It sees that you have declared a web server (nginx or apache) to run as a pod, and now there are no pods running matching this description. So Kubernetes scheduler kicks in and tries to start it again all by itself - automatically. If there are multiple worker nodes available, the scheduler tries to restart the pod on another node! This is a *huge* advantage over any plain docker or docker-compose setup one may have! 

Referring to the example of the PHP based web application and it's backend database, had it being run on a plain docker, (or docker-compose), failure of a service container would not trigger an automatic restart. However in Kubernetes, if the web service pod dies, Kubernetes starts it again, and if the db pod dies Kubernetes starts it again, automatically!

Also, if you want to scale the web application to 50 instances , all you need to do is to declare this to Kubernetes. Kubernetes scheduler will spawn 50 instances of this web service pod. Though you might want to add a load balancer in front of these 50 instances! 

Availability and scaling are just two of the many exciting features of Kubernetes. There are many more interesting features in Kubernetes, which makes the job of managing your applications super easy. e.g. The possibility of having separate namespaces to have isolation of pods, and access controls, labels, selectors, etc, are some other helpful features.


## Pods, Deployments, Labels, Selectors and Services

### Pods:
So from above, we already learned about the pod. i.e. It is the fundamental unit of scheduling in Kubernetes; each pod gets a unique IP address - from the pool of IPs belonging to *Pod Network*; each pod can have one or more containers in it.

### Deployments (formerly Replication Controllers):
An interesting thing is that you cannot directly create a pod in kubernetes! For example, if you want to run nginx web service, you don't create a pod for nginx. Instead, you have to create a *Deployment* (formerly known as *Replication Controller* or *RC*). This deployment will have the nginx pod inside it. Normally when you ask Kubernetes to run a container image without any specific definition/configuration file, then Kubernetes creates a deployment for it, makes the pod part of this deployment and runs the pod - all automatically. The Deployment object can have labels, which you can use in selectors while creating or exposing services, or for other general deployment management tasks. (todo: More on this later.) The deploymnet object is sometimes called *deployment controller* just as it's predecessor was called *replication controller*. 

### Labels and Selectors
Kubernetes enables users and internal components to attach key-value pairs called "labels" to any API object in the system, e.g. pods, deployments, nodes, etc. Correspondingly, "label selectors" are queries against labels that resolve to matching objects. Labels and selectors are the primary grouping mechanism in Kubernetes, and are used to determine which components or objects certain operation will be applied to. The name of the key and the value it contains depend solely on the operator. There are no fixed labels and no reserved words.

For example, if the Pods of an application have labels for "tier" (front-end, back-end, etc.) and "release_track" (canary, production, etc.), then an operation on all of the back-end production nodes could use a selector such as the following:

```
. . .  tier=back-end AND release_track=production
```

### Services

After a deployment is created it is not accissble directly for general use. One way is to query the deployment to check for the IPs of it's backend pods, and then access those IPs to utilize the services offered by those pods. This may work for a deploymnet containing one pod, but for a deployment having 10 or 50 pods behind it, accessing the individual IPs is not smart. Also depending on your cluster/network design, it may not be possible to access the pod IP directly in some cases. Also the pods are getting deleted and recreatd all the time, and their IP addresses keep changing. You need a persistent way of accessing these pods (or deployments to be precise). Enter *Services*. 

When you want to make a deployment available over a persistent IP, you expose the deployment as a *Service*. Exposing a deployment actually creates the Service object, assigns it a cluster IP and a DNS name for Kubernetes internal usage. So now, the web application can access the backend db using a proper DNS name such as "db.default.cluster.local". This way, even if the service's cluster IP is changed because of service deletion and re-creation, the web application will still be able to access it using this DNS name. It may look like that the emphasis is on DNS here, but in reality, the DNS points to the cluster IP, and without the ability to create a Service in the first place, there would be no cluster IP for this deployment! 

So service is the way to make the deployments available for actual use. The service can be exposed in different ways though, or there are few different types of service. The service can be exposed as **ClusterIP** (which is the default), as **NodePort**, or as **LoadBalancer**. The default type ClusterIP obtains an IP address from the pool of the IPs from Services IP range, also known as Cluster Network. Interestingly the Cluster Network is not really a network, rather special type of labels, which is used by kube-proxy software running on the worker nodes to create complex IPTables rules on worker nodes. When a pod tries to access a service using it's cluster IP, the IPtables rules ensure that the trafic from the client pod can reach the service pod. The Cluster IPs are not accessible outside worker nodes. 



## Accessing services: Cluster IPs, Load Balancers, NodePorts:

We already covered Cluster IPs above. Now comes the question that how a service can be accessed from outside the worker nodes, or outside the Kubernetes cluster? e.g. The web application in the example above (in the Serices section) can access the backend DB thanks to the cluster IP. But how do we access the frontend web application? Well, to do that we have two more types of Services, which are NodePort and LoadBalancer. 

### Node Port:
When you expose a deployment as a service and declare it of the type *NodePort*, then Kubernetes exposes the port of the backend pods of the deployment on the worker node, and replicates this on all worker nodes. Now, if you want to access this front end application, you access it using the IP address of any of the worker nodes. The process of exposing the port is not so straight forward internally! For example you want to expose your deployment running 10 nginx pods as a NodePort type of service. Kubernetes will assign it a higher number port (e.g. 33056) , bind that on the worker node, and map that high number port to the port 80 of the deploymnet (or to the ports of the pods inside of the deployment). When you want to access this web aplication, you access it in the form of `<WorkerNode's IP>:33056` . This is a bit awkward, but this is the way with NodePort!

### Load Balancer:
At the moment it is only possible to expose a deployment as a LoadBalancer type of service, IF, you are running your cluster on GCE (Google compute Engine), or AWS. The LoadBalancer type requires that you can map some public IP to the IPs of the backend pods of a deployment object. This is not dependent on ports though. This is the ideal way of accessing the service on your Kubernetes cluster, but so far this is the biggest problem if you are not on GCE or AWS. The good news is that Praqma has created a LoadBalancer just for you! (todo: More on this later!)





## Kubernetes infrastructure components:

Now we come to some infrastructure level components of Kubernetes. Kubernetes uses **Worker Nodes** to run the pods. To manage worker nodes, there are **Controller Nodes**, and all the cluster state (meta data, etc), is maintained in another set of nodes running **etcd**. Though **Load Balancer** is a crucial component of the entire equation, it is technically not a formal part/component of Kubernetes cluster or kubernetes services. 

These components are shown in the diagram in the beginning of this document. 

Each type of node has specific role and thus have special services running on it. Very briefly, these are:

* Etcd node: etcd
* Controller node: kube-apiserver, kube-scheduler, kube-controller-manager. (+ kubectl to manage these services)
* Worker node: kubelet, kube-proxy, docker
* Load Balancer: iptables, haproxy, nginx, etc. (it depends!)


