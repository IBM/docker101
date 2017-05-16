# Lab 3- Introduction to Orchestration

© Copyright IBM Corporation 2017

IBM, the IBM logo and ibm.com are trademarks of International Business Machines Corp., registered in many jurisdictions worldwide. Other product and service names might be trademarks of IBM or other companies. A current list of IBM trademarks is available on the Web at &quot;Cfopyright and trademark information&quot; at www.ibm.com/legal/copytrade.shtml.

This document is current as of the initial date of publication and may be changed by IBM at any time.

The information contained in these materials is provided for informational purposes only, and is provided AS IS without warranty of any kind, express or implied. IBM shall not be responsible for any damages arising out of the use of, or otherwise related to, these materials. Nothing contained in these materials is intended to, nor shall have the effect of, creating any warranties or representations from IBM or its suppliers or licensors, or altering the terms and conditions of the applicable license agreement governing the use of IBM software. References in these materials to IBM products, programs, or services do not imply that they will be available in all countries in which IBM operates. This information is based on current IBM product plans and strategy, which are subject to change by IBM without notice. Product release dates and/or capabilities referenced in these materials may change at any time at IBM&#39;s sole discretion based on market opportunities or other factors, and are not intended to be a commitment to future product or feature availability in any way.

# Overview

In this lab, you will create your docker swarm and schedule services. You will demonstrate some of the problems needed of an orchestration features such as service scheduling, scaling and reconciliation.

## Prerequisites

For this lab, we will be using the multi-node support provided by http://play-with-docker.com

# Step 1: Create your first swarm

1. Navigate to http://play-with-docker.com

2. Click "add new instance" on the lefthand side three times to create three nodes

3. Initialize the swarm on node 1
```sh
$ docker swarm init --advertise-addr eth0
Swarm initialized: current node (zdqbsoxa6x1bubg3jyjdmrnrn) is now a manager.
```
Take note of the ouput: you will use the generated command in the next step.
4. On both node2 and node3, copy and run the `docker swarm join` command that was outputted to the console by the last command

5. Back on node1, run `docker node ls` to verify your 3 node cluster.
```sh
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
7x9s8baa79l29zdsx95i1tfjp     node3               Ready               Active
x223z25t7y7o4np3uq45d49br     node2               Ready               Active
zdqbsoxa6x1bubg3jyjdmrnrn *   node1               Ready               Active              Leader
```
Please note, that all swarm commands must be sent to a manager of the swarm. For this lab, all commands will be sent to Node1.
# Step 2: Deploy your first service

1. Deploy a service using Nginx
```sh
$ docker service create --name nginx1 --publish 80:80 nginx
2i81uxklszm6shjpmupb1hrvi
```

2. Inspect the service
```sh
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
a46cgocdfjj4        nginx1              replicated          1/1                 nginx:latest        *:80->80/tcp
```

3. Check out the running container of the service
```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
k76andqr9197        nginx1.1            nginx:latest        node1               Running             Running 8 seconds ago
```
You can also see the container running if you run `docker container ls` on the node indicated in the output above.

4. Test the service

On **any** node, send a request to nginx running on port 80
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
This command works on any node because the routing mesh that is provided by Docker Swarm automatically redirects traffic to a node that is running the container.

# Step 3: Scale your service

1. Update your service with an updated replicas
```sh
$ docker service update --replicas=10 nginx1
nginx1
```

2. Check the running instances
```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
k76andqr9197        nginx1.1            nginx:latest        node1               Running             Running 5 minutes ago
zfg0xvlnthie        nginx1.2            nginx:latest        node3               Running             Running 13 seconds ago
7u5kosgme345        nginx1.3            nginx:latest        node3               Running             Running 12 seconds ago
tzzki7jial9d        nginx1.4            nginx:latest        node2               Running             Running 14 seconds ago
cdueodzlm8bs        nginx1.5            nginx:latest        node3               Running             Running 12 seconds ago
qvl1jnyswo84        nginx1.6            nginx:latest        node3               Running             Running 13 seconds ago
1lwb1nt5zmh0        nginx1.7            nginx:latest        node2               Running             Running 12 seconds ago
cqvrq868lp8i        nginx1.8            nginx:latest        node1               Running             Running 14 seconds ago
0yc170so5cld        nginx1.9            nginx:latest        node2               Running             Running 13 seconds ago
rueelnboyi6j        nginx1.10           nginx:latest        node1               Running             Running 14 seconds ago
```
The built-in routing mesh will now route requests to any of these running containers. 

