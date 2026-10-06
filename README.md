# Introduction
This article primarily introduces how to install ROS2 Humble on the EPC-WCL Ubuntu 26.04.1 LTS environment provided by Avalue Technology Inc.

# Check Ubuntu 26.04.1 LTS GPU - Accelerated: yes
```bash
sudo apt update
sudo apt install -y mesa-utils
env -u LIBGL_ALWAYS_SOFTWARE glxinfo -B
```

# Install Docker, ROS2 Humble Docker Image
## Update Ubuntu
```bash
sudo apt update
sudo apt upgrade -y
```

## Install Docker Certificates
```bash
sudo apt install -y ca-certificates curl
```

## Install Docker Offical GPG Key
```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## Install Docker Repository
```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

## Install Docker Engine
```bash
sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## Verify Docker Engine
```bash
# Active: active (running)
sudo systemctl status docker

# Hello from Docker!
sudo docker run --rm hello-world
```

## Allow standard users to execute Docker
```bash
sudo usermod -aG docker "$USER"
```

## Reboot to Verify standard users permission
```bash
sudo reboot

# Without Permission Denied
docker ps
```

## Install ROS2 Humble Docker Image
```bash
docker pull osrf/ros:humble-desktop
```

## Start ROS2 Humble Docker Container - Terminal 1
```bash
docker run -it --rm \
  --name ros2_humble_env \
  osrf/ros:humble-desktop \
  bash
  
source /opt/ros/humble/setup.bash

# humble
echo $ROS_DISTRO

# Publishing: 'Hello World: 1'
# Publishing: 'Hello World: 2'
# Publishing: 'Hello World: 3'
# ...
ros2 run demo_nodes_cpp talker
```

## Start ROS2 Demo Node Listener - Terminal 2
```bash
docker exec -it ros2_humble_env bash

source /opt/ros/humble/setup.bash

# I heard: [Hello World: 1]
# I heard: [Hello World: 2]
# I heard: [Hello World: 3]
# ...
ros2 run demo_nodes_cpp listener
```

# Configure GUI Access Permission, ROS2 Humble Docker Container with GUI
## Check Display Environment
```bash
# wayland
echo $XDG_SESSION_TYPE
# :0
echo $DISPLAY
```

## Install X11 Testing Tool
```bash
sudo apt install -y x11-xserver-utils x11-apps

# xhost -si:localuser:root
xhost +si:localuser:root
```

## Test ROS2 Humble Docker Container with GUI without GPU
```bash
docker run -it --rm \
  --name ros2_humble_rviz2 \
  --env DISPLAY="$DISPLAY" \
  --env QT_QPA_PLATFORM=xcb \
  --env QT_X11_NO_MITSHM=1 \
  --env LIBGL_ALWAYS_SOFTWARE=1 \
  --volume /tmp/.X11-unix:/tmp/.X11-unix:ro \
  osrf/ros:humble-desktop \
  bash
  
source /opt/ros/humble/setup.bash  

# If it can't find rviz2, we should install it manually
# apt update
# apt install -y ros-humble-rviz2
which rviz2

rviz2

apt update
apt install -y mesa-utils

# OpenGL renderer string: llvmpipe
glxinfo -B
```

## Create ROS2 Humble Docker Container
### Create Dockerfile
```bash
mkdir -p /home/renity-admin/ros2_humble_workspace

mkdir -p ~/ros2_humble_docker
cd ~/ros2_humble_docker

cat > Dockerfile <<'EOF'
FROM osrf/ros:humble-desktop

ARG USERNAME=ros-admin
ARG USER_UID=1000
ARG USER_GID=1000

# Install ROS Humble - INTEL RealSense, USB Utilities
RUN apt-get update && apt-get install -y \
    ros-humble-librealsense2 \
    ros-humble-realsense2-camera \
    ros-humble-realsense2-camera-msgs \
    ros-humble-realsense2-description \
	ros-humble-rosbridge-server \
	libasio-dev \
	libgoogle-glog-dev \
	libpcap-dev \
	libuvc-dev \
	nlohmann-json3-dev \
	ros-humble-rtcm-msgs \
	ros-humble-bond ros-humble-bondcpp \
	ros-humble-test-msgs \
	ros-humble-filters \
	ros-humble-nav2-msgs \
	ros-humble-navigation2 \
	ros-humble-nav2-bringup \
	ros-humble-image-common \
	ros-humble-pcl-ros \
	ros-humble-pcl-conversions \
	ros-humble-async-web-server-cpp \
	ros-humble-image-publisher \
	ros-humble-image-proc \
	ros-humble-image-transport \
	ros-humble-image-transport-plugins \
	ros-humble-behaviortree-cpp-v3 \
	ros-humble-joint-state-publisher \
	ros-humble-joint-state-publisher-gui \
	ros-humble-imu-filter-madgwick \
	ros-humble-robot-localization \
    usbutils \
    udev \
	sudo \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd --gid "${USER_GID}" "${USERNAME}" \
    && useradd --uid "${USER_UID}" \
       --gid "${USER_GID}" \
       --create-home \
       --shell /bin/bash \
       "${USERNAME}" \
    && { \
       echo "source /opt/ros/humble/setup.bash"; \
       echo "export ROS_DISTRO=humble"; \
       echo "export RMW_IMPLEMENTATION=rmw_fastrtps_cpp"; \
       } >> "/home/${USERNAME}/.bashrc"

