# map_navigation
仅用于实现PC端命令行终端slam建图（cyberdog2）
导航功能相关接口：

启动任务：
接口形式：ros action
接口名字："start_algo_task"
接口文件：protocol/ros/srv/Navigation.action

关闭任务：
接口形式： ros service
接口名字： "stop_algo_task"
接口文件：protocol/ros/srv/StopAlgoTask.srv

任务状态查询
接口形式： ros topic
接口名字： "algo_task_status"
接口文件：protocol/ros/msg/AlgoTaskStatus.msg

实现操作：

1、启动环境
先登录机器狗终端:

ssh mi@xxx.xxx.xxx.xxx(机器狗的IP地址)

然后执行：

source /opt/ros2/cyberdog/setup.bash
（如遇报错：xxxx not found，可以暂时先不管，这个报错不影响后续使用。）
如果你的机器狗上有工作空间并且已经编译过，再执行：

source ~/cyberdog_ws_rolling/install/setup.bash
如果这句提示 No such file or directory，先不用管，说明你当前主要用的是系统安装包。

2、检查导航包能不能找到：
按顺序输入以下三条指令

ros2 pkg list | grep navigation_bringup
ros2 pkg list | grep algorithm_manager
ros2 pkg list | grep protocol

如果能看到这三个包，则继续。

3、确认机器狗命名空间

CyberDog2 通常会把 topic 放到一个命名空间下面，例如：

/mi_desktop_48_b0_2d_5f_be_ac/odom_out

你需要执行以下指令：找到机械狗的命名空间

ros2 action list | grep start_algo_task
ros2 service list | grep stop_algo_task
ros2 service list | grep -E "start_mapping|stop_mapping|get_label|outdoor"
你应该能看到类似：

/mi_desktop_xxx/start_algo_task
/mi_desktop_xxx/stop_algo_task

就设置：

export DOG_NS=mi_desktop_xxx
export NS_PREFIX=/DOG_NS

4、如果找不到第3步的节点，需要重新启动导航总 launch

推荐先启动完整导航栈：

ros2 launch navigation_bringup navigation.launch.py namespace:=$DOG_NS

注意：这个 launch 里很多节点是延迟启动的，algorithm_manager 大约要等 2 分钟左右才起来。不要一看到没反应就关掉，先等 2-3 分钟。

如果新开了一个终端，同样先 source：

source /opt/ros2/cyberdog/setup.bash
source ~/cyberdog_ws_rolling/install/setup.bash 2>/dev/null || true
export DOG_NS=你的命名空间
export NS_PREFIX=/$DOG_NS

如果没有命名空间，就：

export DOG_NS=
export NS_PREFIX=

5、 开始实验室建图

启动室内激光建图：

ros2 action send_goal "$NS_PREFIX/start_algo_task" protocol/action/Navigation "{nav_type: 5, map_name: '', outdoor: false}" --feedback

nav_type不同值分别对应不同功能，具体数字对应功能如下：

1 = AB 点导航

5 = 激光建图 LaserMapping

7 = 激光定位 LaserLocalization

然后用手机 APP 慢慢控制机器狗在实验室走一圈。

建图建议：

先沿实验室墙边走一圈

再走中间区域

速度慢一点，不要急转弯

尽量回到起点附近形成闭环

光照保持稳定

不要让机器狗频繁碰撞桌腿、椅子

这一步的地图还没有真正保存，必须执行下一步。

6、 停止建图并保存地图

建议给地图起一个清楚的名字，例如 lab_map_01：

export MAP_NAME=lab_map_01

停止建图并保存：

ros2 service call "$NS_PREFIX/stop_algo_task" protocol/srv/StopAlgoTask "{task_id: 5, map_name: '$MAP_NAME'}"

保存成功后，地图文件会在机器狗的这个目录：

/home/mi/mapping/

检查地图文件：

ls -lh /home/mi/mapping

正常情况下会生成这些文件：

/home/mi/mapping/lab_map_01.pgm

/home/mi/mapping/lab_map_01.yaml

/home/mi/mapping/lab_map_01.json

其中：

.pgm 是栅格地图图片

.yaml 是地图分辨率、原点、图片路径等配置

.json 是 CyberDog2 自己保存的地图标签/室内外标记信息

查看地图配置：

cat /home/mi/mapping/${MAP_NAME}.yaml

7、 如何保存、备份、删除地图

保存地图就是第 5 步的这条命令：（停止之后，自动保存）

ros2 service call "$NS_PREFIX/stop_algo_task" protocol/srv/StopAlgoTask "{task_id: 5, map_name: '$MAP_NAME'}"

地图位置：

/home/mi/mapping/

备份地图：

mkdir -p /home/mi/mapping_backup

cp -a /home/mi/mapping/${MAP_NAME}.* /home/mi/mapping_backup/

删除某一张地图时，只删同名的三类文件：

rm -i /home/mi/mapping/${MAP_NAME}.yaml

rm -i /home/mi/mapping/${MAP_NAME}.pgm

rm -i /home/mi/mapping/${MAP_NAME}.json


本地工程关键文件：

navigation.launch.py (line 41) 启动导航总栈。

node.laser_mapping.launch.py (line 42) 调用 laser_slam 建图 launch。

Task.toml (line 10) 定义 LaserMapping=5、LaserLocalization=7、NavAB=1。

Navigation.action (line 3) 定义导航 action 的任务编号。

StopAlgoTask.srv (line 1) 定义停止/保存任务服务。

label_store.cpp (line 28) 定义地图目录 /home/mi/mapping/。
