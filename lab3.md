# Lab 3- Introduction to Orchestration

© Copyright IBM Corporation 2017

IBM, the IBM logo and ibm.com are trademarks of International Business Machines Corp., registered in many jurisdictions worldwide. Other product and service names might be trademarks of IBM or other companies. A current list of IBM trademarks is available on the Web at &quot;Copyright and trademark information&quot; at www.ibm.com/legal/copytrade.shtml.


This document is current as of the initial date of publication and may be changed by IBM at any time.

The information contained in these materials is provided for informational purposes only, and is provided AS IS without warranty of any kind, express or implied. IBM shall not be responsible for any damages arising out of the use of, or otherwise related to, these materials. Nothing contained in these materials is intended to, nor shall have the effect of, creating any warranties or representations from IBM or its suppliers or licensors, or altering the terms and conditions of the applicable license agreement governing the use of IBM software. References in these materials to IBM products, programs, or services do not imply that they will be available in all countries in which IBM operates. This information is based on current IBM product plans and strategy, which are subject to change by IBM without notice. Product release dates and/or capabilities referenced in these materials may change at any time at IBM&#39;s sole discretion based on market opportunities or other factors, and are not intended to be a commitment to future product or feature availability in any way.

# Overview

So far you have learned how to run applications using docker on your local machine, but what about running dockerized applications in production? There are a number of problems that come with building an application for production: scheduling services across distributed nodes, maintaining high availability, implementing reconciliation, scaling, and logging... just to name a few.

