---
layout: post
mathjax: true
comments: true
title:  "Working with ROS2 packages: Part 2 "
author: John Z. Li
date:   2021-02-27 19:00:18 +0800
categories: robotics
tags: path-following
---
Recall that ROS in itself is nothing but a collection of libraries that are arranged
to work together. The so called client libraries are provided to users to develop
their own ROS packages. This is ROS's way to call language bindings to their core
libraries. Two language bindings, that of C++ and Python are officially supported,
although there are community contributed bindings to other languages.

The client libraries for C++ is `rclcpp`, that is short for "ROS client libraries
for CPP". And the client libraries for Python is called `rclpy` (ROS client libraries
for Python). Both of which are built on top of `RCL`, that is "ROS core libraries",
which implements features that are supposed to be language-agnostic. The RCL exposes
its interfaces via C ABI, and client libraries are actually a thin wrapper of the
RCL.

## Exchange data between ROS nodes.
### Basic data types
In order to different ROS nodes can communicate with each other, they have to
agree on the message formats they use. Since ROS2 uses DDS/RTPS to pass information
among nodes. (DDS is short for "Data Distribution Service" and RTPS, Real Time Publish Subscribe Protocol,
is the wire protocol on which DDS is built on.

DDS defines many commonly encountered data types and how they are serialize and
de-serialized. To define a message that can be consumed by DDS, one must specify
the type of data that is to be transferred among nodes according to DDS specification.
ROS add a layer of indirection by introducing a intermediate representation language
to specify the format of the data in a language-agnostic way. The language is called
the Interface Definition Language (IDL). The types of IDL is mapped to those of
DDS as below:

```
|Types of IDL            |Types of DDS        |example                   |
|------------------------|--------------------|--------------------------|
|bool                    |boolean             |bool flip_image           |
|byte/int8/uint8         |octet               |byte rx                   |
|char                    |char                |char id                   |
|float32                 |float               |float num                 |
|float64                 |double              |float64 tol               |
|int16                   |short               |int16 count               |
|uint16                  |unsigned short      |uint16 length             |
|int32                   |long                |int32 rc                  |
|uint32                  |unsigned long       |uint32 iteration          |
|int64                   |long long           |int64 key                 |
|uint64                  |unsigned long Long  |uint64 mul                |
|string (unbounded)      |string              |string s                  |
|wstring (unbounded)     |wstring             |wstring user_name         |
|string<=N (bounded)     |string              |string<=10 small_str      |
|T[] (unbounded array)   |sequence            |char[] c_str              |
|T[N] (static array)     |T[N]                |float[4] d4               |
|T[<=N] (bounded array)  |sequence<T,N>       |string[<=5] array_of_str  |
```

For example, the following means an array of string whose length is less or
equal than 10, and each string has 20 wchars at most.

```cpp
wstring<=20[<=10] array_of_wstr
```

The IDL also has restrictions on the name of variables. It is required that the
name of variables only contains lower case characters of "a-z" and numerical characters "0-9".
A name must start with a non-numerical character, underscore can be used but consecutive
underscores or underscore as the last character is forbidden.

Language-wise, if you are using C++, you might want to alias `bool` to `uint8` to
avoid the problems caused by the specialization of `std::vector<bool>`.

Another thing to notice that a type `Time` is is simply an alias of `uint32`, and
a type of `Duration` is simply an alias of `int32`. Their units are unfortunately
dependent on the context. They are both defined in package "builtin_interfaces".

### Constants, default value and comments
It is possible to specify the default value of a variable by adding a tailing literal
to its definition, for example

```
# variable count is of type int32 with default value being 1000
int32 count 1000
# variable name if of type "bounded string with no more than 20 characters,
# and its default value is "John Z. Li".
string<=20 name "John Z. Li"
# The above is equivalent to
string<=20 name 'John Z. Li'
int a[<=4] [1, 2, 3, 4]
```

Notice that string literals are not C/C++ string literals. They are not allowed
to contain escaped characters as of now. Also array literals are enclosed by
square brackets and elements separated by commas. Comment lines are started by
the "sharp(#)" symbol.

Besides setting default values, one can also define constants. Examples are as
below (using the "equal(=)" sign:
```
int32 COUNT=1000
string NAME="John Z. Li"
```

**Notice that names of constants must be of upper case.**

## Three types of description files.
The purpose of the IDL is to define data types used for message passing between
ROS nodes in plain text files, so that corresponding language bindings can be
generated automatically. There are three types of description file in ROS2:
- `msg` files, files with names ended with `.msg`, for definition of data types for ROS2 topics.
- `srv` files, files with names ended with `.srv`, for definition of data types for ROS2 services.
- `action` files, files with names ended with `.action`, for definition of data types for ROS2 actions.

Concerning the directory layout, `msg` files are required to be located in a directory
with the name `msg/` in the root directory of a package, `srv` files are required to
be located in a directory with the name `srv/` in the root directory of a package,
and `action` files are required to be located in a directory with the name `action`
in the root directory of a package. As a good practice, instead of having a directory
structure as below (C++ supposed):
```bash
/workspace_folder/src/my_cpp_ros_project
├── CMakeLists.txt
├── include
│   └── my_cpp_ros_project
├── package.xml
└── src
└── msg
└── srv
└── action
```

A better option is to wrap all the description files in a separate folder, and
the directory layout will be like:
```bash
/workspace_folder/src/my_cpp_ros_project
├── my_package
	├── CMakeLists.txt
	├── include
	│	└── my_package
	├── package.xml
	├── src
├── my_msgs
	└── msg
	└── srv
	└── action
	├── CMakeLists.txt
	├── package.xml
```


By wrapping all the communication data in a separate package, one can expand the
above ROS2 project horizontally by adding other packages in the "my_cpp_ros_project"
folder, each package of which can also use the message definitions in the "my_msgs" package. Of course,
one should give meaningful names to those projects in real projects.

### The file format of `msg` files.
A `msg` file consists of IDL definitions separated by line break. For example,
The `BehaviorTreeStatusChange.msg` defined in the "navigation2" project as below:

```bash
builtin_interfaces/Time timestamp  # internal behavior tree event timestamp. Typically this is wall clock time
string node_name
string previous_status              # IDLE, RUNNING, SUCCESS or FAILURE
string current_status               # IDLE, RUNNING, SUCCESS or FAILURE
```

Note that instead of creating your own `msg` types, one should first check that
whether the types he wants have already been defined in some well known ROS packages.
It is a good practice to use data types that are shared by the community, so that
interfacing with packages written by other people will not need unnecessary castings.
The package `std_msgs` [github link](https://github.com/ros/std_msgs) is well known,
which defines commonly used data structure such as multiarray types, message header types,
and types of empty messages. (**Note that it is not a good idea to use the `Time` and
`Duration` types defined in this package, because those two types are now officially
defined in package `builtin_interfaces` in the project "rcl_interfaces" [github link](https://github.com/ros2/rcl_interfaces).**
As its name suggests, it is supposed to be part of the ROS core libraries.
It contains type definitions of messages (of the types `msg`, `srv` and `action`)
that are common to all ROS developers.
Especially, one should pay attention to the `rcl_interfaces` package. If you are
dealing with ROS parameters or logging, you are going to need this package.
Another relevant project is the "common_msgs", it [github link](https://github.com/ros/common_msgs/tree/noetic-devel/geometry_msgs/msg)
contain message type definitions
for sensor data and navigation, and geometric types on which the former two kinds
are built. Obviously, if you are developing navigation algorithms, you should first
check whether what you need has already been defined in it.

### The file format of `srv` files.
Recall that a ROS service consists of a request and a reply. Thus, a `srv` file
contains two parts, one for the request message, one for the reply message. The two parts are separated
by three dashed (that is, ---). For example, the `GetCostmap` service in package "navigation2"
is defined as

```bash
# Get the costmap

# Specifications for the requested costmap
nav2_msgs/CostmapMetaData specs
---
nav2_msgs/Costmap map
```

The two projects, "rcl_interfaces" and "common_msgs" also contain commonly useful
definition of services.

### The file format of `action` files.
Recall that a ROS action is like a service except that it usually takes some time
to complete and it can be canceled before the action is completed. Besides the request
and the result, there is also continuous stream of feedback messages so that the action
client can know how the action is going. Thus, a `action` file consists three parts
separated by three dashes (that is, ---). For example, the `FollowWaypoints` action
in package "navigation2" is defined as

```bash
#goal definition
geometry_msgs/PoseStamped[] poses
---
#result definition
int32[] missed_waypoints
---
#feedback
uint32 current_waypoint
```

### Naming style
Note that all the type definitions of `msg`, `srv` and `action` follow the Camel case style.
It is unclear whether this is required by ROS, or is just a convention.

### Relevant configuration in `CMakeLists.txt` and `package.xml`.
After defining your own `msg`, `srv` and `action`files, you need to add relevant
settings in the `CMakeLists.txt` file and `package.xml` file so that the building
system can find dependencies for them and generate corresponding language specific code.
- A typical `CMakeLists.txt` configuration is as below.

	```cmake
	# dependencies
	find_package(ament_cmake REQUIRED) # this is an ament_cmake package
	find_package(rosidl_default_generators REQUIRED) # generate language-specific code from IDL descriptions.
	# three commonly used  message types packages.
	find_package(builtin_interfaces REQUIRED)
	find_package(std_msgs REQUIRED) # most commonly used data types
	find_package(action_msgs REQUIRED) # relyies on package action_msgs which is part of rcl_interfaces repository.

	# list all files in msg folder and ended with ".msg"
	set(msg_files
	"msg/<filename_1>.msg"
	"msg/<filename_2>.msg"
	# ...
	"msg/<filename_n>.msg"
	)

	# list all files in srv folder and ended with ".srv"
	set(srv_files
	"srv/<filename_1>.srv"
	"srv/<filename_2>.srv
	# ...
	"srv/<filename_n>.srv"
	)

	# list all files in the action folder and ended with ".action"
	set(action_files
	"action/<filename_1>.action"
	"action/<filename_2>.action"
	# ...
	"action/<filename_n>.action"
	)

   # generate header files and shared libaries that will be that will be
	# installed by "colcon".
	# Notice: you don't have to write "camke install" commands to install them.
	rosidl_generate_interfaces(${PROJECT_NAME}
	${msg_files}
	${srv_files}
	${action_files}
	DEPENDENCIES action_msgs builtin_interfaces std_msgs
	ADD_LINTER_TESTS # optional
	)

	# make the runtime dependency explicit.
	ament_export_dependencies(rosidl_default_runtime)

	```
- Relavant peace in `package.xml`
Typically, one needs to add the following to the file `package.xml`

```xml
  <buildtool_depend>ament_cmake</buildtool_depend>
  <buildtool_depend>rosidl_default_generators</buildtool_depend>
  <depend>action_msgs</depend>
  <depend>builtin_interfaces</depend>
  <depend>std_msgs</depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <test_depend>ament_lint_common</test_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>
```

## Specify Quality of Service (QoS) of messages
since ROS2 use DDS/RTPS to deliver messages between nodes, how a type of message
should be delivered becomes configurable. The main tradeoff is between reliability
and delay. Generally, if you require that a type of message must be delivered to
the subscribers no matter what (otherwise it is considered a fatal error of the
system), it is expected that additional delay might appear with some messages.
On the other hand, if you require that a stream of messages is sent in a realtime
fashion, and occasionally losing some pieces of information is tolerable, you will
want to choose a delivery policy that minimize the delay at the cost of being less reliable.
There are also many middle-ground configurations. The configuration w.r.t. a type
of message is called its Quality of Service, aka QoS.

### Configurable options of QoS
- *History*. Setting this as N means the last N messages will be stored. (It is also
possible to configure the system so that it saves all the history data of a type.
This should be used with caution. One need to make sure there will not be OOM, aka out-of-memory,
errors, or the performance of the system will not degrade too much).
- *Depth*, depth of the out-going message queue. This only takes effect if the "History" policy is set
to "keep last". If the producer node is not exactly synchronized and loss of information is undesirable,
this option is used to provide a buffer for the producer.
- *Reliability*. Setting this to "best effort" means to minimize delay at the cost of
possible loss of information. Setting this to "reliable" means that messages must
be reliably received, at the cost of possible re-sending which leads to delay.
- Durability, either "Transient local" or "volatile". The former means if no node
is listening to the message, the publishing node will wait until the information
is needed by a node. This is useful when a message is supposed to publish only once.
In this case, when the publishing node is set up, the subscribing node might has not yet
finihsed its initialization. The latter simply means sent and forget.
- Deadline, the maximal time that could be tolerated between two consecutive messages.
The name "Deadline" is actually a misnomer. This actually means the largest possible
deviation of intervals between two consecutive messages. Normally, periodic messages
are generated at roughly constant time intervals, and the error between the theoretical
value of the period and actual time intervals between two consecutive messages are normally distributed
with a small standard deviation. If the deviation is not small, it usually means
the realtime constraints of the robotic system has been violated. Thus, the value
of "Deadline" should be derived from design. Setting this value too large will only
cover underneath problems.
- Lifespan, if a message is older than the value of "Lifespan", it becomes useless.
- Lease Duration, a publisher has to notify the DDS that how long it will be remain
alive, and this value is called "Lease Duration". Before the lease duration of a
publisher expires, if the publisher does not renew it, the DDS considers the publisher
being dead.
- Liveliness, setting this to "Automatic" means a publisher of a node is living, that is,
it is sending messages, all the publisher of the node will be considered alive, meaning being
granted another lease duration period if its current lease has expired.
Setting this to "Manual by topic" means the programmer will manually manage the lifetime of
each publisher (renew lease when necessary).

All these options have default values. One should check the documentation of the underlying
DDS/RTPS used to find them. Generally, if an option denotes a time period, the default value
of it will be indefinitely long.

### use profiles to simplify QoS settings.
Usually a user does not need all the configurability provided by the DDS/RTPS middleware.
In most cases, we want services to be reliable while sensible to be streamed in realtime.
The following three profiles of QoS are most useful in ROS application.

- Services, reliable and volatile (missed service request will not be served, maybe because
the server node has restarted.).
- Sensor data, best effort and small depth (minimal delay is more important).
- Parameters, reliable, volatile and be with a larger queue than that of services (A larger queue means
    it is not a good idea to
discard user inputs even if there are many).

### Dealing with QoS constraints violations.
The DDS/RTPS software provide users with mechanics to detect possible violations of
QoS constraints in the form of callback functions. The events that might trigger
the callback functions are:
- Prespecified deadline missed (of a publisher) or requested deadline missed (of a subscriber)
- Liveliness lost (of a publisher) or Liveliness changed (on the subscriber side)
- Offered incompatible QoS (of a publisher) or Requested incompatible QoS (of a subscriber)

The names are quite self-explanatory.

To check that the QoS settings of a robotic system is workable, one can test-run
the system along with the `libstatistics_collector` package [github link](https://github.com/ros-tooling/libstatistics_collector).
This package will create a topic named `/statistics` to publish lively updated statistics of another topic (of course, by
let `libstatistics_collector` subscribe to the topic that needs to be monitored.).`

## An example
Create a ROS workspace and initilize a C++ package inside it, using the method in part 1 of this post.
we have a workspace which has the following directory layout.

```bash
# folder "src" is in the root of the workspace.
src
└── cpp_pubsub
    ├── CMakeLists.txt
    ├── include
    │   └── cpp_pubsub
    │       └── cpp_pub_sub.hpp
    ├── package.xml
    └── src
        ├── publisher_member_function.cpp
        └── subscriber_member_function.cpp
```

Few things to notice:
- The package name is "cpp_pubsub", the same with the name of directory where "CMakeLists.txt"
and "package.xml" reside.
- Header files that are supposed to be consumed by other pckages should be put into
"src/include/<package-name>. In our mini example, we do not expose an interface to other
packages. A dummy header file is created though to illustrate the package arrangement.

### The "publisher_member_function.cpp" file

```cpp
#include <chrono>
#include <functional>
#include <memory>
#include <string>

#include "rclcpp/publisher.hpp"
#include "rclcpp/qos.hpp"
#include "rclcpp/rclcpp.hpp"
#include "rclcpp/timer.hpp"
#include "std_msgs/msg/string.hpp"
#include "std_msgs/msg/string__struct.hpp"

using namespace std::chrono_literals;

class MinimalPublisher : public rclcpp::Node {
  using MsgString = std_msgs::msg::String;
  using PublisherType = rclcpp::Publisher<MsgString>::SharedPtr;
  using TimerType = rclcpp::TimerBase::SharedPtr;
  using SensorQoS = rclcpp::SensorDataQoS;

public:
  MinimalPublisher(std::string msg_header) : Node{"minimal_publisher"} {

    publisher_ = this->create_publisher<MsgString>("talker_msg", SensorQoS{});
    timer_ =
        this->create_wall_timer(500ms, [this]() { this->timer_callback(); });
    msg_header_ = msg_header;
  }

private:
  void timer_callback() {
    auto message = std_msgs::msg::String();
    message.data = msg_header_ + std::to_string(count_++);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", message.data.c_str());
    publisher_->publish(message);
  }
  TimerType timer_;
  PublisherType publisher_;
  size_t count_ = 0;
  std::string msg_header_ = "Hello World";
};

int main(int argc, char *argv[]) {
  rclcpp::init(argc, argv);
  if (argc == 2)
    rclcpp::spin(std::make_shared<MinimalPublisher>(argv[1]));
  else
    rclcpp::spin(std::make_shared<MinimalPublisher>("Hello world"));
  rclcpp::shutdown();
  return 0;
}
```

### The "subscriber_member_function.cpp" file

```cpp
#include <functional>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "rclcpp/subscription.hpp"
#include "std_msgs/msg/string.hpp"
#include "std_msgs/msg/string__struct.hpp"

using std::placeholders::_1;

class MinimalSubscriber : public rclcpp::Node {
  using MsgString = std_msgs::msg::String;
  using MsgType = MsgString::SharedPtr;
  using SubscriptionType = rclcpp::Subscription<MsgString>::SharedPtr;
  using SensorQoS = rclcpp::SensorDataQoS;

public:
  MinimalSubscriber() : Node("minimal_subscriber") {

    subscription_ = this->create_subscription<MsgString>(
        "talker_msg", SensorQoS{},
        [this](const MsgType msg) { this->topic_callback(msg); });
  }

private:
  void topic_callback(const MsgType msg) const {
    RCLCPP_INFO(this->get_logger(), "I heard: '%s'", msg->data.c_str());
  }
  SubscriptionType subscription_;
};

int main(int argc, char *argv[]) {
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<MinimalSubscriber>());
  rclcpp::shutdown();
  return 0;
}
```

### the "package.xml" file

```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
        <name>cpp_pubsub</name>
        <version>1.0</version>
        <description>
                A simple pub-sub example
        </description>
        <maintainer email="lizhao.johnlee@gmail.com">
                John Z. Li
        </maintainer>
        <license>None</license>

        <buildtool_depend>ament_cmake</buildtool_depend>

        <depend>rclcpp</depend>
        <depend>std_msgs</depend>

        <test_depend>ament_lint_auto</test_depend>
        <test_depend>ament_lint_common</test_depend>

        <export>
                <build_type>ament_cmake</build_type>
        </export>
</package>
```

### The "CmakeLists.txt" file

```cmake
cmake_minimum_required(VERSION 3.5)
project(cpp_pubsub VERSION 1.0
                  DESCRIPTION "A tiny project"
                  LANGUAGES CXX)

# Default to C++14
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

# find extra headers
include_directories(
  include/${PROJECT_NAME}
)

add_executable(talker src/publisher_member_function.cpp)
ament_target_dependencies(talker rclcpp std_msgs)
add_executable(listener src/subscriber_member_function.cpp)
ament_target_dependencies(listener rclcpp std_msgs)

# use add_library() if you also want to add some libraries.

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  # the following line skips the linter which checks for copyrights
  # uncomment the line when a copyright and license is not present in all source files
  #set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # uncomment the line when this package is not in a git repo
  #set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

# install
## copy headers in the include direcoty to /
install(
  DIRECTORY include/
  DESTINATION include/
)

# install the whole project.
# only install those executables that don't rely on ROS to manage its dependences to /bin
# if an executalbe is supposed to be run by "ros2 run", put it to /lib
install(
  TARGETS talker
  # ARCHIVE DESTINATION lib/${PROJECT_NAME}
  # LIBRARY DESTINATION lib/${PROJECT_NAME}
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)
install(
  TARGETS listener
  # ARCHIVE DESTINATION lib/${PROJECT_NAME}
  # LIBRARY DESTINATION lib/${PROJECT_NAME}
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)

# to be found and used by other package
ament_export_include_directories(include)
# if the project also provides a library
# ament_export_libraries(${PROJECT_NAME})
ament_package()
```

**Notice that unlike non-ament CMake projects, the executables should be installed
to "lib/${PROJECT_NMAE}" folder in the "install" direcotory of the workspace, so that
tools like "ros2 run" can find them.**

After building and sourcing the "setup.zsh" script, one can lauch the node "talker" use
the following from the root directory of the workspace:

```bash
ros2 run cpp_pubsub talker "Hi "
```

Here "Hi " is used as the message header of the topic. If you ommit the message header
in command line, it will defaulted to "Hello World".

The topic can be monitorred using

```bash
ros2 run cpp_pubsub listener
```
