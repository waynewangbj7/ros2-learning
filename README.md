# ROS 2 学习与实机实验

WSL2 + Ubuntu 24.04 + ROS 2 Jazzy 环境下的学习记录，包含自己写的节点、真实摄像头接入、双节点链路与服务调用，以及完整的排错过程。

> 本仓库只保留**自己写的代码、文档和实验数据**。课程示例源码（古月居 `ros2_21_tutorials`）不入库，仅在文档中注明出处与修改点。

---

## 环境

| 项目 | 版本 |
|---|---|
| Windows | 11 25H2（build 26200） |
| WSL | 2.6.2.0，内核 6.6.87 |
| Ubuntu | 24.04.5 LTS |
| ROS 2 | Jazzy（含 desktop、ros_gz、usb_cam） |
| 编辑器 | VS Code + WSL Remote + clangd |
| 图形 | WSLg（Gazebo 走 D3D12 硬件渲染） |

---

## 目录

```
ros2-learning/
├─ 01-my-first-pkg/          自写 C++ 功能包（定时发布者节点）
├─ 02-vision-red-detection/  摄像头接入 + HSV 红色区域识别（单节点）
├─ 03-dual-node-vision/      图像发布-订阅双节点 + 用 usb_cam 替换发布端
├─ 04-orange-service/        橙色识别 + ROS 2 服务返回像素坐标
├─ 05-topic-object-position/ 同一份坐标改用话题持续发布（含补丁）
└─ 排错记录.md               排查记录（累计）
```

| 子项目 | 内容 | 涉及 ROS 2 机制 | 状态 |
|---|---|---|---|
| [01-my-first-pkg](01-my-first-pkg/) | C++ 定时发布者 | **话题（发布）** | ✅ |
| [02-vision-red-detection](02-vision-red-detection/) | usbipd 接摄像头 + HSV 红色分割 | 单节点 + OpenCV | ✅ 定性通过 |
| [03-dual-node-vision](03-dual-node-vision/) | 图像话题双节点，发布端换 `usb_cam` | **话题（发布+订阅）** | ✅ 两种发布端均通过 |
| [04-orange-service](04-orange-service/) | 橙色检测 + 服务查询坐标 | **服务（请求+应答）** | ✅ 返回 `(214, 157)` |
| [05-topic-object-position](05-topic-object-position/) | 同一份坐标改用话题持续发布 | **服务 → 话题的改造** | ⚠️ 话题通，叠字待恢复 |

---

## 时间线

| 日期 | 内容 |
|---|---|
| 11-17 | 搭建 WSL + Ubuntu 24.04 + ROS 2 Jazzy（rootfs 导入路线） |
| 11-18 | turtle 仿真、`rqt_graph`、话题/服务实验 |
| 11-19 | tmux、Linux 权限、git 身份配置、VS Code 接 WSL |
| 11-20 | 第一个 C++ 功能包与发布者节点 |
| 11-22 | 接入真实摄像头，单节点 HSV 红色识别 |
| 11-23 | 双节点图像链路；发布端换成 `usb_cam`；服务查询橙色目标坐标 |
| 11-26 | 把坐标从「服务一问一答」改成「话题持续发布」，对比两种机制 |

---

## 从这几个实验里得到的结论

1. **话题是解耦点**：识别端只依赖 `/image_raw`，换掉发布端（课程 OpenCV 节点 → `usb_cam`）后一行代码都不用改。
2. **"有日志"不等于"有数据"**：发布端打印 `Publishing video frame` 可能写在取帧判断之外；判断有没有帧只能用 `ros2 topic echo`。
3. **服务返回 `(0, 0)` 不等于没检测到**：只能说明服务应答了，要看画面和返回值是否同步更新。
4. **编译通过 ≠ 能跑**：`colcon` 显示 `Finished` 也可能运行时抛 `IndentationError` / `NameError`，改完先 `py_compile`。
5. **服务与话题各有场合**：同一份坐标，连续跟踪用话题（订阅后一直收），按需查询用服务（问一次答一次）。第 9 讲的客户端每运行一次只拿一个坐标，跟不上移动；改成话题后才持续流出。

---

## 说明

- 所有实验记录都是**实际操作的产物**，包括失败和返工（见 [排错记录](排错记录.md)）
- 数字和结论只写验证过的部分，未量化的地方明确标注"定性"
- 这是学习仓库，不是产品代码；工程化程度会随学习进度改进

---

## 参考

- [古月居 ROS 2 教程](https://github.com/guyuehome/ros2_21_tutorials)
- [微软 WSL USB 接入文档](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)
- [OpenCV HSV 颜色空间说明](https://docs.opencv.org/4.13.0/df/d9d/tutorial_py_colorspaces.html)
- [usb_cam（ROS 2 驱动）](https://github.com/ros-drivers/usb_cam)
