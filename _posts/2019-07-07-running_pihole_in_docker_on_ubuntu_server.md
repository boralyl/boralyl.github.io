---
title: Installing Pi-hole on Ubuntu Server with Docker
excerpt: >
  How to install Pi-hole on an Ubuntu Server using Docker.
categories:
  - software
tags:
  - pi-hole
  - docker
  - docker-compose
  - ubuntu
---

I recently decided to try out [Pi-hole](https://pi-hole.net/) after seeing it
pop up quite a bit in conversations in home automation forums.  I did not have
a spare raspberry pi (although I have a RPI4 on the way).  So I decided to use
the [docker image](https://github.com/pi-hole/docker-pi-hole/#running-pi-hole-docker)
that is available and run it on my Intel NUC that runs home assistant a bunch
of other services.  I added the following to my `docker-compose.yml`:

```yaml
version: '3.6'
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "80:80/tcp"
      - "443:443/tcp"
    environment:
      TZ: ${TZ}
      WEBPASSWORD: 'super-secure-web-password'
    # Volumes store your data between container upgrades
    volumes:
       - ${USERDIR}/docker/pihole/etc-pihole:/etc/pihole/
       - ${USERDIR}/docker/pihole/etc-dnsmasq.d/:/etc/dnsmasq.d/
    dns:
      - 127.0.0.1
      - 1.1.1.1
    restart: unless-stopped
```

I then attempted to bring up the container and was met with an error:

```bash
$ docker-compose up pihole
Creating network "docker_default" with the default driver
Pulling pihole (pihole/pihole:latest)...
latest: Pulling from pihole/pihole
fc7181108d40: Pull complete
d7926f6f8fe0: Pull complete
83d7b8d6f2b7: Pull complete
1a9901e26e3f: Pull complete
99dd63aa37fe: Pull complete
167794b7a49c: Pull complete
a69449b843e3: Pull complete
495bdd313aea: Pull complete
aed3a8b2d5ae: Pull complete
c579f8e9f8d2: Pull complete
472c32e9da06: Pull complete
Creating pihole ... error

ERROR: for pihole  Cannot start service pihole: driver failed programming external connectivity on endpoint pihole (af0755eaf81ab800fd336644122a0a7db48a0628b16980b096fe9d4b285c2927): Error starting userland proxy: listen tcp 0.0.0.0:53: bind: address already in use
```

It turns out that by default, ubuntu (as of server version 18.04) will run
`dnsmasq` which listens on port 53.  The solution is pretty straight forward, you need
to simply disable the service then stop it.  This can be done by running:

```bash
$ sudo systemctl disable systemd-resolved.service
$ sudo systemctl stop systemd-resolved.service
```

Then the container starts up with no problem:

```bash
$ docker-compose up -d pihole
Starting pihole ... done
```
