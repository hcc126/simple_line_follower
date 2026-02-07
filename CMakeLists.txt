cmake_minimum_required(VERSION 3.5)
project(simple_line_follower)

# -----------------------------------------------------------------------------
# 文件说明（中文注释）
# 这个 CMakeLists.txt 用于构建 `simple_line_follower` 包（C++ 实现），包含：
# - 设置 C/C++ 标准和编译选项
# - 查找并声明 ROS 2 运行时/编译时依赖
# - 编译库和可执行文件（节点），并安装到合适位置作为运行入口
# - 安装配置文件（放到 share/<pkg>），并配置测试相关内容
# -----------------------------------------------------------------------------

# Default to C99
if(NOT CMAKE_C_STANDARD)
  # 如果没有指定 C 标准，默认使用 C99
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++14
if(NOT CMAKE_CXX_STANDARD)
  # 如果没有指定 C++ 标准，默认使用 C++14
  set(CMAKE_CXX_STANDARD 14)
endif()

# 如果使用 GNU 或 Clang 编译器，添加常用警告选项（有助于发现潜在问题）
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# -----------------------------------------------------------------------------
# 查找依赖（find_package）
# 这些是编译/链接时以及运行时需要的 ROS 2 包
# -----------------------------------------------------------------------------
find_package(rclcpp REQUIRED)            # ROS 2 C++ 客户端库
find_package(geometry_msgs REQUIRED)     # 消息类型
find_package(tf2_ros REQUIRED)           # TF2 框架
find_package(tf2_geometry_msgs REQUIRED) # 帮助在 TF 与 geometry_msgs 之间转换
find_package(nav_msgs REQUIRED)          # 导航相关消息

# 将 include/ 添加到头文件搜索路径（用于 #include "simple_line_follower/..."）
include_directories(include)

# 一个方便的变量，列出上面声明的依赖（用于 ament_target_dependencies）
set(dependencies
  rclcpp
  geometry_msgs
  tf2_ros
  tf2_geometry_msgs
  nav_msgs
)

# -----------------------------------------------------------------------------
# 构建库（library）
# 将实现封装成库有助于单元测试（测试库中的函数），以及复用代码
# -----------------------------------------------------------------------------
# 把 line_follower.cpp 构建成一个共享库（方便测试和链接到可执行文件）
add_library(line_follower_library SHARED
  src/line_follower.cpp
)
# 把上面列出的 ROS 依赖与该库关联（确保头文件和链接库可用）
ament_target_dependencies(line_follower_library 
  ${dependencies}
)

# 同样为 naive 实现创建一个库（作为对比和单元测试对象）
add_library(naive_line_follower_library SHARED
  src/naive_line_follower.cpp
)
ament_target_dependencies(naive_line_follower_library 
  ${dependencies}
)

# -----------------------------------------------------------------------------
# 可执行文件（节点）
# add_executable + install(TARGETS ... DESTINATION lib/${PROJECT_NAME}) 是 C++ 包的 "入口"
# 可执行目标会被安装到 lib/<pkg>，之后可以通过 launch 文件或 `ros2 run <pkg> <exe>` 启动
# -----------------------------------------------------------------------------
add_executable(naive_line_follower_node src/naive_line_follower.cpp)
# 将 naive node 链接到对应的库
target_link_libraries(naive_line_follower_node naive_line_follower_library)

add_executable(line_follower_node src/line_follower.cpp)
# 将主节点链接到其库（这样逻辑复用并便于测试）
target_link_libraries(line_follower_node line_follower_library)

# 为可执行目标声明依赖（确保在编译时可以找到相应的头文件和链接库）
ament_target_dependencies(naive_line_follower_node
  rclcpp
  geometry_msgs
  tf2_ros
  tf2_geometry_msgs
  nav_msgs
)

ament_target_dependencies(line_follower_node
  rclcpp
  geometry_msgs
  tf2_ros
  tf2_geometry_msgs
  nav_msgs
)

# 有时需要显式链接到 rclcpp 的库（取决于平台和 CMake 配置）
target_link_libraries(line_follower_node ${rclcpp_LIBRARIES})

# 指定库的私有头文件搜索路径（供编译器在构建该库时使用）
target_include_directories(line_follower_library PRIVATE
  ${rclcpp_INCLUDE_DIRS}
  ${geometry_msgs_INCLUDE_DIRS}
  ${tf2_ros_INCLUDE_DIRS}
  ${tf2_geometry_msgs_INCLUDE_DIRS}
  ${nav_msgs_INCLUDE_DIRS}
)

# -----------------------------------------------------------------------------
# 安装规则（使其它包或运行时能找到构建产物与配置）
# - 可执行文件通过 install(TARGETS ...) 安装到 lib/${PROJECT_NAME}（成为运行入口）
# - config 放到 share/${PROJECT_NAME}/config，供运行时加载参数
# - include/ 安装到全局 include 目录，供其它包引用头文件
# -----------------------------------------------------------------------------
install(TARGETS
  naive_line_follower_node
  line_follower_node
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY
  config
  DESTINATION share/${PROJECT_NAME}/
)

install(
  DIRECTORY include/
  DESTINATION include
)

# -----------------------------------------------------------------------------
# 测试相关：当启用 BUILD_TESTING 时，添加测试依赖并编译 test 子目录
# -----------------------------------------------------------------------------
if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  find_package(ament_cmake_gtest REQUIRED)
  find_package(rclcpp REQUIRED)

  ament_lint_auto_find_test_dependencies()

  add_subdirectory(test)  
endif()

# 最后调用 ament_package() 导出包的元数据（让 colcon/ament 知道这是一个 ament 包）
ament_package()
