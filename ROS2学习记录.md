## 节点介绍
### 每个节点只负责单独的模块化的功能
![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1730786776975-697b095d-208a-4695-8868-9441cf244bef.png)

### 节点的四种通信方式 --第五章
+ 话题 topics
    - 话题有发布者和订阅者， 单向过程
+ 服务 services
    - 请求和回复，双向过程
+ 动作 action
+ 参数 parameters

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1730786966148-b79f05c2-e594-409b-b8b1-6e8b19fede51.png)



### 通过命令行运行节点
ros2 run 包名称 可执行文件名称

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1730787191435-2a2cd0bb-5093-4fba-8099-a673823ddd16.png)

ros2 node list		查看节点列表

ros2 node info <node_name>		查看节点信息



---

## 工作空间
![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1730787905673-e14c17e4-77b9-467e-92ab-81af825a41d7.png)

工作空间是**包含若干个功能包的目录**，一开始把工作空间理解成一个文件夹就行了。这个文件夹包含下有**src** 。所以一般新建一个工作空间的操作就像下面一样!  
mkdir: 创建一个目录，mkdir -p:递归创建目录,即使上级目录不存在,会按目录层级自动创建目录

```bash
mkdir -p turtle_ws/src
cd turtle_ws/src
```



### 功能包
功能包可以理解为存放节点的地方，ROS2中功能包根据编译方式的不同分为三种类型。

+ ament_python，适用于python程序
+ cmake，适用于C++
+ ament_cmake，适用于C++程序,是cmake的增强版

#### 功能包获取的两种方式
+ 安装获取

```bash
sudo apt install ros-<version>-package_name
```

+ 手动编译获取（详见通信机制案例）

#### 功能包相关指令
+ ros2  pkg executables   --列出可用功能包
+ ros2 pkg executables turtlesim --列出指定功能包(海龟模拟器)
+ ros2 pkg list	--列出所有包
+ ros2 pkg prefix <pkg-name>	--输出某个包所在路径的前缀
+ ros2 pkg xml <pkg-name>	--输出包的清单描述文件



---

## ROS2构建工具 colcon
**colcon --功能包的构建工具，用来编译代码**

安装命令：

```bash
sudo apt-get install python2-colcon-common-extebsions
```



---

## 创建一个功能包
```bash
ros2 pkg create village_li --build-type ament_python --dependencies rclpy
```

+ pkg create 	--创建包
+ --build-type	--指定包的编译类型，一共三个可选项 ament_python, ament_cmake, cmake，缺省则默认为ament_cmake
+ --dependencies	--此功能包的依赖包（rclpy： ROS Client Library Python ROS2的python客户端接口）

步骤：

+ 创建功能包后cd 进入功能包路径（含__init__.py）
+ 编辑功能代码
+ 编辑setup.py的功能入口
+ 回到工作空间路径
+ colcon build进行编译
+ source install/setup.bash 进行安装
+ ros2 run 功能包  节点名称 



## ROS2通信机制
### 话题
Node----Publish--->Topic---Subscrib--->Node

可以一对一，一对多，多对1，多对多

规则：

+ 话题名字是关键，发布和订阅的接口类型要相同，发布的是字符串，订阅也要用字符串
+ 同一个节点可以订阅多个话题，同时也可以发布多个话题
+ 同一个话题可以有多个发布者

#### 工具
##### RQT工具之rqt_graph
在运行的过程中，通过命令看到节点和节点之间的数据关系

例：

```bash
ros2 run demo_nodes_py listener
ros2 run demo_node_cpp talker
rqt_graph
```

得到如下直观节点之间通信关系

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1731981858436-ea273ac3-0ad7-419f-b923-0d6103a5c3cb.png)

##### 话题相关命令行界面（CLI）工具
```bash
ros2 topic -h
```

