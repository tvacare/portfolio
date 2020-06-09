---
title: Round Robin DNS with Docker
date: 2018-11-10T23:00:00.000+00:00
type: blog
author: Thiago Vacare
hero: "/images/docker-1.jpg"

---
For starters, we need to know what is round-robin DNS and how it works:

> Round-robin load balancing is one of the simplest methods for distributing client requests across a group of servers. Going down the list of servers in the group, the round-robin load balancer forwards a client request to each server in turn. When it eaches the end of the list, the load balancer loops back and goes down the list again sends the next request to the first listed server, the one after that to the second server, and so on). -- [What Is Round-Robin Load Balancing?](https://www.nginx.com/resources/glossary/round-robin-load-balancing/)

So knowing now what round-robin DNS is, let's do a practical example of how it works, using [Docker](https://www.docker.com/):

* The first step is to create a Docker network; I will call it `round-robin`:

```docker
  docker network create round-robin
```

* Now let's run two elastic-search containers (passing in the "--net" flag pointing to the network just created and the alias "search" that will be our DNS). There is no need to specify ports as we are not using the services outside the Docker network:

```docker
docker container run  --net round-robin --net-alias search --name elastic1 -d elasticsearch
```

Open another terminal:

```docker
docker container run  --net round-robin --net-alias search --name elastic2 -d elasticsearch
```

Use the following command to check if both containers are up and running:

```docker
docker container ls
```

    CONTAINER ID    IMAGE              COMMAND                  CREATED         STATUS        PORTS                NAMES
    da83546da7a9    elasticsearch      "/docker-entrypoint.…"   1 minutes ago   Up 1 minutes  9200/tcp, 9300/tcp   elastic1
    131221be44b2    elasticsearch      "/docker-entrypoint.…"   1 minutes ago   Up 1 minutes  9200/tcp, 9300/tcp   elastic2

* Let's run the command `nslookup` inside the alpine container, using the same network, and you'll see what happens:

```docker
docker container run --rm --net round-robin alpine nslookup search
```

Awesome! as you can see the DNS is pointing to both containers:

    Name:      search
    Address 1: 172.27.0.2 elastic1.round-robin
    Address 2: 172.27.0.3 elastic2.round-robin

* Try to `curl` from another container in the same network. You'll see that sometimes it points to a container and sometimes it points to the other (I used centos in the example, but you can use any other Linux image that contains `curl` command installed):

```docker
docker container run --rm --net round-robin centos curl -s search:9200
```

The result:

```json
{
  "name": "Cybelle",
  "cluster_name": "elasticsearch",
  "cluster_uuid": "OvAZoQS8RJyhNgDkwX3pXA",
  "version": {
    "number": "2.4.6",
    "build_hash": "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp": "2017-07-18T12:17:44Z",
    "build_snapshot": false,
    "lucene_version": "5.5.4"
  },
  "tagline": "You Know, for Search"
}
```

And if you run again:

```json
{
  "name": "Omega",
  "cluster_name": "elasticsearch",
  "cluster_uuid": "-by3GXTNQwOsAOgGSNRDkA",
  "version": {
    "number": "2.4.6",
    "build_hash": "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp": "2017-07-18T12:17:44Z",
    "build_snapshot": false,
    "lucene_version": "5.5.4"
  },
  "tagline": "You Know, for Search"
}
```

The round-robin DNS using Docker is working, and each time the algorithm points to a different container to load balance the requests.

There are many other algorithms used for load-balancing, such as Weighted Round Robin, Least Connection, Chained Failover, and others. I recommend you to see the differences between these algorithms and understand what fits best for your case.