# Set User Password, Grant sudo Privileges
# Please remember to change: your_password 
RUN echo "${USERNAME}:your_password" | chpasswd \
    && usermod -aG sudo "${USERNAME}"

USER ${USERNAME}

WORKDIR /home/${USERNAME}/ros2_humble_workspace

CMD ["/bin/bash"]
EOF
```

### Create docker-compose.yml
```bash
cd ~/ros2_humble_docker
nano docker-compose.yml
```

```text
services:
  ros2_humble:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        USERNAME: ros-admin
        USER_UID: ${HOST_UID:-1000}
        USER_GID: ${HOST_GID:-1000}

    image: ros2-humble-desktop:latest
    container_name: ros2_humble

    stdin_open: true
    tty: true

    # ROS 2 DDS networking
    network_mode: host
    ipc: host

    environment:
      # ROS 2
      - ROS_DOMAIN_ID=0
      - ROS_LOCALHOST_ONLY=0

      # Wayland Host -> Xwayland -> RViz2
      - DISPLAY=${DISPLAY}
      - QT_QPA_PLATFORM=xcb
      - QT_X11_NO_MITSHM=1

    volumes:
      # ROS2 Workspace, Avalue AMR
      - ./ros2_humble_workspace:/home/ros-admin/ros2_humble_workspace
      - ./ros2_humble_amr_avalue:/home/ros-admin/ros2_humble_amr_avalue

      # Xwayland display
      - /tmp/.X11-unix:/tmp/.X11-unix:rw

      # Host USB / V4L2 / Serial devices
      - /dev:/dev

    device_cgroup_rules:
      # RealSense V4L2 video nodes
      - 'c 81:* rmw'

      # RealSense USB devices
      - 'c 189:* rmw'

      # RPLIDAR USB serial
      - 'c 188:* rmw'

      # STM32 / IMU ttyACM
      - 'c 166:* rmw'

    # Use numeric GIDs from the Host.
    # Replace these examples with the actual values.
    group_add:
      - "${DIALOUT_GID}"
      - "${VIDEO_GID}"

    working_dir: /home/ros-admin/ros2_humble_workspace

    command: /bin/bash
```

### Build Docker Container
```bash
# Enter Docker Directory
cd /home/renity-admin/ros2_humble_docker

# Get Host user UID / GID
export HOST_UID=$(id -u)
export HOST_GID=$(id -g)

# Get USB Device Group GIDs
export DIALOUT_GID=$(getent group dialout | cut -d: -f3)
export VIDEO_GID=$(getent group video | cut -d: -f3)

# Check environment variables
echo "HOST_UID=$HOST_UID"
echo "HOST_GID=$HOST_GID"
echo "DIALOUT_GID=$DIALOUT_GID"
echo "VIDEO_GID=$VIDEO_GID"

# Build Docker Image, Start Docker Container
docker compose up -d --build
```

### Rebuild Docker Container
```bash
# Enter Docker Directory
cd /home/renity-admin/ros2_humble_docker

# Get Host user UID / GID
export HOST_UID=$(id -u)
export HOST_GID=$(id -g)

# Get USB Device Group GIDs
export DIALOUT_GID=$(getent group dialout | cut -d: -f3)
export VIDEO_GID=$(getent group video | cut -d: -f3)

# Check environment variables
echo "HOST_UID=$HOST_UID"
echo "HOST_GID=$HOST_GID"
echo "DIALOUT_GID=$DIALOUT_GID"
echo "VIDEO_GID=$VIDEO_GID"

# Rebuild Compose Service, if needs, Rebuild Image too
docker compose down
docker compose up -d --build --force-recreate

# Rebuild Image without Cache
docker compose down
docker compose build --no-cache
```

### Dump Docker Container Logs
```bash
docker compose logs -f ros2_humble_rviz2
```

### Stop Docker Container
```bash
docker compose stop ros2_humble_rviz2
```

### Remove Docker Container
```bash
docker compose down --remove-orphans

docker rm -f ros2_humble_rviz2 2>/dev/null || true

# Verify Result
docker ps -a --filter name=ros2_humble_rviz2
```

### Remove Docker Image
```bash
docker image rm ros2-humble-desktop:latest 2>/dev/null || true

# Verify Result
docker images | grep ros2-humble
```

### Open Terminal in the Running ROS2 Humble Docker Container
```bash
# Check Docker Container Status
docker compose ps

# Enter Docker Directory
cd /home/renity-admin/ros2_humble_docker

# Get Host user UID / GID
export HOST_UID=$(id -u)
export HOST_GID=$(id -g)

# Get USB Device Group GIDs
export DIALOUT_GID=$(getent group dialout | cut -d: -f3)
export VIDEO_GID=$(getent group video | cut -d: -f3)

