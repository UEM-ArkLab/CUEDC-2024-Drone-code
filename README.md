# CUEDC-2024 自主无人机定位与航点导航系统

2024 年全国大学生电子设计竞赛（CUEDC）无人机赛题方案。基于 **Fast-LIO 激光 SLAM** 实现室内自主定位，融合 **PX4 飞控**、**STM32 下位机路径规划** 与 **OpenMV 视觉识别**，完成自主航点飞行与目标检测任务。

> 已验证完成 2024 年电赛题目，得分 **100/120**。

## 演示视频

[![演示视频](https://img.shields.io/badge/Bilibili-演示视频-blue)](https://b23.tv/Q0KBtWE)

---

## 硬件清单

| 部件 | 型号/规格 |
|------|-----------|
| 飞控 | CUAV-V5+ （PX4 固件 v1.13） |
| 机载计算机 | 运行 ROS Noetic 的 Linux 板卡（树莓派 / Jetson 等） |
| 激光雷达 | Livox MID-360 |
| 电机 | 2207 6 寸 |
| 电池 | 4S 9000mAh |
| 机架 | 330mm 轴距全碳板 |
| 视觉模块 | OpenMV4 Plus（H7） |
| 下位机 | STM32F103（矩阵按键 + 航线规划 + 串口屏） |
| 中继 | STM32F405（OpenMV 数据桥接） |

![硬件](https://github.com/user-attachments/assets/b4e4fbb1-d6cc-46fa-8001-700f310332dc)

---

## 软件依赖

### 上位机（机载计算机）

- **Ubuntu 20.04** + **ROS Noetic**
- [Fast-LIO](https://github.com/hku-mars/FAST_LIO) — 激光惯性里程计
- [livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2) — Livox LiDAR ROS 驱动
- [MAVROS](https://github.com/mavlink/mavros) — PX4 通信桥接
- `serial` 库 — 串口通信（`sudo apt install ros-noetic-serial`）

### 下位机（STM32）

- Keil MDK (ARM Compiler 5/6)
- STM32CubeMX (HAL 库 + FreeRTOS CMSIS-RTOS v1)

### 视觉模块

- OpenMV IDE
- Edge Impulse FOMO 模型（`trained.tflite`）

---

## 目录结构

```
CUEDC-2024-Drone-code/
├── README.md
├── run_all.sh                 # 一键启动所有 ROS 节点
├── src/                       # ROS 节点源码
│   ├── lio-to-mavros_node.cpp # Fast-LIO 里程计 → PX4 坐标系转换
│   ├── offboard_node.cpp      # PX4 Offboard 模式航点控制
│   └── uart_node.cpp          # 串口通信（下位机 ↔ ROS）
├── down/                      # STM32F103 下位机固件（Keil 工程）
│   ├── Core/                  # HAL 初始化 + FreeRTOS 配置
│   ├── HARDWARE/              # 矩阵按键、LCD 屏、串口驱动
│   └── TASK/                  # 按键/航线规划/显示 任务
├── uper/                      # STM32F405 中继固件（Keil 工程）
├── openmv/                    # OpenMV H7 Plus 视觉识别代码
│   ├── main.py                # 目标检测主程序
│   ├── trained.tflite         # Edge Impulse FOMO 模型权重
│   └── labels.txt             # 类别标签
└── 3D maker/                  # 3D 打印件（电池座、光流支架）
```

---

## 快速开始

### 1. 创建工作空间

```bash
mkdir -p ~/drone_ws/src
cd ~/drone_ws/src
# 将本仓库所有内容放入 src 目录
git clone <your-repo-url> .
```

### 2. 安装依赖与编译

```bash
# 安装 Fast-LIO 与 MAVROS（手动克隆到 src 目录）
cd ~/drone_ws/src
git clone https://github.com/hku-mars/FAST_LIO.git
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
sudo apt install ros-noetic-mavros ros-noetic-mavros-extras ros-noetic-serial
# 安装 GeographicLib 数据集（MAVROS 依赖）
wget https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh
sudo bash install_geographiclib_datasets.sh

# 编译
cd ~/drone_ws
catkin_make
source devel/setup.bash
```

### 3. 硬件连接

| 连接 | 说明 |
|------|------|
| 机载计算机 ↔ PX4 | USB / TELEM2 串口（MAVROS） |
| 机载计算机 ↔ STM32F103 | USB 转 TTL（`/dev/ttyUSB0`，115200 bps） |
| STM32F103 ↔ STM32F405 | UART 直连 |
| STM32F405 ↔ OpenMV | UART3（115200 bps） |
| STM32F103 ↔ LCD 串口屏 | UART2 |
| STM32F103 ↔ 4×4 矩阵按键 | GPIO（PB12-15 列，PD8-11 行） |

### 4. 一键启动

```bash
cd ~/drone_ws/src
chmod +x run_all.sh
./run_all.sh
```

`run_all.sh` 将依次启动：
1. `roscore`
2. Livox LiDAR 驱动（`msg_MID360.launch`）
3. Fast-LIO（`mapping_mid360.launch`）
4. MAVROS（`px4.launch`）
5. `lio-to-mavros_node` — 坐标系转换
6. `offboard_node` — 航点控制
7. `uart_node` — 串口通信

如需开机自启，将 `run_all.sh` 加入系统启动项即可。

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│  上位机（机载计算机 Linux + ROS）                             │
│                                                             │
│  Livox LiDAR ──▶ livox_ros_driver2 ──▶ Fast-LIO ──▶ /Odometry
│                                                     │        │
│  lio-to-mavros_node ◀───────────────────────────────┘        │
│       │                                                     │
│       ▼                                                     │
│  /mavros/vision_pose/pose ──▶ MAVROS ──▶ PX4 EKF2           │
│                                                             │
│  uart_node ◀── /dev/ttyUSB0 ──▶ STM32F103（下位机）           │
│       │                                                     │
│       ▼                                                     │
│  /uart_data ──▶ offboard_node ──▶ /setpoint_position/local   │
│                       │                                     │
│                       ▼                                     │
│                   MAVROS ──▶ PX4 OFFBOARD 模式               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  下位机（STM32 + OpenMV）                                    │
│                                                             │
│  4×4 矩阵按键 ──▶ STM32F103 ──▶ DFS/BFS 航线规划              │
│                       │                                     │
│                       ├──▶ UART1 ──▶ 上位机（航点坐标）        │
│                       ├──▶ UART2 ──▶ LCD 串口屏（显示）       │
│                       └──▶ 接收 STM32F405 ◀─▶ OpenMV         │
│                                                             │
│  OpenMV H7 Plus: Edge Impulse FOMO 动物目标检测              │
└─────────────────────────────────────────────────────────────┘
```

### ROS 话题流转

| 话题 | 方向 | 说明 |
|------|------|------|
| `/Odometry` | Fast-LIO → lio-to-mavros | LiDAR 里程计 |
| `/mavros/vision_pose/pose` | lio-to-mavros → PX4 | 视觉定位位姿 |
| `/uart_data` | uart_node → offboard_node | 下位机发送的网格航点 |
| `/mavros/setpoint_position/local` | offboard_node → PX4 | OFFBOARD 位置指令 |
| `/mavros/local_position/odom` | PX4 → offboard_node | 无人机当前位置 |
| `/mavros/state` | PX4 → offboard_node | 飞控状态 |

### 航线规划逻辑（STM32F103）

- 7×9 网格（行 0-6，列 0-8），起降点固定为 `(6, 8)`
- 通过矩阵按键标记障碍物单元格
- **DFS** 算法规划巡检路径，**BFS** 计算最短返航路线
- 到达新单元格时等待 OpenMV 动物检测结果（约 1.4s 超时）
- 任务结束后在 LCD 屏上输出各动物统计数量

---

## 下位机烧录

### STM32F103（down/）

1. 用 Keil MDK 打开 `down/MDK-ARM/flydiansai.uvprojx`
2. 编译下载到 STM32F103 开发板
3. 如需修改引脚配置，用 STM32CubeMX 打开 `down/flydiansai.ioc`

### STM32F405（uper/）

1. 用 Keil MDK 打开 `uper/MDK-ARM/666.uvprojx`
2. 编译下载到 STM32F405 开发板

### OpenMV（openmv/）

1. 用 OpenMV IDE 打开 `openmv/main.py`
2. 将 `trained.tflite` 与 `labels.txt` 存入 OpenMV 存储
3. 运行脚本

---

## 视觉识别说明

当前 OpenMV 方案使用 Edge Impulse 训练的 **FOMO** 轻量模型，识别 5 类目标：

| 类别 | 标签 |
|------|------|
| 大象 | elephant |
| 老虎 | tiger |
| 狼 | wolf |
| 猴子 | monkey |
| 孔雀 | peacock |

> **注意**：OpenMV H7 算力有限，FOMO 模型效果一般。README 原作者建议改用 **OpenCV + 更强算力板卡** 的检测方案以获得更高识别精度。

---

## 3D 打印件

`3D maker/` 目录包含 SolidWorks 源文件（`.sldprt`），可根据实际需求修改：

- `330电池.sldprt` — 330mm 机架电池座
- `光流支架.sldprt` — 光流传感器安装支架

---

## 致谢

感谢以下同学对该项目的支持与开发：

- UEM Arklab Yusiyuan
- UEM storm Lixiaojia
- UEM storm ZhouZhenquan
- UEM strom The captain of Tomato

祝师弟师妹们勇创佳绩！
