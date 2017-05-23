# Lab 1 - Running Your First Container

© Copyright IBM Corporation 2017

IBM, the IBM logo and ibm.com are trademarks of International Business Machines Corp., registered in many jurisdictions worldwide. Other product and service names might be trademarks of IBM or other companies. A current list of IBM trademarks is available on the Web at &quot;Copyright and trademark information&quot; at www.ibm.com/legal/copytrade.shtml.

This document is current as of the initial date of publication and may be changed by IBM at any time.

The information contained in these materials is provided for informational purposes only, and is provided AS IS without warranty of any kind, express or implied. IBM shall not be responsible for any damages arising out of the use of, or otherwise related to, these materials. Nothing contained in these materials is intended to, nor shall have the effect of, creating any warranties or representations from IBM or its suppliers or licensors, or altering the terms and conditions of the applicable license agreement governing the use of IBM software. References in these materials to IBM products, programs, or services do not imply that they will be available in all countries in which IBM operates. This information is based on current IBM product plans and strategy, which are subject to change by IBM without notice. Product release dates and/or capabilities referenced in these materials may change at any time at IBM&#39;s sole discretion based on market opportunities or other factors, and are not intended to be a commitment to future product or feature availability in any way.

# Overview

In this lab, you will create your first Docker container. Docker containers are isolated via linux namespaces. In this lab you will be able to inspect a container to see the effect of linux namespaces. 

One advantage of the isolation properties of containers is that you can run multiple containers on the same host without conflicts. In this lab, you will see this in action by running multiple containers on your local machine.

## Prerequisites

Completed Lab 0: You must have docker installed, or be using http://play-with-docker.com.


# Step 1: Run your first container

First, we are going to use the Docker CLI to run your first container. We will run an ubuntu container, and run the `bash` process inside that container. 

1. Open a terminal on your local computer

2. Run `docker run -it ubuntu bash`

Use the `docker run` command to run the ubuntu container with the `bash` command. The `-it` flags start the container in "interactive" mode, so that we can interact with the container directly from our local terminal.

```sh
$ docker run -it ubuntu bash
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
aafe6b5e13de: Pull complete 
0a2b43a72660: Pull complete 
18bdd1e546d2: Pull complete 
8198342c3e05: Pull complete 
f56970a44fd4: Pull complete 
Digest: sha256:f3a61450ae43896c4332bda5e78b453f4a93179045f20c8181043b26b5e79028
Status: Downloaded newer image for ubuntu:latest
root@4fb6b0acc681:/ 
```
Notice the change in the prefix of your terminal. e.g. `root@4fb6b0acc681:/`. This is one indication that you are "inside" of a container. In the case of the output above, I am running as the `root` user inside a container with ID: `4fb6b0acc681`. Your container id will be different.

3. Run `ps -ef` to inspect the running processes inside the container.

```sh
root@4fb6b0acc681:/ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 21:43 ?        00:00:00 bash
root        10     1  0 21:44 ?        00:00:00 ps -ef
```

Containers use linux namespaces to provide isolation of system resources from other containers or the host. The PID namespace provides isolation for process IDs. If you run `ps -ef` while inside the container, you will notice that it prints out only the processes within the PID namespace of the container, which is much different than what you can see if you ran `ps` on the host.

4. Run `exit` to exit the container

Notice that after running `exit`, the prefix in your terminal will revert back to what it normally is on your host. i.e. you are not inside the container anymore.

```sh
root@4fb6b0acc681:/ exit
exit
$ #Outside the container now
```
5. Run `ps -ef` again to see the processes running on the host



```sh
$ ps -ef
# Lots of processes!
```
Just for comparison, running `ps -ef` outside the container will show you all the processes running on your host. 

PID is just one of the linux namespaces that provides containers with isolation to system resources. Other linux namespaces include:
- MNT - Mount and unmount directories without affecting other namespaces
- NET - Containers have their own network stack
- IPC - Isolated interprocess communication mechanisms such as message queues.
- User - Isolated view of users on the system
- UTC - Set hostname and domain name per container

These namespaces together provide the isolation for containers that allow them to run together securely and without conflict with other containers running on the same system. Next we will demonstrate this benefit by running multiple containers on the same host.


# Step 2: Run Multiple Containers

1. Run `docker run –-detach nginx`

Now we are going to run a new container: `nginx`. We will use the `--detach` flag to detach the container process from our terminal.

