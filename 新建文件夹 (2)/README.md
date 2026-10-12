# ROS 2 系统状态监视器：两个 Python 包 + Qt 窗口

目标环境：**Ubuntu 22.04 + ROS 2 Humble + 系统 Python 3.10 + PyQt5**。

本工程包含且仅包含两个 ROS 2 功能包。第一个包采集和发布主机状态，第二个包订阅后显示。不是两个互不通信的脚本，GUI 也不会自行采集本机数据。

```text
system_status_monitor                 system_status_gui
┌─────────────────────┐              ┌────────────────────────┐
│ psutil + ROS 定时器  │ ──话题─────> │ ROS 订阅 + Qt 主事件循环 │
│ status_publisher    │ /system/status│ status_gui             │
└─────────────────────┘              └────────────────────────┘
          └── 可选追加保存 JSONL 日志
```

## 1. 功能对应

| 要求 | 采集方式 / 显示方式 |
|---|---|
| 记录信息的时间 | 发布端实际采样时间，ISO 8601 UTC；GUI 转换为显示端本地时区 |
| 主机名 | `socket.gethostname()`，显示发布端主机名 |
| CPU 使用率 | `psutil.cpu_percent(interval=None)`，百分比和进度条 |
| 内存使用率 | `psutil.virtual_memory().percent`，百分比和进度条 |
| 内存总大小 | `virtual_memory().total`，自动显示 GiB 等单位 |
| 剩余内存 | `virtual_memory().available`，可供应用使用的内存；悬停显示 `free` |
| 网络接收 / 发送数据量 | 网卡累计接收 / 发送字节数，并显示增量计算的实时速率 |
| 简单小窗口 | PyQt5，默认 680 × 620，可调整大小 |

附加功能：可配置采样周期、指定网卡、日志记录、断流提示、坏消息校验、组合启动文件。

## 2. 目录结构

```text
ros2_system_monitor_ws/
├── README.md
├── docs/preview.png                 # 示例数据渲染的界面预览
├── TESTING.md                       # 实际验证结果与限制
├── conftest.py
├── pytest.ini
└── src/
    ├── system_status_monitor/
    │   ├── package.xml
    │   ├── setup.py / setup.cfg
    │   ├── resource/system_status_monitor
    │   ├── system_status_monitor/
    │   │   ├── __init__.py
    │   │   ├── metrics.py           # 系统数据采集和网速计算
    │   │   ├── protocol.py          # JSON 编解码和校验
    │   │   └── publisher.py         # ROS 发布节点
    │   └── test/test_metrics.py
    └── system_status_gui/
        ├── package.xml
        ├── setup.py / setup.cfg
        ├── resource/system_status_gui
        ├── launch/system_monitor.launch.py
        ├── system_status_gui/
        │   ├── __init__.py
        │   ├── window.py            # Qt 小窗口
        │   └── gui_node.py          # ROS 订阅与 Qt 事件循环
        └── test/
            ├── test_window.py
            └── test_ros_pipeline.py
```

## 3. 安装依赖并构建

以下操作在 **Ubuntu 22.04 的桌面终端**进行，前提是已经安装 ROS 2 Humble。使用系统 Python，不要在 Conda 环境构建。

将压缩包解压后，把 `ros2_system_monitor_ws` 放到主目录。若放在别处，相应修改 `cd` 路径。

```bash
source /opt/ros/humble/setup.bash
sudo apt update
sudo apt install -y python3-colcon-common-extensions python3-psutil \
  python3-pyqt5 python3-pytest fonts-noto-cjk \
  ros-humble-rclpy ros-humble-rcl-interfaces ros-humble-std-msgs \
  ros-humble-launch ros-humble-launch-ros

cd ~/ros2_system_monitor_ws
colcon build --symlink-install
source install/setup.bash
```

`package.xml` 同时声明了依赖；已有 rosdep 的环境也可以在工作空间根目录运行 `rosdep install --from-paths src --ignore-src -r -y`。

## 4. 一条命令运行

```bash
ros2 launch system_status_gui system_monitor.launch.py
```

默认每秒采样一次。第一次有效数据约在采集器启动一秒后出现；CPU 首次调用只用于建立基准。关闭窗口会同时结束该 launch 启动的采集进程。终端按 `Ctrl+C` 也可退出。

指定网卡、改变周期并保存记录：

```bash
# 先查看真实网卡名，例如 enp0s3、wlp2s0。
ip -brief link

# enp0s3 仅为示例，请替换为自己的网卡名。
ros2 launch system_status_gui system_monitor.launch.py \
  period:=0.5 interface:=enp0s3 log_file:="$HOME/system-status.jsonl"
```

`log_file` 默认留空，不写磁盘；启用后每条 JSON 单独占一行并追加写入，已有记录会保留。日志不自动轮转，长时间运行请自行管理文件大小。

## 5. 也可以在两个终端分别启动

