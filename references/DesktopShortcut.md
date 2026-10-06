# Create MAP
## Create Create MAP Launch Script
```bash
mkdir -p /home/renity-admin/bin

nano /home/renity-admin/bin/launch_create_map.sh
```

```text
#!/bin/bash

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

cd /home/renity-admin/ros2_humble_docker || exit 1

exec docker compose exec ros2_humble_rviz2 \
    bash -lc "source /opt/ros/humble/setup.bash && \
              source /home/ros-admin/ros2_humble_workspace/install/setup.bash && \
			  source /home/ros-admin/ros2_humble_amr_avalue/install/setup.bash && \
              exec ros2 launch slam_gmapping slam_gmapping.launch.py"
```

## Grant Create MAP Launch Script Execution Permissions
```
chmod +x /home/renity-admin/bin/launch_create_map.sh
```

## Create Create MAP Launch Script Shortcut
```bash
nano "$(xdg-user-dir DESKTOP)/CreateMAP.desktop"
```

```text
[Desktop Entry]
Version=1.0
Type=Application
Name=Create MAP
Comment=Create ROS2 MAP - SLAM Gmapping in Docker
Exec=/home/renity-admin/bin/launch_create_map.sh
Icon=utilities-terminal
Terminal=true
Categories=Development;
```

## Grant Create MAP Launch Script Shortcut Execution Permissions
```bash
chmod +x "$(xdg-user-dir DESKTOP)/CreateMAP.desktop"
```

# Save MAP
## Create Save MAP Launch Script
```bash
mkdir -p /home/renity-admin/bin

nano /home/renity-admin/bin/launch_save_map.sh
```

```text
#!/bin/bash

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

cd /home/renity-admin/ros2_humble_docker || exit 1

exec docker compose exec ros2_humble_rviz2 \
    bash -lc "source /opt/ros/humble/setup.bash && \
              source /home/ros-admin/ros2_humble_workspace/install/setup.bash && \
			  source /home/ros-admin/ros2_humble_amr_avalue/install/setup.bash && \
              exec ros2 launch avalue_robot_nav2 save_map.launch.py"
```

## Grant Save MAP Launch Script Execution Permissions
```
chmod +x /home/renity-admin/bin/launch_save_map.sh
```

## Create Save MAP Launch Script Shortcut
```bash
nano "$(xdg-user-dir DESKTOP)/SaveMAP.desktop"
```

```text
[Desktop Entry]
Version=1.0
Type=Application
Name=Save MAP
Comment=Save ROS2 MAP - SLAM Gmapping in Docker
Exec=/home/renity-admin/bin/launch_save_map.sh
Icon=utilities-terminal
Terminal=true
Categories=Development;
```

## Grant Save MAP Launch Script Shortcut Execution Permissions
```bash
chmod +x "$(xdg-user-dir DESKTOP)/SaveMAP.desktop"
```

# Start Navigation
## Create Start Navigation Launch Script
```bash
mkdir -p /home/renity-admin/bin

nano /home/renity-admin/bin/launch_start_navigation.sh
```

```text
#!/bin/bash

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

cd /home/renity-admin/ros2_humble_docker || exit 1

exec docker compose exec ros2_humble_rviz2 \
    bash -lc "source /opt/ros/humble/setup.bash && \
              source /home/ros-admin/ros2_humble_workspace/install/setup.bash && \
			  source /home/ros-admin/ros2_humble_amr_avalue/install/setup.bash && \
              exec ros2 launch avalue_robot_nav2 avalue_robot_nav2.launch.py"
```

## Grant Start Navigation Launch Script Execution Permissions
```
chmod +x /home/renity-admin/bin/launch_start_navigation.sh
```

## Create Start Navigation Launch Script Shortcut
```bash
nano "$(xdg-user-dir DESKTOP)/StartNavigation.desktop"
```

```text
[Desktop Entry]
Version=1.0
Type=Application
Name=Start MAP Navigation
Comment=Start ROS2 MAP Navigation - SLAM Gmapping in Docker
Exec=/home/renity-admin/bin/launch_start_navigation.sh
Icon=utilities-terminal
Terminal=true
Categories=Development;
```