![](https://cdn.nlark.com/yuque/0/2024/png/34610865/1731982179638-de361c43-f441-4280-991f-70a1b4c6ab1f.png)

```bash
ros2 interface show		--查看消息类型
```



```bash
ros2 topic pub 话题名称 话题内容		--手动发布话题
例：ros2 topic pub /chatter std_msgs/msg/String '{data: 'wowwwwwww'}		--发布字符串话题
```





### 服务
客户端发送请求，服务端响应请求

运行一个服务端：（运行节点）

```bash
ros2 run 功能包 可执行文件名称
例：ros2 run examples_rclpy_minimal_service service
```

注意事项：

+ 同一个服务（名称相同）有且只有一个节点来提供
+ 同一个服务可以被多个客户端调用



#### 工具
查看服务列表：

```bash
ros2 service list	--查看服务列表
```

手动调取服务：

```bash
ros2 service call  /add_two_ints example_interfaces/srv/AddTwoInts '{a: 44, b: 33}'
/ 的意思是ros2框架下的全局
```

##### rqt命令
rqt--->Service Caller    rqt界面调用服务

##### 服务相关命令行
```bash
ros2 service type /add_two_ints		--查看服务接口类型

ros2 service list --列出可用服务  配合  ros2 service type --查询具体类型 使用 
```

```bash
ros2 interface show 是 ROS2 中用于查看消息、服务或动作类型的具体定义的命令。
例：ros2 interface show example_interfaces/srv/AddTwoInts   
```

服务类型文件（以 `.srv` 为后缀）定义了**请求**和**响应**的数据格式。例如，`example_interfaces/srv/AddTwoInts.srv` 的内容如下：

```plain
int64 a
int64 b
---
int64 sum
```

+ `---` 将请求和响应分隔开。
+ 请求部分包含两个整数 `a` 和 `b`。
+ 响应部分返回一个整数 `sum`。

### 参数
**参数就是ROS2节点的配置值**， 参数提供了一种灵活的方式，让开发者可以在运行时修改节点的配置，而无需重启或重新编译节点。  

**参数特征：**参数以 `key-value` 的形式存在，其中 key 是参数的名字，value 是对应的值。例如：

```plain
param_name: "robot_speed"
param_value: 1.5
```

支持多种数据类型：

+ 标量型：int, float, bool, string
+ 数组型：int[], float[], bool[], string[], byte[]

#### 工具
```bash
ros2 param get 节点名 参数名		--得到相关参数的内容
ros2 param set 节点名 参数名		--设置相关参数值（并不会保存，仅当前有效）
ros2 param dump 节点名					--保存当前参数
```



### action
 在 ROS2 的通信机制中，**Action（动作）** 是一种专门设计用于处理长时间任务的通信方式。它允许客户端向服务端发起任务请求，同时支持**目标状态反馈**和**任务取消**等功能，非常适合需要持续监控任务状态或可中断的任务场景。  

#### **Action 的核心概念**
1. **Action 客户端**  
向服务端发送动作目标（Goal）请求，可以随时检查任务的执行状态，接收中间反馈或取消任务。
2. **Action 服务端**  
负责接收客户端的目标请求，执行目标任务，并在完成时返回结果或在任务过程中提供反馈。
3. **Action 的结构**  
Action 的接口由以下三部分组成：
    - **Goal（目标）**：客户端发送的任务请求参数。
    - **Feedback（反馈）**：服务端在任务执行过程中提供的中间状态信息。
    - **Result（结果）**：服务端在任务完成后返回的最终结果。

---

#### **Action 的通信流程**
1. **客户端发送 Goal**  
Action 客户端向服务端发送任务请求（Goal）。
2. **服务端接收并执行任务**  
服务端收到目标后开始任务执行，并可能发送中间反馈。
3. **任务完成或取消**
    - 当任务完成时，服务端返回结果。
    - 如果客户端取消了任务，服务端会终止执行。

---

#### **Action 的使用场景**
+ 机器人导航（如移动到指定位置）
+ 抓取任务（如机械臂抓取物体）
+ 图像处理任务







## **ROS2 run 和 ROS2 launch区别**
| 特性 | `ros2 run` | `ros2 launch` |
| --- | --- | --- |
| **用途** | 启动单个节点 | 启动多个节点或复杂系统 |
| **配置能力** | 不支持参数文件、复杂配置 | 支持动态参数、参数文件、依赖管理等 |
| **适用场景** | 测试和调试单个节点 | 系统级别运行，多个节点协作 |
| **文件支持** | 无需额外文件，直接运行可执行文件 | 需要 `.launch.py`<br/> 配置文件 |
| **灵活性** | 简单直接 | 高度灵活 |


---

+ `**ros2 run**`：适用于简单、直接运行单节点的场景。
+ `**ros2 launch**`：适用于复杂场景，尤其是需要运行多个节点、加载参数文件和环境配置时。

