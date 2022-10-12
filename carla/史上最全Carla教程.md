### （一）Carla的基本架构

#### Client-Server的交互形式

Carla主要分为Server与Client两个模块，Server端用来建立这个仿真世界，而Client端则是由用户控制，用来调整、变化这个仿真世界。

1.Server: Server端负责任何与仿真本身相关的事情：从3D渲染汽车、街道、建筑，传感器模型的构建，到物理计算等等。它就像一个**造物主，** 将整个世界建造出来，并且根据Client 的外来指令更新这个世界。

2.Client: 如果server构造了整个世界，那么这个世界不同时刻到底该如何运转（比如天气是什么样，有多少辆车在跑，速度是多少）则是由Client端控制的。用户通过书写Python脚本（最新版本C++ 也可以）来向Server端输送指令指导世界的变化，Server根据用户的指令去执行。（可以理解为Client端耍耍嘴皮子下个指令，咱们的造物主亲力亲为去执行这些指令。 另外，Client端也可以接受Server端的信息，譬如某个照相机拍到的路面图片。）

#### Carla核心模块

1. **Traffic Manager**: 自动驾驶之所以难搞，很核心的一个原因就是现实世界车太多了！试想如果整个世界就你一辆车在大马路上跑，自动驾驶恐怕早实现了。因此，Carla专门构造了Traffic Manager这个模块来模拟类似现实世界负责的交通环境。通过这个模块，用户可以定义N多不同车型、不同行为模式、不同速度的车辆在路上愉快地与你的自动驾驶汽车（Ego-Vehicle）一起玩耍。这个模块后面会详细讲解。
2. **Sensors:** Carla里面有各种各样模拟真实世界的传感器模型，包括相机、激光雷达、声波雷达、IMU、GNSS等等。为了让仿真更接近真实世界，它里面的相机拍出的照片甚至还有畸变和动态模糊效果。用户一般将这些Sensor attach到不同的车辆上来收集各种数据。
3. **Recorder：** 俗话说的好，不能复现的仿真不是好仿真。这个模块就是用来记录仿真每一个时刻（Step)的状态，可以用来回顾、复现等等。
4. **ROS bridge：** 这个模块可以让Carla与ROS还有Autoware交互，正是这个模块的存在使得在仿真里测试你的自动驾驶系统变得可能，十分重要，后面也会详细讲解。
5. **Open Assest**：这个模块可以允许你为仿真世界添加customized的物体库，比如你可以在默认的汽车蓝图里再加一个真实世界不存在、外形酷炫的小飞汽车，用来给Client端调用。

### （二）Carla安装

### （三）Carla基础API使用

#### 0. 将Carla Library安装到你的python3里

#### 1. Client and World

这两个词将会贯穿我们整个教程。

Client我们在第一节提到过，用户通过Client载体与python API与仿真环境交互，所以我们第一部就是要创建Client，并且设置一个timeout时间防止连接时间过久。

```python3
client = carla.Client('localhost', 2000)
client.set_timeout(2.0)
```

其中2000是端口，2.0是秒数。

接下来我们就要通过这个构建的Client来获取仿真世界（World)。我们如果想让仿真世界有任何变化，都要对这个获取的world进行操作。

```text
world = client.get_world()
```

作为一个简单明了的示例，我们可以先试着改变一下世界的天气。

```text
weather = carla.WeatherParameters(cloudiness=10.0,
                                  precipitation=10.0,
                                  fog_density=10.0)
world.set_weather(weather)
```

#### 2. Actor 与 Blueprint

创立了世界之后，就要开始安放我们的主角——actor了。顾名思义，Actor意味演员，在仿真世界里则代表可以移动的物体，包括汽车，传感器（因为传感器要安在车身上）以及行人。

- **2.1. 生成（spawn) Actor**

如果我们想生成一个Actor, 必须要先定义它的蓝图（Blueprint），这就好比造房子前要先话设计图一样。

```text
# 拿到这个世界所有物体的蓝图
blueprint_library = world.get_blueprint_library()
# 从浩瀚如海的蓝图中找到奔驰的蓝图
ego_vehicle_bp = blueprint_library.find('vehicle.mercedes-benz.coupe')
# 给我们的车加上特定的颜色
ego_vehicle_bp.set_attribute('color', '0, 0, 0')
```

构建好蓝图以后，下一步便是选定它的出生点。我们可以给固定的位子，也可以赋予随机的位置，不过这个位置必须是空的位置，比如你不能将奔驰扔在一棵树上。

```text
# 找到所有可以作为初始点的位置并随机选择一个
transform = random.choice(world.get_map().get_spawn_points())
# 在这个位置生成汽车
ego_vehicle = world.spawn_actor(ego_vehicle_bp, transform)
```

- **2.2 操纵（Handling）Actor**

汽车生成以后，我们便可以随意挪动它的初始位置，定义它的动态参数。

```text
# 再给它挪挪窝
location = ego_vehicle.get_location()
location.x += 10.0
ego_vehicle.set_location(location)
# 把它设置成自动驾驶模式
ego_vehicle.set_autopilot(True)
# 我们可以甚至在中途将这辆车“冻住”，通过抹杀它的物理仿真
# actor.set_simulate_physics(False)
```

- **2.3 注销（Destroy) Actor**

当这个脚本运行完后要记得将这个汽车销毁掉，否则它会一直存在于仿真世界，可能影响其他脚本的运行哦。

```text
# 如果注销单个Actor
ego_vehicle.destroy()
# 如果你有多个Actor 存在list里，想一起销毁。
client.apply_batch([carla.command.DestroyActor(x) for x in actor_list])
```

#### 3.Sensor搭建

