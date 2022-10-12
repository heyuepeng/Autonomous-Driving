### Client-Server的交互形式

Carla主要分为Server与Client两个模块，Server端用来建立这个仿真世界，而Client端则是由用户控制，用来调整、变化这个仿真世界。

1.Server: Server端负责任何与仿真本身相关的事情：从3D渲染汽车、街道、建筑，传感器模型的构建，到物理计算等等。它就像一个**造物主，** 将整个世界建造出来，并且根据Client 的外来指令更新这个世界。

2.Client: 如果server构造了整个世界，那么这个世界不同时刻到底该如何运转（比如天气是什么样，有多少辆车在跑，速度是多少）则是由Client端控制的。用户通过书写Python脚本（最新版本C++ 也可以）来向Server端输送指令指导世界的变化，Server根据用户的指令去执行。（可以理解为Client端耍耍嘴皮子下个指令，咱们的造物主亲力亲为去执行这些指令。 另外，Client端也可以接受Server端的信息，譬如某个照相机拍到的路面图片。）

### 多个client同时开多辆车

**单个client开单辆车**

```python
    # First of all, we need to create the client that will send the requests, assume port is 2000
    client = carla.Client('localhost', 2000)
    client.set_timeout(5.0)
    # Retrieve the world that is currently running
    world = client.get_world()
    # 拿到这个世界所有物体的蓝图
    blueprint_library = world.get_blueprint_library()
    # 从浩瀚如海的蓝图中找到bmw的蓝图
	ego_vehicle_bp = blueprint_library.find('vehicle.bmw.grandtourer')
    ego_vehicle_bp.set_attribute('color', '0, 0, 0')
    # 找到所有可以作为初始点的位置并随机选择一个
	transform = random.choice(world.get_map().get_spawn_points())
	# 在这个位置生成汽车
	ego_vehicle = world.spawn_actor(ego_vehicle_bp, transform)
    # 把它设置成自动驾驶模式，道路的其它车辆可以设置为自动驾驶模式
	ego_vehicle.set_autopilot(True)
```

给车子装上sensor，比如相机与激光雷达。与汽车类似，我们先创建蓝图，再定义位置，然后再选择我们想要的汽车安装上去。不过，这里的位置都是相对汽车中心点的位置（以米计量）

```
	# add a camera
	camera_bp = blueprint_library.find('sensor.camera.rgb')
	camera_transform = carla.Transform(carla.Location(x=1.5, z=2.4))
	camera = world.spawn_actor(camera_bp, camera_transform, attach_to=ego_vehicle)
```

```
    # set the callback function
    camera.listen(lambda image: image.save_to_disk(os.path.join(output_path, '%06d.png' % image.frame)))
```

```
	# we also add a lidar on it
	lidar_bp = blueprint_library.find('sensor.lidar.ray_cast')
    lidar_bp.set_attribute('channels', str(32))
    lidar_bp.set_attribute('points_per_second', str(90000))
    lidar_bp.set_attribute('rotation_frequency', str(40))
    lidar_bp.set_attribute('range', str(20))
    lidar_location = carla.Location(0, 0, 2)
    lidar_rotation = carla.Rotation(0, 0, 0)
    lidar_transform = carla.Transform(lidar_location, lidar_rotation)
    lidar = world.spawn_actor(lidar_bp, lidar_transform, attach_to=ego_vehicle)
    lidar.listen(
            lambda point_cloud: point_cloud.save_to_disk(os.path.join(output_path, '%06d.ply' % 			 	point_cloud.frame)))
	
```

**单个client开多辆车**

```
#多生成几辆ego_vehicle就行了，比如：
ego_vehicle_bmw_bp = blueprint_library.find('vehicle.bmw.grandtourer')
ego_vehicle_audi_bp = blueprint_library.find('vehicle.audi.a2')
ego_vehicle_jeep_bp = blueprint_library.find('vehicle.jeep.wrangler_rubicon')
transform1 = random.choice(world.get_map().get_spawn_points())
transform2 = random.choice(world.get_map().get_spawn_points())
transform3 = random.choice(world.get_map().get_spawn_points())

ego_vehicle_bmw = world.spawn_actor(ego_vehicle_bmw_bp, transform1)
ego_vehicle_audi = world.spawn_actor(ego_vehicle_audi_bp, transform2)
ego_vehicle_jeep = world.spawn_actor(ego_vehicle_jeep_bp, transform3)
#车辆多的话，也可以设置一个ego_vehicle列表ego_vehicles_list[i]
```

