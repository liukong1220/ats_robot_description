# ats_robot_description

ATS 2026 哨兵及相关机型的机器人描述包。描述文件使用 [xmacro](https://github.com/gezp/xmacro)
编写的 SDF（`*.sdf.xmacro`），本包不含手写 URDF/xacro；URDF 由 launch 在运行时经
`sdformat_tools` 从生成的 SDF 转换得到。上游北极熊战队的原始说明保留在
[de_README.md](./de_README.md)，其中部分默认值（如 `robot_name`）已与当前 launch 不一致，以本文件和源码为准。

## 内容

| 路径 | 说明 |
| :--- | :--- |
| `resource/xmacro/ats_sentry_robot.sdf.xmacro` | ATS 哨兵：四舵轮底盘 + Mid360 + 工业相机，Gazebo 导航入口默认使用 |
| `resource/xmacro/simulation_robot.sdf.xmacro`、`simulation_nav_robot.sdf.xmacro` | 基于 `rm25_example_robot` 的仿真机型（模型名同为 `ats_sentry_robot`） |
| `resource/xmacro/ats_infantry_robot.sdf.xmacro` | 步兵机型，仅挂工业相机 |
| `resource/models/ats_swerve_chassis` | 四驱四转舵轮底盘 xmacro block |
| `resource/models/mid360`、`industrial_camera`、`rplidar_a2` | 传感器模型 block |
| `launch/robot_description_launch.py` | xmacro -> SDF -> URDF，启动 joint_state_publisher、robot_state_publisher、可选 RViz |
| `params/robot_description.yaml` | 两个 publisher 的参数 |
| `rviz/visualize_robot.rviz` | 可视化配置，Fixed Frame 为 `chassis` |
| `env-hooks/gazebo.dsv.in` | 把 `share/ats_robot_description/resource/models` 加入 `IGN_GAZEBO_RESOURCE_PATH`、`SDF_PATH`、`IGN_FILE_PATH` |

## 坐标系（ats_sentry_robot）

```text
base_footprint -> chassis -> gimbal_yaw_odom -> gimbal_pitch_odom -> gimbal_yaw -> gimbal_pitch
                     |               |                                                 |
                     |               +-> front_mid360（fixed）                          +-> front_industrial_camera
                     |                                                                 |     -> front_industrial_camera_optical_frame
                     +-> {front,rear}_{left,right}_steer_link -> *_wheel               +-> speed_monitor
```

| 关节 | 类型 | 说明 |
| :--- | :--- | :--- |
| `base_to_chassis` | fixed | `chassis` 在 `base_footprint` 上方 `chassis_height = 0.10 m` |
| `gimbal_yaw_odom_joint` | revolute，z 轴 | 大 yaw，导航/定位参考轴，位于 `chassis` 上方 `0.10 m` |
| `gimbal_pitch_odom_joint` | fixed | 小 yaw 相对大 yaw 偏置 `(0.061, 0, 0.030)` |
| `gimbal_yaw_joint` | revolute，z 轴 | 小 yaw |
| `gimbal_pitch_joint` | revolute，y 轴 | 相对 `gimbal_yaw` 偏置 `(-0.053, 0, 0.045)` |
| `*_steer_joint` / `*_wheel_joint` | revolute，z / y 轴 | 4 个舵轮模块，命名与 MuJoCo `swerve_chassis.xml` 一致 |

坐标约定：x 前、y 左、z 上，欧拉角顺序 roll pitch yaw（rad）。

## 底盘与足迹

- `chassis` 的 visual/collision 为 `0.58 x 0.58 x 0.10 m` 的盒体，质量 23.0 kg。
- 舵轮模块安装在 `(±0.270, ±0.270)`，轮半径 `0.0425 m`，轮心高度等于轮半径。
- `base_footprint` 是贴地的空 link；本包不定义导航足迹多边形，导航足迹由导航参数决定。
- `ats_sentry_robot.sdf.xmacro` 中的 `chassis_length`(0.60)、`chassis_width`(0.50) 未传入底盘 block，不影响几何。
- Gazebo 插件 `AtsSwerveDrive4WS` 接收车体系 `[vx, vy, wz]`，`command_timeout` 0.5 s，超时按零速处理。

## 传感器挂载（ats_sentry_robot）

| 传感器 | 父 frame | pose（x y z roll pitch yaw） | 参数 |
| :--- | :--- | :--- | :--- |
| Mid360（`front_mid360`，gpu_lidar + imu） | `gimbal_yaw_odom` | `-0.2 0 0 0 0 -61°` | 32 线，水平采样 `nav_livox_horizontal_samples`=625，`nav_livox_update_rate_hz`=10.0，量程 0.1-40 m，IMU 200 Hz |
| 工业相机（`front_industrial_camera`） | `gimbal_pitch` | `0.1 0 0.045 0 0 0` | 1920x1080，30 Hz，`horizontal_fov`=1 |
| 底盘 IMU（`chassis_imu`） | `chassis` | - | 200 Hz |

`simulation_robot` / `simulation_nav_robot` 中 Mid360 挂载为 `-0.1 0.245 0.325 75° 0 -161°`、20 Hz、1875 采样，
与哨兵机型不同。Gazebo 导航入口的 `gimbal_yaw_odom -> front_mid360` 静态 TF 需与实际使用的 xmacro 保持一致。

## 构建与运行

```bash
cd /home/ats/ATS_2026_snetry_test
colcon build --base-paths src --packages-select ats_robot_description
source install/setup.bash
ros2 launch ats_robot_description robot_description_launch.py
```

Launch 参数：`namespace`（默认空）、`use_sim_time`（`False`）、`robot_name`（`ats_sentry_robot`）、
`robot_xmacro_file`（默认 `resource/xmacro/<robot_name>.sdf.xmacro`，优先于 `robot_name`）、
`params_file`、`rviz_config_file`、`use_rviz`（`True`）、`use_respawn`（`False`）、`log_level`（`info`）。

`params/robot_description.yaml` 中 joint_state_publisher 以 200 Hz 汇总 `serial/gimbal_joint_state`
（`source_list`），robot_state_publisher `publish_frequency` 为 200 Hz。

依赖：`xmacro`（pip）、`sdformat_tools`、`rmoss_gz_resources`（提供 `rm21_*` armor/灯条/测速模块与
`rm25_example_robot`）、`nav2_common`；launch 还会启动 `joint_state_publisher`、`robot_state_publisher`、`rviz2`。
`dependencies.repos` 列出 `joint_state_publisher`（`feat-offset-timestamp` 分支）、
`rmoss_gz_resources`、`sdformat_tools`。
