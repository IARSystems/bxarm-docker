# Tutorial: Docker images for IAR Build tools on Linux hosts

The [IAR build tools for Linux hosts][iar-bx-url] requires a separate license. Please feel free to [contact us](https://www.iar.com/about-us/contact) if you would like to learn how to get access to them.

If you have questions, you can also check the [__bxarm-docker wiki__][repo-wiki-url], or [here][repo-old-issue-url] for earlier questions.
If you have a new question, post it [here][repo-new-issue-url].

This tutorial provides the fundamental steps on how to create a __BXARM Docker image__ and how to run it on a [_Docker Container_][docker-container-url]. 

Before you start the walkthrough, make sure you have a non-root superuser account - *a user with __sudo__ privileges* - if you need to install the Docker engine on the Ubuntu Host. If you already have the Docker engine in place, the standard user account has to belong to the __docker__ group. This tutorial also assumes that you already have familiarity on basic usage of *Ubuntu, Bash, Git* and *Docker*.

### Additional Resources
If you are new to CI/CD, Docker, Jenkins and Self-Hosted Runners or just want to learn more and see the IAR tools in action, you can find an useful selection of recorded webinars about automated building and testing in Linux-based environments [here!][iar-bx-url]

### Table of Contents

* [Installing Docker](#installing-docker)
   * [Setup the Official Docker Repository](#setup-the-official-docker-repository)
   * [Install the Docker Engine](#install-the-docker-engine)
* [Clone the bxarm-docker repository](#clone-the-bxarm-docker-repository)
* [Build a BXARM Docker image](#build-a-bxarm-docker-image)
* [Setup host environment](#setup-host-environment)
* [Host license configuration](#host-license-configuration)
* [Running BXARM Docker](#running-bxarm-docker)
* [Summary](#summary)

## Installing Docker

The following steps are the typical ones needed to make Docker ready to be used on the Ubuntu host which will hold the Docker images and run the containers. These installation instructions are based on the official ones available in the [_Docker Documentation_][docker-docs-url].

### Setup the Official Docker Repository
Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```sh
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Launch the Ubuntu's Bash shell and add Docker's official repository GPG key to the package management keyring.
```sh
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -   
```

Use the following command to set up the __stable__ repository.
```sh
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

### Install the Docker Engine
Update the `apt` package index, and install the _latest_ version of Docker Engine and containerd.
```sh 
$ sudo apt update && sudo apt -y install docker-ce docker-ce-cli containerd.io
```

In order to use Docker as a non-root user, add the user (referred by the `$USER` environment variable) to the "docker" group.
```sh
$ sudo usermod -aG docker $USER
```
Now log out and log back in so thay your group membership is re-evaluated.

As in many Linux distributions using `systemd` to manage which services to start when the system boots, the Docker service can be configured with `systemctl` to start on boot. To automatically start Docker with the system, enable the service.
```
$ sudo systemctl enable docker && sudo systemctl start docker
```

Verify that your user can run `docker` commands without requiring `sudo` powers.
```sh
$ docker run hello-world
```
This command downloads a test image and runs it in a container. When the container runs, it prints an informational message and exits.

## Clone the bxarm-docker repository

Now it is time to clone the __bxarm-docker__ repository. The repository contains a template __Dockerfile__, some scripts and a example project.
```sh
$ git clone https://github.com/IARSystems/bxarm-docker.git
```

## Build a BXARM Docker image

The commands in this section are for building a __BXARM Docker image__ based on the provided [__Dockerfile__](images/Dockerfile).

Copy the bxarm-`<version>`.deb package to `./bxarm-docker/images/`, the same folder where the __Dockerfile__ is located.
```sh
$ cp -v <path-to>/bxarm-<version>.deb ./bxarm-docker/images/
```
* Update `<path-to>` and `<version>` of the install package accordingly. For this tutorial, we will use __bxarm-8.50.6.deb__.

Build the Docker image with the `docker build` command as below. The image will be tagged as `iarsystems/bxarm-8.50.6`.
```sh
$ docker build --build-arg USER_ID=$(id -u) -t iarsystems/bxarm-8.50.6 --rm=true --force-rm=true ./bxarm-docker/images
```

In the end, you should have your __BXARM Docker image__ successfully built and tagged as `iarsystems/bxarm-8.50.6:latest`.
> ```
> ...
> ...
> Removing intermediate container 2c2c85303c4f
> ---> 9c4098069d9b
> Step 15/15 : WORKDIR ${IAR_BUILD_PATH}
> ---> Running in 9f2cb70b7dbc
> Removing intermediate container 9f2cb70b7dbc
> ---> 42fa81e5d7a7
> Successfully built 42fa81e5d7a7
> Successfully tagged iarsystems/bxarm-8.50.6:latest
> ```


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

> __Note__: The [provided aliases](scripts/bxarm-docker-aliases-set) make the usage of the build tools seamless with projects located on the current directory (`$PWD`) and its subdirectories. Keep in mind that all these aliases are customizable and ultimately optional. If you need to unset them, it is possible to quickly do so with the [__bxarm-docker-aliases-unset__](scripts/bxarm-docker-aliases-unset) script:
> ```sh
> # Unset the standard bxarm-docker aliases 
> $ source bxarm-docker/scripts/bxarm-docker-aliases-unset
> ```

For this tutorial we will use the aliases from the provided script when running the build tools from the BXARM Docker _image_.

## Host license configuration

This section shows how to configure the license on the Host for when using the build tools from a BXARM Docker _container_.

> __Note__
> * The Host license configuration requires an [__IAR License Server__][lms2-url] already __up__, loaded with __activated__ BXARM licenses and __reachable__ from the Host.

Executing the build tools prior properly setting up the license will result in a fatal error message: _"No license found."_
```
$ iccarm --version
```
> ```
>    IAR ANSI C/C++ Compiler V8.50.6.265/LNX for ARM
>    Copyright 1999-2020 IAR Systems AB.
> Fatal error[LMS001]: License check failed. Use the IAR License Manager to
>           resolve the problem.
> No license found. [LicenseCheck:2.17.3.J190,
>           RMS:9.4.0.0023, Feature:ARMBX.EW.COMPILER, Version:1.18]
> Fatal error detected, aborting.
> ```

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
> * Access the [__Installation and Licensing User Guide for Linux__][ug-lms2-lx-1-url] for more information.

[ug-lms2-lx-1-url]: https://netstorage.iar.com/SuppDB/Public/UPDINFO/014853/common/doc/lightlicensemanager/UserGuide_LMS2_LX.ENU.pdf


Once the license is properly setup, it should be possible to run all the build tools.
```
$ iccarm --version
```
> ```
> IAR ANSI C/C++ Compiler V8.50.6.265/LNX for ARM
> ```


## Running BXARM Docker

Now it is finally time to test your brand new __BXARM Docker image__ to its fully extent, by actually building projects.

In this section, you will find some examples on how to use the build tools with a ready-made project named [c-stat.ewp](tests/c-stat).

It is straightforward to build an `.ewp` project with `iarbuild`.
```sh
# Syntax: iarbuild <project>.ewp [command] <build-configuration> [-parallel <cpu-cores>] [other-options]

# For example, run:
$ iarbuild bxarm-docker/tests/c-stat/c-stat.ewp "Debug" 
```

> __Notes__
> * The __`[command]`__ parameter is __optional__. If ommited, it will default to `-make`. Other commands commonly used when build projects are `-build` or `-clean`.
> * The __`<build-configuration>`__ parameter is __mandatory__. Typically it will be `Debug` or `Release`. This parameter accepts multiple comma-separated _build configurations_ such as `Debug,Release[,MyAnotherCustomBuildConfiguration,...]`. Ultimately this parameter accepts the __` * `__ as wildcard. The wildcard will address all the _build configurations_ in the `<project>`.
> * The __`-parallel <cpu-cores>`__ parameter is __optional__. It can significantly reduce the required time for building when the host PC has 2 or more CPU cores.
> * Invoke `iarbuild` with no parameters for a more extensive description on its parameter options.


---
With `iarbuild`, it is also possible to perform static analysis with [C-STAT](https://www.iar.com/cstat). 
```sh
# Syntax: iarbuild <project>.ewp -cstat_analyze <build-configuration> [-parallel <cpu-cores>]

# For example, run:
$ iarbuild bxarm-docker/tests/c-stat/c-stat.ewp -cstat_analyze "Debug" -parallel $(nproc)
```

By default, the [C-STAT](https://www.iar.com/cstat) static analysis outputs a SQLite database named `cstat.db`. Then, use `ireport` to process the database and generate an automatic _full HTML report_ containing all the warnings about coding violations for the `<project>`'s selected checks:
```sh
# Syntax: ireport --db <path-to>/cstat.db --project <path-to>/<project>.ewp [--full] [--output <path-to>/<report>.html]

# For example, run:
$ ireport --db bxarm-docker/tests/c-stat/Debug/Obj/cstat.db \
          --project bxarm-docker/tests/c-stat/c-stat.ewp \
          --full \
          --output bxarm-docker/tests/c-stat/c-stat-report.html
```
The output will be:
> ```
> HTML report generated: bxarm-docker/tests/c-stat/c-stat-report.html
> ```

> __Note__
> * When used in conjunction with `iarbuild`, [C-STAT](https://www.iar.com/cstat) will look for a file named `<project>.ewt` in the `<project>`'s folder. This file is automatically generated by the __IAR Embedded Workbench IDE__ when rulesets other than its _Standard Checks_ were selected for the `<project>`. 

---
The [bxarm-docker-alias-set](scripts/bxarm-docker-aliases-set) script brings the `bxarm-docker-interactive` alias to spawn a container in _interactive mode_:
```
$ bxarm-docker-interactive
```
> ```
> To run a command as administrator (user "root"), use "sudo <command>".
> See "man sudo_root" for details.
> ```
```
iaruser@96a4986f8535:/build$ iarbuild bxarm-docker/tests/c-stat/c-stat.ewp -build "*" -parallel $(nproc)
```
> ```
> ...
> ...
> Total number of errors: 0
> Total number of warnings: 0
> ```

Type `exit` to exit from the interactive container:
```
iaruser@96a4986f8535:/build$ exit
```

## Summary

And that is how it can be done. Keep in mind that this mini tutorial is only intended to provide you with a quick start towards our [IAR Build Tools for Linux hosts][iar-bx-url] within Docker scenarios. It is definitely not a replacement for the extensive [_Docker Documentation_][docker-docs-url]. The steps described for creating a bare minimal __BXARM Docker image__ sum up as a cornerstone for many existing build server topologies. The best aspect of starting with a [__Dockerfile__](images/Dockerfile) is that, taking it as a template for creating your very own image from scratch, also brings the benefit of empowering you towards a wide range of image customization possibilites, in accordance with your specific build server topology needs.

[iar-bx-url]: https://www.iar.com/bx/
[iar-myp-url]: https://iar.com/mypages/
[docker-url]: https://www.docker.com/
[docker-docs-url]: https://docs.docker.com/
[docker-container-url]: https://www.docker.com/resources/what-container/
[docker-docs-ubuntu-url]: https://docs.docker.com/engine/install/ubuntu/
[lms-port-url]: https://www.iar.com/support/tech-notes/licensing/iar-license-server-how-to-open-udp-5093/
[lms2-url]: https://www.iar.com/support/tech-notes/licensing/iar-license-server-tools-lms2/
[repo-wiki-url]: https://github.com/IARSystems/bxarm-docker/wiki
[repo-new-issue-url]: https://github.com/IARSystems/bxarm-docker/issues/new
[repo-old-issue-url]: https://github.com/IARSystems/bxarm-docker/issues?q=is%3Aissue+is%3Aopen%7Cclosed
