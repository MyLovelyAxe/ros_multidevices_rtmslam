# Real-time SLAM based on multiple devices through ROS2 humble

This project implements real-time SLAM pipeline cooperated with multiple devices (i.e. Raspberry Pi and GPU-equipped PC) which are connected with ROS humble.

## Table of Contents

- [Introduction](#introduction)
- [Requirements](#requirements)
- [Setup](#setup)
- [Usage](#usage)

---

## Introduction

Neural networks have shown great performance for 3D scene reconstruction which has great potential in real-time SLAM task. However, NN models usually require powerful GPU to take effect, even though some of them can work on CPU, the performance still suffers from losing precision and less computing resource. 

Meanwhile, Raspberry Pi is popular for deployment on edge devices, which is flexible and modular to fit different robotic tasks, e.g. deployed on mobile robot. However, Raspberry Pi is still strongly restricted when GPU is necessary for complex task.

In order to combine the advantages of both GPU-equipped device (e.g. PC) and Raspberry Pi, i.e. possess powerful computing resource and flexibility for deployment, this project implement cooperation between PC and Raspberry Pi through ROS humble under the same WIFI environment, where:

- Raspberry Pi offers input data (i.e. image in this project) without dealing with NN

- GPU-equipped PC recontruct 3D scene in real-time based on live-stream images

- ROS humble connects both for image transmission

The following diagram illustrates how different components exchange message and work together:

> **Attention:**  
> On the PC end, since ROS humble runs under system Python 3.10, while most state-of-the-art NN-based models rely on Python 3.11 and PyTorch, this project uses ZMQ sockets to exchange messages between ROS and NN models. This approach avoids forcing ROS and NN models to run under the same Python environment.

<img src="slam_center_diagram.svg" width="800"/>

1. Raspberry Pi and PC connects with ROS humble with the same ROS domain;

2. Raspberry Pi captures images with camera module v3 and transfers them to `ROS image topic`;

3. PC receives images from `ROS image topic` and sends them to ZMQ sockets;

4. NN model receives images from ZMQ sockets and reconstruct for 3D scene;


---

## Requirements

This project is tested on the following hardware & software configuration:

- PC
    - Ubuntu22.04
    - System python 3.10 (for [ROS2 humble](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)) 
    - Conda env python 3.11 (for neural network-based 3D scene reconstruction)
- Raspberry Pi 5
    - [Rasbian OS 64Bit (Bookworm)](https://www.raspberrypi.com/software/)
    - System python 3.10 in docker container (for ROS2 humble)
    - [Camera Module v3](https://www.raspberrypi.com/products/camera-module-3/)

---

## Setup

#### Step 1: Prepare IP address

Since ROS humble replies on **IP address** to make sure Raspberry Pi and PC can find each other under the same WIFI environment, firstly find the IP addresses for both. Run this command on each of device separately, take the returned IP addresses:

```bash
hostname -I
```

E.g. the return is: `192.168.178.41 2003:de:4f13:200:e402:17a2:4b02:1f6f 2003:de:4f13:200:91f7:32d8:15da:1541`, the **`192.168.178.41`** is the IP address of current device.

#### Step 2: Raspberry Pi

1. Raspberry Pi OS

Ensure Rasbian OS 64Bit (Bookworm) is installed on Raspberry Pi 5.

2. Setup ROS humble and camera

Refer to the instruction in repository [rpi5_ros_docker_collection](https://github.com/MyLovelyAxe/rpi5_ros_docker_collection/tree/ros_rpios_humble_camera#ros-humble-in-docker-on-raspberry-pi-5-with-rasbian-bookworm-64bit-with-camera_ros) to setup both ROS humble and camera in docker container (which is Tier1 support according to [ROS2 documentation](https://docs.ros.org/en/foxy/How-To-Guides/Installing-on-Raspberry-Pi.html)).

#### Step 3: GPU-equipped PC

1. Setup ROS humble

Install ROS humble following [ROS2 documentation instruction - installation](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html).

2. Install `slam_center`

Setup ROS package [slam_center](https://github.com/MyLovelyAxe/slam_center/tree/main) which exchanges messages between Raspberry Pi and NN model, including:

- Input: compressed images
- Output: 3D point cloud, camera poses

3. Install NN model for 3D scene reconstruction

In order to integrate into ROS, this project uses a forked and refactored MASt3R-SLAM, refer to `README.md` of repo [MASt3R-SLAM-ROS](https://github.com/MyLovelyAxe/MASt3R-SLAM-ROS) to setup.

[TODO: add installation of pyzmq into repo MASt3R-SLAM-ROS]

#### Step 4: Connect devices with ROS humble

Setup communication between Raspberry Pi 5 and PC with ROS humble under the **same WIFI environment**, in order to transfer images from camera module v3 to PC in live-stream.

1. Raspberry Pi 5

On Raspberry Pi 5, refer to repository [rpi5_ros_docker_collection](https://github.com/MyLovelyAxe/rpi5_ros_docker_collection/tree/ros_rpios_humble_camera#ros-humble-in-docker-on-raspberry-pi-5-with-rasbian-bookworm-64bit-with-camera_ros), enter a container, make sure the following commands in `docker_entrypoint.sh`:

```bash
source /opt/ros/${ROS_DISTRO}/setup.bash # source underlay
source /app/install/setup.bash # source overlay
export ROS_DOMAIN_ID=0 # make sure both devices have the same domain id
export ROS_LOCALHOST_ONLY=0 # make sure ROS not only connect its local host network
export ROS_IP=<IP.address.of.pc>
```

Then source `docker_entrypoint.sh` which makes sure Raspberry Pi 5 is able to find PC by ROS humble:

```bash
source docker_entrypoint.sh
```

2. PC

On PC, Add the following commands into `~/.bashrc`:

```bash
source /opt/ros/humble/setup.bash # source underlay
export ROS_DOMAIN_ID=0 # make sure both devices have the same domain id
export ROS_LOCALHOST_ONLY=0 # make sure ROS not only connect its local host network
export ROS_IP=<IP.address.of.raspberry_pi>
```

Then source `.bashrc` which makes sure PC is able to find Raspberry Pi 5 by ROS humble:

```bash
source .bashrc
```

## Usage

#### 1) Raspberry Pi: offer live-stream images

On Raspberry Pi, enter a ROS humble docker container, and start `camera_ros` node, make the camera on Raspberry Pi keep publishing compressed images to topic `/camera/image_raw/compressed`, according to [here](https://github.com/MyLovelyAxe/rpi5_ros_docker_collection/tree/ros_rpios_humble_camera#terminal-1-start-camera_ros-node):

```bash
docker exec -it <current_container_name> bash
source docker_entrypoint.sh
ros2 run camera_ros camera_node
```

#### 2) PC: transfer images

On PC, open a terminal, ensure not to enter any virtual env, in order to use system python for ROS humble. Start slam_center node to transfer images, i.e. subscribe to `/camera/image_raw/compressed` to get compressed images and publish to a ZMQ socket for NN model:

```bash
cd path/to/ros_workspace/
source install/setup.bash # source underlay and overlay
ros2 run slam_center send_comp_img
```

#### 3) PC: real-time SLAM

On PC, open a new terminal, enter the conda env for [MASt3R-SLAM-ROS](https://github.com/MyLovelyAxe/MASt3R-SLAM-ROS):

```bash
conda activate mast3r
```

Start process of MASt3R-SLAM which subscribes live-stream compressed images from ZMQ socket and reconstructs 3D scene in real-time:

```bash
python main.py
```
