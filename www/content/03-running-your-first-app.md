---
title: Running your first Docker app
weight: 3
menu: true
---

In this section, we will run some pre-existing Docker containers
that are based on public Docker images hosted on [Docker Hub](https://hub.docker.com/).

## Hello World

We can use `docker run` to run containers from Docker images.
Let's start with a simple "hello world" container.

    $ docker run hello-world
    Unable to find image 'hello-world:latest' locally
    latest: Pulling from library/hello-world
    1b930d010525: Pull complete 
    Digest: sha256:2557e3c07ed1e38f26e389462d03ed943586f744621577a99efb77324b0fe535
    Status: Downloaded newer image for hello-world:latest

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
    1. The Docker client contacted the Docker daemon.
    2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
        (amd64)
    3. The Docker daemon created a new container from that image which runs the
        executable that produces the output you are currently reading.
    4. The Docker daemon streamed that output to the Docker client, which sent it
        to your terminal.

    To try something more ambitious, you can run an Ubuntu container with:
    $ docker run -it ubuntu bash

    Share images, automate workflows, and more with a free Docker ID:
    https://hub.docker.com/

    For more examples and ideas, visit:
    https://docs.docker.com/get-started/

Images for containers are automatically pulled, if they don't exist locally.

## Ubuntu in Docker

We can run Ubuntu in a container.

    $ docker run ubuntu:18.04 uname -a
    Linux 4dd6fed5a8a9 4.19.8-200.fc28.x86_64 #1 SMP Mon Dec 10 15:43:40 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

We can also run it in interactive mode to get a shell using the `-i` (interactive) and `-t` (tty) flags.

    $ docker run -it ubuntu:18.04
    root@fad321cefd0a:/# ls
    bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
    boot  etc  lib   media  opt  root  sbin  sys  usr
    root@fad321cefd0a:/# exit

## Web server

Let's run a long-running process like a web server.
We can use [NGINX](https://www.nginx.com/) for that.

    $ docker run -p 8080:80 nginx

NGINX opens port 80 for HTTP. We can use the `-p` option to map the port to a host port like 8080.

We can test the container by opening URL <http://localhost:8080/> in our browser or with `curl` in another terminal.

    $ curl http://localhost:8080/
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...

You should see NGINX write logs to the terminal where you launched NGINX.

    172.17.0.1 - - [01/Feb/2019:13:12:58 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:64.0) Gecko/20100101 Firefox/64.0" "-"
    2019/02/01 13:12:58 [error] 6#6: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "localhost:8080"
    172.17.0.1 - - [01/Feb/2019:13:12:58 +0000] "GET /favicon.ico HTTP/1.1" 404 153 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:64.0) Gecko/20100101 Firefox/64.0" "-"
    172.17.0.1 - - [01/Feb/2019:13:14:33 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.59.0" "-"

Containers can also be run detached from the terminal with the `-d` flag.
Let's also give the container a name we can refer to later.
Kill the container with Ctrl+C and run the following command.

    $ docker run -d --name web -p 8080:80 nginx
    624a8853afe59b69a7a4ba3f35fa274ba28247bd931cef1f505a9129aad8fa5b

The printed ID is the container ID, which we will look into later.
Check that the web server still responds.

    $ curl http://localhost:8080/
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...

## Freestyle: Find and run a Docker image

Spend some time browsing Docker images from [Docker Hub](https://hub.docker.com/),
and try to get one running with your instructor's help.
