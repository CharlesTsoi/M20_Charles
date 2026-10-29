# M20 机器狗外参标定

前后两台 RoboSense Airy 激光雷达的外参已经用实车 rosbag 核对过。结论是：`config.yaml` 里的外参数值可以继续用，这次没有改数字。

标定日期：2026-10-28  
数据包：`rosbag2_2026_10_28-05_54_03`（时长 152.9 s）  
外参文件：`rslidar_ws/src/rslidar_sdk/config/config.yaml`

## 1. 结论

两台雷达的点云在录包时已经被驱动外参变到同一车体坐标系。用车体 IMU 分别做 FAST-LIO 建图后，前后地图重合，轨迹也一致。静止时地面高度一致到 1 cm，相对倾角约 0.9°。这个倾角换到后续扫描上并不稳定，所以没有把 CAD 外参改掉。

| 雷达 | 组播地址 | 话题 | x, y, z (m) | roll, pitch, yaw (rad) |
|---|---|---|---|---|
| 前 Airy | 224.10.10.201 | `/rslidar_points_front` | 0.32028, 0, -0.013 | 0, -1.570795, 3.1415926 |
| 后 Airy | 224.10.10.202 | `/rslidar_points_rear` | -0.32028, 0, -0.013 | 0, -1.570795, 0 |

旋转顺序与 RoboSense 驱动一致：先 yaw，再 pitch，最后 roll。平移单位是米，角度单位是弧度。前雷达 yaw 为 π，后雷达 yaw 为 0，相当于两台半球雷达分别朝前、朝后，再变到同一车体坐标系。

## 2. 录包内容

驱动按 `config.yaml` 编译时打开了 `ENABLE_TRANSFORM`，录进包里的点云已经乘过上表外参。两台雷达都是 RSAIRY，点云字段为 `x y z intensity ring timestamp`，约 10 Hz，96 线。

| 话题 | 类型 | 条数 | 说明 |
|---|---|---|---|
| `/rslidar_points_front` | PointCloud2 | 1528 | 前雷达，已做外参变换 |
| `/rslidar_points_rear` | PointCloud2 | 1528 | 后雷达，已做外参变换 |
| `/IMU` | Imu | 30585 | 车体 IMU，200 Hz，加速度单位 m/s²，静止时重力在 +Z |
| `/LIDAR/IMU201` | Imu | 30558 | 前雷达内置 IMU，单位约 g，重力在 -Y |
| `/LIDAR/IMU202` | Imu | 30558 | 后雷达内置 IMU，单位约 g，重力在 -Y |

三路 IMU 的角速度在时间上对齐，滞后为 0。点云头时间与 `/IMU` 使用同一套时钟。建图用 `/IMU`，不用雷达内置 IMU，因为点云已经不在雷达 IMU 坐标系里。

## 3. 核对方法

单帧里两台半球雷达几乎没有公共视场，前雷达主要看见 +X，后雷达主要看见 -X，所以不能靠一帧点云直接配准。核对分成两步。

1. 静止段拟合地面。比较两台雷达地面法向和高度，检查 roll、pitch 和 z。
2. 用同一套车体 IMU，分别只喂前雷达、只喂后雷达，跑 FAST-LIO。车体在这段数据里走了约 19 m，并转了约 168°。两台雷达因此扫过同一片环境。比较两条轨迹和两张地图，检查 yaw 以及水平平移。

FAST-LIO 使用 hku-mars 仓库的 `ROS2` 分支，工作空间在 `fast_lio_ws`。点云外参在 FAST-LIO 里设为单位阵，因为驱动已经把点云变到与车体 IMU 对齐的车体坐标系。

## 4. 结果

静止地面：

| 雷达 | 地面高度 z (m) | 相对水平面倾角 |
|---|---|---|
| 前 | -0.555 | 1.4° |
| 后 | -0.554 | 0.6° |

FAST-LIO，各保留 1525 / 1528 帧：

| 项目 | 前雷达 | 后雷达 |
|---|---|---|
| 轨迹长度 | 19.02 m | 18.96 m |
| 航向变化 | 约 168° | 约 168° |
| 单帧耗时 | 约 8 ms | 约 8 ms |

前后对比：

- 位姿差：平均 7.6 cm，90% 的帧小于 10 cm，最大 17.6 cm。
- 地图在 0.3 m 体素下的重合率：配准前 **98.6%**。
- 把后雷达地图配到前雷达地图：yaw **-0.11°**，平移 **0.7 cm、2.3 cm、-0.4 cm**，配准均方距离 2.7 cm。

地面相对倾角大约 0.9°。只把后雷达 pitch 加 0.74° 再看后面的扫描，有的帧变好，有的帧变差，所以没有写入 `config.yaml`。

## 5. 外参是否修改

数值保持第 1 节的表。`config.yaml` 里只加了核对说明，没有改 x、y、z、roll、pitch、yaw。

残留差来自两台雷达看见的区域不同，以及约 19 m 轨迹上的里程计漂移，小于这次数据能稳定分开的外参误差。

## 6. 复现

```bash
source /opt/ros/foxy/setup.bash
source ~/fast_lio_ws/install/setup.bash

# 前雷达
ros2 run fast_lio fastlio_mapping --ros-args \
  --params-file ~/fast_lio_ws/src/FAST_LIO/config/m20_front.yaml
ros2 bag play ~/M20_bag/rosbag2_2026_10_28-05_54_03 \
  --topics /rslidar_points_front /IMU

# 后雷达把配置换成 m20_rear.yaml，话题换成 /rslidar_points_rear
```

配置要点：`lidar_type: 5`（Airy，使用点云里的绝对时间戳），`imu_topic: /IMU`，`extrinsic_T` 为 0，`extrinsic_R` 为单位阵。
