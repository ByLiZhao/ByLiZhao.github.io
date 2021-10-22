---
layout: post
mathjax: true
comments: true
title:  "Difference between ROS and ROS2"
author: John Z. Li
date:   2021-09-24 19:00:18 +0800
categories: ros
tags: ros, ros2
---
ROS is sometimes called by the name ROS1, though I believe it is a misnomer.
Throughout this post, I will refer to ROS distributions before ROS2 as ROS.
The current actively maintained distribution of ROS is "ROS Noetic Ninjemys".
## Workspace and packages
### Initialize  a workspace
1. ROS2:
After creating a workspace, assuming the workspace contains a sub-directory named "src".
After entering the "src" sub-directory, the workspace can be initialized using the
following:
```bash
# CMake project
ros2 pkg create --build-type ament_cmake <package_name>
# Python project
ros2 pkg create --build-type ament_python <package_name>

## Or use the --node-name option to specify the entry point of the package, that is
## the package provides an executable named after "node_name"
ros2 pkg create --build-type ament_cmake --node-name <node_name> <package_name>
ros2 pkg create --build-type ament_python --node-name <node_name> <package_name>
```
2. ROS:
After creating an empty workspace, execute the following in the workspace directory:
```bash
catkin_make
```
This command will initialize a CMake project and creating three sub-directories
including "build", "devel" and "src".
To create a package, after entering the "src" subdirectory, execute the following:
```bash
catkin_create_pkg <package_name> <package_dep_1> <package_dep_2> ... <package_dep_k>
```

### Build a workspace
1. ROS2:
To build a workspace, execute the following at the root directory of the workspace:
```bash
# ROS2
colcon build
```
It is possible to build only a single ROS2 package or a set of selected packages with
```bash
colcon build --packages-select <pkg_1> <pkg_2> ... <pkg_k>
```
2. ROS:
To build a workspace, execute the following at the root directory of the workspace:
```bash
#ROS
catkin_make
```
It is impossible to only build a single package of ROS, because all packages in a workspace
is managed by a single CMake file.