有了车子之后，我们就可以往上放各种各样的sensor啦！在这里先简单介绍下Carla有哪些Sensors，  在[Python API reference](https://link.zhihu.com/?target=https%3A//carla.readthedocs.io/en/latest/python_api/) 里有对每一种数据格式类别的详细介绍。

| Sensor         | Sensor输出数据形式             | 功能                                       | 类别     |
| -------------- | ------------------------------ | ------------------------------------------ | -------- |
| RGB Camera     | carla.Image                    | 普通的RGB相机                              | Cameras  |
| 深度相机       | carla.Image                    | 深度相机，以灰度图形式储存                 | Cameras  |
| 分割相机       | carla.Image                    | 直接输出景物分割图，不同颜色代表不同的种类 | Cameras  |
| Collision      | carla.CollisionEvent           | 汽车发生碰撞时启动，会将事故的信息记录下来 | Detector |
| Lane invasion  | carla.LaneInvasionEvent        | 汽车变道时启动，将Lane ID与汽车ID记录下来  | Detector |
| Obstacle       | carla.ObstacleDetectionEvent   | 将可能挡在前方行驶道路上的物体记下         | Detector |
| GNSS           | carla.GNSSMeasurement          | 记录车子的地理位置                         | Other    |
| IMU            | carla.IMUMeasurement           | 记录汽车的轴加速度与角加速度               | Other    |
| LIDAR          | carla.LidarMeasurement         | 激光雷达                                   | Other    |
| Radar          | carla.RadarMeasurement         | 声波雷达                                   | Other    |
| Semantic LIDAR | carla.SemanticLidarMeasurement | 除了3D点云外，还提供额外的Semantic信息     | Other    |

在这个教程里，我们会建立Camera与激光雷达。

- **Camera构建**

与汽车类似，我们先创建蓝图，再定义位置，然后再选择我们想要的汽车安装上去。不过，这里的位置都是相对汽车中心点的位置（以米计量）。

```python3
camera_bp = blueprint_library.find('sensor.camera.rgb')
camera_transform = carla.Transform(carla.Location(x=1.5, z=2.4))
camera = world.spawn_actor(camera_bp, camera_transform, attach_to=ego_vehicle)
```

我们还要对相机定义它的callback function,定义每次仿真世界里传感器数据传回来后，我们要对它进行什么样的处理。在这个教程里我们只需要简单地将文件存在硬盘里。

```text
camera.listen(lambda image: image.save_to_disk(os.path.join(output_path, '%06d.png' % image.frame)))
```

- **Lidar构建**

Lidar可以设置的参数比较多，对Lidar模型不熟也没有关系，我在后面会另开文章详细介绍激光雷达模型，现在就知道我们设置了一些常用参数就好。

```text
lidar_bp = blueprint_library.find('sensor.lidar.ray_cast')
lidar_bp.set_attribute('channels', str(32))
lidar_bp.set_attribute('points_per_second', str(90000))
lidar_bp.set_attribute('rotation_frequency', str(40))
lidar_bp.set_attribute('range', str(20))
```

接着把lidar放置在奔驰上, 定义它的callback function.

```text
lidar_location = carla.Location(0, 0, 2)
lidar_rotation = carla.Rotation(0, 0, 0)
lidar_transform = carla.Transform(lidar_location, lidar_rotation)
lidar = world.spawn_actor(lidar_bp, lidar_transform, attach_to=ego_vehicle)
lidar.listen(lambda point_cloud: \
            point_cloud.save_to_disk(os.path.join(output_path, '%06d.ply' % point_cloud.frame)
```

#### 4. 观察者（spectator）放置

当我们去观察仿真界面时，我们会发现，自己的视野并不会随我们造的小车子移动，所以经常会跟丢它。解决这个问题的办法就是把spectator对准汽车，这样小汽车就永远在我们的视野里了！

```text
spectator = world.get_spectator()
transform = ego_vehicle.get_transform()
spectator.set_transform(carla.Transform(transform.location + carla.Location(z=20),
                                                    carla.Rotation(pitch=-90)))
```

放置观察者后效果如下，我们将以俯视的角度观察它的运行：

![img](https://pic1.zhimg.com/80/v2-cfb62ed565d68630b17efb02f3648464_720w.webp)

#### **5. 查看储存的照片与3D点云**

储存的RGB相机图如下：



![img](https://pic1.zhimg.com/80/v2-b70222938b5acf5c681beb754245b488_720w.webp)

查看点云图需要另外安装meshlab, 然后进入meshlab后选择import mesh：

```text
sudo apt-get update -y
sudo apt-get install -y meshlab
meshlab
```

![img](https://pic4.zhimg.com/80/v2-05f7966ef808dd049bda5d45f968648b_720w.webp)



#### 6.完整代码

欢迎浏览我的github获取完整代码！这个repo目前正在不断更新中，将会与这个blog同步进行。

[https://github.com/DerrickXuNu/Learn-Carlagithub.com/DerrickXuNu/Learn-Carla](https://link.zhihu.com/?target=https%3A//github.com/DerrickXuNu/Learn-Carla)

#### 7. 下集预告

不知道小伙伴有没有发现自己储存的照片是不连续的，存在掉帧现象？这是因为Carla Simulation Server 与 Client默认为异步进行，也就是说server并不会等client处理完手头的事再进行模拟，它会以最快的速度进行模拟，这就导致你的client如果处理过慢，还没来得及储存第K个data, simulation早就进行到了第K+10步，也就会产生掉帧了。

所以在下一节，我会介绍Carla里很重要的一个模块——**同步模式，**以此来解决我们掉帧的问题。

### （四）同步模式

我也模仿鲁迅先生的话说一句：**Carla仿真不等客户，除非你会设置同步。**

在上一个[blog](https://zhuanlan.zhihu.com/p/340031078)里，我们成功地在仿真世界造了一辆奔驰小轿车，设置了自动驾驶模式，并安置了相机与lidar不停地收集数据储存在硬盘里。但是我们发现，储存的相机照片有着严重的掉帧现象，这是为什么呢？

一句话简单总结原因，因为咱们的**仿真server默认为异步模式，它会尽可能快地进行仿真，根本不管客户是否跟上了它的步伐**。接下来我会更详细地介绍这个原因，并提出解决方案。

#### 仿真世界里的时间步长

在simulation里的时间与真实世界是不同的，simulation里没有“一秒”的概念，只有“一个time-step"的概念。这个time-step相当于仿真世界进行了一次更新（比如小车们又往前挪了一小步，天气变阴了一丢丢），它在真实世界里的时间可能只有几毫秒。

仿真世界里的这个time-step其实有两种，一种是Variable time-step, 另一种是Fixed time-step.

- Variable time-step. 顾名思义，仿真每次步长所需要的真实时间是不一定的，可能这一步用了3ms, 下一步用了5ms, 但是它会竭尽所能地快速运行。这是仿真默认的模式：

```text
settings = world.get_settings()
settings.fixed_delta_seconds = None # Set a variable time-step
world.apply_settings(settings)
```

- Fixed time-step. 在这种时间步长设置下，每次time-step所消耗的时间是固定的，比如永远是5ms. 设置代码如下：

```text
settings = world.get_settings()
settings.fixed_delta_seconds = 0.05 #20 fps, 5ms
world.apply_settings(settings)
```

#### 同步模式

看到这里，我相信各位小伙伴们已经猜到了，carla simulation**默认模式为异步模式**+**variable time-step**, 而**同步模式则对应fixed time-step**.

在异步模式下, server会自个跑自个的，client需要跟随它的脚步，如果client过慢，可能导致server跑了三次，client才跑完一次, 这就是为什么咱们照相机储存的照片会掉帧的原因。

而在同步模式下，simulation会等待客户完成手头的工作后，再进行下一次更新。假设simulation每次更新只需要固定的5ms,但我们客户端储存照片需要10ms, 那么simulation就会等照片储存完才进行下一次更新，也就是说，一个真正cycle耗时10ms(simulation更新与照片储存是同时开始进行的）。设置代码**关键部分**如下：

```text
def sensor_callback(sensor_data, sensor_queue, sensor_name):
    if 'lidar' in sensor_name:
        sensor_data.save_to_disk(os.path.join('../outputs/output_synchronized', '%06d.ply' % sensor_data.frame))
    if 'camera' in sensor_name:
        sensor_data.save_to_disk(os.path.join('../outputs/output_synchronized', '%06d.png' % sensor_data.frame))
    sensor_queue.put((sensor_data.frame, sensor_name))

settings = world.get_settings()
settings.synchronous_mode = True
world.apply_settings(settings)

camera = world.spawn_actor(blueprint, transform)
sensor_queue = queue.Queue()
camera.listen(lambda image: sensor_callback(image, sensor_queue, "camera"))

while True:
    world.tick()
    data = sensor_queue.get(block=True)
```

这段代码首先注意到的是 **world.tick()** 这个函数。它只出现于同步模式，意思是让simulation更新一次。然后我们还会发现这里用了python自带的Queue, queue.get有一个功效，就是在它把列队里所有内容都提取出来之前，会阻止任何其他进程越过自己这一步，相当于一个blocker。如果没有这个queue，你会发现仿真虽然设置成了同步模式，还是照样会自个跑自个的。

所以你可以这样理解，**settings.synchronous_mode = True** 让仿真的更新要通过这个client来唤醒，但这并不能保证它会等该client其他进程运行完，必须要再加一个queue来阻挡一下它，逼迫它等着该客户其他线程搞定。也就是说，启动同步模式，让你的server学会等待客户的必要条件有三个：

1. **settings.synchronous_mode = True**
2. **world.tick()**
3. **Thread Blocker(such as Queue)**

当你将上述代码加到我们上一次的程序里，会发现，照片是不掉帧了，但是小车一动也不动了。这里是因为同步模式下汽车要使用autopilot必须依附于**开启同步模式的traffic manager**.至于这个traffic manager是何方神圣，咱们下期会详细讲述，现在你只需要如何操作：

traffic_manager = client.get_trafficmanager(8000)
traffic_manager.set_synchronous_mode(True)

ego_vehicle = world.spawn_actor(ego_vehicle_bp, transform)
ego_vehicle.set_autopilot(True, 8000)

#### 注意事项

1. 目前carla只支持**单客户同步**，也就是说，如果你有N个python scripts(N个client), **只能在其中一个client里设置同步模式, 而其他client只能异步模式.** 这些处于异步模式的客户首先通过world.wait_for_tick() 等待server 更新*，*一旦更新了它们会立刻通过*world.*on_tick 里的callback 来提取这个更新的wordsnapshot里面的信息（比如timestamp， 这个on_tick()我会在以后的分享里详细说明举例，现在暂且用不到）.

```text
# Wait for the next tick and retrieve the snapshot of the tick.
world_snapshot = world.wait_for_tick()

# Register a callback to get called every time we receive a new snapshot.
world.on_tick(callback)
```

2. 在你设置了同步模式的client完成了它的任务准备停止/销毁时，千万别忘了**将世界设置回异步模式**，否则server会因为找不到它的同步客户而卡死。

```text
try:
  .......
finally:
  settings = world.get_settings()
  settings.synchronous_mode = False
  settings.fixed_delta_seconds = None
  world.apply_settings(settings)
```

#### 完整代码：

完整代码已经上线啦！

[https://github.com/DerrickXuNu/Learn-Carla/blob/main/python/synchronize.pygithub.com/DerrickXuNu/Learn-Carla/blob/main/python/synchronize.py](https://link.zhihu.com/?target=https%3A//github.com/DerrickXuNu/Learn-Carla/blob/main/python/synchronize.py)

#### 总结

同步/异步模式因为涉及到多线程的问题，设置和使用的时候要格外小心，一般设置同步模式的客户主要是用来做数据储存与采集的。在下一期里，我将会讲到如何**通过神秘的Traffic Manager, 让街道充满各种不同行为模式的汽车，为你的无人汽车模拟真实的交通环境**。

### （五）交通管理器

今天我们要学习的，就是CARLA里面及其重要的一个模块——Traffic Manager(交通管理器）。在上一章中，我们成功地让小车在城市里以同步模式自由地穿梭，然而有一个严重的问题仍然没有解决——我们的小车太孤单，整个城市只有它自己，孑然一身（脑海里想起了这首歌“空荡的街景，想找个人放感情......").

为了解决小车的单身问题，顺便模拟真实的交通环境，我们需要在simulation server里先放置一些其他的车辆。但是这些车辆的行为该怎如何去管理、控制来模拟真实世界里的交通场景呢？这就要用到我们的**Traffic Manager**了！

#### 1. 什么是Traffic Manager?

Traffic Manager 简称TM，是仿真里用来控制车辆行为的模块。它是纯C++构造包装被Python调用的，所以如果用户想要了解其内部构造是无法从PythonAPI里面看到的，需要查看C++代码： ~/carla/Libcarla/source/trafficmanager/TrafficManager.h。 它最大的作用就是可以帮用户**集体管理一个车群**，比如说设定所有的奔驰和林肯车辆的限速和最小安全车距。有了它，用户就可以轻易地定义某些群体的共性行为。另外很重要的一点要指出，**在同步模式下，设置AutoPilot的车辆必须依附于设置为同步模式的traffic manager才能跑起来**。

#### 2. Traffic Manager的内部架构

Traffic Manager的内部架构如下图所示，是不是看起来特别头大？不要担心，让小飞帮你重点梳理！

- **第一阶段： Vehicle Registry**. 首先Server端的Lifecycle管理器（ALSM）会扫描simulation里所有的汽车、行人的信息，删除掉已经died的物体，然后Traffic Manager会把部分车辆注册到自己旗下（至于哪些会被选中是由用户决定的), 这些被选中的车辆的行为从此以后就会被该traffic manager控制了!（可以有多个Traffic manager同时存在，后面会详细讲述）
- **第二阶段：Localization Stage**. In Memory Map这个外部模块会缓存整张地图的拓扑结构以及每辆汽车的短期路线规划（实质上是一系列的waypoint), TM在这一阶段做的就是读取in memory map，把自己管辖的车辆的位置、路径规划存到内部模块path buffer & vehicle tracking(PBVT)里面，这样方便后面的函数快速便捷读取相关信息。
- **第三阶段：Collision Stage.** 这一阶段TM会根据每个管辖车辆的周边环境和他们的路线规划来判断是否有潜在车祸可能，如果有的话会将此信息传递给motion planner阶段。
- **第四阶段： Traffic Light Stage.** 与上一阶段类似，TM会探测每一辆车是否接近红灯或路口，将信息传给下一阶段。
- **第五阶段： Motion Planner.** 将上个两阶段的信息整合到一起，制定每一辆所管辖的车下一步的举动（比如变道、急刹或者正常行驶），设立下一个目标点，将该点传给PID控制器，得到方向盘应该转动的角度和踩油门的力度、刹车的力度。
- **第六阶段：Send Command Array.** 将上述每一辆车的控制命令一次性分发到每辆车上，实现控制。



![img](https://pic4.zhimg.com/80/v2-cb32de5de2cab0bfb48b0b9d2d71abd3_720w.webp)

#### 3. Traffic Manager的使用方法

- 创建traffic manager. 其中这个tm_port参数代表我们的TM是从哪个口接入server的，默认为8000

```text
world = client.get_world()
 traffic_manager = client.get_trafficmanager(args.tm_port)
```

- 设置该traffic manager管辖车群的整体行为模式。特别说明两个点，一个是traffic model里的hybrid physics mode. 一旦这个模式被设置，只有在ego-vehicle（可以特别设定哪一辆车是ego-vehicle) 附近一定范围的车辆会开启物理特性。所谓的物理特性就是在运动时会考虑一些真实的物理限制，比如轮胎摩擦、道路曲率等等。这个混合物理模式就是为了减少计算消耗，只让ego-vehicle附近的交通更具有真实性。另外一个点就是tm默认所有的车辆限速为30公里每小时，通过设置global_percentage_speed_difference可以改变默认速度。80%意思是默认限速为24，-80%则意味着默认限速为30*180%=54km/h.

```text
# tm里的每一辆车都要和前车保持至少3m的距离来保持安全
traffic_manager.set_global_distance_to_leading_vehicle(3.0)
# tm里面的每一辆车都是混合物理模式
traffic_manager.set_hybrid_physics_mode(True)
# tm里面每一辆车都是默认速度的80%
 traffic_manager.global_percentage_speed_difference(80)
```

- 设置TM同步模式如果该client是同步模式

```text
if args.sync:
        settings = world.get_settings()
        traffic_manager.set_synchronous_mode(True)
         if not settings.synchronous_mode:
            synchronous_master = True
            settings.synchronous_mode = True
                # 20fps
            settings.fixed_delta_seconds = 0.05
            world.apply_settings(settings)
```

- 将我们的一部分车辆分配给TM。这里我们用了carla.command, 因为我们TM管理许多车辆，用指令的方式可以方便用户一次性创立所有车辆并且将这些车辆分配给TM。 SpawnActor(blueprint, transform).then(SetAutopilot(FutureActor, True, traffic_manager.get_port())) 的意思是先按照蓝图和位置创建这辆汽车，然后再把这辆车分配给咱们的TM，其中FutureActor就是一个整数值0，意味着self.

```text
# Use command to apply actions on batch of data
SpawnActor = carla.command.SpawnActor
SetAutopilot = carla.command.SetAutopilot
# this is equal to int 0
FutureActor = carla.command.FutureActor

batch = []

for n, transform in enumerate(spawn_points):
       if n >= args.number_of_vehicles:
                break

       blueprint = random.choice(blueprints_vehicle)

       if blueprint.has_attribute('color'):
                color = random.choice(blueprint.get_attribute('color').recommended_values)
                blueprint.set_attribute('color', color)
       if blueprint.has_attribute('driver_id'):
                driver_id = random.choice(blueprint.get_attribute('driver_id').recommended_values)
                blueprint.set_attribute('driver_id', driver_id)

       # set autopilot
       blueprint.set_attribute('role_name', 'autopilot')

       # spawn the cars and set their autopilot all together
       batch.append(SpawnActor(blueprint, transform)
                         .then(SetAutopilot(FutureActor, True, traffic_manager.get_port())))
```

- 执行提前设置好的指令。因为使用同步模式，所以使用apply_batch_sync执行命令。在这些指令没有执行完之前，main thread会被block住。对于每一个指令都会有一个response返回，通过response我们可以得知命令是否下达成功以及回应的汽车的id

```text
# excute the command
for (i, response) in enumerate(client.apply_batch_sync(batch, synchronous_master)):
  if response.error:
       logging.error(response.error)
  else:
       print("Fucture Actor", response.actor_id)
       vehicles_id_list.append(response.actor_id)
```

到了这里，traffic manager的基本用法已经结束了，我在

[DerrickXuNu/Learn-Carlagithub.com/DerrickXuNu/Learn-Carla](https://link.zhihu.com/?target=https%3A//github.com/DerrickXuNu/Learn-Carla)

里有在这个基础用法上加一点点好玩的东西，我还设置了crazy vehicle, 它们不会保持和前车的距离并且开的飞快，以此来模拟现实生活中那些开车不要命的人。除此之外，我还在里面放了咱们的ego-vehicle --- 不再寂寞的奔驰小车，同时放上了rgb camera, 用户可以实时看到相机里拍摄到的内容。由于这些额外的内容不过是重复使用咱们已经讲过的方法，这里就不详细列出了，想要了解的欢迎到我的repo里看完整代码。

#### 4. 多个Traffic Manager

在carla里，你可以构建多个traffic manager, 大概有以下几个原则：

- 你可以创立多个tm, 这几个tm可以在一个client里，也可以不在一个client里创造
- 只有一个tm可以设置为同步模式

#### 5. 下集预告———高能预警！

在接下来的一章，我会通过carla官方示例automatic_control.py 来详细阐述carla里行为规划是如何实现的，**这将全世界仅有的一个详细教程喔！**

### （六）一文解析CARLA的行为规划（上）

大家好，欢迎回到小飞仿真课堂！ 前段时间由于自己在忙着搭建一个基于CARLA-SUMO 合作仿真的**多车协同驾驶的框架（不久就会开源，敬请期待！），**导致自己迟迟没有更新，在这里和各位小伙伴说抱歉了！

今天，我将会带来一篇非常硬的干货，这将是**全网唯一一篇**详细阐述carla里**自动驾驶行为规划是如何实现**的文章！

众所周知，Carla的官方代码里给出过一个automatic_control.py的例子来展示行为规划的实现过程，不过这个例子里也包含了许多pygame相关的class object来单独弹出一个窗口展示画面，读起来十分不清爽，所以我在它的基础上进行了一些改动，**只留下了与行为规划直接相关的代码，**其他的一律剔除，方便大家理解**。**所以各位如果想获得最佳阅读体验**，请先打开下面链接里给出的修改版automatic_control.py来与本文进行一一对照。**

[https://github.com/DerrickXuNu/Learn-Carla/blob/main/python/automatic_control_revised.pygithub.com/DerrickXuNu/Learn-Carla/blob/main/python/automatic_control_revised.py](https://link.zhihu.com/?target=https%3A//github.com/DerrickXuNu/Learn-Carla/blob/main/python/automatic_control_revised.py)

先上本文大致目录来瞧一瞧：

![img](https://pic1.zhimg.com/80/v2-909173e4d208545b5a7ba5883b89bec4_720w.webp)

#### 1. Carla行为规划简介

Carla里的behavior planning大执分为全局路线规划、行为规划、轨迹规划与底层控制四大部分。automatic_control.py做的事其实很简单，随机在地图上生成一辆小汽车，随机给个目的地，让汽车自己在限速内安全、丝滑地开到目的地。

#### 2. Behavior Agent初始化

在automatic_control.py里，最重要的不是隶属于Actor class里的vehicle, 而是BehaviorAgent class.它就像vehicle的大脑，一切指令由它下达，vehicle只管按照这个指令去行走。 这个class本身并不是像traffic_manager那样由C++封装好的，它位于PythonAPI/carla/agent中，纯粹由python构成。为了方便大家阅读和测试代码，我把agent相关的代码直接copy到了我的repo里。Agent构建这一块代码**只会在最开始被呼叫，不存在于while循环中。**

#### 2.1 构建Behavrior Agent Class

BehaviorAgent 在构建时需要两个输入，一个是属于Actor class的vehicle, 另外一个就是车辆驾驶风格（string type).

```text
# automatic_control_revised.py
agent = BehaviorAgent(world.player, behavior='normal')
```

如果我们走进这个class里面看（**agents.navigation.behavior_agent**.），可以看到如下**部分代码：**

```text
# behavior_agent.py
class BehaviorAgent(Agent):
  def __init__(self, vehicle, ignore_traffic_light=False, behavior='normal'):
    self.vehicle = vehicle
    if behavior == 'cautious':
        self.behavior = Cautious()

    elif behavior == 'normal':
        self.behavior = Normal()

    elif behavior == 'aggressive':
        self.behavior = Aggressive()
```

如代码所展示，输入不同的驾驶风格字符串，agent的成员behavior会选择不同的class进行初始化。而Cautious(), Normal()和Agressive()这三个成员来源于agents/navigation/types_behavior. 在这里我以Normal()和Agressive（)举例：

```text
class Normal(object):
    """Class for Normal agent."""
    max_speed = 50
    speed_lim_dist = 3
    speed_decrease = 10
    safety_time = 3
    min_proximity_threshold = 10
    braking_distance = 5
    overtake_counter = 0
    tailgate_counter = 0

class Aggressive(object):
    """Class for Aggressive agent."""
    max_speed = 70
    speed_lim_dist = 1
    speed_decrease = 8
    safety_time = 3
    min_proximity_threshold = 8
    braking_distance = 4
    overtake_counter = 0
    tailgate_counter = -1
```

由此可见，types_behavior里主要是定义汽车的限速（max_speed,)、与前车保持的安全时间（safety_time, 基于ttc,后面会详细讲述），与前车的最小安全距离（braking_distance）等安全相关的参数，agressive的车相对normal的车来说速度更快，跟车更紧。

当然了，agent里面还有其他成员变量，后面会一一涉及，在这里暂时就不全部罗列解释了。

#### 2.2 全局路径规划

就像我们人类开车旅游一样，我们在手机上输入一个目的地，算法会自动给算出一个最优全局路线，我们顺着这些路开就行了。carla里也是一样，想让小车跑起来，就得告诉他跑到哪去，怎么个大概跑法。

```text
# automatic_control_revised.py
agent.set_destination(agent.vehicle.get_location(), destination, clean=True)
```

不错，在automatic_control_revised.py里，**只用了一行代码就把全局规划给做完了**，但如果我们往里面挖掘，便会发现内部包含了四个步骤。

#### 2.2.1 **GlobalRoutePlaner初始化**

在set_destination()被呼叫之后，behavior_agent内部会先初始化一个全局路径规划器. 其中GlobalRoutePlannerDAO在我看来是一个比较多余的class,因为它内部就只有两个成员函数，完全可以和GlobalRoutePlanner合并到一起。这两个成员函数一个是carla里的map--wld.get_map(), 另外一个就是sampling_resolution(默认为4.5米，我们知道所谓的路径不过就是一个个node与link连接起来，而sampling resolution变定义了两个node之间的距离是多少）。

```text
# behavior_agent.py, line 151
dao = GlobalRoutePlannerDAO(wld.get_map(), sampling_resolution=self._sampling_resolution)
grp = GlobalRoutePlanner(dao)
grp.setup()
```

#### 2.2.2 Carla Map Topology提取

上面我们提到了GlobalRoutePlannerDAO，它的主要作用就是提取carla 地图的拓朴结构--> self._topology = self._dao.get_topology()。

```text
# inside the grp.setup()   
def setup(self):
      """
      Performs initial server data lookup for detailed topology
      and builds graph representation of the world map.
      """
      self._topology = self._dao.get_topology()
      self._graph, self._id_map, self._road_id_to_edge = self._build_graph()
      self._find_loose_ends()
      self._lane_change_link()
```

这个get_topology()内部的函数在这里不会细究（我会忽略一些细节，否则内容太多），我们只需要它返回的这个topology是怎样的data format:

```text
[{entry:Waypoint1, exit:Waypoint2, path:[Waypoint1_1, Waypoint1_2,......]},
 {entry:Waypoint1, exit:Waypoint2, path:[Waypoint1_1, Waypoint1_2,......]}, 
 ......]
```

它的本质是一个list of dictionary, 每一个 dictionary 都是一段在carla的提前定义的**lane segment.**如下图所示，entry和exit是一段lane segment的开始和结尾点，path是一个list,里面装着entry 和 exit之间的node, 每个node相距4.5米（即你的sampling resolution)。每一个node都是一个**Waypoint class, 它永远位于lane的中央，**同时还可以记录左右lane上平行的waypoint的位置。

![img](https://pic3.zhimg.com/80/v2-1e3b5a031b5698e7b775e9aac9f0638e_720w.webp)图1

#### 2.2.3 Graph 建立

这一步会利用上面提取的topology(list格式）来构建道路的graph(networkx.Diagraph()格式）。其中每一个entry point或者exit point被看作node, path里的多个point组合起来当作edge。这些edge不仅会记录每一个waypoint的xyz坐标，还会记录entry/exit point的yaw angle,从而知道汽车进入这条lane时车头朝向什么位置。

```text
# inside the grp.setup()   
def setup(self):
  """
  graph: networkx.Diagraph(); id_map:a dictionary, the key is the waypoint transform, and value is its
  node id in the graph; road_id_to_edge: eg.{road_id:{section_id:{lane_id:(node_id, node_id)}}}
  """
  # global_route_planner.py
  self._graph, self._id_map, self._road_id_to_edge = self._build_graph()
```

如以上代码所示，build_graph（）会提取成员变量 topology 返回三个变量。self._graph是我们最终需要的nextoworx.Diagraph()对象，id_map是一个辞典，key是每一个entry/exit waypoint的坐标位置，value是它们在graph里的id. road_id_to_edge 则是一个三层字典，最外成是carla里的道路id,中间一层是carla里的在该道路上的section id, 最后一层是carla 里的lane id， 里面对应着graph里的node id. 这个看起来复杂，其实就是为了做一件事：**为了把networkx graph中的每个node都和carla里的信息一一对应，方便后面做信息互换。**举个例子，node id 10对应着carla地图里 第17号road, 第三号section里的 第二条lane的entry point.至于carla地图如何定义road, section和lane的关系, 请看下图

![img](https://pic2.zhimg.com/80/v2-0663b797ce4684a618574f6cd95de15d_720w.webp)图2

那么build_graph()里面是怎么运作的呢？它是不断地遍历topology list, 然后通过add_node与add_edge来一步步构建完整的图谱。

- graph.add_node:

- - new_id: 由当前已存入的node数量决定，初始为0，每存入一个entry/exit point, id便会加一
  - vertex: 该node对应的xyz carla坐标

- graph.add_edge:

- - n1, n2: lane segment的entry point和exit point在**graph**中的id
  - path: 之前提到过在lane segment中位于entry/exit中间的插入点
  - entry_vector, exit_vector: 以向量的形式记录了车辆如果正常驶入和离开这条lane时它的角度是怎样的。

```python3
#注：这里只提取了关键代码，要看完整代码请查看global_route_planner.py line49-106
graph.add_node(new_id, vertex=vertex)
graph.add_edge(
    n1, n2,
    length=len(path) + 1, path=path,
    entry_waypoint=entry_wp, exit_waypoint=exit_wp,
    entry_vector=np.array(
        [entry_carla_vector.x, entry_carla_vector.y, entry_carla_vector.z]),
    exit_vector=np.array(
        [exit_carla_vector.x, exit_carla_vector.y, exit_carla_vector.z]),
    net_vector=vector(entry_wp.transform.location, exit_wp.transform.location),
    intersection=intersection, type=RoadOption.LANEFOLLOW)
```

看到这里细心的小伙伴会注意到一个问题——我们虽然根据一段段lane建立起了graph, 但是这个graph有缺陷阿: 它只会顺着一个方向建立edge. 拿图2举例子，section A里的lane2会和section B里的lane2建立edge, 但是并不会和section A里的lane1建立edge, 这就会导致一个严重的问题：**全局规划出来的路线只能顺着一条道开到黑，无法进行变道甚至转弯的操作。**所以carla在建立了初步的graph之后，还加了以下代码：

```text
self._graph.add_edge(
                self._id_map[segment['entryxyz']], next_segment[0], entry_waypoint=waypoint,
                exit_waypoint=next_waypoint, intersection=False, exit_vector=None,
                path=[], length=0, type=next_road_option, change_waypoint=next_waypoint)
```

这句代码要做的事情很简单，如果已存在的node里它的左边或右边有可驾驶的lane(node本质是waypoint class, waypoint里会存这个信息），那么读取它左边或右边lane里离自己最近的那个node, 与自己建立起edge. 这样一来，整个graph既有了纵向的链接，又有了横向的链接。

#### 2.2.4 全局导航路线生成

到了这一步，我们终于建立了完整的graph, 从任意一个地方出发，我们都能找到一些列的链接和节点，从而到达目的地。完成这个的函数是紧跟着前面提到的grp.setup()函数之后，给定初始点和目的地，自动产生路线（用的是networkx 里打包的的A* 算法，这里就不细说了）。其中返回的route变量是一个装满tuple的list. tuple里含有两个东西（`waypoint, road_option)`

```text
self._grp.setup()
route = self._grp.trace_route(
        start_waypoint.transform.location,
        end_waypoint.transform.location)
```

之前说过，networkx构建的graph的node是entry/exit point, edge是它们之间的插入点，每个相隔4.5米，总而言之，无论是node还是edge, 本质上都是一系列的Waypoint。所以tuple里的waypoint很好理解，就是让汽车朝着这些点一个个开，最后就能开到目的地。road_option是一个enum class, 主要用来表示这个waypoint是不是涉及到变道，为后面的行为规划提供额外的信息。

#### 2.3 局部路线规划器初始化

全局路线规划好以后，agent会初始化局部路线规划器，方便后面的轨迹规划（carla里的轨迹规划严格意义上讲算不上真的轨迹规划，后面会详细讲到）。

```text
# inside set_destination() function in behavior_agent.py
self._local_planner.set_global_plan(route_trace, clean)
```

这个set_global_plan()里面做了两件事： 第一件事情就是把上面global planner产生的route全部赋值给waypoints_queue（deque类型数据，carla设置的默认缓存长度为1000）。第二件事情，就是把 buffer_size数量（默认为5）的elements(tuple of waypoint and roadoption) 从waypoints_queue里弹出，存到短期缓存waypoint_buffer里（也是deque类型数据）。那么它为什么要用deque 来存waypoint, 又为什么设置两个长短不一的导航点缓存呢？

其实就如同我们人类开车看导航一样，我们不会从当前位置一直看到几千米外，我们一般只关注最多几秒后的路线。同样，waypoint_buffer里存的导航点是carla里短期跟随的路线，它每经过一个点就会把历史数据扔掉，而这种操作用deque/queue数据类型效率最高（FIFO）。当waypoint_buffer里的点都被扔出去变空之后，waypoints_queue就会再次弹出自己的五个点，送给waypoint_buffer。如此循环，随着离目的地越来越近，waypoints_queue会被一点点“掏空”，最终到达目的地时全部清空。

```text
    def set_global_plan(self, current_plan):
        """
        Resets the waypoint queue and buffer to match the new plan. Also
        sets the global_plan flag to avoid creating more waypoints
        :param current_plan: list of (carla.Waypoint, RoadOption)
        :return:
        """

        # Reset the queue
        self._waypoints_queue.clear()
        for elem in current_plan:
            self._waypoints_queue.append(elem)
        self._target_road_option = RoadOption.LANEFOLLOW

        # and the buffer
        self._waypoint_buffer.clear()
        for _ in range(self._buffer_size):
            if self._waypoints_queue:
                self._waypoint_buffer.append(
                    self._waypoints_queue.popleft())
            else:
                break

        self._global_plan = True
```

#### 下集预告

由于要讲的东西很多，而且我想尽量把重要的细节也说出来，所以剩下的章节将会在下一篇文章里叙述完，大概一周后会更新，请各位敬请期待！

### （七）一文解析CARLA的行为规划（下）

在上一章，我们讲解了CARLA自带的官方示例automatic_control.py中的前半部分，主要涉及到了CARLA里是如何给汽车设置目的地产生全局的路线规划，以及CARLA里Waypoint的一些基础特征，忘记了的朋友猛戳下方链接回顾下：

[史上最全Carla教程 |（六）一文解析CARLA的行为规划（上）131 赞同 · 31 评论文章![img](https://pic1.zhimg.com/v2-40ff61e9d5ce56f976025e384e959264_ipico.jpg)](https://zhuanlan.zhihu.com/p/355420522)

在上一章，我们完成了仿真初始化，BehaviorAgent初始化，接下来就是runtime的讲解了！从最外层看来，runtime部分的代码十分简单明了（来源：

[https://github.com/DerrickXuNu/Learn-Carla/blob/main/python/automatic_control_revised.pygithub.com/DerrickXuNu/Learn-Carla/blob/main/python/automatic_control_revised.py](https://link.zhihu.com/?target=https%3A//github.com/DerrickXuNu/Learn-Carla/blob/main/python/automatic_control_revised.py)

）：

```text
while True：
            world.tick()
            agent.update_information(vehicle) 
            # top view
            spectator = world.get_spectator()
            transform = vehicle.get_transform()
            spectator.set_transform(carla.Transform(transform.location + carla.Location(z=40),
                                                    carla.Rotation(pitch=-90)))

            control = agent.run_step(debug=True)
            vehicle.apply_control(control)
```

这个 while True的循环里要做的事一共有四件：1）world.tick()->仿真世界运行一个步长。2）agent.update_information(vehicle): BehaviorAgent更新汽车的实时信息，方便更新行为规划。3）control = agent.run_step(debug=True) -> 最核心的一步，更新BehaviorAgent的信息之后，agent运行新的一步规划，并产生相应的控制命令。4) vehicle.apply_control(control) -> 汽车执行产生的控制命令，在仿真世界运行。

接下来，让我们先讲讲Agent信息实时更新部分。

#### 3. Agent信息实时更新

#### 3.1. 速度信息更新

下方代码是agent.update_information()的第一部分，如代码所示，这第一部分首先更新汽车的速度信息，然后通过world.player.get_speed_limit() 找到汽车的速度上限（这个值一般是30），self._local_planner.set_speed(self.speed_limit) 是用来设置轨迹规划的目标速度，这个速度就是速度上限本身，然后self.direction是你的agent里面waypoint_buffer（忘记这个是什么的朋友可以看一下前一张）里面最近的一个waypoint的road_option, 用于告诉用户这个点是否涉及到变道。

```text
# behavior_agent.py update_information function
self.speed = get_speed(self.vehicle)
self.speed_limit = world.player.get_speed_limit()
self._local_planner.set_speed(self.speed_limit)
self.direction = self._local_planner.target_road_option
```

#### 3.2. Incoming waypoint更新

小伙伴们看到upda_information()这一部分代码时一定很纳闷，这个incoming_waypoint和look_ahead_steps是做什么的呢？ 其实很简单，假设look_ahead_steps是3，它就是拿取你waypoint_queue里面第三个点，这个点代表着你在接下来很短的时间将会到达的地方。它在后面要使用这个incoming_waypoint和incoming_direction来检测是否即将到达交叉路口。

```text
self.look_ahead_steps = int((self.speed_limit) / 10)
self.incoming_waypoint, self.incoming_direction = self._local_planner.waypoints_queue[look_ahead_steps]
if self.incoming_direction is None:
    self.incoming_direction = RoadOption.LANEFOLLOW
```

#### 3.3. 信号灯、指示牌信息更新

最后一部分是交通信号相关的更新，从server直接读取ego vehicle是否处于交通信号灯附近。

```text
self.is_at_traffic_light = world.player.is_at_traffic_light()
if self.ignore_traffic_light:
    self.light_state = "Green"
else:
    # This method also includes stop signs and intersections.
    self.light_state = str(self.vehicle.get_traffic_light_state())
```

#### 4. 万事具备，只欠规划

在前面三个小节里，我们初始化了BehaviorAgent, 制定好了全局路线，并将当前server的重要信息更新给了agent, 现在所有的信息都一应俱全，我们的agent终于开始行为规划了！CARLA中的行为规划可大致分为五步：1）针对交通信号灯的行为规划，2）针对行人的行为规划，3）针对路上其它车辆的行为规划，4）交叉口的行为规划， 5）正常驾驶的行为规划。如下方代码所示，最外层的代码只有两行。

```text
control = agent.run_step(debug=True)
vehicle.apply_control(control)
```

#### 4.1. 正常驾驶的行为规划

虽然在CARLA的官方示例里，会先针对交通信号灯进行决策，在这里我还是先讲一下“正常行驶”的行为规划。此处的正常行驶指的是附近没有任何障碍物，没有交叉口，没有交通灯，你可以理解为一个人在茫茫的大高速上独自驾驶。

正常行驶的代码在 run_step里最后两行，它呼叫了local_planner.run_step()，并以限速作为目标速度来控制longitudinal方向的控制。

```text
control = self._local_planner.run_step(
                target_speed= min(self.behavior.max_speed, self.speed_limit - self.behavior.speed_lim_dist), debug=debug)
```

让我们进到local_planner里面看一下！首先我们可以看到，如果之前没有给目标速度，这里也会默认给限速，所以在**这套官方代码里，你如果不做任何修改，它最终速度永远只能到30.** 接着它会对waypoint_queue做一个长度判断，因为这个deque是用来装长期全局路线的（还记得waypoint_buffer是做短期全局路线的吗？），所有当它等于了0，代表着我们到了目的地了，那么我们就把control command里面转弯、加速设为0，刹车设为1，意思是车要急刹停下结束这一切了。 如果小车还没跑到目的地，那么我们就判断一下装短期全局路线的deque是不是还有目标点存在，如果没有，那就从waypoint_queue里不停pop目标点出来，装到waypoint_buffer里。

```text
# inside local_planner_behavior.py run_step()
def run_step(self, target_speed=None, debug=False):
  if target_speed is not None:
            self._target_speed = target_speed
        else:
            self._target_speed = self._vehicle.get_speed_limit()

        if len(self.waypoints_queue) == 0:
            control = carla.VehicleControl()
            control.steer = 0.0
            control.throttle = 0.0
            control.brake = 1.0
            control.hand_brake = False
            control.manual_gear_shift = False
            return control

        # Buffering the waypoints
        if not self._waypoint_buffer:
            for i in range(self._buffer_size):
                if self.waypoints_queue:
                    self._waypoint_buffer.append(
                        self.waypoints_queue.popleft())
                else:
                    break
```

接着local planner会提取waypoint_buffer里最近的一个点来作为下一个目标点（target_waypoint), 然后根据当前速度来对pid控制器进行参数设置。由于pid控制器本身是一块较大的内容，这里就不详述了，小伙伴只需要知道，pid控制器会根据你自身位置和下一个目标点来产生lateral方向的转弯程度，根据当前速度与目标速度来产生longitudinal方向的下一步踩油门程度，这两个就是返回的control command里面的steering, throttle(踩油门）/brake(刹车）属性。

```text
        # Current vehicle waypoint
        self._current_waypoint = self._map.get_waypoint(self._vehicle.get_location())

        # Target waypoint
        self.target_waypoint, self.target_road_option = self._waypoint_buffer[0]

        if target_speed > 50:
            args_lat = self.args_lat_hw_dict
            args_long = self.args_long_hw_dict
        else:
            args_lat = self.args_lat_city_dict
            args_long = self.args_long_city_dict

        self._pid_controller = VehiclePIDController(self._vehicle,
                                                    args_lateral=args_lat,
                                                    args_longitudinal=args_long)

        control = self._pid_controller.run_step(self._target_speed, self.target_waypoint)
```

产生了相应的control command后，接着我们将根据自身的位置和waypoint_buffer里现存的目标点位置做对比，凡是距离离当前位置太近的目标点，都一律被剔除，这样就保证了汽车不会“吃回头草”。从以上的这些步骤大家可以看出，CARLA里严格意义来说，**是没有轨迹规划的，**因为它直接从一个目标点直线冲向另外一个数米外的目标点，没有任何插值和弧度可言。最后这个local planner会返回control command, 然后小车会applyControl来实现相应的运动。

```text
        # Purge the queue of obsolete waypoints
        vehicle_transform = self._vehicle.get_transform()
        max_index = -1

        for i, (waypoint, _) in enumerate(self._waypoint_buffer):
            if distance_vehicle(
                    waypoint, vehicle_transform) < self._min_distance:
                max_index = i
        if max_index >= 0:
            for i in range(max_index + 1):
                self._waypoint_buffer.popleft()

        if debug:
            draw_waypoints(self._vehicle.get_world(),
                           [self.target_waypoint], 1.0)
        return control
```

#### 4.2. 针对交通信号灯的行为规划

CARLA里对交通信号灯的行为规划十分简单粗暴。

```text
# behavior_agent.py run_step()
if self.traffic_light_manager(ego_vehicle_wp) != 0:
    return self.emergency_stop()
```

如果汽车所在位置处于交叉口，附近的信号灯为红灯（这个信息是update_information那一步拿到的），并且汽车本身并没有设置无视红灯的话，那么它就要做一个急刹，尽快停下来。

```text
    def traffic_light_manager(self, waypoint):
        """
        This method is in charge of behaviors for red lights and stops.
        WARNING: What follows is a proxy to avoid having a car brake after running a yellow light.
        This happens because the car is still under the influence of the semaphore,
        even after passing it. So, the semaphore id is temporarely saved to
        ignore it and go around this issue, until the car is near a new one.
            :param waypoint: current waypoint of the agent
        """

        light_id = self.vehicle.get_traffic_light().id if self.vehicle.get_traffic_light() is not None else -1

        if self.light_state == "Red":
            if not waypoint.is_junction and (self.light_id_to_ignore != light_id or light_id == -1):
                return 1
            elif waypoint.is_junction and light_id != -1:
                self.light_id_to_ignore = light_id
        if self.light_id_to_ignore != light_id:
            self.light_id_to_ignore = -1
        return 0
```

#### 4.3. 针对行人的行为规划

agent会检测自己未来的行驶路径上是否有行人，如果有行人且距离小于一定的阈值，那么它会立刻做一个急刹。

```text
walker_state, walker, w_distance = self.pedestrian_avoid_manager(
            ego_vehicle_loc, ego_vehicle_wp)
if walker_state:
       # Distance is computed from the center of the two cars,
       # we use bounding boxes to calculate the actual distance
       distance = w_distance - max(
       walker.bounding_box.extent.y, walker.bounding_box.extent.x) - max(
       self.vehicle.bounding_box.extent.y, self.vehicle.bounding_box.extent.x)

       # Emergency brake if the car is very close.
       if distance < self.behavior.braking_distance:
            return self.emergency_stop()
```

在 pedestrian_avoid_manager()这个函数里面，我们可以看到以下代码：

```text
walker_list = self._world.get_actors().filter("*walker.pedestrian*")
def dist(w):
    return w.get_location().distance(waypoint.transform.location)

walker_list = [w for w in walker_list if dist(w) < 10]
walker_state, walker, distance = self._bh_is_vehicle_hazard(waypoint, location, walker_list, max(
    self.behavior.min_proximity_threshold, self.speed_limit / 2), up_angle_th=90, low_angle_th=0, lane_offset=-1)
```

walker_list 是从server里直接拿到的和小车距离十米以内的行人， 而_bh_is_vehicle_hazard是一个很重要的函数，它的argument里的参数解释如下：

- waypoint: ego vehicle waypoint
- location: ego vehicle的location, 类型为carla.Location()
- walker_list： 装着附近的人的列单。
- proximity_th：一个阈值， 如果ego vehiclie和某个行人的距离大于这个阈值，那么这个行人就不会影响汽车的规划。
- up_angle_th, low_angle_th：角度阈值。用来判断行人会不会出现在汽车的驾驶路线上。比如汽车在往前走，它正后方一米的行人不应该在它的紧急避范围之内。
- lane_offset： -1, 0 或者1. 它代表着小车下一个waypoint是会向左变道，保持原车道还是向右边道。举个例子，如果车子要向左变道，但是行人在右方车道，那么agent也会判断为安全。

其实看完输入变量的解释，大家应该已经大致明白函数里面做了些什么，受篇幅限制，这里就不走到函数里面看了。这个函数最后返回了walker_state, walker, distance这三个变量，它们分别代表着1）是否有行人可能与小车发生碰撞，2）如果有，返回该行人，3）该行人与车的距离, 如果该距离过小，则立刻紧急刹车。

#### 4.4. 针对其他车辆的行为规划

CARLA里检测是否有其他汽车可能与ego vehicle相撞的函数与行人如出一辙，都是使用了 _bh_is_vehicle_hazard() 这个函数，返回vehicle_state, vehicle, distance三个变量，如下方代码所示：

```text
vehicle_list = self._world.get_actors().filter("*vehicle*")
def dist(v):
  return v.get_location().distance(waypoint.transform.location)

vehicle_list = [v for v in vehicle_list if dist(v) < 45 and v.id != self.vehicle.id]
vehicle_state, vehicle, distance = self._bh_is_vehicle_hazard(
                waypoint, location, vehicle_list, max(
                    self.behavior.min_proximity_threshold, self.speed_limit / 2), up_angle_th=180, lane_offset=-1)           
```

不过不同的是，当ego vehicle检测到有车辆处于自己的驾驶路线上时，它可以做的选择不再是仅仅刹车，还可以选择超车、跟车。

#### 超车模式

满足进入超车模式的条件有如下几个：1）前方障碍汽车离己方较近，挡在了行驶路线上。2）汽车不在交叉路口。 3）ego vehicle速度大于10， 且大于前方障碍汽车。

```text
if vehicle_state and self.direction == RoadOption.LANEFOLLOW and \
        not waypoint.is_junction and self.speed > 10 \
        and self.behavior.overtake_counter == 0 and self.speed > get_speed(vehicle):
    self._overtake(location, waypoint, vehicle_list)
```

self._overtake()是管理具体超车行为的函数，其代码如下所示。这里可以看出，ego vehicle会优先选择从左道超车（因为left turn判断语句放在了right turn前方），而能从左方超车的条件为：1）ego左边的车道是允许从当前车道变道的（left_turn == carla.LaneChange.Left or left_turn == carla.LaneChange.Both) , 并且属于车行道（left_wpt.lane_type == carla.LaneType.Driving）2）左边车道和当前车道是同方向的（waypoint.lane_id * left_wpt.lane_id > 0， 如果<0代表这俩车道是对向的。 3）左边车道上没有离自己太近的汽车（ new_vehicle_state, _, _ = self._bh_is_vehicle_hazard(waypoint, location, vehicle_list, max( self.behavior.min_proximity_threshold, self.speed_limit / 3), up_angle_th=180, lane_offset=-1)）。满足了这三条后，ego就可以从左边超车了！

```text
    def _overtake(self, location, waypoint, vehicle_list):
        """
        This method is in charge of overtaking behaviors.
            :param location: current location of the agent
            :param waypoint: current waypoint of the agent
            :param vehicle_list: list of all the nearby vehicles
        """

        left_turn = waypoint.left_lane_marking.lane_change
        right_turn = waypoint.right_lane_marking.lane_change

        left_wpt = waypoint.get_left_lane()
        right_wpt = waypoint.get_right_lane()

        if (left_turn == carla.LaneChange.Left or left_turn ==
            carla.LaneChange.Both) and waypoint.lane_id * left_wpt.lane_id > 0 and left_wpt.lane_type == carla.LaneType.Driving:
            new_vehicle_state, _, _ = self._bh_is_vehicle_hazard(waypoint, location, vehicle_list, max(
                self.behavior.min_proximity_threshold, self.speed_limit / 3), up_angle_th=180, lane_offset=-1)
            if not new_vehicle_state:
                print("Overtaking to the left!")
                self.behavior.overtake_counter = 200
                self.set_destination(left_wpt.transform.location,
                                     self.end_waypoint.transform.location, clean=True)
        elif right_turn == carla.LaneChange.Right and waypoint.lane_id * right_wpt.lane_id > 0 and right_wpt.lane_type == carla.LaneType.Driving:
            new_vehicle_state, _, _ = self._bh_is_vehicle_hazard(waypoint, location, vehicle_list, max(
                self.behavior.min_proximity_threshold, self.speed_limit / 3), up_angle_th=180, lane_offset=1)
            if not new_vehicle_state:
                print("Overtaking to the right!")
                self.behavior.overtake_counter = 200
                self.set_destination(right_wpt.transform.location,
                                     self.end_waypoint.transform.location, clean=True)
```

CARLA里超车的具体方式很简单粗暴，直接使用set_destination(left_waypoint, self.end_waypoint), 也就是重新规划一下全局路线，由于新的全局路线是从左车道开始，ego自然会变到左车道，完成超车。

#### 跟车模式

当汽车不满足超车模式，离障碍车辆距离未到紧急刹车时，就会进入跟车模式。

```text
# Emergency brake if the car is very close.
if distance < self.behavior.braking_distance:
     return self.emergency_stop()
else:
     control = self.car_following_manager(vehicle, distance)
```

跟车模式的核心要点就一个——和前方车辆保持固定的距离。CARLA里实现方式十分简单，如下方代码所示，如果ego和前方汽车的ttc太小，ego就会把速度控制到比前方汽车速度还小，如果ttc适中，那就保持和前车一样的速度，如果ttc较大了，那就按原定的目标速度行驶。

```text
      def car_following_manager(self, vehicle, distance, debug=False):
        """
        Module in charge of car-following behaviors when there's
        someone in front of us.
            :param vehicle: car to follow
            :param distance: distance from vehicle
            :param debug: boolean for debugging
            :return control: carla.VehicleControl
        """

        vehicle_speed = get_speed(vehicle)
        delta_v = max(1, (self.speed - vehicle_speed) / 3.6)
        ttc = distance / delta_v if delta_v != 0 else distance / np.nextafter(0., 1.)

        # Under safety time distance, slow down.
        if self.behavior.safety_time > ttc > 0.0:
            control = self._local_planner.run_step(
                target_speed=min(positive(vehicle_speed - self.behavior.speed_decrease),
                                 min(self.behavior.max_speed, self.speed_limit - self.behavior.speed_lim_dist)),
                debug=debug)
        # Actual safety distance area, try to follow the speed of the vehicle in front.
        elif 2 * self.behavior.safety_time > ttc >= self.behavior.safety_time:
            control = self._local_planner.run_step(
                target_speed=min(max(self.min_speed, vehicle_speed),
                                 min(self.behavior.max_speed, self.speed_limit - self.behavior.speed_lim_dist)),
                debug=debug)
        # Normal behavior.
        else:
            control = self._local_planner.run_step(
                target_speed=min(self.behavior.max_speed, self.speed_limit - self.behavior.speed_lim_dist), debug=debug)

        return control
```

#### 5. 小结

我们这期的CARLA课堂就到这里了！由于内容较多，强烈建议读者结合着源代码来看这一系列的课程：

[DerrickXuNu/Learn-Carlagithub.com/DerrickXuNu/Learn-Carla![img](https://pic1.zhimg.com/v2-02a8bf745945997d7b8075803555e704_180x120.jpg)](https://link.zhihu.com/?target=https%3A//github.com/DerrickXuNu/Learn-Carla)

读者们在阅读完这篇文章后一定会觉得，CARLA里的**行为规划实在是粗糙至极，甚至没有一个像样的轨迹规划！**

不要着急，小飞马上就要开放一个基于CARLA-SUMO联合仿真的**多车协同代码框架（纯Python)！**在这套框架里，你可以看到一个完整的pipeline在carla里运作，包括**一键场景构建**，**感知**（相机加Lidar)**、定位**(gps+imu)**、行为规划、轨迹规划、控制、以及数个多车协同应用**，支持用户轻松地将自己的算法替换框架中自带的任何算法，同时不会影响其他任何模块，这样用户可以轻易地看到自己开发的算法在车辆行驶整体表现的影响。这里放一个预览图，是CAV之间通过协同规划做到了高速上的协同变道。我将在下一期介绍这个开源框架，大家敬请期待哦！

![动图封面](https://pic4.zhimg.com/v2-726aced29abc8b5be07db93d791ec9f7_b.jpg)
