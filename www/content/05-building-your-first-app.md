---
title: Building your first Docker app
weight: 5
menu: true
---

Having the capability to leverage pre-existing Docker images is awesome,
but often we want to create our own Docker images.
In this section, we'll create our own Docker application.

## Web application example

Let's prepare a simple [Python](https://www.python.org/) web application for containerization.
First, create a directory for your app.

    $ mkdir ~/myapp
    $ cd ~/myapp

Next, createa a Python file named `app.py` to the directory we just created,
and write the following contents to it:

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello World!\n"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

If you have Python and [Flask](http://flask.pocoo.org/) installed,
you can try running the app with command `python app.py` in the app directory.

    ~/myapp $ python app.py
    * Serving Flask app "app" (lazy loading)
    * Environment: production
      WARNING: Do not use the development server in a production environment.
      Use a production WSGI server instead.
    * Debug mode: off
    * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)

Don't worry if it doesn't work just yet.

## Dockerfile

Now that we have a very basic app prepared,
we can package it into a Docker image.
For that, we need to write a Dockerfile.

Create a file named `Dockerfile` in the directory we created earlier.
In that, add the following contents:

```Dockerfile
FROM ubuntu:18.04
WORKDIR /opt/myapp
COPY app.py .
RUN apt-get update
RUN apt-get install -y python python-pip
RUN pip install Flask
CMD ["python", "app.py"]
```

Each line in this file is an instruction for Docker image builder.
The first word of the line is a command,
and the rest of the words are parameters for that command.
Here's what they do:

* `FROM` describes the image to build our image on.
  In this case, we're going to use a plain Ubuntu image as the base image.
* `WORKDIR` sets the working directory in the image
* `COPY` copies our Python file from our host directory `~/myapp` to the image.
* `RUN` runs a Linux program inside the image.
   In this case, we install Python and PIP, a package manager for Python,
   and then install the app dependencies using PIP.
* `CMD` sets `python app.py` to be the default command, when we run the container.

## Build from Dockerfile

Our Docker image is ready for build! Let's use `docker build` to do that.

    ~/myapp $ docker build -t myapp .

Using the option `-t`, we can specify a name for our Docker image similar to `ubuntu:18.04` and `nginx`.
In Docker, the name is called a tag. The same image can have multiple tags.

The positional parameter specifies the context for the Docker build,
i.e. the path where all the resources for the Docker image can be found.
In this case, we use the `~/myapp` as the context directory.

After the build, we should now have a Docker image named `myapp`.

    $ docker images myapp
    REPOSITORY  TAG        IMAGE ID         CREATED           SIZE
    pyapp       latest     5ebae78df742     1 minute ago      455MB

Let's smoke test it!
The app opens port 5000, so we'll need to set a forwarding rule to that port.

    $ docker run -d --name myapp -p 5000:5000 myapp
    $ curl http://localhost:5000/
    Hello World!
    $ docker stop myapp

We now our very first Docker app ready! Awesome!

## Improving our Docker image

Even though the Docker image works,
there's a few improvements that are worth making.

### Caching

Docker caches each instruction and only re-runs them when they're likely to cause a change.
For example, `RUN` instructions are re-run when the command is changed,
and `COPY` instructions are rerun when the source file changes.

Additionally, when an instruction needs to be ran again,
all the instructions that follow will need to be run as well.
For example, if we change `app.py` contents and rebuild the image,
`apt-get` and `pip` will be run again.

We can improve the caching by moving the `COPY` instruction to the bottom of the file.

```Dockerfile
FROM ubuntu:18.04
WORKDIR /opt/myapp
RUN apt-get update
RUN apt-get install -y python python-pip
RUN pip install Flask
CMD ["python", "app.py"]
COPY app.py .
```

### Cleaning up

Have a look at the size of the Docker image.

    $ docker images myapp
    REPOSITORY  TAG        IMAGE ID         CREATED           SIZE
    pyapp       latest     5ebae78df742     1 minute ago      455MB

Let's see if we can reduce the size by removing some of the cache files and unused packages.

```Dockerfile
FROM ubuntu:18.04
WORKDIR /opt/myapp
RUN apt-get update
RUN apt-get install -y python python-pip
RUN pip install --no-cache Flask
RUN apt-get remove -y python-pip
RUN apt-get autoremove -y
RUN rm -rf /var/lib/apt/lists/*
CMD ["python", "app.py"]
COPY app.py .
```

Now let's have a look at that image size again...

    ~/myapp $ docker build -t myapp .
    ~/myapp $ docker images myapp
    REPOSITORY   TAG       IMAGE ID        CREATED          SIZE
    myapp        latest    afc654b35aad    30 seconds ago   457MB

Huh?! The image size is actually WORSE!

There's an explanation for this in the feature that supports the instruction caching: layers.
Each instruction creates a new layer of files.
When the image is built, the layers are stacked together to represent the final container file system.
Therefore, when we create an instruction to remove files,
it will just create a layer with those files missing,
but it will not remove them from the previously created layer.