```sh
$ docker run --detach nginx
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
36a46ebd5019: Pull complete 
57168433389f: Pull complete 
332ec8285c50: Pull complete 
Digest: sha256:c15f1fb8fd55c60c72f940a76da76a5fccce2fefa0dd9b17967b9e40b0355316
Status: Downloaded newer image for nginx:latest
5e1bf0e6b926bd73a66f98b3cbe23d04189c16a43d55dd46b8486359f6fdf048
```

Since this is the first time you are running the nginx container, it will pull down the nginx image from the docker hub. Think of a docker `image` as the blueprint from which you can run many docker `containers`.

2. Repeat step 1 several more times, note a new container ID each time. Use the `-d` shorthand of `--detach`.

```sh
$ docker run -d nginx
31617fdd8e5f584c51ce182757e24a1c9620257027665c20be75aa3ab6591740
$ docker run -d nginx
60abd5ee65b1e2732ddc02b971a86e22de1c1c446dab165462a08b037ef7835c
$ docker run -d nginx
7872fd96ea4695795c41150a06067d605f69702dbcb9ce49492c9029f0e1b44b
```

Now that you have the image stored locally. You can create many instances of the nginx container very quickly.


3. Run `docker container ls` to see all the running containers

You now have several nginx containers all running on your local machine. Use the `docker container ls` command to list them. 

```sh
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
7872fd96ea46        nginx               "nginx -g 'daemon ..."   23 seconds ago       Up 22 seconds       80/tcp              laughing_northcutt
60abd5ee65b1        nginx               "nginx -g 'daemon ..."   24 seconds ago       Up 23 seconds       80/tcp              xenodochial_johnson
31617fdd8e5f        nginx               "nginx -g 'daemon ..."   25 seconds ago       Up 24 seconds       80/tcp              elegant_thompson
5e1bf0e6b926        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              kind_stonebraker
```

4. Run `docker image ls` to see the images you have downloaded locally thus far. So far you should have downloaded two images: `nginx`, and `ubuntu`.

```sh
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              46102226f2fd        2 weeks ago         109MB
ubuntu              latest              f7b3f317ec73        2 weeks ago         117MB
```
Running multiple containers on the same host gives us the ability to fully utilize the resources (cpu, memory, etc) available on single host. Because these containers are isolated, we don't have to worry containers running on the same host conflicting with each other.

# Step 3: Clean Up

Completing this lab results in a bunch of running containers on your host. Let's clean these up.

1. Run `docker container stop [container id]` for each container that is running

First get a list of the containers running using `docker container ls`.
```sh
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
7872fd96ea46        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              laughing_northcutt
60abd5ee65b1        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              xenodochial_johnson
31617fdd8e5f        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              elegant_thompson
5e1bf0e6b926        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              kind_stonebraker
```
Then run `docker container stop [container id]` for each container in the list.
```sh
$ docker container stop 787 60a 316 5e1
787
60a
316
5e1
```

2. Remove the stopped containers

`docker system prune` is a really handy command to clean up your system. It will remove any stopped containers, unused volumes and networks, and dangling images.

```sh
$ docker system prune
WARNING! This will remove:
        - all stopped containers
        - all volumes not used by at least one container
        - all networks not used by at least one container
        - all dangling images
Are you sure you want to continue? [y/N] y
Deleted Containers:
7872fd96ea4695795c41150a06067d605f69702dbcb9ce49492c9029f0e1b44b
60abd5ee65b1e2732ddc02b971a86e22de1c1c446dab165462a08b037ef7835c
31617fdd8e5f584c51ce182757e24a1c9620257027665c20be75aa3ab6591740
5e1bf0e6b926bd73a66f98b3cbe23d04189c16a43d55dd46b8486359f6fdf048
4fb6b0acc681bf72dfd81e81c5945d7e430d1957494bff2dbaa5561f312b087d

Total reclaimed space: 12B
```

# Summary

In this lab, you created your first ubuntu and nginx containers.

Key Takeaways
- Containers are composed of linux namespaces that provide isolation from other containers and the host.
- Containers are fast and lightweight, which gives you the flexibility of starting multiple containers quickly.
- Because of the two statements above, you can schedule many containers on a single host without worrying about conflicting dependencies. This gives you the advantage of making full use of the resources allocated to the host.

