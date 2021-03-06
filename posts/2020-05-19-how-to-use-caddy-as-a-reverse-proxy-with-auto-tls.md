---
title: How to use Caddy as a Reverse-Proxy (With auto-TLS)
slug: how-to-use-caddy-as-a-reverse-proxy-with-auto-tls
date_published: 2020-05-19T18:27:00.000Z
date_updated: 2020-05-31T18:27:13.000Z
tags: 
  - devops
  - tls
  - caddy
  - let's encrypt
  - gcp
  - dns
layout: layouts/post.njk
---

Caddy offers TLS encryption by default (https) and it uses Let’s Encrypt’s authority to automatically generate your certificates. In this short tutorial we will run a small backend and a Caddy web server as a reverse proxy, first in local, and then in a virtual machine on the Cloud (because ports 80 and 443 are blocked in my home, please ISP providers, stop that already). We could’ve used a different port to serve https but it’s not as cool.

Let’s pull the code needed for this tutorial:

```bash
git clone [https://github.com/serafdev/reverse-caddy.git](https://github.com/serafdev/reverse-caddy.git) && cd reverse-caddy
```

The only file we need to inspect is the docker-compose.yml file, the others can be ignored, they are only used to build and run a small backend server to test our reverse-proxy situation. Let’s break it down.

```bash
    version: '3'
    volumes:
      caddy_data:
      caddy_config:
    services:
      caddy:
        image: caddy
        volumes:
          - caddy_data:/data
          - caddy_config:/config
        ports:
          - 80:80
          - 443:443
        command: caddy reverse-proxy --from localhost --to backend:8081 
      backend:
        build:
          context: .
```

We have here 2 services, caddy and backend. backend is the backend (woah) and caddy serves as a reverse proxy.

There is 2 volumes attached to caddy, there’s caddy_data and caddy_config, these are used to have minimal persistance, of course I always suggest having all your network as code, but let’s go with this anyway as we are only running a reverse proxy.

We expose ports 80 and 443 to enable redirection and https and we run the caddy command for the reverse-proxy:

```bash
caddy reverse-proxy — from localhost — to backend:8081
```

In a caddy config this looks like:


```bash
localhost

reverse-proxy backend:8081
```

Let’s build this:
 
```bash
docker-compose up -d
```

Now you can visit localhost, it will redirect you to [https://localhost](https://localhost,)
![](/content/images/2020/05/image-23.png)https, noice
If we inspect the certificate, it is a self-signed certificate generated by Caddy. This is normal, for security reasons, a TLS certificate for localhost will never be generated by known Authorities. There’s is many tricks to bypass that, but just don’t.
![](/content/images/2020/05/image-24.png)Caddy Local Authority
Okay, let’s go to the next step, let’s create a virtual machine in any cloud, I will be using GCP. Don’t forget to allow HTTP and HTTPS on the firewall level. Ideally use a Ubuntu Server to keep following, but feel free to do whatever you need.

Go to this link and follow the steps to install docker engine for Ubuntu: [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

Now we’ll need docker-compose:

```bash
sudo apt install docker-compose
```

Double check your installations with docker -v and docker-compose -v, you should have something like this:

```bash
fares@instance-2:~$ docker-compose -v
docker-compose version 1.17.1, build unknown
fares@instance-2:~$ docker -v
Docker version 19.03.9, build 9d988398e7
```

Versions
Okay now install git:

```bash
sudo apt install git
```

And clone the repository we used earlier with the caddy reverse-proxy docker-compose config:

```bash
git clone [https://github.com/serafdev/reverse-caddy.git](https://github.com/serafdev/reverse-caddy.git) && cd reverse-caddy
```

Now we just need to do some modifications in the docker-compose.yml file, on the “command” directive in caddy change “localhost” with one of your own domains. I will pick something from my Cloud DNS Zone (gcp.seraf.dev), let’s go with dydy for caddy (??).

```bash
version: '3'
volumes:
  caddy_data:
  caddy_config:
services:
  caddy:
    image: caddy
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    ports:
      - 80:80
      - 443:443
    command: caddy reverse-proxy --from dydy.gcp.seraf.dev --to backend:8081 
  backend:
    build:
      context: .
```

Now go to your DNS and add an A record for this new virtual machine, for me the domain name is “dydy.gcp.seraf.dev” and I use the external IP “[35.188.19.184](https://35.188.19.184/)”, I will make some pictures on how it looks like on GCP:
![](/content/images/2020/05/image-26.png)Compute Engine
And on Cloud DNS:
![](/content/images/2020/05/image-27.png)Cloud DNS
It’s a good idea to create an NS record for your cloud environment to make small tests like this one as a developer or devops, else you can just do that inside your domain provider’s DNS config.

I hid the SOA and NS record because I don’t know if it’s sensitive, but basically it just tells my domain provider to put gcp.seraf.dev in the GCP Nameserver, so you can create anything.gcp.seraf.dev from within your console.

Okay back to our stuff, now that we setup the A record we can go back to the machine and run our docker-compose. Since we don’t care about this machine and permissions, you can run docker-compose up -d with elevated privileges:

```bash
sudo docker-compose up -d
```

Verify your deployment:
![](/content/images/2020/05/image-28.png)Wow, so small. Just zoom
OHHH yeahhh! We’ve done it! Now visit your url and look at the certificate, so simple right?
![](/content/images/2020/05/image-29.png)Easy to deploy certificate? Check.
All right, hope this helped someone!

For more information go ahead to their website, I am not affiliated with them (wish I was though) [https://caddyserver.com/](https://caddyserver.com/)