We can view these layers using `docker history`.

    $ docker history myapp
    IMAGE               CREATED             CREATED BY                                      SIZE  
    afc654b35aad        5 minutes ago       /bin/sh -c #(nop) COPY file:7348164eda8bae0d…   169B  
    01a4ccfecf8f        5 minutes ago       /bin/sh -c #(nop)  CMD ["python" "app.py"]      0B    
    ea344c5663c7        5 minutes ago       /bin/sh -c rm -rf /var/lib/apt/lists/*          0B    
    64764d84582c        5 minutes ago       /bin/sh -c apt-get autoremove -y                1.18MB
    abd52a23b584        5 minutes ago       /bin/sh -c apt-get remove -y python-pip         1.29MB
    5ff8caeb97bc        5 minutes ago       /bin/sh -c pip install --no-cache Flask         4.02MB
    a05eb9c23c4c        5 minutes ago       /bin/sh -c apt-get install -y python python-…   339MB 
    2e52e5d4b94f        5 minutes ago       /bin/sh -c apt-get update                       24.5MB
    38bb6c7c3d6c        10 minutes ago      /bin/sh -c #(nop) WORKDIR /opt/myapp            0B    
    20bb25d32758        2 hours ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B    
    <missing>           2 hours ago         /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B    
    <missing>           2 hours ago         /bin/sh -c rm -rf /var/lib/apt/lists/*          0B    
    <missing>           2 hours ago         /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   745B  
    <missing>           2 hours ago         /bin/sh -c #(nop) ADD file:38a199e521f5e9007…   87.5MB

How do we get rid of those files then?
The solution is to run all of the steps within a single instruction to create just one layer.

```Dockerfile
FROM ubuntu:18.04
WORKDIR /opt/myapp
RUN apt-get update && \
    apt-get install -y python python-pip && \
    pip install --no-cache Flask && \
    apt-get remove -y python-pip && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
CMD ["python", "app.py"]
COPY app.py .
```

Let's have a look at the history now.

    $ docker history myapp
    IMAGE               CREATED             CREATED BY                                      SIZE  
    aa64ac6a505c        5 minutes ago       /bin/sh -c #(nop) COPY file:7348164eda8bae0d…   169B  
    b647886bd2bb        5 minutes ago       /bin/sh -c #(nop)  CMD ["python" "app.py"]      0B    
    8ac22646ce40        5 minutes ago       /bin/sh -c apt-get update &&     apt-get ins…   42.7MB
    38bb6c7c3d6c        10 minutes ago      /bin/sh -c #(nop) WORKDIR /opt/myapp            0B    
    20bb25d32758        2 hours ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B    
    <missing>           2 hours ago         /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B    
    <missing>           2 hours ago         /bin/sh -c rm -rf /var/lib/apt/lists/*          0B    
    <missing>           2 hours ago         /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   745B  
    <missing>           2 hours ago         /bin/sh -c #(nop) ADD file:38a199e521f5e9007…   87.5MB

There's now less layers, and the huge +300 MB layer is gone. Let's check the full image size now.

    $ docker images myapp
    REPOSITORY    TAG       IMAGE ID       CREATED           SIZE
    myapp         latest    aa64ac6a505c   30 seconds ago    130MB

We managed reduce the size to about 30% of what we had earlier. Not bad!

### Alternative base images

Using Ubuntu as the base image might not always be the most lightweight option.
You might also want to look into alternative base images such as [Debian slim](https://hub.docker.com/_/debian)
or [Alpine](https://hub.docker.com/_/alpine).
There's also [a pre-made image for Python](https://hub.docker.com/_/python),
which includes Python and some Python related tools like PIP pre-installed.

Here's a Dockerfile that's based on Alpine:

```Dockerfile
FROM alpine
WORKDIR /opt/myapp
RUN apk update && \
    apk add python py-pip && \
    pip install --no-cache Flask && \
    apk del py-pip
CMD ["python", "app.py"]
COPY app.py .
```

Let's build it with another tag and compare it to the Ubuntu based image.

    ~/myapp $ docker build -t myapp:alpine .
    ~/myapp $ docker images myapp
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    myapp               alpine              494f4abc243c        39 seconds ago      48.6MB
    myapp               latest              aa64ac6a505c        10 minutes ago      130MB

Just under 50 MB, which is roughly 10% of what we had originally. Nice!

### Running as non-root

By default, all containers run as root by default.
Even though there's plenty of isolation between the container and the host system,
it's a good practice to use a custom user in containers whenever possible.
Optimally, the custom user is defined and set in the Dockerfile.

```Dockerfile
FROM ubuntu:18.04
WORKDIR /opt/myapp
RUN apt-get update && \
    apt-get install -y python python-pip && \
    pip install --no-cache Flask && \
    apt-get remove -y python-pip && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
RUN groupadd -g 999 appuser && \
    useradd -r -u 999 -g appuser appuser
CMD ["python", "app.py"]
COPY app.py .
```

## Experiment with your own Dockerfile

Create your own Dockerfile or extend one of the Dockerfiles shown earlier.
