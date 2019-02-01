---
title: Running your first Docker app
weight: 3
menu: true
---

In this section we will run some pre-existing Docker containers
that are based on public Docker images hosted on [Docker Hub](https://hub.docker.com/).

## Hello World

Let's start with a simple container:

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


## Ubuntu in Docker

We can run an existing Ubuntu container:

    $ docker run ubuntu:18.04 uname -a
    Linux 4dd6fed5a8a9 4.19.8-200.fc28.x86_64 #1 SMP Mon Dec 10 15:43:40 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

We can also run it in interactive mode to get a shell:

    $ docker run -it ubuntu:18.04
    root@fad321cefd0a:/# ls
    bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
    boot  etc  lib   media  opt  root  sbin  sys  usr
    root@fad321cefd0a:/# exit

## Web server
