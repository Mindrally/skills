---
name: ros2-robotics
description: "Best practices for ROS 2 robotics development, covering package structure, nodes, topics/services/actions, launch files, QoS, tf2 transforms, and testing. Use when creating ROS 2 packages, writing nodes in rclpy or rclcpp, defining custom messages/services/actions, writing launch files, configuring QoS profiles, working with tf2 transforms, or building and testing a colcon workspace."
---

# ROS 2 Robotics Development

This skill covers building ROS 2 packages and nodes, including package structure, communication patterns (topics, services, actions), timing and frames, build/test workflows with colcon, and common pitfalls in robotics software.

## Workflow for Building a ROS 2 Package

1. **Scaffold the package** — Run `ros2 pkg create --build-type ament_python <name>` (Python) or `--build-type ament_cmake <name>` (C++) inside a colcon workspace's `src/` directory.
2. **Define interfaces** — Add custom `.msg`, `.srv`, or `.action` files under `msg/`, `srv/`, `action/` only when a standard interface (`std_msgs`, `geometry_msgs`, `sensor_msgs`, `nav_msgs`) doesn't fit.
3. **Implement nodes** — Write focused, composable nodes; declare parameters explicitly; choose topics for streams, services for quick request/response, and actions for long-running goals with feedback and cancellation.
4. **Write launch files** — Compose nodes, parameters, and remappings in a `launch/*.launch.py` file using `launch_ros.actions.Node`.
5. **Set QoS profiles intentionally** — Match publisher/subscriber QoS (reliability, durability, history depth) for sensor data vs. command/control paths.
6. **Build the workspace** — Run `colcon build --symlink-install` from the workspace root, then `source install/setup.bash`.
7. **Test** — Add unit tests (`pytest` for Python, `gtest`/`launch_testing` for C++/integration) and run `colcon test`.
8. **Run and inspect** — Use `ros2 run`, `ros2 launch`, `ros2 topic echo`, `ros2 node info`, and `ros2 doctor` to verify runtime behavior.

## Package Structure

- Keep each package focused on one robot capability or integration boundary (e.g., `lidar_driver`, `path_planner`, `arm_controller`) rather than a monolithic package.
- Use `package.xml` consistently with the build type declared in it, and either `CMakeLists.txt` (ament_cmake, C++) or `setup.py`/`setup.cfg` (ament_python).
- Organize files with `launch/` for launch scripts, `config/` for YAML parameter files, `msg/` for message definitions, `srv/` for services, `action/` for actions, and `urdf/`/`xacro/` for robot descriptions.
- Use namespaces and topic remapping instead of hardcoded topic names whenever a node might run multiple times (e.g., multi-robot setups) or be reused across projects.
- Declare all dependencies explicitly in `package.xml` (`<depend>`, `<exec_depend>`, `<build_depend>`) so the workspace builds reproducibly.

## Nodes and Interfaces

- Keep nodes small and composable — one node per logical responsibility, wired together via topics/services/actions rather than one giant node doing everything.
- Declare ROS 2 parameters explicitly with `self.declare_parameter('rate_hz', 10.0)` and read them with `self.get_parameter('rate_hz').value`; never rely on undeclared parameters.
- Prefer messages (topics) for continuous state streams, services for quick, blocking request/response operations, and actions for long-running goals that report feedback and support cancellation/preemption.
- Use standard message types (`geometry_msgs/Twist`, `sensor_msgs/LaserScan`, `nav_msgs/Odometry`) before creating custom interfaces — custom types fragment the ecosystem and lose compatibility with existing tools.
- Document topic, service, action, frame, and parameter contracts (name, type, units, frame_id, update rate) in the package README or node docstring.

### Example: A Minimal rclpy Publisher Node

```python
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
from geometry_msgs.msg import Twist


class SafeVelocityPublisher(Node):
    """Publishes bounded velocity commands at a fixed rate."""

    def __init__(self) -> None:
        super().__init__("safe_velocity_publisher")

        self.declare_parameter("rate_hz", 10.0)
        self.declare_parameter("max_linear_speed", 0.5)
        rate_hz = self.get_parameter("rate_hz").value
        self._max_linear = self.get_parameter("max_linear_speed").value

        # Reliable QoS for a low-rate command topic; sensor streams would
        # typically use BEST_EFFORT with a shallow history instead.
        qos = QoSProfile(
            reliability=ReliabilityPolicy.RELIABLE,
            history=HistoryPolicy.KEEP_LAST,
            depth=10,
        )
        self._publisher = self.create_publisher(Twist, "cmd_vel", qos)
        self._timer = self.create_timer(1.0 / rate_hz, self._on_timer)
        self._target_linear = 0.0

    def _on_timer(self) -> None:
        msg = Twist()
        msg.linear.x = max(-self._max_linear, min(self._max_linear, self._target_linear))
        self._publisher.publish(msg)


def main(args: list | None = None) -> None:
    rclpy.init(args=args)
    node = SafeVelocityPublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
```

## Timing and Frames

- Use ROS time (`self.get_clock().now()`, and `use_sim_time` when a simulation clock is in play) instead of wall-clock time whenever simulation or bag replay matters.
- Use `tf2` (`tf2_ros.Buffer` + `TransformListener`) for all frame transforms; never hand-roll coordinate transforms between named frames.
- Document frame names (`base_link`, `odom`, `map`, sensor frames) and their conventions (REP-103 for axis orientation, REP-105 for the `map`→`odom`→`base_link` tree).
- Avoid blocking callbacks in a node's executor thread — move long-running work to timers, a `MultiThreadedExecutor`, worker threads, or actions so the node keeps servicing other callbacks.
- Set QoS profiles intentionally: `BEST_EFFORT`/volatile for high-rate sensor data where dropped samples are acceptable, `RELIABLE`/`TRANSIENT_LOCAL` for latched-like configuration topics, and `RELIABLE`/volatile for command paths.

## Build and Test

- Use `colcon build` (typically `--symlink-install` during development) and keep inter-package dependencies explicit in `package.xml`.
- Run the linters and formatters the workspace already uses — `ament_flake8`, `ament_pep257` for Python; `ament_cpplint`, `ament_uncrustify` for C++ — via `colcon test`.
- Add `launch_testing`-based integration tests for multi-node behavior (e.g., "does node A receive what node B publishes within N seconds").
- Use simulation (Gazebo/Ignition), recorded rosbags, or fixture data for repeatable sensor scenarios instead of relying on live hardware for every test run.
- Explicitly test failure cases: missing/late transforms (`tf2.LookupException`), stale sensor data, and services or actions that are unavailable when called.

## Common Mistakes

- Hardcoding absolute filesystem paths instead of resolving them via `ament_index_python.packages.get_package_share_directory`.
- Publishing velocity or actuator commands without validating frame_id, units, and timestamp freshness — a stale or wrong-frame command can move a robot unsafely.
- Creating a custom message/service/action type when an existing standard interface already models the data.
- Ignoring QoS incompatibility warnings between a publisher and subscriber (e.g., a `BEST_EFFORT` publisher with a `RELIABLE` subscriber) — the connection silently never delivers data.
- Doing significant work inside a subscription callback without offloading it, starving the executor and delaying other callbacks (including safety-critical ones).
- Forgetting `use_sim_time:=true` when running against a simulator, causing timestamps and `tf2` lookups to disagree with the simulated clock.
