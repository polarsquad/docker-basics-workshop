---
title: Docker networking
weight: 8
menu: true
---

So far we've used port forwarding to access containerized services.
In this section, we'll briefly look into Docker's various networking models.

Docker includes following network drivers.

* `bridge`: network bridged with the host machine
* `host`: connect container directly to the host network interface
* `overlay`: a network that spans over multiple Docker hosts
* `macvlan`: assign MAC addresses to containers, making them part of the network your host is connected to
* `none`: disable all container networking

It's also possible to install [a custom network plugin](https://docs.docker.com/engine/extend/plugins_services/)
to manage container networking.

By default, Docker will have a `bridge`, a `host`, and a `none` network available.
If not explicitly selected, the containers will use the `bridge` network by default.
This includes all the containers we've run so far.
We can view these networks using the Docker CLI.

    $ docker network ls
    NETWORK ID        NAME       DRIVER       SCOPE
    678c7d72b948      bridge     bridge       local
    4e05fb2c140c      host       host         local
    dbd54eecb19d      none       null         local

## Host network

Let's try connecting the app we created earlier to the `host` network driver.

    $ docker run --rm -d --name myapp --net host myapp
    $ docker ps
    CONTAINER ID     IMAGE    COMMAND             CREATED           STATUS          PORTS     NAMES
    6e805ede534b     myapp    "python app.py"     10 seconds ago    Up 10 seconds             myapp

Notice how we no longer have a port mapping.
This is because the container will open the port directly on the host network.
We can test it with curl.

    $ curl http://localhost:3000/
    Hello World!

If you have the `lsof` command installed,
you should also be able to see Python program connected to the port.

    $ sudo lsof -i TCP:3000
    COMMAND   PID             USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
    python  27365 systemd-coredump    3u  IPv4 35128999      0t0  TCP *:commplex-main (LISTEN)

Neat! Let's clean up the resources.

    $ docker stop myapp

## Custom networks

The default bridge and host networks are great,
but occasionally you may need to create custom networks for your containers.
For example, you can use the custom networks to prevent containers from connecting to each other over the network.

Let's create our own bridge network using the Docker CLI.

    $ docker network create mynet
    $ docker network ls
    NETWORK ID       NAME      DRIVER    SCOPE
    678c7d72b948     bridge    bridge    local
    4e05fb2c140c     host      host      local
    f8ad75fd2622     mynet     bridge    local
    dbd54eecb19d     none      null      local

Let's now run our app in the network we just created.

    $ docker run --rm -d --name myapp --net mynet myapp

We could specify a port forwarding rule to access the app from the host, but for now we can leave it out.
Instead, let's run another container in the same network, that will do the HTTP request for us.
We can use `appropriate/curl` for this, which is a pre-made image with `curl` installed.

    $ docker run --rm --net mynet appropriate/curl -s http://myapp:3000/
    Hello World!

Notice how we used the container name as part of the HTTP request address.
This works because the custom network will create internal DNS records that maps the container name to the container IP address.
If you want to find out the container IP address, you can use the `docker inspect` command.

    $ docker inspect myapp | grep IPAddress
                "SecondaryIPAddresses": null,
                "IPAddress": "",
                        "IPAddress": "172.19.0.2",
    $ docker run --rm --net mynet appropriate/curl -s http://172.19.0.2:3000/
    Hello World!

To confirm the network isolation works as expected,
we can try to access our app using the `curl` container in the default network.

    $ docker run --rm appropriate/curl -m 5 http://myapp:3000/
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (6) Could not resolve host: myapp
    $ docker run --rm appropriate/curl -m 5 http://172.19.0.2:3000/
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:--  0:00:05 --:--:--     0
    curl: (28) Connection timed out after 5000 milliseconds

Fantastic! Let's clean up the resources.

    $ docker stop myapp
    $ docker network rm mynet
