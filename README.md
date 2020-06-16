# Tutorial: Docker Images for IAR Build tools on Linux hosts

The [IAR build tools for Linux hosts](https://www.iar.com/iar-embedded-workbench/build-tools-for-linux/) are downloadable directly from our customer's portal at [IAR My Pages](http://iar.com/mypages). 
Please reach out to [fae@iar.com](mailto:fae@iar.com?subject=Tell%20me%20more%20about%20bxarm-docker)
 if you have any questions or would like to learn how to get access to them.

This tutorial provides the fundamental steps on how to create a __BXARM Docker image__ and how to run it on a *[Docker](https://www.docker.com/) container*. Before you start the walkthrough, make sure you have a non-root superuser account - *a user with __sudo__ privileges* - if you need to install the Docker engine on the Ubuntu host. If you already have the Docker engine in place, the standard user account has to belong to the __docker__ group. This tutorial also assumes that you already have familiarity on basic usage of *Ubuntu, Bash, Git* and *Docker*.

### Table of Contents

* [Install Docker](#install-docker)
* [Clone the bxarm-docker repository](#clone-the-bxarm-docker-repository)
* [Build a BXARM Docker image](#build-a-bxarm-docker-image)
* [Configure the host to use a license from a LMS2 license server](#configure-the-host-to-use-a-license-from-a-lms2-network-license-server)
* [How to run BXARM Docker](#how-to-run-bxarm-docker)
* [Summary](#summary)

## Install Docker

The following steps are the typical ones needed to make Docker ready to be used on the Ubuntu host which will hold the Docker images and run the containers. These installation instructions are based on the official ones available in the [Docker documentation](https://docs.docker.com/engine/install/ubuntu).

```sh
# Adds the Docker repository's authentication to the apt's keyring
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -   

# Adds the official Docker repository to the apt list
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Update the apt package list cache
$ sudo apt update

# Install the needed packages
$ sudo apt install apt-transport-https ca-certificates curl gnupg-agent docker-ce docker-ce-cli containerd.io

# Add the current user to the docker group
$ sudo usermod -a -G docker $USER

# Reload the current $USER so the new group takes effect immediately
$ su - $USER

# Make the Docker service available since boot
$ sudo systemctl start docker
$ sudo systemctl enable docker

# Test the Docker installation
$ docker run hello-world
```

## Clone the bxarm-docker repository

Now it is time to clone the __bxarm-docker__ repository.

```sh
$ git clone https://github.com/IARSystems/bxarm-docker.git
````

## Build a BXARM Docker image

The commands in this section are for building a __BXARM Docker image__ based on the provided [__Dockerfile__](images/Dockerfile).

* Remember to update *path-to, version* and *build* of the tarball accordingly. For this tutorial, we will show it with __BXARM v8.50.4-261__.

```sh
# Copies the official BXARM tarball to the images subdirectory
$ cp -v <path-to>/bxarm-<version>-<build>.tgz ./bxarm-docker/images/bxarm.tgz

# Builds the BXARM Docker image tagged as iar/bxarm-8.50.4-261
$ docker build --build-arg USER_ID=$(id -u) -t iar/bxarm-8.50.4-261 --rm=true --force-rm=true ./bxarm-docker/images
````

## Configure the host to use a license from a LMS2 network license server

This section shows how to configure the host to make use of a __LMS2__ (*License Management System version 2*) server.

* Remember to update the __LMS2 server IP__ to your actual LMS2 server IP address.

```sh
# Enables the execution bit for the license setup script
$ chmod a+x ./bxarm-docker/scripts/lms_setup

# Configure the host to use a license from a LMS2 network license server
$ ./bxarm-docker/scripts/lms_setup -s <LMS2 server IP> 
````

## How to run BXARM Docker

Now it is finally time to test your brand new __BXARM Docker image__ running it within a Docker container. As the Docker command line to run the container is lengthy to type every single time, it is much more convenient to create an alias and make it permanently available on the *.bash_aliases*. The one liner that accomplishes it for the current user is: 

```sh
# Creates an alias as a shorthand to run the BXARM Docker image in a container
$ echo "alias bxdocker='docker run -v $(pwd):/build -v $HOME/.lms:/.lms iar/bxarm-8.50.4-261'" >> ~/.bash_aliases && . ~/.bash_aliases
```

From this point onwards, `bxdocker` becomes the shorthand for the lengthy `docker run ...` command, always available:

```sh
$ bxdocker iccarm ./bxarm-docker/tests/main.c
```
And, in a similar manner, it is also possible to run a container interactively.

```sh
# Creates an alias as a shorthand to interactively run the BXARM Docker image in a container
$ echo "alias bxdockerit='docker run -v $(pwd):/build -v $HOME/.lms:/.lms -it iar/bxarm-8.50.4-261'" >> ~/.bash_aliases && . ~/.bash_aliases

$ bxdockerit
```

## Summary

And that is how it can be done. Keep in mind that this mini tutorial is only intended to provide you with a quick start towards our [IAR Build Tools for Linux hosts](https://www.iar.com/iar-embedded-workbench/build-tools-for-linux) within Docker scenarios. It is definitely not a replacement for the extensive [Docker documentation](https://docs.docker.com/). The steps described for creating a bare minimal __BXARM Docker Image__ sum up as a cornerstone for many existing build server topologies. The best aspect of starting with a [__Dockerfile__](images/Dockerfile) is that, taking it as a template for creating your very own image from scratch, also brings the benefit of empowering you towards a wide range of image customization possibilites, in accordance with your specific build server topology needs.

Questions? Feel free to reach out to us via [fae@iar.com](mailto:fae@iar.com?subject=Tell%20me%20more%20about%20bxarm-docker).