3. Send a bunch of requests to http://localhost:80
```sh
$ curl localhost:80
$ curl localhost:80
$ curl localhost:80
$ curl localhost:80
$ curl localhost:80
```

4. To see which nodes these requests were routed to, check the aggregated logs for the service
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
In addition to seeing whether the request was sent to Node1, Node2, or Node3, you can also see which container on each node that it was sent to. For example `nginx1.5` means that request was sent to container with that same name as indicated in the output of `docker service ps nginx1`.
# Step 4: Reconciliation

We are going to remove a node, and see tasks of our nginx1 service be rescheduled on other nodes automatically.

1. We are going to use `watch` to watch the update from the output of `docker service ps`.
```sh
$ watch -n 1 docker service ps nginx1
```
This should result in a window that looks like this:
```sh
Every 1s: docker service ps nginx1                                                                                                2017-05-12 15:29:20

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
o68mbojcmqqq        nginx1.1            nginx:latest        node1               Running             Running 9 minutes ago
81icw3cvbvgx        nginx1.2            nginx:latest        node2               Running             Running 5 minutes ago
bmw0eu84c8qz        nginx1.3            nginx:latest        node3               Running             Running 5 minutes ago
uqmn9kuk79pw        nginx1.4            nginx:latest        node2               Running             Running 5 minutes ago
mwy6f9jtl3ss        nginx1.5            nginx:latest        node2               Running             Running 5 minutes ago
dbapn72nlwp6        nginx1.6            nginx:latest        node2               Running             Running 5 minutes ago
lq0rp88inh2x        nginx1.7            nginx:latest        node1               Running             Running 6 minutes ago
nvmpbh0m57rr        nginx1.8            nginx:latest        node3               Running             Running 5 minutes ago
xbm3cl1huvhg        nginx1.9            nginx:latest        node3               Running             Running 5 minutes ago
y9ib99s6he7n        nginx1.10           nginx:latest        node1               Running             Running 6 minutes ago
```

2. Click on Node3, and type the command to leave the swarm cluster.
```sh
$ docker swarm leave
```

3. Click on Node1 to watch the magic. You should see that the containers that were running on node3 will be rescheduled to node1 and node2 automatically.
```sh
$ docker service ps nginx1
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               P
ORTS
o68mbojcmqqq        nginx1.1            nginx:latest        node1               Running             Running 11 minutes ago
81icw3cvbvgx        nginx1.2            nginx:latest        node2               Running             Running 8 minutes ago
nfypiigm47w6        nginx1.3            nginx:latest        node1               Running             Running about a minute ago
bmw0eu84c8qz         \_ nginx1.3        nginx:latest        node3               Shutdown            Running about a minute ago
uqmn9kuk79pw        nginx1.4            nginx:latest        node2               Running             Running 8 minutes ago
mwy6f9jtl3ss        nginx1.5            nginx:latest        node2               Running             Running 8 minutes ago
dbapn72nlwp6        nginx1.6            nginx:latest        node2               Running             Running 8 minutes ago
lq0rp88inh2x        nginx1.7            nginx:latest        node1               Running             Running 8 minutes ago
lmo8xk73fben        nginx1.8            nginx:latest        node1               Running             Running about a minute ago
nvmpbh0m57rr         \_ nginx1.8        nginx:latest        node3               Shutdown            Running about a minute ago
x09i3t60sbsx        nginx1.9            nginx:latest        node2               Running             Running about a minute ago
xbm3cl1huvhg         \_ nginx1.9        nginx:latest        node3               Shutdown            Running about a minute ago
y9ib99s6he7n        nginx1.10           nginx:latest        node1               Running             Running 8 minutes ago
```

# Summary

In this lab, you experienced some of the problems that come with orchestrating multi-container applications across multiple nodes such as building applications with requirements for high availability, reconciliation and scaling.

