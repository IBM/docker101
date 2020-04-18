# Lab 3- Introduction to Orchestration

## Overview

So far you have learned how to run applications using docker on your local machine, but what about running dockerized applications in production? There are a number of problems that come with building an application for production: scheduling services across distributed nodes, maintaining high availability, implementing reconciliation, scaling, and logging... just to name a few.

There are several orchestration solutions out there that help you solve some of these problems. One example is the [IBM Kubernetes Service](https://cloud.ibm.com/kubernetes/catalog/create) which uses [Kubernetes](https://kubernetes.io/) to run containers in production.

Before we introduce you to Kubernetes, we will teach you how to orchestrate applications using Docker Swarm. Docker Swarm is the orchestration tool that comes built-in to the Docker Engine.

We will be using a few Docker commands in this lab. For full documentation on available commands check out the [Docker official documentation](https://docs.docker.com/).

### Prerequisites

In order to complete a lab about orchestrating an application that is deployed across multiple hosts, you need... well, multiple hosts.  To make things easier, for this lab we will be using the multi-node support provided by [Play with Docker](http://play-with-docker.com). This is the easiest way to test out Docker Swarm, without having to deal with installing docker on multiple hosts yourself.

## Step 1: Create your first swarm

In this step, we will create our first swarm using play-with-docker.

1. Navigate to [Play with Docker](http://play-with-docker.com)

1. Click "add new instance" on the lefthand side three times to create three nodes

    Our first swarm cluster will have three nodes.

1. Initialize the swarm on node 1

    ```sh
    $ docker swarm init --advertise-addr eth0
    Swarm initialized: current node (vq7xx5j4dpe04rgwwm5ur63ce) is now a manager.

    To add a worker to this swarm, run the following command:

        docker swarm join \
        --token SWMTKN-1-50qba7hmo5exuapkmrj6jki8knfvinceo68xjmh322y7c8f0pj-87mjqjho30uue43oqbhhthjui \
        10.0.120.3:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
    ```

    You can think of docker swarm as a special "mode" that is activated by the command: `docker swarm init`. The `--advertise-addr` specifies the address in which the other nodes will use to join the swarm.

    This `docker swarm init` command generates a join token. The token makes sure that no malicious nodes join our swarm. We will need to use this token to join the other nodes to the swarm. For convenience, the output includes the full command `docker swarm join` which you can just copy/paste to the other nodes.

1. On both node2 and node3, copy and run the `docker swarm join` command that was outputted to YOUR console by the last command.

    You now have a three node swarm!

1. Back on node1, run `docker node ls` to verify your 3 node cluster.

    ```sh
    $ docker node ls
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    7x9s8baa79l29zdsx95i1tfjp     node3               Ready               Active
    x223z25t7y7o4np3uq45d49br     node2               Ready               Active
    zdqbsoxa6x1bubg3jyjdmrnrn *   node1               Ready               Active              Leader
    ```

    This command outputs the three nodes in our swarm. The * next to the ID of the node represents the node that handled that specific command (`docker node ls` in this case).  

    Our node consists of  1 manager node and 2 workers nodes. Managers handle commands and manage the state of the swarm. Workers cannot handle commands and are simply used to run containers at scale. By default, managers are also used to run containers.

    All `docker service` commands for the rest of this lab need to be executed on the manager node (Node1).

    **Note:** While we will control the Swarm directly from the node in which its running, you can control a docker swarm remotely by connecting to the docker engine of the manager via the remote API or activating a remote host from your local docker installation (using the `$DOCKER_HOST` and `$DOCKER_CERT_PATH` environment variables). This will become useful when you want to control production applications remotely instead of ssh-ing directly into production servers.

## Step 2: Deploy your first service

Now that we have our 3 node Swarm cluster initialized, let's deploy some containers. To run containers on a Docker Swarm, we want to create a service. A service is an abstraction that represents multiple containers of the same image deployed across a distributed cluster.

Let's do a simple example using Nginx. For now we will create a service with just 1 running container, but we will scale up later.

1. Deploy a service using Nginx

    ```sh
    $ docker service create --detach=true --name nginx1 --publish 80:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12
    pgqdxr41dpy8qwkn6qm7vke0q
    ```

    This above statement is *declarative*, and docker swarm will actively try to maintain the state declared in this command unless explicitly changed via another `docker service` command. This behavior comes in handy when nodes go down, for example, and containers are automatically rescheduled on other nodes. We will see a demonstration of that a little later on in this lab.

    The `--mount` flag is a neat trick to have nginx print out the hostname of the node it's running on. This will come in handy later in this lab when we start load balancing between multiple containers of nginx that are distributed across different nodes in the cluster, and we want to see which node in the swarm is serving the request.

    We are using nginx tag "1.12" in this command. We will demonstrate a rolling update with version 1.13 later in this lab.

    The `--publish` command makes use of the swarm's built in *routing mesh*. In this case port 80 is exposed on *every node in the swarm*. The routing mesh will route a request coming in on port 80 to one of the nodes running the container.

1. Inspect the service

    You can use `docker service ls` to inspect the service you just created.

    ```sh
    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
    pgqdxr41dpy8        nginx1              replicated          1/1                 nginx:1.12          *:80->80/tcp
    ```

1. Check out the running container of the service

    To take a deeper look at the running tasks, you can use `docker service ps`. A task is yet another abstraction using in docker swarm that represents the running instances of a service. In this case, there is a 1-1 mapping between a task and a container.

    ```sh
    $ docker service ps nginx1
    ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STAT
    E           ERROR               PORTS
    iu3ksewv7qf9        nginx1.1            nginx:1.12          node1               Running             Running 8 mi
    nutes ago
    ```

    If you happen to know which node your container is running on (you can see which node based on the output from `docker service ps`), you can use `docker container ls` to see the container running on that specific node.

1. Test the service

    Because of the routing mesh, we can send a request to any node of the swarm on port 80. This request will be automatically routed to the one node that is running our nginx container.

    Try this on each node:

    ```sh
    $ curl localhost:80
    node1
    ```

    Curling will output the hostname where the container is running. For this example, it is running on "node1", but yours might be different.

## Step 3: Scale your service

In production we may need to handle large amounts of traffic to our application. So let's scale!

1. Update your service with an updated number of replicas

    We are going to use the `docker service` command to update the nginx service we created earlier to include 5 replicas. This is defining a new state for our service.

    ```sh
    $ docker service update --replicas=5 --detach=true nginx1
    nginx1
    ```

    As soon as this command is run the following happens:

    1. The state of the service is updated to 5 replicas (which is stored in the swarms internal storage).
    1. Docker swarm recognizes that the number of replicas that is scheduled now does not match the declared state of 5.
    1. Docker swarm schedules 5 more tasks (containers) in an attempt to meet the declared state for the service.

    This swarm is actively checking to see if the desired state is equal to actual state, and will attempt to reconcile if needed.

1. Check the running instances

    After a few seconds, you should see that the swarm did its job, and successfully started 9 more containers. Notice that the containers are scheduled across all three nodes of the cluster. The default placement strategy that is used to decide where new containers are to be run is "emptiest node", but that can be changed based on your need.

    ```sh
    $ docker service ps nginx1
    ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STAT
    E            ERROR               PORTS
    iu3ksewv7qf9        nginx1.1            nginx:1.12          node1               Running             Running 17 m
    inutes ago
    lfz1bhl6v77r        nginx1.2            nginx:1.12          node2               Running             Running 6 mi
    nutes ago
    qururb043dwh        nginx1.3            nginx:1.12          node3               Running             Running 6 mi
    nutes ago
    q53jgeeq7y1x        nginx1.4            nginx:1.12          node3               Running             Running 6 mi
    nutes ago
    xj271k2829uz        nginx1.5            nginx:1.12          node1               Running             Running 7 mi
    nutes ago
    ```

1. Send a bunch of requests to [localhost:80](http://localhost:80)

    The `--publish 80:80` is still in effect for this service, that was not changed when we ran `docker service update`. However, now when we send requests on port 80, the routing mesh has multiple containers in which to route requests to. The routing mesh acts as a load balancer for these containers, alternating where it routes requests to.

    Let's try it out by curling multiple times. Note, that it doesn't matter which node you send the requests. There is no connection between the node that receives the request, and the node that that request is routed to.

    ```sh
    $ curl localhost:80
    node3
    $ curl localhost:80
    node3
    $ curl localhost:80
    node2
    $ curl localhost:80
    node1
    $ curl localhost:80
    node1
    ```

You should see which node is serving each request because of the nifty `--mount` command we used earlier.

**Limits of the routing Mesh**
The routing mesh can only publish one service on port 80. If you want multiple services exposed on port 80, then you can use an external application load balancer outside of the swarm to accomplish this.

1. Check the aggregated logs for the service

    Another easy way to see which nodes those requests were routed to is to check the aggregated logs. We can get aggregated logs for the service using `docker service logs [service name]`. This aggregates the output from every running container, i.e. the output from `docker container logs [container name]`.

    ```sh
    $ docker service logs nginx1
    nginx1.4.q53jgeeq7y1x@node3    | 10.255.0.2 - - [28/Jun/2017:18:59:39 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.
    52.1" "-"
    nginx1.2.lfz1bhl6v77r@node2    | 10.255.0.2 - - [28/Jun/2017:18:59:40 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.
    52.1" "-"
    nginx1.5.xj271k2829uz@node1    | 10.255.0.2 - - [28/Jun/2017:18:59:41 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.
    52.1" "-"
    nginx1.1.iu3ksewv7qf9@node1    | 10.255.0.2 - - [28/Jun/2017:18:50:23 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.
    52.1" "-"
    nginx1.1.iu3ksewv7qf9@node1    | 10.255.0.2 - - [28/Jun/2017:18:59:41 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.
    52.1" "-"
    nginx1.3.qururb043dwh@node3    | 10.255.0.2 - - [28/Jun/2017:18:59:38 +0000] "GET / HTTP/1.1" 200 6 "-" "curl/7.
    52.1" "-"
    ```

    Based on these logs we can see that each request was served by a different container.

    In addition to seeing whether the request was sent to node1, node2, or node3, you can also see which container on each node that it was sent to. For example `nginx1.5` means that request was sent to container with that same name as indicated in the output of `docker service ps nginx1`.

## Step 4: Rolling Updates

Now that we have our service deployed, let's demonstrate a release of our application. We are going to update the version of Nginx to version "1.13". To do this update we are going to use the `docker service update` command.

```sh
$ docker service update --image nginx:1.13 --detach=true nginx1
1a2b3c
```

This will trigger a rolling update of the swarm. Quickly type in `docker service ps nginx1` over and over to see the updates in real time.

You can fine tune the rolling update using these options:

* `--update-parallelism` will dictate the number of containers to update at once. (defaults to 1)
* `--update-delay` will dictate the delay between finishing updating a set of containers before moving on to the next set.

After a few seconds, run `docker service ps nginx1` to see all the images have been updated to nginx:1.13.

```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STAT
E             ERROR               PORTS
di2hmdpw0j0z        nginx1.1            nginx:1.13          node1               Running             Running 50 s
econds ago
iu3ksewv7qf9         \_ nginx1.1        nginx:1.12          node1               Shutdown            Shutdown 52
seconds ago
qsk6gw43fgfr        nginx1.2            nginx:1.13          node2               Running             Running 47 s
econds ago
lfz1bhl6v77r         \_ nginx1.2        nginx:1.12          node2               Shutdown            Shutdown 49
seconds ago
r4429oql42z9        nginx1.3            nginx:1.13          node3               Running             Running 41 s
econds ago
qururb043dwh         \_ nginx1.3        nginx:1.12          node3               Shutdown            Shutdown 43
seconds ago
jfkepz8tqy9g        nginx1.4            nginx:1.13          node2               Running             Running 44 s
econds ago
q53jgeeq7y1x         \_ nginx1.4        nginx:1.12          node3               Shutdown            Shutdown 45
seconds ago
n15o01ouv2uf        nginx1.5            nginx:1.13          node3               Running             Running 39 s
econds ago
xj271k2829uz         \_ nginx1.5        nginx:1.12          node1               Shutdown            Shutdown 40
seconds ago
```

You have successfully updated your app to the latest version of nginx!

## Step 5: Reconciliation

In the previous step, we updated the state of our service using `docker service update`. We saw Docker Swarm in action as it recognized the mismatch between desired state and actual state, and attempted to solve this issue.

The "inspect->adapt" model of docker swarm enables it to perform reconciliation when something goes wrong. For example, when a node in the swarm goes down it might take down running containers with it. The swarm will recognize this loss of containers, and will attempt to reschedule containers on available nodes in order to achieve the desired state for that service.

We are going to remove a node, and see tasks of our nginx1 service be rescheduled on other nodes automatically.

1. For the sake of clean output, first create a brand new service by copying the line below. We will change the name, and the publish port to avoid conflicts with our existing service. We will also add the `--replicas` command to scale the service with 5 instances.

    ```sh
    $ docker service create --detach=true --name nginx2 --replicas=5 --publish 81:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12
    aiqdh5n9fyacgvb2g82s412js
    ```

1. On Node1, use `watch` to watch the update from the output of `docker service ps`. Note "watch" is a linux utility and might not be available on other platforms.

    ```sh
    $ watch -n 1 docker service ps nginx2
    ok
    ```

    This should result in a window that looks like this:

    ```sh
    Every 1s: docker service ps nginx1 2                                                                                              2017-05-12 15:29:20

    ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STAT
    E            ERROR               PORTS
    6koehbhsfbi7         nginx2.1        nginx:1.12          node3               Running            Running 21 s
    econds ago
    dou2brjfr6lt        nginx2.2            nginx:1.12          node1               Running             Running 26 s
    econds ago
    8jc41tgwowph        nginx2.3            nginx:1.12          node2               Running             Running 27 s
    econds ago
    n5n8zryzg6g6        nginx2.4            nginx:1.12          node1               Running             Running 26 s
    econds ago
    cnofhk1v5bd8        nginx2.5            nginx:1.12          node2               Running             Running 27 s
    econds ago
    [node1] (loc
    ```

1. Click on Node3, and type the command to leave the swarm cluster.

    ```sh
    $ docker swarm leave
    ok
    ```

    This is the "nice" way to leave the swarm, but you can also kill the node and the following behavior will be the same.

1. Click on Node1 to watch the reconciliation in action. You should see that the swarm will attempt to get back to the declared state by rescheduling the containers that were running on node3 to node1 and node2 automatically.

    ```sh
    $ docker service ps nginx2
    ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
    jeq4604k1v9k        nginx2.1            nginx:1.12          node1               Running             Running 5 seconds ago
    6koehbhsfbi7         \_ nginx2.1        nginx:1.12          node3               Shutdown            Running 21 seconds ago
    dou2brjfr6lt        nginx2.2            nginx:1.12          node1               Running             Running 26 seconds ago
    8jc41tgwowph        nginx2.3            nginx:1.12          node2               Running             Running 27 seconds ago
    n5n8zryzg6g6        nginx2.4            nginx:1.12          node1               Running             Running 26 seconds ago
    cnofhk1v5bd8        nginx2.5            nginx:1.12          node2               Running             Running 27 seconds ago
    [node1] (loc
    ```

## Number of nodes

In this lab, our Docker Swarm cluster consists of one master, and two worker nodes. This configuration is not highly available. The manager node contains the necessary information to manage the cluster, so if this node goes down, the cluster will cease to function. For a production application, you will want to provision a cluster with multiple manager nodes to allow for manager node failures.

For manager nodes you want at least 3, but typically no more than 7. Managers implement the raft consensus algorithm, which requires that more than 50% of the nodes agree on the state that is being stored for the cluster. If you don't achieve >50%, the swarm will cease to operate correctly. For this reason, the following can be assumed about node failure tolerance.

* 3 manager nodes tolerates 1 node failure
* 5 manager nodes tolerates 2 node failures
* 7 manager nodes tolerates 3 node failures

It is possible to have an even number of manager nodes, but it adds no value in terms of the number of node failures. For example, 4 manager nodes would only tolerate 1 node failure, which is the same tolerance as a 3 manager node cluster. The more manager nodes you have, the harder it is to achieve a consensus on the state of a cluster.

While you typically want to limit the number of manager nodes to no more than 7, you can scale the number of worker nodes much higher than that. Worker nodes can scale up into the 1000's of nodes. Worker nodes communicate using the gossip protocol, which is optimized to be highly performant under large traffic and a large number of nodes.

If you are using [Play with Docker](http://play-with-docker.com), you can easily deploy multiple manager node clusters using the built in templates. Click the templates icon in the upper left to see what templates are available.

## Summary

In this lab, you got an introduction to problems that come with running container with production such as scheduling services across distributed nodes, maintaining high availability, implementing reconciliation, scaling, and logging. We used the orchestration tool that comes built-in to the Docker Engine- Docker Swarm, to address some of these issues.

Key Takeaways:

* The Docker Swarm schedules services using a declarative language. You declare the state, and the swarm attempts to maintain and reconcile to make sure the actual state == desired state
* Docker Swarm is composed of manager and worker nodes. Only managers can maintain the state of the swarm and accept commands to modify it. Workers have high scability and are only  used to run containers. By default managers can run containers as well.
* The routing mesh built into swarm means that any port that is published at the service level will be exposed on every node in the swarm. Requests to a published service port will be routed automatically to a container of the service that is running in the swarm.
* Many tools out there exist to help solve problems with orchestration containerized applications in production, include Docker Swarm, and the [IBM Kubernetes Service](https://cloud.ibm.com/kubernetes/catalog/create).
