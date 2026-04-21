# map_navigation

仅用于实现 PC 端命令行终端 SLAM 建图（CyberDog2）

---

## 导航功能相关接口

```bash
# 启动任务（接口形式：ros action）
接口名称: start_algo_task
接口文件: protocol/ros/srv/Navigation.action

# 关闭任务（接口形式：ros service）
接口名称: stop_algo_task
接口文件: protocol/ros/srv/StopAlgoTask.srv

# 任务状态查询（接口形式：ros topic）
接口名称: algo_task_status
接口文件: protocol/ros/msg/AlgoTaskStatus.msg
```

---

## 使用步骤

### 1️ 启动环境

先登录机器狗终端：

```bash
ssh mi@xxx.xxx.xxx.xxx   # 机器狗IP
```

加载环境：

```bash
source /opt/ros2/cyberdog/setup.bash
```

> 如遇报错 `xxx not found` 可以暂时忽略，不影响使用

如果存在工作空间：

```bash
source ~/cyberdog_ws_rolling/install/setup.bash
```

> 若提示 `No such file or directory` 可忽略

---

### 2️ 检查导航相关包

```bash
ros2 pkg list | grep navigation_bringup
ros2 pkg list | grep algorithm_manager
ros2 pkg list | grep protocol
```

能查到说明环境正常。

---

### 3️ 确认机器狗命名空间

查询接口：

```bash
ros2 action list | grep start_algo_task
ros2 service list | grep stop_algo_task
ros2 service list | grep -E "start_mapping|stop_mapping|get_label|outdoor"
```

示例输出：

```bash
/mi_desktop_xxx/start_algo_task
/mi_desktop_xxx/stop_algo_task
```

设置命名空间：

```bash
export DOG_NS=mi_desktop_xxx
export NS_PREFIX=/$DOG_NS
```

> 注意：不要写成 `//xxx`，否则会报 topic 名错误

如果没有命名空间：

```bash
export DOG_NS=
export NS_PREFIX=
```

---

### 4️ 启动导航系统（如未运行）

```bash
ros2 launch navigation_bringup navigation.launch.py namespace:=$DOG_NS
```

> ⏳ 注意：
> - algorithm_manager 可能需要 **2分钟左右启动**
> - 不要过早关闭

新终端需要重新加载环境：

```bash
source /opt/ros2/cyberdog/setup.bash
source ~/cyberdog_ws_rolling/install/setup.bash 2>/dev/null || true

export DOG_NS=你的命名空间
export NS_PREFIX=/$DOG_NS
```

---

### 5️⃣ 开始建图（室内激光 SLAM）

```bash
ros2 action send_goal "${NS_PREFIX}/start_algo_task" \
protocol/action/Navigation \
"{nav_type: 5, map_name: '', outdoor: false}" \
--feedback
```

---

### nav_type 功能说明

| 数值 | 功能 |
|------|------|
| 1    | AB 点导航 |
| 5    | 激光建图（LaserMapping） |
| 7    | 激光定位（LaserLocalization） |

---

### 建图建议

- 沿墙走一圈
- 再走中间区域
- 速度慢，不急转弯
- 尽量形成闭环
- 光照稳定
- 避免频繁碰撞

---

### 6️ 停止建图并保存地图

```bash
export MAP_NAME=lab_map_01
```

```bash
ros2 service call "${NS_PREFIX}/stop_algo_task" \
protocol/srv/StopAlgoTask \
"{task_id: 5, map_name: '$MAP_NAME'}"
```

---

### 地图保存路径

```bash
/home/mi/mapping/
```

检查文件：

```bash
ls -lh /home/mi/mapping
```

生成文件：

```bash
lab_map_01.pgm
lab_map_01.yaml
lab_map_01.json
```

说明：

- `.pgm` → 地图图像
- `.yaml` → 地图配置
- `.json` → 标签/室内外信息

查看配置：

```bash
cat /home/mi/mapping/${MAP_NAME}.yaml
```

---

### 7️ 地图管理

#### 备份地图

```bash
mkdir -p /home/mi/mapping_backup
cp -a /home/mi/mapping/${MAP_NAME}.* /home/mi/mapping_backup/
```

#### 删除地图

```bash
rm -i /home/mi/mapping/${MAP_NAME}.yaml
rm -i /home/mi/mapping/${MAP_NAME}.pgm
rm -i /home/mi/mapping/${MAP_NAME}.json
```

---

## 本地工程关键文件

```bash
navigation.launch.py        # 启动导航总栈
node.laser_mapping.launch.py # 激光建图 launch
Task.toml                  # 定义任务编号（5/7/1）
Navigation.action          # action 定义
StopAlgoTask.srv           # 停止任务接口
label_store.cpp            # 地图目录定义
```
