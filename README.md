# ROS 2 Humble 部署指南（WSL2 / Ubuntu 22.04）

> 基于对 [fishros/install](https://github.com/fishros/install) 源码的完整分析，本文档给出适合你当前 WSL2 环境的手动部署步骤，避免一键脚本中可能破坏现有配置的操作。

---

## 一、fishros 一键安装流程分析

当执行 `wget http://fishros.com/install -O fishros && . fishros` 并选择"安装 ROS2 Humble"时，背后实际执行了以下步骤：

### 1. 入口脚本 (`install`)
```bash
# 下载 install.py 到 /tmp/fishinstall/
# 安装 python3-distro, python3-yaml
# 以 sudo 执行 install.py
```

### 2. `install.py` — 主菜单
- 下载 `tools/base.py` 和翻译模块
- 展示分类工具菜单
- 选择 **选项 1（一键安装 ROS）** 会依次触发：
  - **选项 5**：更换系统源（`tool_config_system_source.py`）
  - **选项 4**：配置 ROS 环境变量（`tool_config_rosenv.py`）
  - **选项 1**：实际安装 ROS（`tool_install_ros.py`）

### 3. `tool_config_system_source.py` — ⚠️ 高风险步骤
这是 **最危险的环节**，它会：
- **删除 `/etc/apt/sources.list`**（选项 2 还会删除整个 `/etc/apt/sources.list.d/`）
- 自动测速选择最快的 Ubuntu 镜像源（清华/中科大/中山大学/官方）
- 用新源重写 `sources.list`
- 执行 `sudo apt update`

> **WSL 风险**：你已配置清华源且可能包含 CUDA 等第三方源，此步骤会破坏这些配置。

### 4. `tool_install_ros.py` — ROS 核心安装
```
① 检查系统支持 (jammy ∈ ros2_dist_dic ✅)
② 询问是否换系统源（推荐选"是"，但可以跳过）
③ 添加 ROS GPG 密钥（从 gitee/github 获取 ros.asc）
④ 创建 /etc/apt/sources.list.d/ros-fish.list
   内容: deb [arch=amd64] http://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu/ jammy main
⑤ 搜索可用 ROS 版本 → 选择 humble
⑥ 选择桌面版 (ros-humble-desktop) 或基础版 (ros-humble-ros-base)
⑦ apt install 对应包
⑧ 安装额外依赖: python3-colcon-common-extensions, python3-argcomplete, python3-rosdep
```

### 5. `tool_config_rosenv.py` — 环境配置
- 扫描 `/opt/ros/*/setup.bash`
- 在 `~/.bashrc` 中添加 `source /opt/ros/humble/setup.bash`

### 6. `tool_config_rosdep.py` — rosdepc（选项 3）
- 安装 `python3-pip`
- 通过 pip 安装 `rosdepc`（小鱼的 rosdep 替代品）
- 执行 `rosdepc init` + `rosdepc fix-permissions`

---

## 二、WSL2 手动部署步骤（推荐）

> 以下步骤**完全跳过**换系统源操作，保留你现有的 APT 配置。

### Step 1：确认系统环境

```bash
# 确认 Ubuntu 22.04 + amd64
lsb_release -a
# 应输出: Ubuntu 22.04.5 LTS, Codename: jammy

dpkg --print-architecture
# 应输出: amd64
```

### Step 2：配置 ROS 2 APT 源（仅添加，不替换系统源）

```bash
# 1. 添加 ROS 2 GPG 密钥
sudo apt update && sudo apt install curl gnupg2 -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/ros.gpg
sudo chmod 644 /etc/apt/trusted.gpg.d/ros.gpg

# 2. 添加 ROS 2 清华镜像源（你已有清华源，速度最快）
echo "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu/ jammy main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list

# 3. 更新 APT 索引
sudo apt update
```

> 备选镜像（如果清华源不可用）：
> - 中科大：`https://mirrors.ustc.edu.cn/ros2/ubuntu/`
> - 华为云：`https://repo.huaweicloud.com/ros2/ubuntu/`
> - 官方源：`http://packages.ros.org/ros2/ubuntu/`

### Step 3：安装 ROS 2 Humble

```bash
# 桌面版（推荐，含 RViz、Gazebo 等 GUI 工具）
sudo apt install ros-humble-desktop -y

# 如果遇到依赖问题，尝试 aptitude：
# sudo apt install aptitude -y
# sudo aptitude install ros-humble-desktop
# （出现提示时先选 n，再选 y 用 aptitude 的方案解决依赖）

# 额外工具
sudo apt install python3-colcon-common-extensions python3-argcomplete python3-rosdep -y
```

> **基础版**（无 GUI，更小）：`sudo apt install ros-humble-ros-base -y`

### Step 4：配置 ROS 2 环境

```bash
# 将 Humble 环境自动加载到 bashrc
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
source ~/.bashrc

# 验证安装
echo $ROS_DISTRO
# 应输出: humble

# 查看具体包版本
apt show ros-humble-desktop 2>/dev/null | grep -E "^(Version|APT-Sources)"

# 测试（ROS 2 没有 roscore，用节点测试）
ros2 run demo_nodes_cpp talker --ros-args -r __node:=talker_demo &>/dev/null &
timeout 3 ros2 run demo_nodes_cpp listener
# 看到 "Hello World" 来回传输即成功（3 秒后自动退出）
kill %1 2>/dev/null
```

### Step 5：安装 rosdep（官方版 / rosdepc 可选）

```bash
# 推荐使用官方 rosdep（需网络代理，否则 GitHub raw 可能超时）
sudo apt install python3-rosdep -y
sudo rosdep init
rosdep update
```

> 国内直连 GitHub 不稳定时，可用鱼香ROS的 **rosdepc** 替代（规则文件镜像到 Gitee，无需代理）：
> ```bash
> sudo apt install python3-pip -y
> sudo pip3 install rosdepc --break-system-packages
> sudo rosdepc init
> rosdepc update
> sudo rosdepc fix-permissions
> ```

---

## 三、Vision 项目依赖安装

根据项目 [README.md](README.md) 和你的 WSL2 环境：

```bash
# colcon 构建工具
sudo apt install python3-colcon-common-extensions -y

# ROS 2 相关依赖
sudo apt install ros-humble-asio-cmake-module -y
sudo apt install ros-humble-foxglove-bridge -y

# 系统依赖
sudo apt install libgpiod-dev -y

# 安装项目 ROS 依赖（在 ros2-framework 目录下）
cd ~/code/vision
rosdep install --from-paths src --ignore-src -r -y
```

---

## 四、WSL2 特殊注意事项

### 4.1 GUI 程序（RViz2 / Gazebo）
WSL2 支持 GUI 转发，但需要：
```bash
# 在 Windows 侧安装 VcXsrv 或使用 WSLg（Windows 11 默认支持）
# 测试 GUI：
ros2 run rviz2 rviz2
```

如果无法显示，检查 `DISPLAY` 变量：
```bash
echo $DISPLAY
# 通常应为 :0 或 hostname:0
export DISPLAY=:0  # 手动设置
```

### 4.2 硬件设备访问
WSL2 对 USB、CAN 等硬件设备的直通支持有限：
- **CAN 总线**：`sudo ip link set can0 ...` 在 WSL2 中通常不可用，需要在 Windows 侧配置
- **摄像头**：需要 `usbipd` 工具将 USB 设备从 Windows 映射到 WSL2
- **GPU (CUDA)**：WSL2 已原生支持，你已安装 CUDA 13.3 ✅

### 4.3 网络
WSL2 使用 NAT 网络，与 Windows 共享 IP。如果需要 ROS 2 跨设备通信：
```bash
# 查看 WSL IP
ip addr show eth0 | grep inet

# 设置 ROS_DOMAIN_ID 避免多机干扰
export ROS_DOMAIN_ID=42  # 0-232 之间任意值
echo 'export ROS_DOMAIN_ID=42' >> ~/.bashrc
```

### 4.4 systemd 支持
Ubuntu 22.04 WSL2 默认可能未启用 systemd。如需开机自启等服务：
```bash
# 检查 systemd
ps --no-headers -o comm 1
# 如果输出 init 而非 systemd，则在 /etc/wsl.conf 中添加：
# [boot]
# systemd=true
```

---

## 五、完整安装命令汇总（可直接执行）

```bash
#!/bin/bash
# ROS 2 Humble 完整安装脚本（WSL2 安全版）
set -e

echo "=== Step 1: 添加 ROS 2 源 ==="
sudo apt update && sudo apt install curl gnupg2 -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/ros.gpg
sudo chmod 644 /etc/apt/trusted.gpg.d/ros.gpg
echo "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu/ jammy main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list
sudo apt update

echo "=== Step 2: 安装 ROS 2 Humble 桌面版 ==="
sudo apt install ros-humble-desktop python3-colcon-common-extensions \
  python3-argcomplete python3-rosdep -y

echo "=== Step 3: 配置环境 ==="
grep -q "humble/setup.bash" ~/.bashrc || \
  echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
source ~/.bashrc

echo "=== Step 4: 安装 rosdep（需网络代理） ==="
sudo apt install python3-rosdep -y
sudo rosdep init
rosdep update

echo "=== Step 5: 安装项目依赖 ==="
sudo apt install ros-humble-asio-cmake-module ros-humble-foxglove-bridge libgpiod-dev -y
cd ~/code/vision
rosdep install --from-paths src --ignore-src -r -y

echo "=== 安装完成! ==="
echo "ROS Distro: $ROS_DISTRO"
apt show ros-humble-desktop 2>/dev/null | grep "^Version"
```

---

## 六、与 fishros 一键安装的差异总结

| 操作 | fishros 一键安装 | 本指南手动方案 |
|------|-----------------|---------------|
| 换系统源 | ✅ 会重写 `/etc/apt/sources.list` | ❌ 跳过，保留现有配置 |
| 清理第三方源 | 可选（选项2会删除整个 `sources.list.d/`） | ❌ 不操作 |
| 添加 ROS 源 | 写入 `ros-fish.list` | 写入 `ros2.list` |
| GPG 密钥 | apt-key（已废弃）或 gpg | gpg dearmer（现代方式） |
| 安装版本 | `ros-humble-desktop` | `ros-humble-desktop` |
| 环境配置 | 支持多版本选择脚本 | 直接 source 单版本 |
| rosdep | rosdepc（小鱼修改版） | 官方 rosdep（需代理），rosdepc 备选 |
| shell 兼容 | bash / zsh 自动检测 | 仅指导 bash |

---

## 七、故障排除

| 问题 | 解决方案 |
|------|---------|
| `gpg: no valid OpenPGP data found` | 换用 `curl` 代替 `wget`；或手动下载 `ros.asc` |
| `Unable to locate package ros-humble-desktop` | 检查 `/etc/apt/sources.list.d/ros2.list` 是否存在且源 URL 正确 |
| `unmet dependencies` | `sudo apt --fix-broken install` 或 `sudo aptitude install ros-humble-desktop` |
| `rosdep update` 超时 | 使用 `rosdepc` 替代，或配置代理 |
| GUI 程序无法启动 | 检查 WSLg 是否运行；尝试 `export DISPLAY=:0` |
| CAN 设备不可用 | WSL2 内核不支持 SocketCAN，需在 Windows 侧处理 |

---

## 八、许可证

本教程采用 [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) 协议。

你可以在任何媒介上自由分享、改编本作品，只需保留原作者署名。

> 完整法律条文见 <https://creativecommons.org/licenses/by/4.0/legalcode>
