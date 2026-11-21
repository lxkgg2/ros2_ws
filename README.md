# ROS 2 QoS Debugger — 发布/订阅 QoS 排查实验

<div align="center">

![ROS 2](https://img.shields.io/badge/ROS%202-Humble-blue)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20Jammy-E95420)
![C++](https://img.shields.io/badge/C%2B%2B-17-00599C)
![colcon](https://img.shields.io/badge/build-colcon-success)

**从"一条消息都收不到"到"100 Hz 零丢包"，一步步拆解 ROS 2 QoS 的坑**

</div>

---

## 📖 项目简介

ROS 2 里最让人抓狂的问题之一：**两个节点都正常启动，话题也在，但订阅端一条消息都收不到，而且不报任何错。**

这个工程就是为这类问题准备的最小复现 + 排查实验。核心是一对发布/订阅节点，通过参数可以自由切换 QoS 配置，用来观察：

- QoS 不兼容时如何**静默失败**，以及怎么用 `--verbose` 把真相挖出来
- 为什么 QoS 队列 `depth` 治不了真正的丢包问题
- 丢包率统计和帧率统计各自该怎么做才不出错

### 用一句话理解 QoS

> QoS 就像双方签的**通信合同**：发布者只能提供某个档次的服务，如果订阅者的要求比这更高，合同签不成——DDS 直接不建立连接，**不报错**。

---

## 🗂 工程结构

```
ros2_ws/
└── homework/                       # colcon 工作空间根目录
    └── src/
        ├── nav_hw_interfaces/      # 接口包：只放 .msg，无业务代码
        │   └── msg/SensorData.msg
        └── qos_debugger/           # 业务节点包
            └── src/
                ├── qos_debugger_pub.cpp   # 发布 /sensor_data
                └── qos_debugger_sub.cpp   # 订阅 /sensor_data
```

**消息定义 `SensorData.msg`**

```
std_msgs/Header header
uint32 seq                    # 自增序号，丢包检测靠它
float32 distance
float32[] ranges
```

---

## 🔧 环境要求

| 组件 | 版本 |
|------|------|
| 操作系统 | Ubuntu 22.04 (Jammy) |
| ROS 版本 | ROS 2 Humble |
| 编译工具 | colcon |
| 编译器 | g++ (C++17) |

确认环境：

```bash
ls /opt/ros/humble/setup.bash   # 存在即说明 ROS 2 已安装
```

---

## 🚀 编译与运行

```bash
# 1. 编译
cd ~/ros2_ws/homework
colcon build
source install/setup.bash

# 2. 开两个终端（每个终端都要 source）
# 终端 A：发布端
ros2 run qos_debugger qos_debugger_pub

# 终端 B：订阅端
ros2 run qos_debugger qos_debugger_sub
```

**预期结果**：订阅端每秒打印一次统计，且没有黄色警告。

```
[INFO] [sensor_subscriber]: 累计: 收到 100 条, 丢失 0 条, 丢包率 0.00%
[INFO] [sensor_subscriber]: 帧率: 100.00 Hz （最近 1.00 秒内收到 100 条）
```

---

# 实验报告

## 一、实验环境

| 项目 | 内容 |
|------|------|
| 操作系统 | Ubuntu 22.04 (Jammy) |
| ROS 版本 | ROS 2 Humble |
| 编译工具 | colcon |

---

## 二、任务一：实现 pub 和 sub 的通信

### 1. 排查过程与命令

运行两个节点后，发现订阅端**收不到任何消息**。使用以下命令排查：

```bash
# 查看当前系统中的所有话题，确认 /sensor_data 存在，说明发布节点正常运行
ros2 topic list

# 关键命令：查看话题两端的 QoS 详细配置
ros2 topic info /sensor_data --verbose

# 验证两端的参数默认值
ros2 param get /sensor_publisher reliability
ros2 param get /sensor_subscriber reliability
```

**命令讲解**

`ros2 topic info --verbose` 是排查 ROS 2 通信问题的利器。普通模式下它只告诉你发布者/订阅者的数量；加上 `--verbose` 后，它会打印出两端完整的 QoS 配置（可靠性、历史深度、持久性等），而这正是决定它们能不能建立通信的东西。

### 2. 问题原因

执行命令后发现：**发布端是 `BEST_EFFORT`，订阅端是 `RELIABLE`。**

根据 ROS 2 的 QoS 兼容性规则：

> 订阅者的可靠性要求**不能高于**发布者能提供的可靠性。

- 发布者只承诺"尽力而为"（像把外卖放楼下就走）
- 订阅者要求"必须可靠送达"（必须送到手上）

DDS 判定两者不兼容，直接拒绝建立连接，**且没有任何报错提示**，所以表现为一条消息都收不到。

### 3. 修复方法

修改 `qos_debugger_pub.cpp` 中的一处代码，把发布端默认可靠性改为 `reliable`：

```cpp
// 修改前
this->declare_parameter("reliability", "best_effort");
// 修改后
this->declare_parameter("reliability", "reliable");
```

修改后重新编译运行，订阅端开始正常打印消息。

### 4. 自己的理解

ROS 2 的 QoS 就像双方的"通信合同"，发布者只能提供基础服务，如果订阅者要求太高（不兼容），系统干脆就不让它们通信。这种**静默失败**的设计在工程上需要特别注意，一定要用 `--verbose` 去查看底层配置，而不能只看"节点是否启动"。

---

## 三、任务二：为什么收到的消息会丢包？

### 1. 排查过程与命令

通信建立后，订阅端大量打印黄色警告（检测到丢包），并且丢包率一直显示为 `0.00%`。使用以下命令排查：

```bash
# 列出节点的所有参数
ros2 param list

# 查看发布端的发布频率
ros2 param get /sensor_publisher rate

# 查看订阅端的回调延迟
ros2 param get /sensor_subscriber callback_delay_ms
```

**命令讲解**

`ros2 param get` 可以实时查看正在运行节点的参数配置。通过它发现：发布端 `rate` 为 **100.0 Hz**，而订阅端 `callback_delay_ms` 为 **30 ms**。

### 2. 问题原因

| 环节 | 数值 | 说明 |
|------|------|------|
| 生产速度 | 100 Hz | 每 10 ms 发一条 |
| 消费速度 | 30 ms/条 | 每秒最多处理约 33 条 |
| 队列限制 | KeepLast(10) | 只能缓冲 10 条 |

**消费速度远远低于生产速度**，队列瞬间被塞满，新消息直接把老消息挤掉，导致丢包。

另外还发现一个隐藏 bug：原代码在检测到丢包后只打印警告，**没有累加 `lost_count_`**，导致统计出来的丢包率永远是 0%。

### 3. 修复方法

修改 `qos_debugger_sub.cpp` 两处：

**① 把 `callback_delay_ms` 默认值改为 0，消除人为延迟**

```cpp
// 修改前
this->declare_parameter("callback_delay_ms", 30);
// 修改后
this->declare_parameter("callback_delay_ms", 0);
```

**② 在丢包分支补上 `lost_count_ += lost;`**

```cpp
else if (msg->seq > expected_seq_) {
    const uint32_t lost = msg->seq - expected_seq_;
    lost_count_ += lost;   // 补上这一行
    RCLCPP_WARN(...);
}
```

> **替代方案**：如果不改代码，也可以在节点运行时用 `ros2 param set /sensor_subscriber callback_delay_ms 0` 动态改参数，立刻生效。

### 4. 自己的理解

QoS 里的 `depth` 队列只是**缓冲**，治标不治本。只要消费速度跟不上，再深的队列也会被塞满。要真正解决丢包，要么提升消费者的处理速度，要么允许数据丢失时改用 `best_effort`，要么用多线程并行处理。

---

## 四、任务三：计算并打印帧率

### 1. 实现方法

在 `qos_debugger_sub.cpp` 的 `report()` 函数中，利用已有的两个成员变量 `last_received_count_` 和 `last_report_time_`，用"计数差值 ÷ 时间差值"计算帧率：

```cpp
// 记录当前时间
const auto now = std::chrono::steady_clock::now();
// 计算距离上次打印过了多少秒
const double dt = std::chrono::duration<double>(now - last_report_time_).count();
// 这段时间内新收到的消息条数
const uint32_t interval_count = received_count_ - last_received_count_;
// 帧率 = 条数 / 秒数
const double fps = (dt > 0.0) ? static_cast<double>(interval_count) / dt : 0.0;

RCLCPP_INFO(this->get_logger(),
            "帧率: %.2f Hz （最近 %.2f 秒内收到 %u 条）", fps, dt, interval_count);

// 更新基准，供下一秒计算
last_received_count_ = received_count_;
last_report_time_ = now;
```

### 2. 自己的理解

这里使用 `steady_clock`（单调时钟）而不是系统时间，是因为系统时间可能因网络校时突然跳变，导致时间差 `dt` 异常。`steady_clock` 保证时间单调递增，测量时间间隔更准确。

---

## 五、完整操作流程与验证结果

```bash
# 1. 编译
cd ~/ros2_ws/homework
colcon build
source install/setup.bash

# 2. 开两个终端运行节点
# 终端 1
ros2 run qos_debugger qos_debugger_pub
# 终端 2
ros2 run qos_debugger qos_debugger_sub

# 3. 命令行验证
ros2 topic info /sensor_data --verbose   # 确认两端 QoS 都是 RELIABLE
ros2 topic hz /sensor_data               # 确认话题频率约为 100 Hz
```

**最终运行结果**

- 发布端正常以 **100 Hz** 发布消息
- 订阅端每秒打印：`累计: 收到 100 条, 丢失 0 条, 丢包率 0.00%` 以及 `帧率: 100.00 Hz`
- 没有出现任何黄色丢包警告

---

## 🧠 踩坑速查表

| 现象 | 原因 | 排查/修复 |
|------|------|-----------|
| 一条消息都收不到，且无报错 | 订阅端 QoS 要求高于发布端（RELIABLE vs BEST_EFFORT） | `ros2 topic info <topic> --verbose`，让发布端改成 `reliable` |
| 黄色丢包警告刷屏，丢包率却是 0% | 消费速度跟不上 + `lost_count_` 没累加 | 降低/去掉回调延迟；补上 `lost_count_ += lost;` |
| 帧率算出来忽大忽小 | 用了系统时间（可能跳变） | 改用 `std::chrono::steady_clock` |

---

## 📚 参考资料

- [ROS 2 QoS 官方文档 — About QoS settings](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Quality-of-Service-Settings.html)
- [ROS 2 QoS 兼容性规则](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Quality-of-Service-Settings.html#qos-compatibilities)
- [rclcpp::QoS API](https://docs.ros.org/en/humble/p/rclcpp/generated/classrclcpp_1_1QoS.html)

---

<div align="center">

**nav_lecture4 小作业 · QoS Debugger**

</div>