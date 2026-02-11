cmake_minimum_required(VERSION 3.5)
# 指定运行此构建所需的最低 CMake 版本（语法兼容性保证）。
# 如果当前 CMake 版本低于 3.5，配置阶段会失败并提示错误。

project(simple_line_follower)
# 定义项目名称为 `simple_line_follower`。
# `project()` 会设置诸如 `PROJECT_NAME`、`PROJECT_SOURCE_DIR` 等变量，
# 并根据系统自动启用常见语言（C、CXX 等），除非显式指定 `LANGUAGES`。

# -----------------------------------------------------------------------------
# 文件说明（中文注释）
# 本文件用于构建 `simple_line_follower` 包（C++ 实现），主要包含：
# - 设置 C/C++ 标准及编译选项（可被外部覆盖）
# - 查找并声明 ROS 2 运行时/编译时依赖（find_package）
# - 构建库（add_library）和可执行文件（add_executable）
# - 定义安装规则（install）和测试配置（BUILD_TESTING 条件）
# -----------------------------------------------------------------------------

# -----------------------------------------------------------------------------
# 设置默认的 C / C++ 标准（可被用户通过 -D 参数覆盖）
# -----------------------------------------------------------------------------
if(NOT CMAKE_C_STANDARD)
  # 检查变量是否已由外部（或父级）设置，若未设置则赋默认值
  set(CMAKE_C_STANDARD 99)
  # 这里意味着：如果没有显式指定 C 标准，则使用 C99。
endif()

if(NOT CMAKE_CXX_STANDARD)
  # 同上：如果没有显式指定 C++ 标准，则使用 C++14 作为默认
  set(CMAKE_CXX_STANDARD 14)
endif()

# 如果使用 GNU (g++) 或 Clang 编译器，则添加常用警告标志
# 这些警告有助于在开发阶段发现潜在问题
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  # add_compile_options 会将选项添加到全局编译选项列表
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# -----------------------------------------------------------------------------
# 查找并声明包依赖（find_package）
# -----------------------------------------------------------------------------
# 说明：find_package(<name> REQUIRED) 会尝试找到 <name> 的配置（Config）文件
# 或 `Find<name>.cmake` 模块，成功后会设置一些变量（例如 <name>_FOUND）
# 并且通常会提供可直接链接的导入目标（例如 `rclcpp::rclcpp`）。
find_package(rclcpp REQUIRED)            # 查找 rclcpp（ROS 2 C++ 客户端库）
find_package(geometry_msgs REQUIRED)     # 查找 geometry_msgs（消息类型包）
find_package(tf2_ros REQUIRED)           # 查找 tf2_ros（TF2 框架）
find_package(tf2_geometry_msgs REQUIRED) # 查找 tf2_geometry_msgs（方便 geometry_msgs 与 tf2 互换）
find_package(nav_msgs REQUIRED)          # 查找 nav_msgs（导航相关消息）

# 将本包的 include/ 目录添加到全局头文件搜索路径。
# 注意：更现代/更细粒度的做法是对目标使用 target_include_directories()
# （这样可以将 include 作用域限制到某个目标）。此处为简单用法。
include_directories(include)

# 把上面的依赖列成一个变量，便于后面统一传递给 ament_target_dependencies
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
# -----------------------------------------------------------------------------
# 构建库：把实现封装成库（SHARED 表示动态链接库）。
# 这样有助于：单元测试（测试库中的函数）、复用代码，以及让可执行文件更轻量。
# -----------------------------------------------------------------------------
add_library(line_follower_library SHARED
  src/line_follower.cpp
)
# add_library(<name> <type> <sources>)：创建一个 CMake 目标（目标名为 line_follower_library），
# - SHARED 表示生成共享库（动态库），也可以使用 STATIC 或 OBJECT
# - 这里指定了源文件 src/line_follower.cpp，CMake 会为该目标生成编译与链接规则

# 把上面列出的 ROS 依赖与该库关联（确保头文件和链接库可用）
# ament_target_dependencies 会：
# - 将依赖的 include 路径添加到目标
# - 将依赖的库链接到目标
# - 传播必要的接口属性（例如编译特性）
# 使用这一接口可以避免手动管理 ${..._INCLUDE_DIRS}/${..._LIBRARIES} 等变量
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
# target_link_libraries(<exe> <lib>)：把库链接到可执行目标上。
# 注意链接顺序在某些平台编译器/链接器中会影响符号解析。
target_link_libraries(line_follower_node line_follower_library)

# 为可执行目标声明依赖（确保在编译时可以找到相应的头文件和链接库）
# ament_target_dependencies(<target> <deps...>) 是 ament 提供的便捷接口，等同于
# 对目标做 target_link_libraries + 添加 include 路径等操作，但会处理好接口传播。
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

# 下面这行是传统/兼容写法：使用 find_package 设置的变量来链接库（不推荐）
# 推荐改为直接使用导入 target（例如 rclcpp::rclcpp）或仅使用 ament_target_dependencies
# 保留它以兼容某些老旧 CMake/平台配置；如无必要可删除。
target_link_libraries(line_follower_node ${rclcpp_LIBRARIES})

# 指定库的私有头文件搜索路径（供编译器在构建该库时使用）
# target_include_directories(<target> PRIVATE <dirs>)：PRIVATE 表示只有本目标自身编译时使用这些 include。
# 在现代 CMake 中，推荐使用导入 target 的 INTERFACE_INCLUDE_DIRECTORIES 而不是手动传变量。
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
# install(TARGETS ...) 会在安装时把目标复制到指定目录，通常是 lib/<pkg>/
# 这使得安装后的可执行文件可以通过 `ros2 run <pkg> <exec>` 被运行。

install(DIRECTORY
  config
  DESTINATION share/${PROJECT_NAME}/
)
# 安装 config 目录到 share/<pkg>/config，运行时或 launch 文件可以从这里读取参数文件。

install(
  DIRECTORY include/
  DESTINATION include
)
# 把头文件安装到全局 include 目录（系统级 include），供其他包在编译时引用。

# -----------------------------------------------------------------------------
# 测试相关：当启用 BUILD_TESTING 时，添加测试依赖并编译 test 子目录
# -----------------------------------------------------------------------------
if(BUILD_TESTING)
  # ament 的自动 lint 工具（用于静态检查）
  find_package(ament_lint_auto REQUIRED)
  # ament 对 gtest 的封装，方便添加 gtest 单元测试
  find_package(ament_cmake_gtest REQUIRED)
  # 测试中可能也需要 rclcpp（例如创建节点进行集成测试）
  find_package(rclcpp REQUIRED)

  # 自动查找并添加测试时所需的 lint 依赖
  ament_lint_auto_find_test_dependencies()

  # 把 test 子目录的 CMakeLists.txt 包含进来以编译测试
  add_subdirectory(test)  
endif()

# 最后调用 ament_package() 导出包的元数据（让 colcon/ament 知道这是一个 ament 包）
# ament_package() 会生成一些包信息和 CMake 配置，方便被其它包 find_package()
ament_package()

# github学习测试(1)
# （原作者的测试注释，保留作为记录，不影响构建）