---
title: "Rootless Reverse Proxy with Podman"
date: 2021-07-31
tags:
  - podman
  - containers
  - networking
  - reverse-proxy
draft: true
---

I've switched almost entirely to [Podman](Podman.io) for all of my container needs, thanks primarily to it's
rootless-first structure, but there was still one use case with some hurdles - rootless reverse proxy to other pods (not
just containers) running on the system.
There are two issues that stand out for this use case:

* Service discovery

    [Traefik + Docker/docker-compose](https://doc.traefik.io/traefik/providers/docker/) seems to be the gold standard
    for this kind of container orchestration.
    Just add a couple of labels to your container and traefik will automatically keep track of that container how to
    route to it.
    We won't get quite that convenient, but I think I've come up with a reasonable alternative.

* Exposing on port 80/443

    By default, only root processes can bind to TCP or UDP ports below 1024. This might be a minor issue to you if you
    are okay with running your http(s) services on port 8080/8443, but that is not a great user experience.
    This is not an issue with Docker since the docker socket runs as root and likes to "helpfully" punch holes in
    your firewall for you.
    Working around this will require root priviledges, but I've minimized what has to be done, and it doesn't require
    the container itself to have root.


## Service Discovery

Service discovery is arguably the more annoying of the two, since it's obvious workarounds are very manual.
Since it also doesn't require any root priviledges, we'll start there.

The key here is using [Podman container networks DNS](https://podman.io/getting-started/network#using-dns-in-container-networks).
All *new* bridge networks (also rootless!) created with podman have the [dnsname
plugin](https://github.com/containers/dnsname) enabled by default.
This works like so:

{{< mermaid >}}
flowchart LR

  subgraph n01[Podman Network]
    subgraph rp[Rev Proxy Pod]
      rp01(Rev Proxy);
    end

    subgraph pod01["Service Pod 01 (`pod01`)"]
      service01(container01 *:80)
    end

    subgraph pod02["Service Pod 02 (`pod02`)"]
      service02(container02 *:8080) --> data[(Data)]
    end


    rp-- "pod01.dns.podman:80" -->pod01;
    rp-- "pod02.dns.podman:8080" -->pod02;
  end
  style n01 fill:white
{{< /mermaid >}}

Each pod and/or container is created within (or added to) a single podman network with the dnsname plugin.  This means
that within the network, pods can refer to each other using `{pod_name}.dns.podman` (domain configurable).  All you need
now is to make sure that the pod is exposing the ports for your container and create a simple configuration in your
reverse proxy to route to this pod.

E.g. To route `service01.mydomain.com` to `container01` running on port 80 in `pod01` you just need to proxy your desired
url to `pod01.dns.podman:80`.



### Example

Here's a simple bash script and nginx configuration snippet to spin up a docker registry and nginx instance and have
nginx proxy the url `registry.mydomain.com` to the registry that is running on port 80 in it's pod.
We have to:
* create a network
* create a pod for each application we want
* create/start the containers in their respective pods

```bash
#!/bin/bash

podname="container-registry"
network="pod_network"

podman network create "$network"

# Create a pod connected to the network
podman pod create \
      --name "$podname" \
      -p 80 \
      --net $network

# Create a container in that pod
podman run \
      --detach \
      --init \
      --name "${podname}-registry" \
      --pod "$podname" \
      --userns keep-id \
      --volume "./docker-registry/:/var/lib/registry/" \
      --volume "./registry.config.yml:/etc/docker/registry/config.yml" \
      docker.io/registry:2

# Create a second pod and container for nginx
podman pod create \
        --name "nginx" \
        -p 8080 \
        -p 8443 \
        --net "$network"

podman create \
        --name "nginx-reverse" \
        --pod "nginx" \
        --expose 8080 \
        --expose 8443 \
        --volume "./nginx.conf.d/:/etc/nginx/conf.d/" \
        --volume "./nginx.conf:/etc/nginx/nginx.conf" \
        nginx:stable
```

```nginx
server {
    listen 8080;

    server_name registry.mydomain.com;

    return 301 https://registry.mydomain.com$request_uri;
}
```


## Exposing on Port 80/443

Doing this requires root access no matter which way you slice it.
The simplest way is to low the minimum port that non-root processes are allowed to access to 80, but that's just a bad
idea, so I'm not going to bother telling you how to do that (not to mention I've also forgotten that detail).

Something we can do with a far more limited scope is to have our firewall redirect port 80/443 on our desired input
interface or IP to port 8080/8443 (or your desired ports) on our system.

The basic idea is to add the following rules to the `nat` chain, and they should be among the first rules to be
evaluated by your firewall.
For example, if you are running Ubuntu with [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall), you could add the
following block to the top of `/etc/ufw/before.rules`:

```plain
*nat
:PREROUTING ACCEPT [0:0]
-F PREROUTING

-A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8080
-A PREROUTING -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 8443
COMMIT
```

This will append two rules to the `PREROUTING` chain in the `NAT` table, which tell `iptables` to redirect tcp packets
arriving on interface `eth0` on port 80 and 443 to ports 8080 and 8443, respectively, on the host system.
Then all you have to do is have your web server/reverse proxy listening on 8080 and 8443.

**Warning:** Be careful when using `-F PREROUTING` as this command will clear all existing commands from the
`PREROUTING` chain in this table. This is useful here because I know that the `PREROUTING` chain is (or should be) clear
at this point and prevents re-appending the same rules multiple times (e.g. if you happened reload to reload/restart UFW
without first removing the original rules).