There are several orchestration solutions out there that help you solve some of these problems. One example is the [IBM Bluemix Container Service](https://www.ibm.com/cloud-computing/bluemix/containers) which uses [Kubernetes](https://kubernetes.io/) to run containers in production. 

Before we introduce you to Kubernetes, we will teach you how to orchestrate applications using Docker Swarm. Docker Swarm is the orchestration tool that comes built-in to the Docker Engine.

## Prerequisites

In order to complete a lab about orchestrating an application that is deployed across multiple hosts, you need... well, multiple hosts.  To make things easier, for this lab we will be using the multi-node support provided by http://play-with-docker.com. This is the easiest way to test out docker swarm, without having to deal with installing docker on multiple hosts yourself.

# Step 1: Create your first swarm

In this step, we will create our first swarm using play-with-docker.

1. Navigate to http://play-with-docker.com

2. Click "add new instance" on the lefthand side three times to create three nodes

Our first swarm cluster will have three nodes.

3. Initialize the swarm on node 1
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

4. On both node2 and node3, copy and run the `docker swarm join` command that was outputted to YOUR console by the last command.

You now have a three node swarm!

5. Back on node1, run `docker node ls` to verify your 3 node cluster.
```sh
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
7x9s8baa79l29zdsx95i1tfjp     node3               Ready               Active
x223z25t7y7o4np3uq45d49br     node2               Ready               Active
zdqbsoxa6x1bubg3jyjdmrnrn *   node1               Ready               Active              Leader
```

This command outputs the three nodes in our swarm. The * next to the ID of the node represents the node that handled that specific command (`docker node ls` in this case).  

Our node has 1 manager and 2 workers. Managers handle commands and manage the state of the swarm. Workers cannot handle commands and are simply used to run containers at scale. By default, managers are also used to run containers.

Note, all `docker service` commands for the rest of this lab need to be executed on the manager node (Node1).

**Note:** You can control a docker swarm remotely by connecting to the docker engine of the manager via the remote API or activating a remote host from your local docker installation (using the `$DOCKER_HOST` and `$DOCKER_CERT_PATH` environment variables).

# Step 2: Deploy your first service

Now that we have our 3 node swarm initialized, let's deploy some containers. To run containers on a Docker Swarm, we want to create a service. A service is an abstraction that represents multiple containers of the same image deployed across a distributed cluster.

Let's do a simple example using Nginx. For now we will create a service with just 1 running container, but we will scale up later.
 
1. Deploy a service using Nginx
```sh
$ docker service create --detach=true --name nginx1 --publish 80:80  --mount source=/etc/hostname,target=/usr/share/nginx/html/index.html,type=bind,ro nginx:1.12
2i81uxklszm6shjpmupb1hrvi
```

This above statement is *declarative*, and docker swarm will actively try to maintain the state declared in this command unless explicitly changed via another `docker service` command. This behavior comes in handy when nodes go down, for example, and containers are automatically rescheduled on other nodes. We will see a demonstration of that a little later on in this lab.

The `--mount` command is a neat trick to have nginx print out the hostname of the node it's running on. This will come in handy when we want to see which node in our swarm serves our request a little later on in the lab.

We are using nginx tag "1.12" in this command. We will demonstrate a rolling update with version 1.13 later in this lab.

The `--publish` command makes use of the swarm's built in *routing mesh*. In this case port 80 is exposed on *every node in the swarm*. The routing mesh will route a request coming in on port 80 to one of the nodes running the container.

2. Inspect the service

You can use `docker service ls` to inspect the service you just created.

```sh
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
a46cgocdfjj4        nginx1              replicated          1/1                 nginx:1.12        *:80->80/tcp
```

3. Check out the running container of the service

To take a deeper look at the running tasks, you can use `docker service ps`. A task is yet another abstraction using in docker swarm that represents the running instances of a service. In this case, there is a 1-1 mapping between a task and a container. 

```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
k76andqr9197        nginx1.1            nginx:1.12        node1               Running             Running 8 seconds ago
```

If you happen to know which node your container is running on (you can see which node based on the output from `docker service ps`), you can use `docker container ls` to see the container running on that specific node.

4. Test the service

Because of the routing mesh, we can send a request to any node of the swarm on port 80. This request will be automatically routed to the one node that is running our nginx container.

Try this on each node:

```sh
$ curl localhost:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

# Step 3: Scale your service

In production we may need to handle large amounts of traffic to our application. So let's scale!

1. Update your service with an updated number of replicas

We are going to use the `docker service` command to update the nginx service we created earlier to include 5 replicas. This is defining a new state for our service.
```sh
$ docker service update --replicas=5 nginx1
nginx1
```
As soon as this command is run the following happens:
1) The state of the service is updated to 5 replicas (which is stored in the swarms internal storage).
2) Docker swarm recognizes that the number of replicas that is scheduled now does not match the declared state of 5.
3) Docker swarm schedules 5 more tasks (containers) in an attempt to meet the declared state for the service.

This swarm is actively checking to see if the desired state is equal to actual state, and will attempt to reconcile if needed.

2. Check the running instances

After a few seconds, you should see that the swarm did its job, and successfully started 9 more containers. Notice that the containers are scheduled across all three nodes of the cluster. The default placement strategy that is used to decide where new containers are to be run is "emptiest node", but that can be changed based on your need.

```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
k76andqr9197        nginx1.1            nginx:1.12        node1               Running             Running 5 minutes ago
zfg0xvlnthie        nginx1.2            nginx:1.12        node3               Running             Running 13 seconds ago
7u5kosgme345        nginx1.3            nginx:1.12        node3               Running             Running 12 seconds ago
tzzki7jial9d        nginx1.4            nginx:1.12        node2               Running             Running 14 seconds ago
cdueodzlm8bs        nginx1.5            nginx:1.12        node3               Running             Running 12 seconds ago
qvl1jnyswo84        nginx1.6            nginx:1.12        node3               Running             Running 13 seconds ago
1lwb1nt5zmh0        nginx1.7            nginx:1.12        node2               Running             Running 12 seconds ago
cqvrq868lp8i        nginx1.8            nginx:1.12        node1               Running             Running 14 seconds ago
0yc170so5cld        nginx1.9            nginx:1.12        node2               Running             Running 13 seconds ago
rueelnboyi6j        nginx1.10           nginx:1.12        node1               Running             Running 14 seconds ago
```

3. Send a bunch of requests to http://localhost:80

The `--publish 80:80` is still in effect for this service, that was not changed when we ran `docker service update`. However, now when we send requests on port 80, the routing mesh has multiple containers in which to route requests to. The routing mesh acts as a load balancer for these containers, alternating where it routes requests to. 

Let's try it out by curling multiple times. Note, that it doesn't matter which node you send the requests. There is no connection between the node that receives the request, and the node that that request is routed to.  

```sh
$ curl localhost:80
$ curl localhost:80
$ curl localhost:80
$ curl localhost:80
$ curl localhost:80
```

**Limits of the routing Mesh**
The routing mesh can only publish one service on port 80. If you want multiple services exposed on port 80, then you can use an external application load balancer outside of the swarm to accomplish this.

4. Check the aggregated logs for the service

One easy way to see which nodes those requests were routed to is to check the aggregated logs. We can get aggregated logs for the service using `docker service logs [service name]`. This aggregates the output from every running container, i.e. the output from `docker container logs [container name]`.

```sh
$ docker service logs nginx1
nginx1.1.k76andqr9197@node1     | 10.255.0.2 - - [12/May/2017:14:22:22 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
nginx1.1.k76andqr9197@node1     | 10.255.0.3 - - [12/May/2017:14:22:40 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
nginx1.3.7u5kosgme345@node3     | 10.255.0.2 - - [12/May/2017:14:27:30 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
nginx1.6.qvl1jnyswo84@node3     | 10.255.0.2 - - [12/May/2017:14:27:33 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
nginx1.9.0yc170so5cld@node2     | 10.255.0.2 - - [12/May/2017:14:27:32 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
nginx1.7.1lwb1nt5zmh0@node2     | 10.255.0.2 - - [12/May/2017:14:27:28 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
nginx1.5.cdueodzlm8bs@node3     | 10.255.0.2 - - [12/May/2017:14:27:31 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.52.1" "-"
```
Based on these logs we can see that each request was served by a different container.

In addition to seeing whether the request was sent to Node1, Node2, or Node3, you can also see which container on each node that it was sent to. For example `nginx1.5` means that request was sent to container with that same name as indicated in the output of `docker service ps nginx1`.

# Step 4: Rolling Updates

Now that we have our service deployed, lets demonstrate a release of our application. We are going to update the version of Nginx to version "1.13". To do this update we are going to use the `docker service update` command.

```sh
docker service update --image nginx:1.13 nginx1
```

This will trigger a rolling update of the swarm. Quickly type in `docker service ps nginx1` over and over to see the updates in real time. You can fine tune the rolling update using these options:
`--update-parallelism` will dictate the number of containers to update at once. (defaults to 1)
`--update-delay` will dictate the delay between finishing updating a set of containers before moving on to the next set.

After a few seconds, run `docker service ps nginx1` to see all the images have been updated to nginx:1.13. 

```sh
oj1tvux0yccn        nginx1.4            nginx:1.13          node3               Running             Running 9 seconds ago

v5op2bz1f0mt         \_ nginx1.4        nginx:1.12          node3               Shutdown            Shutdown 10 seconds ago

94o4gwvixtyi        nginx1.5            nginx:1.13          node2               Running             Running 14 seconds ago

rloi3m4ytk99         \_ nginx1.5        nginx:1.12          node2               Shutdown            Shutdown 17 seconds ago

dl0cu17h0fr7        nginx1.6            nginx:1.13          node1               Running             Running 25 seconds ago

0mj2qavxu91p         \_ nginx1.6        nginx:1.12          node1               Shutdown            Shutdown 27 seconds ago

olrwocmp8oo3        nginx1.7            nginx:1.13          node1               Running             Running 33 seconds ago

lf7hfyqzalgg         \_ nginx1.7        nginx:1.12          node1               Shutdown            Shutdown 36 seconds ago

taxuoctg5p3p        nginx1.8            nginx:1.13          node2               Running             Running 41 seconds ago

eotvoinl4xwm         \_ nginx1.8        nginx:1.12          node2               Shutdown            Shutdown 44 seconds ago

bs946b1gmwca        nginx1.9            nginx:1.13          node3               Running             Running 21 seconds ago

i6f109cu7ap7         \_ nginx1.9        nginx:1.12          node3               Shutdown            Shutdown 23 seconds ago

nof6eyz9zlep        nginx1.10           nginx:1.13          node1               Running             Running 11 seconds ago

sp88fcvgfj3p         \_ nginx1.10       nginx:1.12          node1               Shutdown            Shutdown 12 seconds ago
```

You have successfully updated your app to the latest version of nginx!

# Step :5 Reconciliation

In the previous step, we updated the state of our service using `docker service update`. We saw Docker Swarm in action as it recognized the mismatch between desired state and actual state, and attempted to solve this issue. 

The "inspect->adapt" model of docker swarm enables it to perform reconciliation when something goes wrong. For example, when a node in the swarm goes down it might take down running containers with it. The swarm will recognize this loss of containers, and will attempt to reschedule containers on available nodes in order to achieve the desired state for that service.
 
We are going to remove a node, and see tasks of our nginx1 service be rescheduled on other nodes automatically.

1. On Node1, use `watch` to watch the update from the output of `docker service ps`.
```sh
$ watch -n 1 docker service ps nginx1
```
This should result in a window that looks like this:
```sh
Every 1s: docker service ps nginx1                                                                                                2017-05-12 15:29:20

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
o68mbojcmqqq        nginx1.1            nginx:1.12        node1               Running             Running 9 minutes ago
81icw3cvbvgx        nginx1.2            nginx:1.12        node2               Running             Running 5 minutes ago
bmw0eu84c8qz        nginx1.3            nginx:1.12        node3               Running             Running 5 minutes ago
uqmn9kuk79pw        nginx1.4            nginx:1.12        node2               Running             Running 5 minutes ago
mwy6f9jtl3ss        nginx1.5            nginx:1.12        node2               Running             Running 5 minutes ago
dbapn72nlwp6        nginx1.6            nginx:1.12        node2               Running             Running 5 minutes ago
lq0rp88inh2x        nginx1.7            nginx:1.12        node1               Running             Running 6 minutes ago
nvmpbh0m57rr        nginx1.8            nginx:1.12        node3               Running             Running 5 minutes ago
xbm3cl1huvhg        nginx1.9            nginx:1.12        node3               Running             Running 5 minutes ago
y9ib99s6he7n        nginx1.10           nginx:1.12        node1               Running             Running 6 minutes ago
```

2. Click on Node3, and type the command to leave the swarm cluster.
```sh
$ docker swarm leave
```
This is the "nice" way to leave the swarm, but you can also kill the node and the following behavior will be the same.

3. Click on Node1 to watch the reconciliation in action. You should see that the swarm will attempt to get back to the declared state by rescheduling the containers that were running on node3 to node1 and node2 automatically.
```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               P
ORTS
o68mbojcmqqq        nginx1.1            nginx:1.12        node1               Running             Running 11 minutes ago
81icw3cvbvgx        nginx1.2            nginx:1.12        node2               Running             Running 8 minutes ago
nfypiigm47w6        nginx1.3            nginx:1.12        node1               Running             Running about a minute ago
bmw0eu84c8qz         \_ nginx1.3        nginx:1.12        node3               Shutdown            Running about a minute ago
uqmn9kuk79pw        nginx1.4            nginx:1.12        node2               Running             Running 8 minutes ago
mwy6f9jtl3ss        nginx1.5            nginx:1.12        node2               Running             Running 8 minutes ago
dbapn72nlwp6        nginx1.6            nginx:1.12        node2               Running             Running 8 minutes ago
lq0rp88inh2x        nginx1.7            nginx:1.12        node1               Running             Running 8 minutes ago
lmo8xk73fben        nginx1.8            nginx:1.12        node1               Running             Running about a minute ago
nvmpbh0m57rr         \_ nginx1.8        nginx:1.12        node3               Shutdown            Running about a minute ago
x09i3t60sbsx        nginx1.9            nginx:1.12        node2               Running             Running about a minute ago
xbm3cl1huvhg         \_ nginx1.9        nginx:1.12        node3               Shutdown            Running about a minute ago
y9ib99s6he7n        nginx1.10           nginx:1.12        node1               Running             Running 8 minutes ago
```

# How many nodes?
Managers and workers scale differently. 

For managers you want at least 3, but typically no more than 7. Managers implement the raft consensus algorithm, which requires that more than 50% of the nodes agree on the state that is being stored for the cluster. If you don't achieve >50%, the swarm will cease to operate correctly. For this reason, the following can be assumed about node failure tolerance.

- 3 manager nodes tolerates 1 node failure 
- 5 manager nodes tolerates 2 node failures
- 7 manager nodes tolerates 3 node failures

It is possible to have an even number of manager nodes, but it adds no value in terms of the number of node failures. For example, 4 manager nodes would only tolerate 1 node failure, which is the same tolerance as a 3 manager node cluster. The more manager nodes you have, the harder it is to achieve a consensus on the state of a cluster.

Worker nodes, on the other hand, can scale up into the 1000's of nodes. Worker nodes communicate using the gossip protocol, which is optimized to be highly performant under large traffic and a large number of nodes.


# Summary

In this lab, you got an introduction to problems that come with running container with production such as scheduling services across distributed nodes, maintaining high availability, implementing reconciliation, scaling, and logging. We used the orchestration tool that comes built-in to the Docker Engine- Docker Swarm, to address some of these issues.

Key Takeaways
- The Docker Swarm schedules services using a declarative language. You declare the state, and the swarm attempts to maintain and reconcile to make sure the actual state == desired state
- Docker Swarm is composed of manager and worker nodes. Only managers can maintain the state of the swarm and accept commands to modify it. Workers have high scability and are only  used to run containers. By default managers can run containers as well.
- The routing mesh built into swarm means that any port that is published at the service level will be exposed on every node in the swarm. Requests to a published service port will be routed automatically to a container of the service that is running in the swarm.
- Many tools out there exist to help solve problems with orchestration containerized applications in production, include Docker Swarm, and the [IBM Bluemix Container Service](https://www.ibm.com/cloud-computing/bluemix/containers).

