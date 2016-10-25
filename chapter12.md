Chapter 12: Kubectl from outside the network
* Show how kubectl will access controller nodes.

This has to do something with the HA we setup at the controller nodes, and the HA we setup at proxy level.
Also specify how kubectl will access the controller nodes from outside the cluster. This is especially important, because we need to specify this on the edge router, whether the incoming traffic for 6443 will go to load balancer VIP or the VIP of the controller nodes.


