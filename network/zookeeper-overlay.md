# Create a Multi-Host network using Zookeeper

You will need to do several things:
* create a zookeeper node
* create 2 docker hosts
* Put the 2 hosts on the same overlay network
* start up several containers and show they can reach each other over our network

## Setting up Zookeeper

This is the easiest step. We'll use Docker-Machine to create a new node, then start zookeeper in a docker container on that node.

```bash
docker-machine create -d virtualbox zookeeper
docker $(docker-machine config zookeeper) run -d -e MYID=1 -p 2181:2181 --name=zookeeper mesoscloud/zookeeper
```

That's it!

## Creating a Docker hosts backed by Zookeeper

We need to create a docker daemon that has the following two options set:

```
cluster-store     # Describes the location of the KV service.
cluster-advertise # Advertises containers created by the HOST on the network.
```

Docker-Machine gives you a really easy way to do this:

```bash
docker-machine create -d virtualbox --engine-opt="cluster-store=zk://$(docker-machine ip zookeeper):2181" --engine-opt="cluster-advertise=eth1:2376" overlay1
docker-machine create -d virtualbox --engine-opt="cluster-store=zk://$(docker-machine ip zookeeper):2181" --engine-opt="cluster-advertise=eth1:2376" overlay2
```

This creates two docker daemons, adding the flags to the daemon.
It looks like to me that the engine-opt flag tells the daemon to store in Zookeeper the ip address of that interface.
Then other nodes will use that interface to communicate.

## Create the network

Now we create the network. We only have to do this on one of the two nodes, and the other will pull the network config from zookeeper.

```bash
docker $(docker-machine config overlay1) network create -d overlay testnet
docker $(docker-machine config overlay2) network ls
```

You should see something like this:

```
joeljohnson@mbp $ docker $(docker-machine config overlay2) network ls
NETWORK ID          NAME                DRIVER
1d8df6f2f62d        bridge              bridge              
  ...              
8ed76b8f546f        testnet             overlay 
```

## Run some containers

```bash
docker $(docker-machine config overlay1) run -d --net=testnet --name=webserver nginx
docker $(docker-machine config overlay2) run -it --net=testnet --name=pinger busybox ping webserver.testnet
```

You should see something like this:

```
PING webserver.testnet (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: seq=0 ttl=64 time=0.993 ms
64 bytes from 10.0.0.4: seq=1 ttl=64 time=0.312 ms
64 bytes from 10.0.0.4: seq=2 ttl=64 time=0.335 ms
64 bytes from 10.0.0.4: seq=3 ttl=64 time=0.272 ms
^C
--- webserver.testnet ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.272/0.478/0.993 ms
```

Enjoy!