## Grant Start Navigation Launch Script Shortcut Execution Permissions
```bash
chmod +x "$(xdg-user-dir DESKTOP)/StartNavigation.desktop"
```

# Open Rviz2 MAP
## Create Open Rviz2 MAP Launch Script
```bash
mkdir -p /home/renity-admin/bin

nano /home/renity-admin/bin/launch_open_rviz2_map.sh
```

```text
#!/bin/bash

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

#
# Use the display of the desktop which launched this script.
# Local Ubuntu desktop: usually :0
# TightVNC XFCE:        usually :10.0
#
HOST_DISPLAY="${DISPLAY:-:0}"

#
# RViz2 config inside container
#
CONFIG_FILE="/home/ros-admin/ros2_humble_amr_avalue/avalue_ros2_humble.rviz"

cd /home/renity-admin/ros2_humble_docker || exit 1

exec docker compose exec \
    -e DISPLAY="$HOST_DISPLAY" \
    -e XAUTHORITY=/home/ros-admin/.Xauthority \
    -e QT_QPA_PLATFORM=xcb \
    ros2_humble_rviz2 \
    bash -lc "source /opt/ros/humble/setup.bash && \
              source /home/ros-admin/ros2_humble_workspace/install/setup.bash && \
              source /home/ros-admin/ros2_humble_amr_avalue/install/setup.bash && \
              exec rviz2 -d \"$CONFIG_FILE\""
```

## Grant Open Rviz2 MAP Launch Script Execution Permissions
```
chmod +x /home/renity-admin/bin/launch_open_rviz2_map.sh
```

## Create Open Rviz2 MAP Launch Script Shortcut
```bash
nano "$(xdg-user-dir DESKTOP)/OpenRviz2MAP.desktop"
```

```text
[Desktop Entry]
Version=1.0
Type=Application
Name=Open Rviz2 MAP
Comment=Open Rviz2 MAP in Docker
Exec=/home/renity-admin/bin/launch_open_rviz2_map.sh
Icon=utilities-terminal
Terminal=true
Categories=Development;
```

## Grant Open Rviz2 MAP Launch Script Shortcut Execution Permissions
```bash
chmod +x "$(xdg-user-dir DESKTOP)/OpenRviz2MAP.desktop"
```

# Follow Waypoints Click
## Create Follow Waypoints Click Launch Script
```bash
mkdir -p /home/renity-admin/bin

nano /home/renity-admin/bin/launch_follow_waypoints_click.sh
```

```text
#!/bin/bash

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

cd /home/renity-admin/ros2_humble_docker || exit 1

exec docker compose exec ros2_humble_rviz2 \
    bash -lc "source /opt/ros/humble/setup.bash && \
              source /home/ros-admin/ros2_humble_workspace/install/setup.bash && \
			  source /home/ros-admin/ros2_humble_amr_avalue/install/setup.bash && \
              python3 /home/ros-admin/ros2_humble_amr_avalue/reference/follow_waypoints_click.py && \
			  exec bash"
```

## Grant Follow Waypoints Click Launch Script Execution Permissions
```
chmod +x /home/renity-admin/bin/launch_follow_waypoints_click.sh
```

## Create Follow Waypoints Click Launch Script Shortcut
```bash
nano "$(xdg-user-dir DESKTOP)/FollowWaypointsClick.desktop"
```

```text
[Desktop Entry]
Version=1.0
Type=Application
Name=Follow Waypoints Click
Comment=Execute Follow Waypoints Click in Docker
Exec=/home/renity-admin/bin/launch_follow_waypoints_click.sh
Icon=utilities-terminal
Terminal=true
Categories=Development;
```

## Grant Follow Waypoints Click Launch Script Shortcut Execution Permissions
```bash
chmod +x "$(xdg-user-dir DESKTOP)/FollowWaypointsClick.desktop"
```
