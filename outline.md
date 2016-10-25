This document / ebook will show you how to setup a Kubernetes cluster on bare-metal setup. The setup may also consist of VMs instead of real bare-metal machines. You can also implement the same concepts on AWS and GCE clouds, or any other cloud for that matter, such as Digital Ocean, Zetta, etc.

**Note:** The order of the chapters can change. This outline will change heavily in the coming days / weeks. 

# Chapter 1: What is Kubernetes?
* (and why not plain docker, or docker swarm, etc?)
* Concepts and terminology of Kubernetes.

# Chapter 2: Selection of infrastructure and network technologies
* Write about what type of hardware is needed. If not physical hardware, then what size of VMs are needed. etc.
* Discuss what type of network technologies are we going to use. Such as flannel or CIDR, etc.
* This will be a relatively short chapter.

# Chapter 3: Provisioning of machines, Network setup
* Here we provision our machines, and also setup networking.
* This will have a couple of diagrams

# Chapter 4: SSL certificates

# Chapter 5: etcd nodes
* Talk about what etcd is and how to set it up, including it's installation 
* Also show how to to setup etcd in HA mode.

# Chapter 6: Kubernetes Master/Controller nodes
* Talk about how kubernetes master node is setup
* also talk about HA for controller nodes.
* Include access control and Authentication/Authorization, etc.

# Chapter 7: HA for Kubernetes Control Plane
* Here we setup LVS/IPVS/Pacemaker to provide HA to Kubernetes.

# Chapter 8: Kubernetes Worker nodes
* Setup Kubernetes worker nodes. 
* Including docker
* Setup networking (flannel or CIDR)
* Setup remote access with Kubectl


# Chapter 9: Verify various cluster components
* short chapter.
* Just verification of components.
* What to expect in logs, etc.

# Chapter 10: Working with Kubernetes
* Setting up a work computer to use kubectl and talk to kubernetes master.
* Creating a simple nginx RC/Deployment
* Scaling a Deployment
* Accessing a pod from within pod network, using pod IPs
* Creating a service using cluster IP and accessing it from within pod network
* Creating a service using external IP and accessing it from outside the cluster network and also outside of kubernetes cluster.

# Chapter 11: Praqma Load Balancer

# Chapter 12: Accessing HA Kubernetes service from outside the network 

# Chapter 13: Cluster add-ons: 

	# Chapter 14: Monitoring and Alerting
* Some Visualizers (CAdvisor, fedora CockPit, kubernetes visualizer, etc)
* Alerting?

Appendix A: DNS (dnsmasq)


 




