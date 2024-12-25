# Isaac ROS Nvblox

Nvblox ROS 2 integration for local 3D scene reconstruction and mapping.

<div align="center"><a class="reference internal image-reference" href="https://media.githubusercontent.com/media/NVIDIA-ISAAC-ROS/.github/main/resources/isaac_ros_docs/repositories_and_packages/isaac_ros_nvblox/isaac_sim_nvblox_humans.gif/"><img alt="image" src="https://media.githubusercontent.com/media/NVIDIA-ISAAC-ROS/.github/main/resources/isaac_ros_docs/repositories_and_packages/isaac_ros_nvblox/isaac_sim_nvblox_humans.gif/" width="600px"/></a></div>

# Quickstart

## Requirements
This repo is tested on ubuntu 22.04, ROS Humble, Cuda 12.1, Nvidia RTX 3060 Ti, Nvidia driver 535.

## Set Up Development Environment
Set up your development environment by following the instructions in getting started. 

- Install the nvidia-container-toolkit using the [instructions](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt).
- Configure nvidia-container-toolkit for Docker using the [instructions](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-docker).

Restart Docker:
```
sudo systemctl daemon-reload && sudo systemctl restart docker
```
Install Git LFS to pull down all large files:
```
sudo apt-get install git-lfs
git lfs install --skip-repo
```
Create a ROS 2 workspace for experimenting with Isaac ROS:
```
mkdir -p  ~/workspaces/isaac_ros-dev/src
echo "export ISAAC_ROS_WS=${HOME}/workspaces/isaac_ros-dev/" >> ~/.bashrc
source ~/.bashrc
```
Clone isaac_ros_common under ${ISAAC_ROS_WS}/src.
```
cd ${ISAAC_ROS_WS}/src
git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common.git isaac_ros_common
```

Make sure required libraries are installed.
```
sudo apt-get install -y curl jq tar
```
Run the following commands to download the `rosbags` from NGC:
```
NGC_ORG="nvidia"
NGC_TEAM="isaac"
PACKAGE_NAME="isaac_ros_nvblox"
NGC_RESOURCE="isaac_ros_nvblox_assets"
NGC_FILENAME="quickstart.tar.gz"
MAJOR_VERSION=3
MINOR_VERSION=2
VERSION_REQ_URL="https://catalog.ngc.nvidia.com/api/resources/versions?orgName=$NGC_ORG&teamName=$NGC_TEAM&name=$NGC_RESOURCE&isPublic=true&pageNumber=0&pageSize=100&sortOrder=CREATED_DATE_DESC"
AVAILABLE_VERSIONS=$(curl -s \
    -H "Accept: application/json" "$VERSION_REQ_URL")
LATEST_VERSION_ID=$(echo $AVAILABLE_VERSIONS | jq -r "
    .recipeVersions[]
    | .versionId as \$v
    | \$v | select(test(\"^\\\\d+\\\\.\\\\d+\\\\.\\\\d+$\"))
    | split(\".\") | {major: .[0]|tonumber, minor: .[1]|tonumber, patch: .[2]|tonumber}
    | select(.major == $MAJOR_VERSION and .minor <= $MINOR_VERSION)
    | \$v
    " | sort -V | tail -n 1
)
if [ -z "$LATEST_VERSION_ID" ]; then
    echo "No corresponding version found for Isaac ROS $MAJOR_VERSION.$MINOR_VERSION"
    echo "Found versions:"
    echo $AVAILABLE_VERSIONS | jq -r '.recipeVersions[].versionId'
else
    mkdir -p ${ISAAC_ROS_WS}/isaac_ros_assets && \
    FILE_REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/\
versions/$LATEST_VERSION_ID/files/$NGC_FILENAME" && \
    curl -LO --request GET "${FILE_REQ_URL}" && \
    tar -xf ${NGC_FILENAME} -C ${ISAAC_ROS_WS}/isaac_ros_assets && \
    rm ${NGC_FILENAME}
fi
```

