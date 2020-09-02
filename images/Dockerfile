# 
# Copyright (c) 2020 IAR Systems AB
#
# BXARM Dockerfile
#
# Creates a bare minimal BXARM Docker image
#
# See LICENSE.md for detailed license information
#

# Create a Docker Image for BXARM based 
#  on the Docker Hub Ubuntu image
FROM ubuntu:18.04 as bxarm-8.50.6

# The USER_ID argument will allow to the Docker Image user 
#  to inherit the same UID and GID from the user creating the Docker image
ARG  USER_ID

# These environment variables will also become available
#  when running a container. The IAR build tools 
#  are going to be on the search path
ENV  IAR_PATH="/opt/iarsystems"
ENV  IAR_BXARM_PATH="${IAR_PATH}/bxarm-8.50.6"
ENV  IAR_LMS_SETTINGS_DIR="/.lms"
ENV  IAR_BUILD_PATH="/build"
ENV  PATH="${IAR_BXARM_PATH}/arm/bin:${IAR_BXARM_PATH}/common/bin:$PATH"

# Updates the Ubuntu packages, install sudo and cleanup
RUN  apt-get update && \
     apt-get install sudo libsqlite3-0 && \
     apt-get clean autoclean && \
     apt-get autoremove -y && \
     rm -rf /var/lib/apt/lists/*

# Add an user to the image with the same UID and GID  
#  from the user creating the BXARM Docker image
#  the user is also added to the sudoers group on the image
RUN  useradd --create-home --uid ${USER_ID} iaruser && \
     echo "iaruser:iaruser" | chpasswd && \
     adduser iaruser sudo 

# Copies the .deb package from the host to the image
COPY bxarm-8.50.6.deb /tmp

# Install the .deb package and then the installer from the image
RUN  dpkg --install /tmp/bxarm-8.50.6.deb && \ 
     rm /tmp/bxarm-8.50.6.deb

# Creates and sets the build directory
RUN  mkdir ${IAR_BUILD_PATH} && \
     chmod u+rwx -R ${IAR_BUILD_PATH} && \
     chmod g+rwx -R ${IAR_BUILD_PATH} && \
     chown iaruser:iaruser ${IAR_BUILD_PATH}

# Creates and sets the lms directory
RUN  mkdir -p ${IAR_LMS_SETTINGS_DIR} && \
     chmod u+rwx -R ${IAR_LMS_SETTINGS_DIR} && \
     chmod g+rwx -R ${IAR_LMS_SETTINGS_DIR} && \
     chown iaruser:iaruser ${IAR_LMS_SETTINGS_DIR} 

# When the container is started, it will start from the ${IAR_BUILD_PATH} directory
USER iaruser 
WORKDIR ${IAR_BUILD_PATH}
