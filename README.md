# smac_hybrid_planner_navigation
This is a **ROS 2 Humble + Gazebo Classic** navigation stack: a diff-drive "cardbot" navigating a tight indoor environment (1.2m corridors, 0.7m doorways, pillar zones) using **Nav2 AMCL** localization + **Smac Hybrid-A*** (Reeds-Shepp) global planning + Regulated Pure Pursuit local control.

## Setup + Build

```bash
cd diffbot_nav_ws
chmod +x build.sh run.sh
./build.sh
source install/setup.bash
```
(`build.sh` installs the needed apt packages for Nav2, SLAM Toolbox, Gazebo, etc., then runs `colcon build`.)

## Run — Option A: use the pre-built map, go straight to navigation (recommended)

```bash
./run.sh
```
This starts Gazebo + Nav2 (AMCL + Smac Hybrid-A*) + RViz + an AMCL quality monitor, all in tmux.

Then in RViz:
1. Click **"2D Pose Estimate"** → click on the map at `(-0.5, 0.0)` pointing right
2. Wait ~3 seconds of driving for AMCL particles to converge
3. Click **"Nav2 Goal"** in RViz, or run the mission script below

## Run — Option B: build your own map first

```bash
# Terminal 1 — SLAM + Gazebo
ros2 launch diffbot_navigation slam.launch.py

# Terminal 2 — drive manually to map the space
ros2 run teleop_twist_keyboard teleop_twist_keyboard

# Terminal 3 — save the map when done
ros2 run diffbot_bringup map_saver.py --ros-args -p output_path:=src/diffbot_navigation/maps/constrained_indoor

# Rebuild so the new map is installed
colcon build --symlink-install
```

## Run the autonomous mission (after Nav2 is running and AMCL is localized)

```bash
ros2 run diffbot_bringup waypoint_mission.py                              # 14-waypoint circuit, once
ros2 run diffbot_bringup waypoint_mission.py --ros-args -p loop:=true    # loop continuously

# In parallel, watch localization quality
ros2 run diffbot_bringup amcl_monitor.py
```

## Manual per-component launches (alternative to run.sh)

```bash
ros2 launch diffbot_sim gazebo.launch.py
ros2 launch diffbot_navigation bringup.launch.py     # Gazebo + Nav2 together
ros2 launch diffbot_navigation navigation.launch.py  # AMCL + Nav2 only
ros2 launch diffbot_navigation slam.launch.py        # mapping phase
ros2 launch diffbot_description display.launch.py    # URDF viewer, no sim
```

## Troubleshooting quick reference

```bash
# Republish initial pose manually if AMCL diverges
ros2 topic pub /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
  "{ header: {frame_id: map}, pose: { pose: { position: {x: -0.5, y: 0.0} } } }" --once
```
- Robot stuck in doorways → check `inflation_radius < 0.35` and `robot_radius = 0.17` in `nav2_params.yaml`
- Smac planner timing out → raise `max_planning_time` or lower `max_iterations` in the config
- Gazebo slow on a VM → `LIBGL_ALWAYS_SOFTWARE=1` is already set in the launch files

Note: needs a real Ubuntu 22.04 + ROS 2 Humble + Gazebo machine — it won't run in this sandbox.
