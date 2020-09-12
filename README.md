# Tutorial: Docker images for IAR Build tools on Linux hosts

The [IAR build tools for Linux hosts][iar-bx-url] are downloadable directly from our customer's portal at [IAR My Pages][iar-myp-url]. 
Please feel free to reach out to [fae@iar.com][fae-mail] if you would like to learn how to get access to them.

If you have questions, you can also check the [__bxarm-docker wiki__][repo-wiki-url], or [here][repo-old-issue-url] for earlier questions.
If you have a new question, post it [here][repo-new-issue-url].

This tutorial provides the fundamental steps on how to create a __BXARM Docker image__ and how to run it on a [_Docker Container_][docker-container-url]. 

Before you start the walkthrough, make sure you have a non-root superuser account - *a user with __sudo__ privileges* - if you need to install the Docker engine on the Ubuntu Host. If you already have the Docker engine in place, the standard user account has to belong to the __docker__ group. This tutorial also assumes that you already have familiarity on basic usage of *Ubuntu, Bash, Git* and *Docker*.

### Table of Contents

* [Install Docker](#install-docker)
* [Clone the bxarm-docker repository](#clone-the-bxarm-docker-repository)
* [Build a BXARM Docker image](#build-a-bxarm-docker-image)
* [Setup host environment](#setup-host-environment)
* [Host license configuration](#host-license-configuration)
* [Running BXARM Docker](#running-bxarm-docker)
* [Summary](#summary)

## Install Docker

The following steps are the typical ones needed to make Docker ready to be used on the Ubuntu host which will hold the Docker images and run the containers. These installation instructions are based on the official ones available in the [_Docker Documentation_][docker-docs-url].
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
```
$ git clone https://github.com/IARSystems/bxarm-docker.git
```

## Build a BXARM Docker image

The commands in this section are for building a __BXARM Docker image__ based on the provided [__Dockerfile__](images/Dockerfile).

Copy the bxarm-`<version>`.deb package to `./bxarm-docker/images/`, the same folder where the __Dockerfile__ is located.
```sh
# Copies the official BXARM .deb package to the images subdirectory
$ cp -v <path-to>/bxarm-<version>.deb ./bxarm-docker/images/
```
* Update `<path-to>` and `<version>` of the install package accordingly. For this tutorial, we will use __bxarm-8.50.6.deb__.

Build the image with the `docker build` command below.
```sh
# Builds the BXARM Docker image tagged as iarsystems/bxarm-8.50.6
$ docker build --build-arg USER_ID=$(id -u) -t iarsystems/bxarm-8.50.6 --rm=true --force-rm=true ./bxarm-docker/images
```

In the end, you should have your __BXARM Docker image__ successfully built and tagged as `iarsystems/bxarm-8.50.6:latest`.
```
...
...
Removing intermediate container 2c2c85303c4f
---> 9c4098069d9b
Step 15/15 : WORKDIR ${IAR_BUILD_PATH}
---> Running in 9f2cb70b7dbc
Removing intermediate container 9f2cb70b7dbc
---> 42fa81e5d7a7
Successfully built 42fa81e5d7a7
Successfully tagged iarsystems/bxarm-8.50.6:latest
```


## Setup Host environment

In this section, we will take advantage of the Bash __aliases__ so we can simplify how we run every build tool in the BXARM Docker Container.

The __bxarm-docker__ standard aliases are set with the [__bxarm-docker-aliases-set__](scripts/bxarm-docker-aliases-set) script:
```sh
# Set the standard bxarm-docker aliases 
$ source bxarm-docker/scripts/bxarm-docker-aliases-set
```

From this point onwards, when invoking the build tools, those will all refer to the BXARM Docker Container. You can always check which aliases are currently set in the Host with the __alias__ command:
```sh
$ alias
```

> __Note__: The [provided aliases](scripts/bxarm-docker-aliases-set) make the usage of the build tools seamless with projects located on the current directory (`$PWD`) and its subdirectories. Keep in mind that all these aliases are customizable and ultimately optional. If you need to unset them, it is possible to quickly do so by using the [__bxarm-docker-aliases-unset__](scripts/bxarm-docker-aliases-unset) script:
> ```sh
> # Unset the standard bxarm-docker aliases 
> $ source bxarm-docker/scripts/bxarm-docker-aliases-unset
> ```

For this tutorial we will use the aliases from the provided script when running the build tools from the BXARM Docker image.

## Host license configuration

This section shows how to configure the license on the Host for when using it containerized with Docker.

It requires an [__IAR License Server__][lms2-url] already up, loaded with activated BXARM licenses and reachable from the Host.

Executing the build tools prior properly setting up the license will result in a fatal error message: _"No license found."_
```
$ iccarm --version

   IAR ANSI C/C++ Compiler V8.50.6.265/LNX for ARM
   Copyright 1999-2020 IAR Systems AB.
Fatal error[LMS001]: License check failed. Use the IAR License Manager to
          resolve the problem.
No license found. [LicenseCheck:2.17.3.J190,
          RMS:9.4.0.0023, Feature:ARMBX.EW.COMPILER, Version:1.18]
Fatal error detected, aborting.
```

The Host license setup is performed with `lightlicensemanager` as follows:
```sh
# Override the Host's default license settings directory and make it persistent
$ echo "export IAR_LMS_SETTINGS_DIR=$HOME/.lms" >> ~/.bashrc && source ~/.bashrc

# Make the directory for the license settings
$ mkdir $IAR_LMS_SETTINGS_DIR

# Setup the Host license to point to the IAR License Server (LMS2)
$ lightlicensemanager setup -s <LMS2.server.IP> 
```

> __Notes__:
> * Update the `<LMS2.server.IP>` with the actual IP address to the __IAR License Server__. 
> * It is possible to customize the `IAR_LMS_SETTINGS_DIR` environment variable to point a different location. The location does not necessarily need to belong to the _build tools user_, but requires read/write/execute (`rwx`) access permissions. If the chosen location is volatile, such as `/tmp/.lms`, the Host license setup will need to be run every time after the Host reboots.
> * There are cases where a Firewall could be preventing the Host from reaching the IAR License Server. IAR Systems provides a [__Tech Note__][lms-port-url] covering such cases.


Once the license is properly setup, it should be possible to run all the build tools.
```
$ iccarm --version
IAR ANSI C/C++ Compiler V8.50.6.265/LNX for ARM
```


## Running BXARM Docker

Now it is finally time to test your brand new __BXARM Docker image__ to its fully extent, by actually building projects.

In this section, you will find many examples on how to use the build tools with a ready-made project named [c-stat.ewp](tests/c-stat).

For a more complete description of the existing options for any build tool, invoke the tool without any parameters.

To build an `.ewp` project, use `iarbuild`.
```sh
# Syntax: iarbuild <project>.ewp -build <configuration> [-parallel <cpu-cores>]

# For example, run:
$ iarbuild bxarm-docker/tests/c-stat/c-stat.ewp -build "Debug" -parallel 2
```

With `iarbuild`, it is also possible to run C-STAT, the Static Analysis tool. It will use the same checks that were selected and saved on the `<project>.ewt` file.
```sh
# Syntax: iarbuild <project>.ewp -c_stat_analyze <configuration> [-parallel <cpu-cores>]

# For example, run:
$ iarbuild bxarm-docker/tests/c-stat/c-stat.ewp -cstat_analyze "Debug" -parallel 2
```

After running __C-STAT__, a SQLite database is generated. With the `cstat.db` contents, you can now quickly generate a _full HTML report_.
```sh
# Syntax: ireport --db <path-to>/cstat.db --project <path-to>/<project>.ewp [--full] [--output <path-to>/<report-output>.html]

# For example, run:
$ ireport --db bxarm-docker/tests/c-stat/Debug/Obj/cstat.db --project bxarm-docker/tests/c-stat/c-stat.ewp --full --output bxarm-docker/tests/c-stat/c-stat-report.html 
HTML report generated: bxarm-docker/tests/c-stat/c-stat-report.html
```

There is an alias readily available for running the container in interactive mode:
```
$ bxarm-docker-interactive
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

iaruser@96a4986f8535:/build$ iarbuild bxarm-docker/tests/c-stat/c-stat.ewp -build "*" -parallel 2
...
...
Total number of errors: 0
Total number of warnings: 0
iaruser@96a4986f8535:/build$ exit
```

* To speed up the build time, replace `<cpu-cores>` by the amount of cores in the Host's CPU.

## Summary

And that is how it can be done. Keep in mind that this mini tutorial is only intended to provide you with a quick start towards our [IAR Build Tools for Linux hosts][iar-bx-url] within Docker scenarios. It is definitely not a replacement for the extensive [_Docker Documentation_][docker-docs-url]. The steps described for creating a bare minimal __BXARM Docker image__ sum up as a cornerstone for many existing build server topologies. The best aspect of starting with a [__Dockerfile__](images/Dockerfile) is that, taking it as a template for creating your very own image from scratch, also brings the benefit of empowering you towards a wide range of image customization possibilites, in accordance with your specific build server topology needs.

[iar-bx-url]: https://www.iar.com/bx/
[iar-myp-url]: https://iar.com/mypages/
[fae-mail]: mailto:fae@iar.com?subject=Tell%20me%20more%20about%20bxarm-docker
[docker-url]: https://www.docker.com/
[docker-docs-url]: https://docs.docker.com/
[docker-container-url]: https://www.docker.com/resources/what-container/
[docker-docs-ubuntu-url]: https://docs.docker.com/engine/install/ubuntu/
[lms-port-url]: https://www.iar.com/support/tech-notes/licensing/iar-license-server-how-to-open-udp-5093/
[lms2-url]: https://www.iar.com/support/tech-notes/licensing/iar-license-server-tools-lms2/
[repo-wiki-url]: https://github.com/IARSystems/bxarm-docker/wiki
[repo-new-issue-url]: https://github.com/IARSystems/bxarm-docker/issues/new
[repo-old-issue-url]: https://github.com/IARSystems/bxarm-docker/issues?q=is%3Aissue+is%3Aopen%7Cclosed
