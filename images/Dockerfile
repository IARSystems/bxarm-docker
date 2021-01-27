# 
# Copyright (c) 2020-2021 IAR Systems AB
#
# BXARM Dockerfile
#
# Creates a bare minimal BXARM Docker image
#
# See LICENSE for detailed license information
#

# Create a Docker Image for BXARM based 
#  on the Docker Hub Ubuntu image
FROM ubuntu:18.04 as bxarm-8.50.6

# The USER_ID argument will allow to the Docker Image user 
#  to inherit the same UID and GID from the user creating the Docker image
ARG  USER_ID

# Optional - configure Ubuntu's timezone
ENV TZ=Europe/Stockholm
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Updates the Ubuntu packages, install sudo and cleanup
RUN  apt-get update && \
     apt-get install sudo libsqlite3-0 && \
     apt-get clean autoclean && \
     apt-get autoremove -y && \
     rm -rf /var/lib/apt/lists/*
     
# Optional - configure the Ubuntu's locale (useful for CMake users)
ENV LC_ALL=C

# The environment variables set with ENV will become available for when running the container
# The IAR Build Tools are going to be appended to the tip of the search PATH
# When BXARM is installed, a symbolic link named `bxarm` is automatically created in IAR_PATH
# The IAR_LMS_SETTINGS_DIR can be used to override the default license configuration path (default: /usr/local/etc/IARSystems)
ENV  IAR_PATH="/opt/iarsystems"
ENV  IAR_BXARM_PATH="${IAR_PATH}/bxarm"
ENV  IAR_LMS_SETTINGS_DIR="/.lms"
ENV  IAR_BUILD_PATH="/build"
ENV  PATH="${IAR_BXARM_PATH}/arm/bin:${IAR_BXARM_PATH}/common/bin:$PATH"

# Create the image with an user named `iaruser` using the same UID from the host's user
# For convenience, the user `iaruser` will be added to the "sudo" group 
RUN  useradd --create-home --uid ${USER_ID} iaruser && \
     echo "iaruser:iaruser" | chpasswd && \
     adduser iaruser sudo 

# Copy the .deb package from the host to the image's /tmp directory
COPY bxarm-8.50.6.deb /tmp

# Install the .deb package 
# And then remove the installer from the image
RUN  dpkg --install /tmp/bxarm-8.50.6.deb && \ 
     rm /tmp/bxarm-8.50.6.deb

# Setup the IAR_BUILD_PATH
RUN  mkdir ${IAR_BUILD_PATH} && \
     chmod u+rwx -R ${IAR_BUILD_PATH} && \
     chmod g+rwx -R ${IAR_BUILD_PATH} && \
     chown iaruser:iaruser ${IAR_BUILD_PATH}

# Setup the IAR_LMS_SETTINGS_DIR
RUN  mkdir -p ${IAR_LMS_SETTINGS_DIR} && \
     chmod u+rwx -R ${IAR_LMS_SETTINGS_DIR} && \
     chmod g+rwx -R ${IAR_LMS_SETTINGS_DIR} && \
     chown iaruser:iaruser ${IAR_LMS_SETTINGS_DIR} 

# Setup the initial user and work directory for the image
USER iaruser 
WORKDIR ${IAR_BUILD_PATH}
