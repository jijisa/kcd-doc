
kubernetes
- steep learning curve to install. hard to install
- can use other container engine like rkt from coreos
  so docker independent container orchestration tool
- multi-host networking
- slower than docker swarm
- use etcd for data storage and for cluster consensus.
- use Fleet for cluster management. Fleet is from CoreOS.
  . Fleet builds on top of systemd. 
  . systemd provides system and service initialization for a machine.
  . Fleet does the same thing for a cluster of machines.
  . Fleet supports "socket activation" - a container spins up when 
    the connection is on a given port.
- components
  . Pods: groups of containers
  . Flat Networking Space: all pods can talk to each other without NAT.
  . Labels: key-value pair attached to objects like pods "type":"redis"
  . Services: a group of pods. can be connected to pods with labels
    ex) "cache" service connect to several "redis" pods 
    with label "type":"redis". 
    Then, the service will automatically round-robin requests between pods.
  . Replication Controllers: control and monitor the # of running pods for
    the service.

docker swarm
- run in a container. no install at all.
- can use only docker engine
  so docker dependent container orchestration tool
- multi-host networking using overlay network driver (ver 1.9)
- faster than kubernetes


mesos and marathon
- most scalable (thousands of nodes) but overkill for small nodes.


https://blog.docker.com/2016/03/swarmweek-docker-swarm-exceeds-kubernetes-scale/

http://www.infoworld.com/article/3042573/application-virtualization/docker-swarm-beats-kubernetes-not-so-fast.html

https://www.oreilly.com/ideas/swarm-v-fleet-v-kubernetes-v-mesos

