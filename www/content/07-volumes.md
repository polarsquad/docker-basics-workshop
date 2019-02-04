---
title: Mounting files
weight: 7
menu: true
---

So far all our files are built into the images.
Let's see how we can mount files to containers from the host system,
and what we can do with them.

## Static web pages for our web server

Docker can mount files from anywhere in the host system to the container.

Let's create some custom content for the NGINX web server we ran earlier.

    $ mkdir web
    $ echo "Hello world!" > web/hello.txt

Let's run the web server again with the directory mounted.

    $ docker run --rm -v "$PWD/web:/usr/share/nginx/html" -d --name web -p 8080:80 nginx

Here we use the `-v` option to add a volume mount in format `host_path:container_path`.
We'll use the directory we just created as the host path we want to mount,
and `/usr/share/nginx/html` as the target path in the container.
In this case, the target path is the place where NGINX serves static files from.
Note that the host path must be specified as an absolute path,
which is why the example uses the `$PWD` environment variable.

Let's see if we can access the file we just created and mounted.

    $ curl http://localhost:8080/hello.txt
    Hello world!

Very cool! Let's clean up the resources.

    $ docker stop web

## Persisting state

The mount we created earlier is called a *bind mount*.
We can also create dedicated, persistent volumes that are managed by Docker,
and mount them to containers.
They are primarily used for ensuring critical data written by the containers is persisted safely.
For example, the local Docker registry can use volumes to store the published images.

Let's create our own volume!

    $ docker volume create mystuff
    $ docker volume ls
    DRIVER      VOLUME NAME
    local       mystuff

The name of our volume is `mystuff`.
By default, it uses a local driver, which means that all the data is stored in a host directory.
We could also use an alternative driver that stores the data in a different way.

Let's mount the volume to a plain Ubuntu container,
and use the container to write some data to the volume.
We can mount the volume using its name.

    $ docker run --rm -v mystuff:/data ubuntu:18.04 \
        bash -c 'apt-get update && apt-get install -y cowsay && /usr/games/cowsay Hello world! > /data/message.txt'

We should now have a file in the `mystuff` volume.
Let's see if we can find it from the host system.

    $ docker volume inspect mystuff
    [
        {
            "CreatedAt": "2019-02-02T22:58:46+02:00",
            "Driver": "local",
            "Labels": {},
            "Mountpoint": "/var/lib/docker/volumes/mystuff/_data",
            "Name": "mystuff",
            "Options": {},
            "Scope": "local"
        }
    ]

From the `docker volume inspect` output,
we can see that the files are located in the `/var/lib/docker/volumes/mystuff/_data` on the host system.
The files there are read protected from normal users, so we'll have to use `sudo` to access them.

    $ sudo ls /var/lib/docker/volumes/mystuff/_data
    message.txt
    $ sudo cat /var/lib/docker/volumes/mystuff/_data/message.txt
     ______________
    < Hello world! >
     --------------
            \   ^__^
             \  (oo)\_______
                (__)\       )\/\
                    ||----w |
                    ||     ||

We can also re-mount the volume to another container to read the data there.

    $ docker run --rm -v mystuff:/usr/share/nginx/html -d --name web -p 8080:80 nginx
    $ curl http://localhost:8080/message.txt
     ______________
    < Hello world! >
     --------------
            \   ^__^
             \  (oo)\_______
                (__)\       )\/\
                    ||----w |
                    ||     ||
    $ docker stop web

Great! Let's clean up the environment and delete the volume.

    $ docker volume rm mystuff

## Using containers for building software

A common use-case for containers and directory mounts is to build software and get the build artifacts.
The reason for this is because containers can easily produce isolated and reproducible environments, and stay lightweight at the same time.
This is especially useful in Continuous Integration servers,
where concurrent build jobs may easily affect each other when there's no proper isolation in place.

As an exercise, let's create a Docker image for building [a curl C++ example program](https://curl.haxx.se/libcurl/c/htmltitle.html).
The program downloads and parses the title from HTML files.

First, we should create a directory for the example and download the source code.

    $ mkdir ~/curl-example
    $ cd ~/curl-example
    ~/curl-example $ curl -sfLO https://github.com/curl/curl/raw/master/docs/examples/htmltitle.cpp

Let's also create a simple build script and name it `build.sh`:

```bash
#!/usr/bin/env bash
g++ -Wall \
    $(pkg-config --cflags libxml-2.0) $(pkg-config --cflags libcurl) \
    htmltitle.cpp -o htmltitle \
    $(pkg-config --libs libxml-2.0) $(pkg-config --libs libcurl)
```

Next, we need a Dockerfile for the builder image.
The image will contain the build tools and libraries we need.
By default, the builder will run the build script from the `/var/build` directory.

```Dockerfile
FROM ubuntu:18.04
WORKDIR /var/build
RUN apt-get update && \
    apt-get install -y g++ libcurl4-openssl-dev libxml2-dev
CMD ["./build.sh"]
```

Now we can build the image.

    ~/curl-example $ docker build -t builder .

We're now ready to run the builder with the files mounted.
Instead of specifying the user and group in the Dockerfile,
we'll provide them them using the `-u` flag.

    ~/curl-example $ docker run --rm -v "$PWD:/var/build" -u $(id -u):$(id -g) builder

We should have a binary file named `htmltitle` in our directory. Let's test it!

    ~/curl-example $ ls
    build.sh  Dockerfile  htmltitle  htmltitle.cpp
    ~/curl-example $ ./htmltitle http://polarsquad.github.io/docker-basics-workshop/
    Title: Docker Basics Workshop
