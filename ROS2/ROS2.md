# 工作空间

## 功能包

工作空间为 ROS2 的基本单位，一个工作空间下可以有多个功能包，一个功能包内对应多个节点。

### 创建

创建基础功能包和首个节点：

``` bash
# --node-name 可以省略，不创建节点
ros2 pkg create 包名 --build-type 构建类型 --dependencies 依赖列表 --node-name 节点（可执行程序）名称
```

- --build-type：	是指功能包的构建类型，有cmake、ament_cmake、ament_python三种类型可选；
- --dependencies： 所依赖的功能包列表；
- --node-name：     可执行程序的名称，会自动生成对应的源文件并生成配置文件。

---

创建其他节点：

在`pkg_multi_node/src`目录下，创建 节点的`.cpp`文件；

在`pkg_multi_node_cpp/CMakeLists.txt`，在文件末尾（`ament_package()`之前）添加为每个节点单独配置编译 / 安装规则：

``` cmake
# 1. 编译talker节点：生成名为talker的可执行文件
add_executable(talker src/talker.cpp)
# 链接依赖（rclcpp + std_msgs）
ament_target_dependencies(talker rclcpp std_msgs)


# 2. 编译listener节点：生成名为listener的可执行文件
add_executable(listener src/listener.cpp)
# 链接依赖
ament_target_dependencies(listener rclcpp std_msgs)

# 3. 安装两个可执行文件（放到lib/${PROJECT_NAME}目录）
install(TARGETS
  talker   # 第一个节点
  listener # 第二个节点
  DESTINATION lib/${PROJECT_NAME}
)
```



### 编译

``` bash
colcon build
```

```
colcon build --packages-select 功能包列表
```

前者会构建工作空间下的所有功能包，后者可以构建指定功能包。

### 查找

在`ros2 pkg`命令下包含了多个查询功能包相关信息的参数。

```bash
ros2 pkg executables [包名]  # 输出所有功能包或指定功能包下的节点。
ros2 pkg list 				# 列出所有功能包
ros2 pkg prefix 包名 		   # 列出功能包路径
ros2 pkg xml 			    # 输出功能包的package.xml内容
```

### 运行

执行命令语法如下：

```bash
ros2 run 功能包 可执行程序 参数
```

> ***小提示：***
>
> 可以通过`命令 -h`或`命令 --help`来获取命令的帮助文档。