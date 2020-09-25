# Lab 3 - Manage data in containers

## Overview

By default all files created inside a container are stored on a writable container layer. That means that:

* If the container no longer exists, the data is lost,
* The container's writable layer is tightly couples to the host machine, and
* To manage the file system, you need a storage driver that provides a union file system, using the Linux kernel. This extra abstraction reduces performance compared to `data volumes` which write directly to the filesystem.

Docker provides two options to store files in the host machine: `volumes` and `bind mounts`. If you're running Docker on Linux, you can also use a `tmpfs mount`, and with Docker on Windows you can also use a `named pipe`.

![Types of Mounts](../.gitbook/images/types-of-mounts.png)

* `Volumes` are stored in the host filesystem that is managed by Docker.
* `Bind mounts` are stored anywhere on the host system.
* `tmpfs mounts` are stored in the host memory only.

## Volumes

A `data volume` or `volume` is a directory that bypasses the `Union File System` of Docker. To understand what a Docker volume is, it helps to understand how layers and the filesystem work in Docker.

There are three types of volumes:

* anonymous volume,
* named volume, and 
* host volume.

### Anonymous Volume

Let's create an instance of a popular open source NoSQL database called CouchDB and use an `anonymous volume` to store the data files for the database.

To run an instance of CouchDB, use the CouchDB image from Docker Hub at [https://hub.docker.com/_/couchdb](https://hub.docker.com/_/couchdb). The docs say that the default for CouchDB is to `write the database files to disk on the host system using its own internal volume management`.

Run the following command,

```console
docker run -d -p 5984:5984 --name my-couchdb -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=passw0rd1 couchdb:3.1
```

CouchDB will create an anonymous volume and generated a hashed name. Check the volumes on your host system,

```console
$ docker volume ls
DRIVER    VOLUME NAME
local    f543c5319ebd96b7701dc1f2d915f21b095dfb35adbb8dc851630e098d526a50
```

Set an environment variable `VOLUME` with the value of the generated name,

```console
export VOLUME=f543c5319ebd96b7701dc1f2d915f21b095dfb35adbb8dc851630e098d526a50
```

And inspect the volume that was created, use the hash name that was generated for the volume,

```console
$ docker volume inspect $VOLUME
[
    {
        "CreatedAt": "2020-09-24T14:10:07Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/f543c5319ebd96b7701dc1f2d915f21b095dfb35adbb8dc851630e098d526a50/_data",
        "Name": "f543c5319ebd96b7701dc1f2d915f21b095dfb35adbb8dc851630e098d526a50",
        "Options": null,
        "Scope": "local"
    }
]
```

You see that Docker has created and manages a volume in the Docker host filesystem under `/var/lib/docker/volumes/$VOLUME_NAME/_data`. Note that this is not a path on the host machine, but a part of the Docker managed filesystem.

Create a new database `mydb` and insert a new document with a `hello world` message.

```console
curl -X PUT -u admin:passw0rd1 http://127.0.0.1:5984/mydb
curl -X PUT -u admin:passw0rd1 http://127.0.0.1:5984/mydb/1 -d '{"msg": "hello world"}'
```

You can share an anonymous volume with another container by using the `--volumes-from` option.

Create a `busybox` container with an anonymous volume mounted to a directory `/data` in the container and write a message to a log file.

```console
$ docker run -it --name busybox1 -v /data busybox sh
/ # echo "hello from busybox1" > /data/hi.log
/ # ls /data
hi.log
/ # exit
```

Make sure the container `busybox1` is stopped but not removed.

```console
$ docker ps -a
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS    PORTS    NAMES
437fb4a271c1    busybox    "sh"    18 seconds ago    Exited (0) 4 seconds ago    busybox1
```

Then create a second `busybox` container using the `--volumes-from` option,

```console
$ docker run --rm -it --name busybox2 --volumes-from busybox1 busybox sh
/ # ls -al /data
/ # cat /data/hi.log
hello from busybox1
/ # exit
```

Docker created the anynomous volume that you were able to share using the `--volumes-from` option.

```console
$ docker volume ls
DRIVER    VOLUME NAME
local    83a3275e889506f3e8ff12cd50f7d5b501c1ace95672334597f9a071df439493
```

Cleanup the existing volumes and container.

```console
docker stop my-couchdb
docker rm my-couchdb
docker rm busybox1
docker volume rm $(docker volume ls -q)
docker system prune -a
```

### Named Volume

A `named volume` and `anonymous volume` are similar in that Docker manages where they are located. However, a `named volume` can be referenced by name when mounting it to a container directory. This is helpful if you want to share a volume across multiple containers.

First, create a `named volume`,

```console
docker volume create my-couchdb-data-volume
```

Verify the volume was created,

```console
$ docker volume ls
DRIVER    VOLUME NAME
local    my-couchdb-data-volume
```

Now create the CouchDB container using the `named volume`,

```console
docker run -d -p 5984:5984 --name my-couchdb -v my-couchdb-data-volume:/opt/couchdb/data -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=passw0rd1 couchdb:3.1
```

Create a new database `mydb` and insert a new document with a `hello world` message.

```console
curl -X PUT -u admin:passw0rd1 http://127.0.0.1:5984/mydb
curl -X PUT -u admin:passw0rd1 http://127.0.0.1:5984/mydb/1 -d '{"msg": "hello world"}'
```

It now is easy to share the volume with another container. For instance, read the content of the volume using the `busybox` image, and share the `my-couchdb-data-volume` volume by mounting the volume to a directory in the `busybox` container.

```console
$ docker run --rm -it --name busybox -v my-couchdb-data-volume:/myvolume busybox sh
/ # ls -al /myvolume/
total 40
drwxr-xr-x    4 5984    5984    4096 Sep 24 17:11 .
drwxr-xr-x    1 root    root    4096 Sep 24 17:14 ..
drwxr-xr-x    2 5984    5984    4096 Sep 24 17:11 .delete
-rw-r--r--    1 5984    5984    8388 Sep 24 17:11 _dbs.couch
-rw-r--r--    1 5984    5984    8385 Sep 24 17:11 _nodes.couch
drwxr-xr-x    4 5984    5984    4096 Sep 24 17:11 shards
/ #
```

Cleanup,

```console
docker stop my-couchdb
docker rm my-couchdb
docker volume rm my-couchdb-data-volume
docker system prune -a
docker volume prune
```

### Host Volume

When you want to access the volume directory easily from the host machine, you can create a `host volume`.

Let's use a directory in the current working directory (indicated with the command `pwd`) called `data`, or choose your own data directory on the host machine, e.g. `/home/couchdb/data`. We let docker create the `$(pwd)/data` directory if it does not exist yet. We mount the `host volume` inside the CouchDB container to the container directory `/opt/couchdb/data`, which is the default data directory for CouchDB.

Run the following command,

```console
docker run -d -p 5984:5984 --name my-couchdb -v $(pwd)/data:/opt/couchdb/data -e COUCHDB_USER=admin -e COUCHDB_PASSWORD=passw0rd1 couchdb:3.1
```

Verify that a directory `data` was created,

```console
$ ls -al
total 16
drwxrwsrwx 3 root users 4096 Sep 24 16:27 .
drwxrwxr-x 1 root root  4096 Jul 16 20:04 ..
drwxr-sr-x 3 5984  5984 4096 Sep 24 16:27 data
```

and that CouchDB has created data files here,

```console
$ ls -al data
total 32
drwxr-sr-x 3 5984  5984 4096 Sep 24 16:27 .
drwxrwsrwx 3 root users 4096 Sep 24 16:27 ..
-rw-r--r-- 1 5984  5984 4257 Sep 24 16:27 _dbs.couch
drwxr-sr-x 2 5984  5984 4096 Sep 24 16:27 .delete
-rw-r--r-- 1 5984  5984 8385 Sep 24 16:27 _nodes.couch
```

Also check that now, no managed volume was created by docker, because we are now using a `host volume`.

```console
docker volume ls
```

Create a new database `mydb` and insert a new document with a `hello world` message.

```console
curl -X PUT -u admin:passw0rd1 http://127.0.0.1:5984/mydb
curl -X PUT -u admin:passw0rd1 http://127.0.0.1:5984/mydb/1 -d '{"msg": "hello world"}'
```

Note that CouchDB created a folder `shards`,

```console
$ ls -al data
total 40
drwxr-sr-x 4 5984  5984 4096 Sep 24 16:49 .
drwxrwsrwx 3 root users 4096 Sep 24 16:49 ..
-rw-r--r-- 1 5984  5984 8388 Sep 24 16:49 _dbs.couch
drwxr-sr-x 2 5984  5984 4096 Sep 24 16:49 .delete
-rw-r--r-- 1 5984  5984 8385 Sep 24 16:49 _nodes.couch
drwxr-sr-x 4 5984  5984 4096 Sep 24 16:49 shards
```

List the content of the `shards` directory,

```console
$ ls -al data/shards
total 16
drwxr-sr-x 4 5984 5984 4096 Sep 24 16:49 .
drwxr-sr-x 4 5984 5984 4096 Sep 24 16:49 ..
drwxr-sr-x 2 5984 5984 4096 Sep 24 16:49 00000000-7fffffff
drwxr-sr-x 2 5984 5984 4096 Sep 24 16:49 80000000-ffffffff
```

and the first shard,

```console
$ ls -al data/shards/00000000-7fffffff/
total 20
drwxr-sr-x 2 5984 5984 4096 Sep 24 16:49 .
drwxr-sr-x 4 5984 5984 4096 Sep 24 16:49 ..
-rw-r--r-- 1 5984 5984 8346 Sep 24 16:49 mydb.1600966173.couch
```

A [shard](https://docs.couchdb.org/en/stable/cluster/sharding.html) is a horizontal partition of data in a database. Partitioning data into shards and distributing copies of each shard to different nodes in a cluster gives the data greater durability against node loss. CouchDB automatically shards databases and distributes the subsets of documents among nodes.

Cleanup,

```console
docker stop my-couchdb
docker rm my-couchdb
sudo rm -rf $(pwd)/data
docker system prune -a
```

## Bind Mounts

The `mount` syntax is recommended by Docker over the `volume` syntax. Bind mounts have limited functionality compared to volumes. A file or directory is referenced by its full path on the host machine when mounted into a container. Bind mounts rely on the host machine’s filesystem having a specific directory structure available and you cannot use the Docker CLI to manage bind mounts. Note that bind mounts can change the host filesystem via processes running in a container.

Instead of using the `-v` syntax with three fields separated by colon separator (:), the `mount` syntax is more verbose and uses multiple `key-value` pairs:

* type: bind, volume or tmpfs,
* source: path to the file or directory on host machine,
* destination: path in container,
* readonly,
* bind-propagation: rprivate, private, rshared, shared, rslave, slave,
* consistency: consistent, delegated, cached,
* mount.

```console
mkdir data
docker run -it --name busybox --mount type=bind,source="$(pwd)"/data,target=/data busybox sh
/ # echo "hello busybox" > /data/hi.txt
/ # exit
cat data/hi.txt
```
