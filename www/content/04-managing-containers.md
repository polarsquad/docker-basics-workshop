---
title: Managing Docker containers
weight: 4
menu: true
---

Usually we want some feedback on how our containers are doing.
In this section, we'll look into a few ways we can examine and manage Docker containers.

## Listing containers

There are two ways to list currently running containers.
One is to use `docker container ls` and the other `docker ps`.
Both commands produce the same output.

    $ docker ps
    CONTAINER ID    IMAGE    COMMAND                  CREATED         STATUS          PORTS                  NAMES
    624a8853afe5    nginx    "nginx -g 'daemon of…"   3 seconds ago   Up 2 seconds    0.0.0.0:8080->80/tcp   web

If you still have your web server from the previous exercise running,
you should see it listed.

You can also list all past containers with the help of the `-a` flag:

    $ docker ps -a
    CONTAINER ID    IMAGE           COMMAND                  CREATED          STATUS                     PORTS                  NAMES
    624a8853afe5    nginx           "nginx -g 'daemon of…"   2 minutes ago    Up 2 minutes               0.0.0.0:8080->80/tcp   web
    ee73eeb1a2e3    nginx           "nginx -g 'daemon of…"   3 minutes ago    Exited (0) 2 minutes ago                          festive_mestorf
    8c6a15958a2e    ubuntu:18.04    "uname -a"               5 minutes ago    Exited (0) 5 minutes ago                          jovial_sinoussi
    4cbcaffa8ca5    hello-world     "/hello"                 6 minutes ago    Exited (0) 6 minutes ago                          suspicious_pascal

We can later use the container IDs and names to refer to these containers for further operations.
Both of them are automatically generated for us, but we can also set the name as done in the earlier exercise.

## Printing logs

When we ran the NGINX container in foreground earlier,
we could see the logs printed on the terminal directly.
Now that it's running in the background,
we need another way to view them.

This is where the `docker logs` command comes in handy.
We can use container name we defined earlier or the container ID to get logs from the web server:

    $ docker logs web
    172.17.0.1 - - [01/Feb/2019:13:12:58 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:64.0) Gecko/20100101 Firefox/64.0" "-"
    ...
    $ docker logs 624a8853afe5
    172.17.0.1 - - [01/Feb/2019:13:12:58 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:64.0) Gecko/20100101 Firefox/64.0" "-"
    ...

You can also follow the flow of logs with the help of the `-f` flag:

    $ docker logs -f web

## Examining container processes

We can use the `docker top` command to examine the processes running inside a container.
Let's see what the NGINX container is running:

    $ docker top web
    UID     PID      PPID     C     STIME     TTY    TIME       CMD
    root    32541    32524    0     20:00     ?      00:00:00   nginx: master process nginx -g daemon off;
    101     32583    32541    0     20:00     ?      00:00:00   nginx: worker process

The `docker top` command accepts Linux `ps` command parameters,
which you can use to tune the `top` output.
See `man ps` for more `ps` options.

## Executing commands on containers

Sometimes we may want to inspect the container by running arbitrary commands inside it.
We can use `docker exec` for that.

For example, if we want to check the configuration used in the NGINX container,
we can use `docker exec` with simple `cat` and a path to the config file.

    $ docker exec web cat /etc/nginx/nginx.conf

    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    ...

We can also combine `docker exec` with the `-it` flags we used earlier to get an interactive shell inside the container:

    $ docker exec -it web bash
    root@1aabce753f41:/# ls
    bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
    root@1aabce753f41:/#

## Stopping a container

Once we are done with the container, we can stop it with `docker stop`.

    $ docker stop web

If the container is stuck, we can also stop it by force using `docker kill`.

    $ docker kill web

## Listing images

Containers are created from Docker images.
The appropriate images are automatically downloaded when you run a container.
You can list the downloaded images with `docker images` or `docker image ls`.

    $ docker images
    REPOSITORY      TAG        IMAGE ID         CREATED        SIZE
    nginx           latest     42b4762643dc     1 hour ago     109MB
    ubuntu          18.04      20bb25d32758     1 hour ago     87.5MB
    hello-world     latest     fce289e99eb9     1 hour ago     1.84kB

## Cleaning your Docker environment

After the container process stops,
the container is still left around for further inspection.
You can view the stopped containers using `docker ps -a`.

To get rid of the stopped containers, you can use the `docker rm` or `docker container rm` command.

    $ docker rm web

If you want the container to be automatically removed after it has stopped,
you can add the `--rm` flag to `docker run`.

    $ docker run --rm --name web -p 8080:80 nginx
    $ docker stop web

If you want to also get rid of the Docker image, you can run the `docker rmi` or `docker image rm` command.

    $ docker rmi nginx

To automatically get rid of unused containers and images, you can use the following commands:

    $ docker container prune
    $ docker image prune
