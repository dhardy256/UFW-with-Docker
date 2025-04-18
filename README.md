# Configuring UFW for use with Docker

## Preface

This project was heavily inspired by [chaifeng/ufw-docker](https://github.com/chaifeng/ufw-docker).

## Disclaimer

This code comes with absolutely no warranty express or implied.<br>
Do not run in a production environment.<br>
**Run at your own risk!**

## Assumptions

You've already got a **dev VM** running **Docker** with **UFW** installed.<br>
You've already got UFW configured to allow SSH (port 22) from your workstation.<br>
There are **no critical services running on this dev VM**.

## Goals

Configure UFW to filter Docker network traffic the same way it filters host traffic with three basic rules:

1. Filter Docker network traffic based on user-created ufw rules (the rules listed with `sudo ufw status`)
2. Allow established/related Docker network traffic
3. Log and drop all other Docker network traffic

## Docker image building limitations

During testing it was discoverd that the Docker image building process uses the default Docker network (172.17.0.0/16) and is impacted by these UFW filtering rules.

The options here are:

1. Explicitly allow DNS, 80, and 443 traffic on the default Docker network (172.17.0.0/16) <br> **and avoid using the default Docker network for all containers** by

   a. defining rules in /etc/ufw/after.rules and keep user-defined rules clean <br>
   b. creating user-defined rules to make default Docker network exceptions more obvious

2. Do **not** allow DNS, 80, and 443 out and **build images on another device**

The solution below is for **option 1a**.<br>
Omit sections dns, 80, and 443 of **after.rules** file append for **option 2**.

## UFW Configuration Update

Edit UFW's after.rules file - [one of 3 files loaded on boot](https://manpages.ubuntu.com/manpages/xenial/man8/ufw-framework.8.html).

```
sudo nano /etc/ufw/after.rules
```

**Append** to bottom of file:

```
# BEGIN UFW CONFIG FOR DOCKER
*filter
:ufw-user-forward - [0:0]
:DOCKER-USER - [0:0]

# filter Docker network traffic based on user-created ufw rules
-A DOCKER-USER -j ufw-user-forward

# sections dns, 80, and 443 enable the docker service to build images on default bridge network

# dns
-A DOCKER-USER -s 172.17.0.0/16 -p udp -m udp --sport 1024:65535 --dport 53 -j LOG --log-prefix "[UFW DOCKER DNS] "
-A DOCKER-USER -s 172.17.0.0/16 -p udp -m udp --sport 1024:65535 --dport 53 -j RETURN

# 80
-A DOCKER-USER -s 172.17.0.0/16 -p tcp --dport 80 -j LOG --log-prefix "[UFW DOCKER RETURN EXTERNAL] "
-A DOCKER-USER -s 172.17.0.0/16 -p tcp --dport 80 -j RETURN

# 443
-A DOCKER-USER -s 172.17.0.0/16 -p tcp --dport 443 -j LOG --log-prefix "[UFW DOCKER RETURN EXTERNAL] "
-A DOCKER-USER -s 172.17.0.0/16 -p tcp --dport 443 -j RETURN

# allow established / related traffic, stop processing more rules, and RETURN (to FORWARD chain)
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN

# log and drop all other traffic
-A DOCKER-USER -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A DOCKER-USER -j DROP

COMMIT
# END UFW CONFIG FOR DOCKER
```

## Reload UFW after config change

```
sudo ufw reload
```

## Basic testing with Docker

Create a custom Docker network with:

```
sudo docker network create --subnet=172.21.0.0/16 webhosting
```

Create an NGINX container, attached to custom network with:

```
sudo docker run -d --name nginx --net=webhosting -p 80:80 nginx
```

Attempt to access NGINX homepage with a web browser (expect no response).

Verify UFW blocking port 80 requests with:

```
tail -f /var/log/ufw.log
```

NGINX homepage should **NOT** be accessible until port 80 is opened with UFW:

```
sudo ufw allow 80
```

Clean up:

```
sudo docker rm -f nginx
```

```
sudo docker network rm webhosting
```

```
sudo ufw status numbered
```

Use `sudo ufw delete` [number] <br>
to delete 'allow port 80' rules.

## Partial docker-compose example:

```
networks:
  webhosting:
    name: webhosting
    ipam:
      driver: default
      config:
        - subnet: 172.21.0.0/16
          gateway: 172.21.0.1

services:
  nginx:
    image: nginx
    ports:
      - 80:80
    networks:
      webhosting:
```
