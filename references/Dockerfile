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
# Please remember to change: 70604376
RUN echo "${USERNAME}:70604376" | chpasswd \
    && usermod -aG sudo "${USERNAME}"

USER ${USERNAME}

WORKDIR /home/${USERNAME}/ros2_humble_workspace

CMD ["/bin/bash"]