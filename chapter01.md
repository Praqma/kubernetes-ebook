# Chapter 1: Kubernetes introduction

This chapter is all about what Kubernetes is. 

We safely assume that you already know about what containers are, and what Docker is, and now you want to take the next logical step by moving to Kubernetes. We will give you a refresher anyway!


## The golden era of virtual machines:
Before containers came along, we all used virtual machines to setup different (guest) operating systems running on top of main operating system of our computer. XEN, KVM, VMWare, VirtualBox, Microsoft Virtual PC and later Hyper-V are few popular virtualization solutions. Virtual machines allowed us to slice the resources of our computer, mainly RAM and CPU - and hard drive, and give it to different virtual machines connected through a virtual network on that physical machine. This was so because most of the time people were using barely 5-10% of the resources of a given physical machine. Virtual machines were cool, and saved a lot of money for a lot of people! All of a sudden all that wasted hardware resources on each physical server were being utilized by multiple virtual machines. There is something called a vitualization ration, which is ratio of physical server and the number of VMs running on it. With virtualization people achieved 1:20 and more! 

The downside of virtualization was (still is) that each VM would need a full OS installation. Each VM running on one physical machine just shares hardware resources and nothing else. So a full OS was the only way to go. Which may be an advantage too, as it was possible to have Windows and Linux VMs on the same physical hardware. Though the full OS is what makes VM very bulky in software sense. So if one had to run only a small web server say Apache, or Nginx, a full (albeit minimal) installation of Linux OS is required. 

In terms of automation, VMs pose a challenge of requiring full fledge provisioning software, and configuration managers which are required to automate the installation of multipe VMs on one or more physical machines.

So the itch that "why do I need a full blown OS just to run Apache?" , soon resulted in discovery of a solution to this (itch). Linux Kernel 2.6.24 introduced cgroups (Control Groups) , and LXC - Linux Containers, was born, first released in August 2008. 

LXC provides operating system-level virtualization through a virtual environment that has its own process and network space, instead of creating a full-fledged virtual machine. LXC relies on the Linux kernel cgroups functionality. It also relies on other kinds of namespace isolation functionality - such as network namespaces, which were developed and integrated into the mainline Linux kernel.

With LXC, it was suddenly possible to run just the piece of software you wanted to run in an isolated (sort of change-rooted) environment, by just having enough supporting libarires in the container, and sharing the Linux kernel from the host OS running on the physical machine. So, to run Apache, one does not need to have a full blown linux OS anymore! 

Docker seazed the opportunity by creating a very usable and user-friendly container (image) format and provided necessary tooling to create, and manage containers on Linux OS. This brought Docker to limelight, and actually made Docker so popular that many people mistakenly think that Docker is the software (or company) which brough them containers. This is of-course not true. Docker just made it super easy to use containers. 





With that all of a sudden the golden era of VMs came to an end, and the golden era of containers started. 



It must be understood that the sole purpose of a kubernetes (cluster) is to run containers. To do that, (at the moment), it uses Docker as the underlying container engine. Though container is not the basic unit in Kubernetes; it is *Pod*. A pod can be composed of one or more containers. Though two same containers (exposing the same port number) can never be part of the same pod. To elaborate it, we can take an example of a PHP based web-application, which uses a MySQL database. The web application is accessible on port 80 and the database is available on port 3306. When these services are containerized, they can be broken down in two containers - a web/PHP container and a db container. In plain docker setup, even involving docker-compose, these two would have been two containers running separately, each having a different IP address from the IP range belonging to docker0 bridge interface on the docker host. However in Kubernetes, these two could have been in the same pod, and they would actually share an IP address assigned to the pod by the same docker0 bridge! This helps to create applications in a tighter way. Continueing the same PHP/MySQL example, now we do not need to *expose* the mysql port at pod level. The web service can easily access the mysql application using **localhost** . So the only exposed port from this example pod would be port 80 used by the front end web application. 

It is arguably correct that combining the web application and a mysql database in a single pod is not a good idea and defeats the purpose of the availability of a service when one component goes down. That is why it is good idea to use separate pod for the web app and separate pod for db. This way we can independently scale up and down the web app, without worrying about scaling issues - such as - with mysql db.

Perhaps an example of multiple containers in a single pod can be a web service and a cache service such as memcache , etc. Since it is best to have a cache as close to the service as possible. Also, normally cache is stateless, (as is web service (normally)), this pod (containing these two containers), can be scaled up and down.

So a question may arise, why use Kubernetes instead of plain Docker or Docker-Compose? The answer is simple. Kubernetes has all the tooling available to deal with containers , clustering and high availability, which a simple docker or docker-compose setup does not have. 

One of the main design ideas / principles of Kubernetes is the concept of "declared state". When you want to run containers on Kubernetes cluster, you describe, what you want to run. You declare that to Kubernetes. Kubernetes takes that as declared state for that container and ensures that the running state matches the declared state. i.e. Kubernetes runs the container as declared in the declared state. If, for example, you kill the pod yourself, or somehow the pod gets killed accidentally (for whatever reason), Kubernetes sees this as a mismatch between the declared state and the running state. It sees that you have declared a web server (nginx or apache) to run as a pod, and now there are no pods running matching this description. So Kubernetes scheduler kicks in and tries to start it again all by itself - automatically. If there are multiple worker nodes available, the scheduler tries to restart the pod on another node! This is a *huge* advantage over any plain docker or docker-compose setup one may have! 


Now we come to some infrastructure level components of Kubernetes. Kubernetes uses **Worker Nodes** to run the pods. To manage worker nodes, there are **Controller Nodes**, and all the cluster state (meta data, etc), is maintained in another set of nodes running **etcd**. Though **Load Balancer** is a crucial component of the entire equation, it is technically not a formal part/component of Kubernetes cluster or kubernetes services. 

These components are shown in the diagram in the beginning of this document. 

Each type of node has specific role and thus have special services running on it. Very briefly, these are:

* Etcd node: etcd
* Controller node: kube-apiserver, kube-scheduler, kube-controller-manager. (+ kubectl to manage these services)
* Worker node: kubelet, kube-proxy, docker
* Load Balancer: iptables, haproxy, nginx, etc. (it depends!)


