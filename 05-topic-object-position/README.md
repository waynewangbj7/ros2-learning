# 05-topic-object-position

把第 9 讲「服务查询坐标」改造成第 10 讲的「**话题持续发布坐标**」，用来对比同一份数据在两种通信机制下的差别。

实验编号：`ROS2-B-261126-001`
状态：坐标话题已跑通；话题版画面的实时叠字**待恢复**

---

## 为什么要做这个改造

同样的 `(x, y)`，两种传法：

| | 服务（第 9 讲） | 话题（本次） |
|---|---|---|
| 接口 | `/get_target_position` · `srv/GetObjectPosition` | `/object_position` · `msg/ObjectPosition` |
| 客户端 | 每查一次发一次请求 | 订阅后**一直收** |
| 时机 | 由客户端决定 | 由发布端决定（帧来了就发） |
| 适合 | 偶尔问一次 | 持续跟踪、多订阅者 |

**结论**：连续跟踪用话题，按需查询用服务。第 9 讲的客户端每运行一次才拿一个坐标，跟不上物块移动；改成话题后坐标会持续流出。

---

## 链路

```
usb_cam ──/image_raw──▶ interface_object_pub ──/object_position──▶ interface_object_sub
（原始图像，无叠字）      （检测 + 定时发布）      （msg/ObjectPosition）    （持续接收）
```

原有的 `/get_target_position` 服务**保留**，方便对照。

---

## 改动要点

补丁见本目录的 `orange-position-topic.patch`。核心是四件事：

**① 检测从"回调里立即发布"改成"定时发布"**

```python
self.position_updated = False
self.timer = self.create_timer(0.2, self.publish_position)   # 每 0.2 s 查一次

def listener_callback(self, data):
    image = self.cv_bridge.imgmsg_to_cv2(data, 'bgr8')
    self.object_detect(image)
    self.position_updated = True        # 有新帧才允许发布

def publish_position(self):
    if not self.position_updated:
        return                          # 没新帧就不发，避免重复刷同一条
    position = ObjectPosition()
    position.x, position.y = self.objectX, self.objectY
    self.pub.publish(position)
    self.position_updated = False
```

**② HSV 从红色改橙色**

```python
lower_orange = np.array([10, 120, 80])
upper_orange = np.array([25, 255, 255])
```

**③ 每帧先把坐标清零**，避免没有目标时仍发布上一帧的旧值

**④ 加实时叠字**

```python
if (self.objectX, self.objectY) != (0, 0):
    cv2.putText(image, f"x={self.objectX}  y={self.objectY}",
                (10, 25), cv2.FONT_HERSHEY_SIMPLEX, 0.55, (0, 255, 0), 2)
```

---

## 补丁说明

`orange-position-topic.patch` 是最终采用的一版，针对**本地已改过**的源码生成，可以直接应用。

最初还生成过一版（针对课程原始源码），因本地源码的空行差异导致 `git apply --check` 失败、未能写入，已弃用。

**经验**：给别人的代码打补丁，先 `git apply --check` 试跑；空行、缩进差一个字符就会失败。好在**失败时源码不会被改**——这比"以为打上了其实没有"安全得多。

---

## 运行

三个终端，A 和 B 保持不退出。

### A：相机

```bash
source /opt/ros/jazzy/setup.bash
ros2 run usb_cam usb_cam_node_exe --ros-args \
  -p video_device:=/dev/video0 -p pixel_format:=mjpeg2rgb \
  -p image_width:=320 -p image_height:=240
```

### B：检测并发布坐标

```bash
source /opt/ros/jazzy/setup.bash
source ~/guyue_ros2_ws/install/local_setup.bash
ros2 run learning_topic interface_object_pub
```

### C：订阅坐标

```bash
source /opt/ros/jazzy/setup.bash
source ~/guyue_ros2_ws/install/local_setup.bash
ros2 run learning_topic interface_object_sub
```

也可以临时用命令看：

```bash
ros2 topic echo /object_position
```

---

## 验证与待办

**已验证**

- 坐标发布与订阅可运行，订阅端不再需要手动查询
- 补丁设计为每 0.2 s 检查、有新帧才发布；**未实测消息频率**

**待处理**

- 话题版 `object` 预览窗口**没有实时坐标叠字**。诊断：终端导入的 `interface_object_pub` 里 `object_detect()` 未找到 `cv2.putText`。目前未确认是 `src` 源码缺这段，还是 `install/` 副本未更新——**本轮不改代码**

---

## 这一轮学到的排查经验

| 经验 | 说明 |
|---|---|
| `/image_raw` 是**原始**图像 | 别人往里加叠字，你订阅它看不到——叠字是本地画的 |
| `ros2 interface show` 只查**数据结构** | 不代表话题正在运行 |
| 话题有没有真的在发 | 只能靠 `ros2 topic echo` / `ros2 topic hz` |
| 改完源码要 `colcon build` | `install/` 是副本，不同步就会出现"代码里有、跑起来没有" |

---

## 边界

- 坐标是**像素位置**（原点在左上角，`x` 向右、`y` 向下），不是三维位置
- 第 9 讲服务实测返回过 `(214, 157)`，**这不是话题版的固定输出**
- 只验证颜色区域筛选，不区分橙色物体类别