### List all packages in a workspace
1. ROS2:
At the root of the workspace, execute the following: (don't need to source the workspace first).
```bash
# don't include packages in the underlay.
colcon list
```
Use the following to list all the packages that have been sourced: (if the current workspace is not sourced,
packages in the current workspace will not be listed.)
```bash
ros2 pkg list
```
2. ROS:
At the root of the workspace, execute the following: (if the current workspace is not sourced,
packages in the current workspace will not be listed.)
```bash
rospack list
```

### Find information of a package
1. ROS2:
Use the following to get the information of a package, (the package must have been sourced):
```bash
ros2 pkg xml <package_name>
```
And use the following to get the installing path of the package:
```bash
ros2 pkg prefix <package_name>
```
2. ROS:
Use the following to find the installing path of a package
```bash
rospack find <packge_name>
```
After getting the installing path, one can manually check the manifest of the package.

### Check package dependencies
1. ROS2:
```bash
# list all packages in the workspace and their dependencies among each other
colcon graph
# check dependencies in the package.xml file (only first-order dependencies)
ros2 pkg xml <package_name>
```
2. ROS:
```bash
# list the first-order dependencies of a package
rospack depends1 <package_name>
# list all dependencies of a package
rospack depends <package_name>
# list all dependencies, indent output to show the depth of dependencies.
rospack depends-indent <package_name>
```

### Launch ROS/ROS2 applications
1. ROS2:
**Note that once can also write ROS2 launch files using XML or YAML files.
But why not use a full-fledged programming language, python in this case.
There are certain things that can not be achieved via XML or YAML.**
```bash
# launch an executable provided by a package
ros2 run <package_name> <executable_name>
# With ROS2, a launch file is a simple python script.
# with managed nodes of ROS2, it can be guaranteed that nodes are started up
# with some well-behaved lifecycle constraints.
ros2 launch launch.py
```
2 ROS:
```bash
# file.launch is a XML file that specifies what should be launched.
# But no guarantee in what order will all nodes be started up.
roslaunch package_name file.launch
```

### Diagnosis tools
1. ROS2:
```bash
# Check ROS2 setup
ros2 doctor
# With some nodes running, get a report of your ROS2 application
ros2 doctor --report
# check whether the underlying communication mechanism is working
ros2 doctor hello
```
2. ROS:
```bash
# check whether ROS is running correctly
roswtf
```

## ROS node versus ROS2 node
### Hode nodes find each other
1. ROS2:
ROS2 uses DDS (Data Distribution Service) to do service discovery and message transmission.
Because DDS can orchestrate node communication over the same subnetwork, this makes it
possible to deploy ROS2 nodes on multiple computers which all belong to the same
subnetwork. DDS is a well-established international standard and good implementation
of DDS is battle tested. A good DDS implementation usually uses shared memory to
exchange information between two nodes when it is detected that both nodes are
located in the same localhost.
2. ROS:
With ROS, one has start the `roscore` first so that other nodes can find each other
and start to communicate with each other. **Note that `roslaunch` will automatically
detect whether `roscore` is already running and start it if not.**
- `roscore` will starts a ROS Master, which provides a naming registration and finding
services for nodes, topics and services.
- `roscore` will also start a ROS Parameter Server, which is actually runs as a part
of above mentioned ROS Master.
- `roscore` will also start a "rosout" logging node.
After all the other nodes except those included in `roscore` find each other,
they communicate with each other using standard TCP. Because standard TCP is used,
with some network configuration, ROS nodes can also be distributed among multiple
machines. The problem is that the TCP mechanism can not be short-circuited even
if we know in advance that two nodes reside on the same machine. This leads to
a sometimes large performance penalty because the overhead of all the serializing,
deserializing and TCP transportation. If performance becomes an issue, with ROS,
one often needs to rewrite  his application using so-called "nodelets" and use
shared pointers to pass messages around.

### Node-process relations
ROS follows a strict One-Node-Per-Process-Per-Executable model.
Each node in ROS is started by an executable creating a new process. Each node-process
then communicating with each other concurrently.

This design has some implications:
- As each node of ROS is corresponding to a separate executable, a ROS node can
only be defined statically.
- Because each ROS node resides in different OS processes, A ROS application tends
to have more processes running, compared with one implemented with ROS2.
- Because of data serialization and message passing overhead, it is more likely that
there are latency and jitter with ROS applications.

**Note: This does not mean that one can not use multiple threads to improve the performance
of a ROS node, or that some functionalities of a node can not be loaded at runtime. It is
just that you can not conceptually model them after ROS node.**
For example, one can use
-`ros::MultiThreadedSpinner`
- `ros::AsyncSpinner`
-  `allow_concurrent_callbacks = true` to enable execution of multiple callbacks at the same time.
- multiple callback queues and dedicated  threads so that callbacks for high-priority topics
don't get blocked by those of low-priority queues.
If one wants to create node-like entities in within a ROS-node, he can use [nodelet](https://github.com/ros/nodelet_core)
and use shared pointers to exchange data between multiple nodelets. ROS nodelets can be loaded dynamically at runtime.

With ROS2, it is possible to implement multiple nodes in a single process. ROS2 completely decouples
the concept of node can how the node is handled through the `executor` interface.
One can add a node into an `executor`, and specify a thread to run that `executor`.
### Lifecycle management
Because ROS2 completely decouples the concept of Nodes and the threading model that
composes different nodes into a robotic system. It becomes necessary to introduce
the notion of "lifecycle management" to make sure that:
- When a robotic system starts up, all nodes are initialized correctly before each
nodes starts to communicate with other nodes and does its work.
- If a node is shutdown and replaced by another node at runtime. We need to make sure
that the replaced node is shutdown properly and the newly loaded node is configured
correctly according to some runtime information.

ROS does not have this issue because its adherence to single-node-per-process model.
While this makes ROS systems simpler, it also limit its capability.
### Namespace of services and topics
In ROS, topics with the same `.msg` name, and services with the same `.srv` name
collides with each other. In ROS2, topics and services are grouped by the packages
in which they are defined.

## Tools specific to ROS
There are two command line tools that currently are only available for ROS.
1. Use `roscd` to navigate to the directory of a ROS package.
For example, `roscd roscpp` will make the shell enter the root directory of the
"roscpp" package. This [Github issue](https://github.com/ros2/ros2cli/pull/75) why
such a facility is hard to be implemented with ROS2.
**Please note that `roscd log` will change the current directory to the directory
where log files are saves. This command does not mean that "log" is a ROS package.**
2. Use `rosls` to view  directory layout of a ROS package without leaving the current
directory or entering the full path of the package. For example, `rosls roscpp` will
print the directory layout of the "roscpp" package"

