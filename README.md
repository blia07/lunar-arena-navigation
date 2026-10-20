# Lunar Arena Navigation (ROS 2 + Nav2)

A simulated differential-drive robot with a 2D LiDAR that maps a lunar-style arena with SLAM and navigates autonomously with Nav2, with hand-gesture commands for teleoperation and for choosing navigation goals.

I started this project to get hands-on experience with ROS 2, LiDAR, SLAM, and simulation ahead of my team's navigation work for a project in a competition, where I'm the Software Lead. It also builds on my earlier gesture-recognition project (MediaPipe + OpenCV).

## Status

- [x] **Phase 1:** Nav2 baseline running in Gazebo (TurtleBot3 Waffle, AMCL localization, autonomous goals)
- [ ] **Phase 2:** Gesture teleoperation via `/cmd_vel`
- [ ] **Phase 3:** Mapping with `slam_toolbox`
- [ ] **Phase 4:** Custom lunar-style arena world
- [ ] **Phase 5:** Nav2 navigation on the custom map
- [ ] **Phase 6:** Gesture → navigation goals node (C++)
- [ ] **Phase 7:** Evaluation (goal success rate, time to goal)

## Environment

| Component | Version |
|---|---|
| OS | Ubuntu 26.04 on Windows (WSL2 / WSLg, dedicated distro) |
| ROS 2 | Lyrical Luth |
| Simulator | Gazebo (`gz sim`) |
| Navigation | Nav2 1.5.2 |
| SLAM | slam_toolbox 2.10.0 |
| Robot | TurtleBot3 Waffle (sim) |

## Setup

```bash
sudo apt install ros-lyrical-navigation2 ros-lyrical-nav2-bringup \
  ros-lyrical-slam-toolbox ros-lyrical-nav2-minimal-tb3-sim

# Required on WSL so RViz can render (see Issue 1)
echo "export QT_QPA_PLATFORM=xcb" >> ~/.bashrc
source ~/.bashrc
```

## Running

```bash
ros2 launch nav2_bringup tb3_simulation_launch.py headless:=False \
  params_file:=$HOME/lunar_nav_ws/config/nav2_params.yaml
```

Then, in RViz, click **Nav2 Goal** and click-drag on an open area of the map. The tool switches back to Move Camera after each goal, so click it again for every new goal.

## How it fits together

TF tree (from `ros2 run tf2_tools view_frames`):

```
map → odom → base_footprint → base_link → base_scan (LiDAR)
```

- **odom → base_footprint:** wheel odometry from Gazebo's diff-drive plugin. Smooth, but drifts over time.
- **map → odom:** AMCL's correction for that drift, from matching LiDAR scans against the map.
- **base_link → base_scan, imu_link, camera_link, ...:** fixed sensor mounting positions (static transforms).

## Issues and fixes

### 1. RViz crashed and shut down the whole launch

**Issue:** The Gazebo window opened blank, then every node shut down. Even `headless:=True` failed.

**How I found it:** Ran `gz sim shapes.sdf` on its own, which worked, so Gazebo rendering was fine. Then saved the launch output with `tee` and searched it with `grep`, which surfaced:

```
rviz::RenderSystem: error creating render window: RenderingAPIException:
Invalid parentWindowHandle (wrong server or screen)
```

**Cause:** Under WSLg, Qt defaults to Wayland, but RViz's 3D renderer (OGRE) needs X11. The Nav2 bringup launch shuts down every node when RViz exits, so the RViz crash took Gazebo and Nav2 down with it. This made it look like a Gazebo problem.

**Fix:** Force Qt to use X11:

```bash
export QT_QPA_PLATFORM=xcb
```

### 2. Nav2 stuck at "inactive" and goals rejected

**Issue:** The RViz Navigation panel stayed **inactive**, and goals failed with `Action server is inactive. Rejecting the goal.` `ros2 lifecycle get` showed `planner_server` and `bt_navigator` stuck at `inactive`, while `controller_server` was `active`.

**Cause:** Nav2 nodes are lifecycle nodes (unconfigured → inactive → active), brought up in order by a lifecycle manager. The planner's global costmap can't activate until the `map → odom` transform exists, and AMCL only publishes it after it receives an initial pose. On WSL, the simulation was slow enough that the startup timed out waiting, and the lifecycle manager aborted.

Clicking **Startup** in the Nav2 panel afterwards made it worse. It tried to configure nodes that were already active (`Unable to start transition 1 (configure) from current state active`) and aborted again.

**Workaround:** Activate the stuck nodes by hand, with `bt_navigator` last:

```bash
ros2 lifecycle set /planner_server activate
ros2 lifecycle set /behavior_server activate
ros2 lifecycle set /bt_navigator activate
```

**Fix:** Give AMCL the robot's starting pose in its config, so `map → odom` exists as soon as AMCL activates. In `config/nav2_params.yaml`, under `amcl: ros__parameters:`:

```yaml
    set_initial_pose: true
    initial_pose:
      x: -2.0
      y: -0.5
      z: 0.0
      yaw: 0.0
```

With this, Nav2 reaches **active** on its own, with no manual pose or activation needed.

### 3. Wrong initial pose corrupted the costmap

**Issue:** After setting **2D Pose Estimate** several times in the wrong spots, the costmap filled with smeared, rotated copies of the pillars, and Nav2 couldn't plan paths.

**Cause:** AMCL believed the robot was in the wrong place, so LiDAR obstacles were added to the costmap in the wrong positions. The red LiDAR dots in RViz sat offset from the map's walls, which was the visible sign.

**Fix:** Restarted to clear the costmap, then set the pose once at the true spawn point, (-2.0, -0.5) facing +x, using RViz's **Publish Point** tool to read exact coordinates from the status bar. Checked that the LiDAR dots lined up with the walls before sending goals. Issue 2's config fix now makes this step unnecessary.

**Lesson:** Localization is the foundation and most important thing in the setup. A bad pose estimate breaks everything downstream (costmaps, planning, control).

## Known limitations

- A 2D LiDAR scans a single horizontal slice, so it **can't detect negative obstacles such as craters**. A depth camera or 3D LiDAR would be needed for that. The TurtleBot3 Waffle model already includes a depth camera frame (`camera_depth_frame`), which could be a future use on this project.