# Check environment variables
echo "HOST_UID=$HOST_UID"
echo "HOST_GID=$HOST_GID"
echo "DIALOUT_GID=$DIALOUT_GID"
echo "VIDEO_GID=$VIDEO_GID"

# Add Local User into GUI/X11
xhost +SI:localuser:$(whoami)

# Start Docker Compose Service
docker compose up -d ros2_humble_rviz2
# Enter Docker Container
docker compose exec ros2_humble_rviz2 bash
```

# Build & Install - Avalue ROS2 Humble AMR Nodes
## In the Host
```bash
cd ~
mkdir ros2_humble_amr_avalue
cd ros2_humble_amr_avalue
# Clone - ros2.humble.amr.avalue Source Code
git clone https://github.com/Avalue-Technology/ros2.humble.amr.avalue.git .
```

## In the Docker Container - ros2_humble_rviz2
```bash
cd ros2_humble_amr_avalue
# Install Dependency 
rosdep install --from-paths . --ignore-src -r -y
# Clean - build, install, log
rm -rf build/ install/ log/
# Compile - ros2.humble.amr.avalue Source Code - All ROS2 Nodes
colcon build --symlink-install
```

# Ensure Docker will start automatically on boot
```bash
sudo systemctl enable --now docker.service
sudo systemctl enable --now containerd.service

# Output should be enabled
systemctl is-enabled docker
# Output should be active
systemctl is-active docker
```

# Create Docker Container, Compose Service Launch Script
## Create Launch Script
```bash
mkdir -p /home/renity-admin/bin
nano /home/renity-admin/bin/launch_docker_ros2_humble_rviz2.sh
```

```text
#!/bin/bash
set -euo pipefail

COMPOSE_DIR="/home/renity-admin/ros2_humble_docker"

echo "[Docker] ROS2 Humble Startup - Start"
echo "User: $(whoami)"
echo "DISPLAY=${DISPLAY:-<unset>}"
echo "XDG_SESSION_TYPE=${XDG_SESSION_TYPE:-<unset>}"
echo "XDG_RUNTIME_DIR=${XDG_RUNTIME_DIR:-<unset>}"

#
# Wait for X11/Xwayland
#
for i in $(seq 1 30); do
    if [ -n "${DISPLAY:-}" ] && [ -d /tmp/.X11-unix ]; then
        echo "X11/Xwayland ready."
        break
    fi

    echo "Waiting for graphical session... ($i/30)"
    sleep 1
done

if [ -z "${DISPLAY:-}" ]; then
    echo "ERROR: DISPLAY is not set."
    exit 1
fi

#
# Host UID/GID
#
export HOST_UID=$(id -u)
export HOST_GID=$(id -g)

#
# Device Group GIDs
#
export DIALOUT_GID=$(getent group dialout | cut -d: -f3)
export VIDEO_GID=$(getent group video | cut -d: -f3)

echo "HOST_UID=$HOST_UID"
echo "HOST_GID=$HOST_GID"
echo "DIALOUT_GID=$DIALOUT_GID"
echo "VIDEO_GID=$VIDEO_GID"

#
# X11 Permission
#
/usr/bin/xhost +SI:localuser:$(whoami)

#
# Start Compose
#
cd "$COMPOSE_DIR"

/usr/bin/docker compose up -d ros2_humble_rviz2

echo "[Docker] ROS2 Humble Startup - End"
```

## Change Launch Script Permission
```bash
chmod +x /home/renity-admin/bin/launch_docker_ros2_humble_rviz2.sh
```

## Test Launch Script
```bash
/home/renity-admin/bin/launch_docker_ros2_humble_rviz2.sh
```

## Verify Launch Script Running
```bash
export HOST_UID=$(id -u)
export HOST_GID=$(id -g)

export DIALOUT_GID=$(getent group dialout | cut -d: -f3)
export VIDEO_GID=$(getent group video | cut -d: -f3)

docker compose \
  -f /home/renity-admin/ros2_humble_docker/docker-compose.yml \
  ps
```

# Create systemd user service for Launch Script
## Create systemd user service
```bash
mkdir -p ~/.config/systemd/user

nano ~/.config/systemd/user/docker_ros2_humble_rviz2.service
```

```text
[Unit]
Description=ROS2 Humble Docker Compose
After=graphical-session.target

[Service]
Type=oneshot
RemainAfterExit=yes

WorkingDirectory=/home/renity-admin/ros2_humble_docker

ExecStart=/home/renity-admin/bin/launch_docker_ros2_humble_rviz2.sh

ExecStop=/usr/bin/docker compose stop ros2_humble_rviz2

[Install]
WantedBy=graphical-session.target
```

## Reload systemd user service
```bash
systemctl --user daemon-reload

systemctl --user enable docker_ros2_humble_rviz2.service
```

## Test systemd user service
```bash
systemctl --user start docker_ros2_humble_rviz2.service
```

## Verify systemd user service
```bash
systemctl --user status docker_ros2_humble_rviz2.service

docker ps
```