每个新终端都先运行：

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_system_monitor_ws/install/setup.bash
```

终端 A：

```bash
ros2 run system_status_monitor status_publisher
# 或：
# ros2 run system_status_monitor status_publisher --ros-args -p publish_period:=0.5
```

终端 B：

```bash
ros2 run system_status_gui status_gui
```

两个独立 `ros2 run` 启动的进程需要分别关闭。

## 6. ROS 通信与数据含义

话题：`/system/status`。消息类型：`std_msgs/msg/String`，`data` 中存放 JSON。复用标准消息，使两个包都保持为纯 `ament_python` 包，不需要自定义消息生成步骤。工程化扩展时可将此契约改成强类型自定义 `.msg`。

订阅和发布均使用深度为 1 的默认可靠、易失 QoS，只保留最新待处理状态；迟加入的 GUI 在下一次发布时得到数据。

| 字段 | 类型 / 单位 |
|---|---|
| `schema_version` | 整数，目前为 1 |
| `sequence` | 从 1 开始的采样序号，节点重启后重置 |
| `timestamp` | 带时区的 ISO 8601 采样时间 |
| `hostname` | 字符串，采集端主机名 |
| `sample_interval_sec` | 实际两次采样间隔，秒 |
| `cpu_percent`, `memory_percent` | 0–100，百分比 |
| `memory_total_bytes` | 总物理内存，字节 |
| `memory_available_bytes` | 可用物理内存，字节 |
| `memory_free_bytes` | 完全空闲内存，字节，显示在可用内存的悬停提示中 |
| `network_interface` | 选择的网卡名或全部网卡说明 |
| `network_received_bytes`, `network_sent_bytes` | 系统网卡累计接收 / 发送字节数 |
| `network_receive_bytes_per_sec`, `network_send_bytes_per_sec` | 最近采样间隔内的平均网速，字节/秒 |

网络速率公式：`(本次累计字节数 - 上次累计字节数) / 实际间隔秒数`。间隔采用单调时钟，不受系统校时影响。网卡列表变化时重新建立速率基准，计数器回退时跳过受影响网卡的本轮速率，避免负值或虚假峰值。

累计量通常从系统启动或网卡计数器建立时累计，**不等于本程序启动后的流量**。默认汇总全部接口，包含回环、虚拟网卡；同一流量可能经过多个接口，这不是精确的外网流量统计。需要监看外网时，请指定实际联网网卡。

Linux 的 `free` 不包含可回收缓存，而 `available` 更适合表达“还能供程序使用的内存”，所以主界面用 `available`。内存百分比沿用 psutil 的 `(total - available) / total × 100` 口径。容量单位采用 1024 进制（KiB、MiB、GiB）。

采集节点使用稳态时钟定时，记录真实 UTC 时间，不跟随 ROS 仿真时钟。GUI 用收到消息的单调时间判断是否断流；阈值为 `max(stale_timeout, 3 × 上次实际采样间隔)`，断流时保留旧值并明确提示过期。

GUI 每 20 ms 用 Qt `QTimer` 调用一次 `executor.spin_once(timeout_sec=0.0)`。回调和界面更新均在主线程，不阻塞 Qt，不跨线程操作控件。

启动参数为只读，修改后请重启节点：

| 节点 | 参数 | 默认值 |
|---|---|---|
| 采集端 | `publish_period` | `1.0` 秒，至少 `0.1`；命令行请使用小数 |
| 采集端 | `network_interface` | 空字符串，全部接口 |
| 采集端 | `log_file` | 空字符串，关闭文件记录 |
| 两端 | `topic` | `/system/status` |
| 显示端 | `stale_timeout` | `3.0` 秒 |

## 7. 验收与测试

启动后在另一个已 source 环境的终端检查：

```bash
ros2 node list
ros2 topic info /system/status
ros2 topic echo /system/status --once
ros2 topic hz /system/status
```

应看到 `/system_status_publisher` 与 `/system_status_gui` 两个节点；默认发布频率约 1 Hz。窗口应显示真实主机名和更新时间，CPU / 内存随系统负载变化。

源码测试：

```bash
cd ~/ros2_system_monitor_ws
source /opt/ros/humble/setup.bash
source install/setup.bash
QT_QPA_PLATFORM=offscreen python3 -m pytest -q
```

或使用 ROS 工作空间测试流程：

```bash
QT_QPA_PLATFORM=offscreen colcon test --event-handlers console_direct+
colcon test-result --verbose
```

`test_ros_pipeline.py` 会真正创建发布者和订阅者，经 DDS 收到数据后检查 Qt 界面和日志。没有 `rclpy` 的机器会明确跳过这项测试，不能把这种跳过视为 ROS 通信验证通过。

手工断流测试：分别启动两个节点后停止采集端，约 3 秒后 GUI 应显示“数据已过期”；重启采集端后应恢复。鼠标悬停“剩余内存”可查看严格意义上的 `free` 值。

常见情况：

- 找不到包：确认构建成功，并在当前终端 `source install/setup.bash`。
- 一直等待数据：先检查发布端终端报错、两端话题名和 `ROS_DOMAIN_ID` 是否一致。
- 网卡不存在：用 `ip -brief link` 查看名称；不要直接照抄 `enp0s3`。
- Qt 无法连接 display：在 Ubuntu 图形桌面的终端运行；无桌面服务器只能运行采集端，或配置图形显示转发。
- 中文方框：安装 `fonts-noto-cjk` 后重新打开窗口。

## 8. 参考资料

- [ROS 2 Humble 官方 Python 发布节点示例](https://github.com/ros2/examples/blob/humble/rclpy/topics/minimal_publisher/examples_rclpy_minimal_publisher/publisher_member_function.py)
- [rclpy Humble 执行器源码](https://github.com/ros2/rclpy/blob/humble/rclpy/rclpy/executors.py)
- [psutil 官方文档](https://psutil.readthedocs.io/en/latest/)
- [Qt 5 QTimer 官方文档](https://doc.qt.io/archives/qt-5.15/qtimer.html)

可按作业要求修改两个 `package.xml` 和 `setup.py` 中的维护者姓名、邮箱。
