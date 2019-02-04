---
title: Publishing a Docker image
weight: 6
menu: true
---

Now that we have our own Docker image, let's look into how we can share it.

Docker images can be published to registries.
Most common registry is Docker Hub, where you can host both public and private Docker images.
You can also get private Docker registries from providers such as AWS, Azure, and Google Cloud,
or host your own registry.

The registry name is part of the Docker name.
For example, if we hosted our own private registry in address `registry.example.com`,
then our Docker images in that address would have names such as
`registry.example.com/myapp`, `registry.example.com/myapp2`, `registry.example.com/myorg/myapp` etc.

Docker Hub hosted images hide the registry name from the image name.
Their actual names would be like this.

* `ubuntu` is `registry.hub.docker.com/_/ubuntu`
* `nginx` is `registry.hub.docker.com/_/nginx`
* `someuser/someapp` is `registry.hub.docker.com/someuser/someapp`

For this exercise, let's host our own local registry to simulate a custom registry for the app we created in the last section.

    $ docker run --rm -d -p 5000:5000 --name registry registry:2

Before we can push our app, we need to re-tag it to include the registry name.

    $ docker tag myapp localhost:5000/myapp
    $ docker images localhost:5000/myapp
    REPOSITORY              TAG       IMAGE ID        CREATED          SIZE
    localhost:5000/myapp    latest    9415036e1386    25 minutes ago   131MB

Now we can push the image.

    $ docker push localhost:5000/myapp
    The push refers to repository [localhost:5000/myapp]
    ea4105421e40: Pushed 
    b413a7b68cc8: Pushed 
    7113e6f202c2: Pushed 
    0877695240f0: Pushed 
    27a216ffe825: Pushed 
    9e9d3c3a7458: Pushed 
    7604c8714555: Pushed 
    adcb570ae9ac: Pushed 
    latest: digest: sha256:dc97f7c1d2f642fdcc1ed89df29da3147c95acd37647ac0fc26556b862b665f9 size: 1984

Let's see if we can pull the image back from the registry.

    $ docker image rm localhost:5000/myapp
    $ docker image rm myapp
    $ docker pull localhost:5000/myapp
    Using default tag: latest
    latest: Pulling from myapp
    38e2e6cd5626: Already exists 
    705054bc3f5b: Already exists 
    c7051e069564: Already exists 
    7308e914506c: Already exists 
    39f5794675c7: Pull complete 
    7803fc71e96f: Pull complete 
    90379ee10948: Pull complete 
    e0eba736f7d3: Pull complete 
    Digest: sha256:dc97f7c1d2f642fdcc1ed89df29da3147c95acd37647ac0fc26556b862b665f9
    Status: Downloaded newer image for localhost:5000/myapp:latest
    $ docker images localhost:5000/myapp
    REPOSITORY             TAG       IMAGE ID       CREATED          SIZE
    localhost:5000/myapp   latest    9415036e1386   39 minutes ago   131MB

Brilliant! Let's re-tag the image to `myapp` and clean up the resources.

    $ docker tag localhost:5000/myapp myapp
    $ docker stop registry
