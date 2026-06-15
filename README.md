# pairs_uav_trajectory_generation

Turns a sparse list of waypoints (a path) into a smooth, dynamically feasible time-parametrized trajectory for the UAV to track. It plans minimum-snap polynomial segments through the waypoints while respecting the UAV's speed and acceleration limits, so the rest of the PAIRS control stack can follow the result. This is the planning layer that feeds reference trajectories to the control manager.

## Contents

- `PairsTrajectoryGeneration` nodelet — accepts a path (topic or service) and returns a sampled, time-optimal trajectory; reconfigurable at runtime via dynamic_reconfigure.
- `PathRandomFlier` nodelet — generates random paths and feeds them to the generator, useful for testing and stress-testing the planning/control loop.
- Bundled ETH polynomial trajectory-optimization library (`eth_trajectory_generation`), used internally for the minimum-snap optimization.

## Branches

- `ros1` — ROS 1 Noetic (catkin)
- `ros2` — ROS 2 Jazzy (ament_cmake)

## Install (ROS 1 Noetic)

```bash
sudo apt install ros-noetic-pairs-uav-trajectory-generation
```

## Usage

```bash
roslaunch pairs_uav_trajectory_generation trajectory_generation.launch
roslaunch pairs_uav_trajectory_generation path_random_flier.launch
```

## License
BSD 3-Clause. Derived from the CTU-MRS `pairs_uav_trajectory_generation` package; the original
copyright is retained in [LICENSE](LICENSE).
