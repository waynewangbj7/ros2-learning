# ros2_learning_pkg

第一个自己写的 ROS 2 C++ 功能包：一个**定时发布者节点**。

---

## 内容

```
ros2_learning_pkg/
├─ CMakeLists.txt
├─ package.xml
└─ src/
    └─ talker.cpp   每 1 秒在 /chatter 上发布一条字符串
```

## 节点在做什么

```cpp
class Talker : public rclcpp::Node
{
public:
  Talker() : Node("talker"), count_(0)
  {
    pub_   = this->create_publisher<std_msgs::msg::String>("chatter", 10);
    timer_ = this->create_wall_timer(1s, std::bind(&Talker::tick, this));
  }

private:
  void tick()   // 每 1 秒被调用一次
  {
    auto msg  = std_msgs::msg::String();
    msg.data  = "Hello ROS 2: " + std::to_string(count_++);
    pub_->publish(msg);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
  }

  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
  size_t count_;
};
```

| 语句 | 作用 |
|---|---|
| `Node("talker")` | 节点名（出现在 `ros2 node list` 里） |
| `create_publisher<...>("chatter", 10)` | 在 `/chatter` 上发布，队列深度 10 |
| `create_wall_timer(1s, ...)` | 每 1 秒触发一次回调 |
| `publish(msg)` | 真正把消息发出去 |

## 节点的四步骨架

```cpp
int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);                            // 1. 初始化运行时（连 DDS、读环境变量）
  rclcpp::spin(std::make_shared<Talker>());            // 2. 创建节点并持续处理回调
  rclcpp::shutdown();                                  // 3. 清理退出
  return 0;
}
```

`spin` 会一直阻塞，直到收到 `Ctrl+C`。**所有 ROS 2 节点的骨架都是这几步**，区别只在中间注册了什么（定时器、订阅、服务）。

## 编译与运行

```bash
# 放进工作空间
cp -r ros2_learning_pkg ~/ros2_ws/src/ros2_learning_pkg
cd ~/ros2_ws
colcon build
source install/setup.bash

ros2 run ros2_learning_pkg talker
```

预期输出（每秒一行）：

```
[INFO] [...] [talker]: Publishing: 'Hello ROS 2: 0'
[INFO] [...] [talker]: Publishing: 'Hello ROS 2: 1'
```

## 验证

```bash
# 另开一个终端
source ~/ros2_ws/install/setup.bash
ros2 topic echo /chatter            # 每秒收到一条字符串

ros2 node info /talker               # 看它发布/订阅了什么
ros2 param list /talker              # 应只有 use_sim_time
```

`ros2 node info` 的输出里除了 `/chatter`，还会出现 `/rosout`、`/parameter_events` 和 7 个 `*_parameters` 服务——**这些是 ROS 2 为每个节点自动创建的**，本包并没有写。

## CMakeLists 关键三行

```cmake
add_executable(talker src/talker.cpp)
ament_target_dependencies(talker rclcpp std_msgs)
install(TARGETS talker DESTINATION lib/${PROJECT_NAME})
```

三行都必须在 `ament_package()` **之前**。

## 踩过的坑

| 现象 | 原因 | 解法 |
|---|---|---|
| `No executable found` | `install(TARGETS ...)` 漏了，或写在 `ament_package()` 之后（不生效） | 补齐并放对位置 |
| `Package 'ros2_learning_pkg' not found` | 没 `source install/setup.bash` | 每个新终端都要 source |
| 包建到了错误位置 | `ros2 pkg create` 在**当前目录**建包 | 必须在 `src/` 里执行 |
| 编译成功但跑不起来 | 一个没有任何可执行目标的包也能编译通过 | 用 `ros2 pkg executables <包名>` 确认 |