**多个client**

CARLA其实是一个client-server的模式，默认是异步模式。**仿真server默认为异步模式，它会尽可能快地进行仿真，根本不管client是否跟上了它的步伐**。目前carla只支持**单client同步**，也就是说，如果你有N个python scripts(N个client), **只能在其中一个client里设置同步模式, 而其他client只能异步模式.** 这些处于异步模式的客户首先通过world.wait_for_tick() 等待server 更新*，*一旦更新了它们会立刻通过*world.*on_tick 里的callback 来提取这个更新的wordsnapshot里面的信息

```text
# Wait for the next tick and retrieve the snapshot of the tick.
world_snapshot = world.wait_for_tick()

# Register a callback to get called every time we receive a new snapshot.
world.on_tick(callback)
```

**一般设置同步模式的client主要是用来做数据储存与采集的，异步模式的client收集的照片是不连续的，存在掉帧现象**。这是因为Carla Simulation Server 与 Client默认为异步进行，也就是说server并不会等client处理完手头的事再进行模拟，它会以最快的速度进行模拟，这就导致你的client如果处理过慢，还没来得及储存第K个data, simulation早就进行到了第K+10步，也就会产生掉帧了。

**因此，我觉得只需要一个client，设置这个client为同步模式，然后这个client生成多辆车，每辆车都是独立驾驶的，就能满足联邦学习的需要了。**（这个client里的多辆车，可以看作联邦学习论文里的多个client）

### 车辆的input的数据类型

`vehicle.apply_control(self, control)`     为carla.Vehicle类的方法：在下一个刻度上应用control对象，这个control对象包含驾驶参数，如油门、转向或换档。

control是carla.VehicleControl类的对象，使用典型的驾驶控制装置管理车辆的基本运动。

```text
# control对象的实例变量，即驾驶参数：
throttle (float)  #油门
# A scalar value to control the vehicle throttle [0.0, 1.0]. Default is 0.0.  
steer (float)   #转向
# A scalar value to control the vehicle steering [-1.0, 1.0]. Default is 0.0.
brake (float)   #刹车
# A scalar value to control the vehicle brake [0.0, 1.0]. Default is 0.0.
hand_brake (bool)   #确定是否使用手刹
# Determines whether hand brake will be used. Default is False.
reverse (bool)   #确定车辆是否向后移动
# Determines whether the vehicle will move backwards. Default is False.
manual_gear_shift (bool)   #确定是否手动换档
# Determines whether the vehicle will be controlled by changing gears manually. Default is False.
gear (int)  #档位
# States which gear is the vehicle running on.
```

例子如下：

    ego_vehicle = world.spawn_actor(ego_vehicle_bp, transform)
    
    control = carla.VehicleControl()
    control.steer = 0.0
    control.throttle = 0.5
    control.brake = 0.0
    control.hand_brake = False
    control.manual_gear_shift = False
    
    # ego_vehicle在下一个刻度开始应用control对象
    ego_vehicle.apply_control(control)

### 用程序作为output来操控车辆的方法

感觉只要通过ego_vehicle上的传感器：比如相机、激光雷达等收集到的信息，作为模型的输入，然后模型实时输出对应的驾驶参数，比如：throttle、steer、brake等（可以改成carla要求的格式），将这些驾驶参数构造成一个control对象传给ego_vehicle就行。

### **Traffic Manager**

Carla专门构造了Traffic Manager这个模块来模拟类似现实世界负责的交通环境。通过这个模块，用户可以定义N多不同车型、不同行为模式、不同速度的车辆在路上与你的自动驾驶汽车（Ego-Vehicle）一起行驶。它最大的作用就是可以帮用户**集体管理一个车群**，比如说设定所有的奔驰和林肯车辆的限速和最小安全车距。有了它，用户就**可以轻易地定义某些群体的共性行为**。**在同步模式下，设置AutoPilot的车辆必须依附于设置为同步模式的traffic manager才能跑起来**。

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

在carla里，你可以构建多个traffic manager, 大概有以下几个原则：

- 你可以创立多个tm, 这几个tm可以在一个client里，也可以不在一个client里创造
- 只有一个tm可以设置为同步模式