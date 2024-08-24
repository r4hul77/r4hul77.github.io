---
layout: post  
title: Setting Up SDKManager on Any Linux Distro  
date: 2024-08-24 14:14:16  
description: A Quick Guide to Setting Up Docker for Jetson Flashing  
tags: Robotics, Networking, Debugging, Jetson, Docker  
categories: Jetson  
---

# Flashing Jetson Devices on Any Linux Distro Using Docker

NVIDIA's Jetson series represents the cutting edge of embedded computing, offering powerful processing capabilities for AI, robotics, and more. However, flashing the latest JetPack on a Jetson device traditionally requires an Ubuntu machine—specifically one older than version 22.04. For many, this is far from ideal.

In this guide, I'll show you how to bypass this limitation by setting up a Docker container with a GUI, allowing you to flash your Jetson device regardless of your host Linux distribution. Let's dive in!

## Step 0: Install Required Components on Host
We need to install `binfmts` by running `sudo apt install binfmts-support`.

## Step 1: Download SDKManager

Download the latest version of SDKManager from the [NVIDIA Developer website](https://developer.nvidia.com/sdk-manager). Rename the file to `sdkmanager.deb` for convenience.

## Step 2: Build the Docker Image

Next, we'll create a Docker image that can run SDKManager with a GUI interface. Follow these steps:

1. On your Ubuntu machine, create a new directory and move the `sdkmanager.deb` file into it.
2. Create a Dockerfile in the same directory and name it `sdkmanager.Dockerfile`.
3. Copy the following contents into `sdkmanager.Dockerfile`:

    ```Dockerfile
    # Use the official Ubuntu 22.04 as a parent image
    FROM ubuntu:22.04

    # Set environment variables to avoid prompts during package installation
    ENV DEBIAN_FRONTEND=noninteractive

    # Update the package list and install essential packages
    RUN apt-get update && \
        apt-get install -y --no-install-recommends \
        build-essential \
        curl \
        wget \
        vim \
        git \
        ca-certificates \
        software-properties-common && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*


    WORKDIR /workspace
    COPY sdkmanager_2.1.0-11698_amd64.deb sdkmanger.deb

    RUN apt update && chmod 777 sdkmanger.deb && apt-get install -y ./sdkmanger.deb


    RUN apt-get -y install \
        firefox \
        libcanberra-gtk-module \
        libcanberra-gtk3-module
    # Set the working directory

    ARG USERNAME=non-root
    ARG USER_UID=1000
    ARG USER_GID=$USER_UID

    # Create the user
    RUN groupadd --gid $USER_GID $USERNAME \
        && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
        #
        # [Optional] Add sudo support. Omit if you don't need to install software after connecting.
        && apt-get update \
        && apt-get install -y sudo \
        && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
        && chmod 0440 /etc/sudoers.d/$USERNAME

    # ********************************************************
    # * Anything else you want to do like clean up goes here *
    # ********************************************************
    
    RUN printf '#!/bin/sh\nexit 0' > /usr/sbin/policy-rc.d
    RUN apt-get -y install apt-utils usbutils
    # [Optional] Set the default user. Omit if you want to keep the default as root.
    USER $USERNAME
    ```

This Dockerfile sets up an Ubuntu 22.04 environment, installs SDKManager, and creates a non-root user to avoid permission issues during flashing.

4. To build the Docker image, open a terminal in the same directory and run:

    ```bash
    docker build -t sdkmanager:latest . --file sdkmanager.Dockerfile
    ```

## Step 3: Run the Docker Image

Now that we have our Docker image, let's run it. We'll create a bash script to simplify the process.

1. In the same directory, create a new file named `sdkmanager.sh` and paste the following script:

    ```bash
    #!/bin/bash
    xhost +
    docker run -it --privileged -v /dev/bus/usb:/dev/bus/usb/ -v /dev:/dev -v /media/$USER:/media/nvidia:slave --network host -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix sdkmanager:latest
    ```

2. Make the script executable:

    ```bash
    chmod +x sdkmanager.sh
    ```

3. To run the Docker container, simply execute:

    ```bash
    bash sdkmanager.sh
    ```

Once inside the container, start SDKManager by running:

```bash
sdkmanager
```
Follow the on-screen instructions to flash your Jetson device. Happy flashing!