Create a `download_ngc_assets.sh` file inside `${ISAAC_ROS_WS}/src/isaac_ros_common/scripts/` and paste the contents of this one. Then execute like this:
```
./download_ngc_assets.sh
```

## Set Up isaac_ros_nvblox
There are two options for installing nvblox: installation from Debian, and installation from source.

### Installation from Debian
Launch the Docker container using the run_dev.sh script:
```
cd $ISAAC_ROS_WS && ./src/isaac_ros_common/scripts/run_dev.sh
```
Inside the docker container, install `isaac_ros_nvblox` and its dependencies.
```
sudo apt-get update
sudo apt update &&
sudo apt-get install -y ros-humble-isaac-ros-nvblox && \
rosdep update && \
rosdep install isaac_ros_nvblox
```
### Installation from source
Clone isaac_ros_nvblox under ${ISAAC_ROS_WS}/src.
```
cd ${ISAAC_ROS_WS}/src
git clone --recursive -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nvblox.git isaac_ros_nvblox
```
Launch the Docker container using the run_dev.sh script:
```
cd $ISAAC_ROS_WS/src/isaac_ros_common && \
./scripts/run_dev.sh
```
Inside the docker container, use rosdep to install the package’s dependencies.
```
sudo apt-get update
rosdep update && rosdep install -i -r --from-paths ${ISAAC_ROS_WS}/src/isaac_ros_nvblox/ --rosdistro humble -y
```
Inside the docker container, build and source the ROS workspace.
```
cd /workspaces/isaac_ros-dev
colcon build --symlink-install --base-paths ${ISAAC_ROS_WS}/src/isaac_ros_nvblox/
source install/setup.bash
```
## Run Example Launch File
Inside the docker container, run the example with:
```
ros2 launch nvblox_examples_bringup isaac_sim_example.launch.py \
rosbag:=${ISAAC_ROS_WS}/isaac_ros_assets/isaac_ros_nvblox/quickstart \
navigation:=False
```
Verify that you see the robot reconstructing a mesh, with the 2d ESDF slice overlaid on top.

Exit and stop the docker container that you are currently running:
```
Type exit or press Ctrl + D. This will stop the shell, which, if it's the main process of the container, will stop the container as well.
```
Go inside the docker container that you are running from another terminal:
```
docker exec -it isaac_ros_dev-x86_64-container /bin/bash
```
Stop the docker container:
```
docker stop isaac_ros_dev-x86_64-container
```
Remove the container:
```
docker rm isaac_ros_dev-x86_64-container
```

## Run with ZED Camera
Clone the zed-ros2-wrapper repository on the master branch:
```
cd ${ISAAC_ROS_WS}/src && \
git clone --recurse-submodules https://github.com/stereolabs/zed-ros2-wrapper
```
If you are using ZED X or ZED X Mini refer to the appropriate stereolabs setup guide. Plug in the USB cable of your ZED camera before launching the Docker container in the next step.

Launch the Docker container.
```
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh
```
Now, you will install the ZED SDK. Use `install-zed-aarch64.sh` instead of `install-zed-x86_64.sh` below if you are running on Nvidia Jetson.
```
sudo chmod +x ${ISAAC_ROS_WS}/src/isaac_ros_common/docker/scripts/install-zed-x86_64.sh && \
${ISAAC_ROS_WS}/src/isaac_ros_common/docker/scripts/install-zed-x86_64.sh
```
Install the dependencies for the zed_wrapper package and build it:
```
cd ${ISAAC_ROS_WS} && \
sudo apt update && \
rosdep update && rosdep install --from-paths src/zed-ros2-wrapper --ignore-src -r -y && \
colcon build --symlink-install --packages-up-to zed_wrapper
```
After the container image is rebuilt and you are inside the container, you can run the following to check that the ZED camera is connected.
```
/usr/local/zed/tools/ZED_Explorer
```
ROS2 Example live ZED camera launch inside the docker container for nvblox package:
```
ros2 launch nvblox_examples_bringup zed_example.launch.py \
camera:=<ZED_CAMERA_MODEL>
```
ROS2 Example recorded ZED camera launch for nvblox package:
```
ros2 launch nvblox_examples_bringup zed_example.launch.py \
camera:=<ZED_CAMERA_MODEL> rosbag:=<YOUR_DATASET_PATH>
```

