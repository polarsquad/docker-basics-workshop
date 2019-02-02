---
title: Preparation
weight: 2
menu: true
---

First, we'll need to set up an environment for Docker.

This workshop requires you to install the latest version of the Docker Community Edition.

## Installation

The most up-to-date guides can be found from [Docker's documentation site](https://docs.docker.com/install/).

* If you use a Mac, install [Docker for Mac](https://docs.docker.com/docker-for-mac/install/).
* If you use Windows, install [Docker for Windows](https://docs.docker.com/docker-for-windows/install/).
* If you use Linux, select the guide for your distribution:
  * [Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
  * [Debian](https://docs.docker.com/install/linux/docker-ce/debian/)
  * [Fedora](https://docs.docker.com/install/linux/docker-ce/fedora/)
  * [CentOS](https://docs.docker.com/install/linux/docker-ce/centos/)

## Post-installation

You should now have `docker` command available in your terminal.
This tool will be used extensively throughout this workshop.

Make sure you can run `docker`:

    $ docker ps

*Linux users:* If the command fails because of permission issues,
add yourself to the Docker group:

    $ sudo usermod -aG docker $(whoami)

Now re-open your terminal, and try `docker ps` again.
If it still doesn't work, logout and log back into desktop.
If it still doesn't work, consult with the presenter to fix the problem.