## Run with Realsense Camera 
Example Realsense camera launch inside the docker container:
```
ros2 launch nvblox_examples_bringup realsense_example.launch.py
```

## Performance

The following tables provides timings for various functions of
[nvblox core](https://github.com/nvidia-isaac/nvblox) on various platforms.

<table class="docutils align-default">
    <thead>
        <tr class="row-odd">
            <th class="head">Dataset</th>
            <th class="head">Voxel Size (m)</th>
            <th class="head">Component</th>
            <th class="head">x86_64 w/ 3090 (Desktop)</th>
            <th class="head">x86_64 w/ RTX A3000 (Laptop)</th>
            <th class="head">AGX Orin</th>
            <th class="head">Orin Nano</th>
        </tr>
    </thead>
    <tbody>
        <tr class="row-even">
            <td rowspan="5">Replica</td>
            <td rowspan="5">0.05</td>
            <td>TSDF</td>
            <td>0.5 ms</td>
            <td>0.3 ms</td>
            <td>0.8 ms</td>
            <td>2.1 ms</td>
        </tr>
        <tr class="row-odd">
            <td>Color</td>
            <td>0.7 ms</td>
            <td>0.7 ms</td>
            <td>1.1 ms</td>
            <td>3.6 ms</td>
        </tr>
        <tr class="row-even">
            <td>Meshing</td>
            <td>0.7 ms</td>
            <td>1.3 ms</td>
            <td>2.3 ms</td>
            <td>13 ms</td>
        </tr>
        <tr class="row-odd">
            <td>ESDF</td>
            <td>0.8 ms</td>
            <td>1.2 ms</td>
            <td>1.7 ms</td>
            <td>6.2 ms</td>
        </tr>
        <tr class="row-even">
            <td>Dynamics</td>
            <td>1.7 ms</td>
            <td>1.4 ms</td>
            <td>2.0 ms</td>
            <td>N/A(\*)</td>
        </tr>
        <tr class="row-even">
            <td rowspan="5">Redwood</td>
            <td rowspan="5">0.05</td>
            <td>TSDF</td>
            <td>0.2 ms</td>
            <td>0.2 ms</td>
            <td>0.5 ms</td>
            <td>1.2 ms</td>
        </tr>
        <tr class="row-odd">
            <td>Color</td>
            <td>0.5 ms</td>
            <td>0.5 ms</td>
            <td>0.8 ms</td>
            <td>2.6 ms</td>
        </tr>
        <tr class="row-even">
            <td>Meshing</td>
            <td>0.3 ms</td>
            <td>0.5 ms</td>
            <td>0.9 ms</td>
            <td>4.2 ms</td>
        </tr>
        <tr class="row-odd">
            <td>ESDF</td>
            <td>0.8 ms</td>
            <td>1.0 ms</td>
            <td>1.5 ms</td>
            <td>5.1 ms</td>
        </tr>
        <tr class="row-even">
            <td>Dynamics</td>
            <td>1.0 ms</td>
            <td>0.7 ms</td>
            <td>1.2 ms</td>
            <td>N/A(\*)</td>
        </tr>
     </tbody>
</table>

(\*): Dynamics not supported on Jetson Nano.

---

## Overview

[Isaac ROS Nvblox](https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nvblox) contains ROS 2 packages for 3D reconstruction and cost maps for navigation. `isaac_ros_nvblox` processes depth and pose to
reconstruct a 3D scene in real-time and outputs a 2D costmap for [Nav2](https://github.com/ros-planning/navigation2). The costmap is used in planning during navigation as a vision-based solution to avoid
obstacles.

<div align="center"><a class="reference internal image-reference" href="https://media.githubusercontent.com/media/NVIDIA-ISAAC-ROS/.github/main/resources/isaac_ros_docs/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox_nodegraph.png/"><img alt="image" src="https://media.githubusercontent.com/media/NVIDIA-ISAAC-ROS/.github/main/resources/isaac_ros_docs/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox_nodegraph.png/" width="750px"/></a></div>

Above is a typical graph that uses `isaac_ros_nvblox`.
Nvblox takes a depth image, a color image, and a pose as input, with
which it computes a 3D scene reconstruction on the GPU. In this graph
the pose is computed using `visual_slam`, or some other pose estimation
node. The reconstruction
is sliced into an output cost map which is provided through a cost map plugin
into [Nav2](https://github.com/ros-planning/navigation2).
An optional colorized 3D reconstruction is delivered into `rviz`
using the mesh visualization plugin. Nvblox streams mesh updates
to RViz to update the reconstruction in real-time as it is built.

`isaac_ros_nvblox` offers several modes of operation. In its default mode
the environment is assumed to be static. Two additional modes of operation are provided
to support mapping scenes which contain dynamic elements: people reconstruction, for
mapping scenes containing people, and dynamic reconstruction, for mapping
scenes containing more general dynamic objects.
The graph above shows `isaac_ros_nvblox` operating in people reconstruction
mode. The color image corresponding to the depth image is processed with `unet`, using
the PeopleSemSegNet DNN model to estimate a segmentation mask for
persons in the color image. Nvblox uses this mask to separate reconstructed persons into a
separate people-only part of the reconstruction. The [Technical Details](https://nvidia-isaac-ros.github.io/concepts/scene_reconstruction/nvblox/technical_details.html)
provide more information on these three types of mapping.


## Packages

* [`isaac_ros_nvblox`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html)
  * [Quickstart](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#quickstart)
    * [Set Up Development Environment](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#set-up-development-environment)
    * [Download Quickstart Assets](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#download-quickstart-assets)
    * [Set Up `isaac_ros_nvblox`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#set-up-package-name)
    * [Run Example Launch File](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#run-example-launch-file)
  * [Try More Examples](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#try-more-examples)
  * [API](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#api)
    * [ROS Parameters](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/api/parameters.html)
    * [ROS Topics and Services](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/api/topics_and_services.html)
  * [Troubleshooting](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/index.html#troubleshooting)
    * [Isaac Sim Issues](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/troubleshooting/troubleshooting_nvblox_isaac_sim.html)
    * [RealSense Issues](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/troubleshooting/troubleshooting_nvblox_realsense.html)
    * [ROS Communication Issues](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/isaac_ros_nvblox/troubleshooting/troubleshooting_nvblox_ros_communication.html)
* [`nvblox_examples_bringup`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_examples_bringup/index.html)
* [`nvblox_image_padding`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_image_padding/index.html)
* [`nvblox_isaac_sim`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_isaac_sim/index.html)
* [`nvblox_msgs`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_msgs/index.html)
* [`nvblox_nav2`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_nav2/index.html)
* [`nvblox_performance_measurement`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_performance_measurement/index.html)
* [`nvblox_ros`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_ros/index.html)
* [`nvblox_ros_common`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_ros_common/index.html)
* [`nvblox_rviz_plugin`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/nvblox_rviz_plugin/index.html)
* [`realsense_splitter`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/realsense_splitter/index.html)
* [`semantic_label_conversion`](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_nvblox/semantic_label_conversion/index.html)

## Latest

Update 2024-12-10: Optimized performance for always-on dynamic obstacle detection and 1 cm voxels
