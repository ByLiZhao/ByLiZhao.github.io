---
layout: post
mathjax: true
comments: true
title:  "Working with ROS2 packages: Part 1"
author: John Z. Li
date:   2021-02-21 19:00:18 +0800
categories: robotics
tags: ros2
---
Recall that a ROS workspace is nothing but a collection of ROS packages.
Technically, a ROS workspace doesn't have to be a directory on the OS's filesystem.
Though it is a convention to start with a new and empty directory each time you
want to create a new workspace. In this way, we can just use the name of the directory
as the name of workspace.

Like the conception of inheritance in Object Oriented Programming, with ROS, we
can develop a new workspace on top of another workspace. Notice that the ROS
documentation does not call this inheritance. Instead, the "inherited" workspace
is called the *underlay*, and the inheriting workspace is called the *overlay*.
To work on top of an underlay, `source` the setup script of that workspace before
starting work with the overlay. For example, one can use the following to start
an overlay

```bash
mkdir -p  ~/ros_examples/ex1/src
# if you are using bash, use setup.bash
source /opt/ros/dashing/setup.zsh
```

Now the directory of `~/ros_examples/ex1` will be the root of the overlay.

## The structure of a ROS package
`cd` into the root of the newly created ROS workspace, that is, in our case, the
`~/ros_examples/ex1` directory. All source code of each package the workspace is
going to contain is located in the `src` subfolder of the workspace. Typically,
each package occupies its own subfolder under `src`. If one would like to group
packages together, he can contain different one or more packages in different
subfolder of `src`. For example, the following is perfectly OK:

```bash
|src
├──subfolder1
|   ├──package1
|   ├──package2
|──subfolder2
    ├──package3
    ├──package4
    ├──|package5
```
The above will be effectively the same as the below from the perspective of users:

```bash
|src
├──package1
├──package2
├──package3
├──package4
├──package5
```

To understand the structure of a ROS package, we start with some packages that
are created by others.

```bash
cd ~/ros_examples/ex1/src
git clone git clone https://github.com/ros/ros_tutorials.git -b dashing-devel
```

The above create a subfolder called `ros_tutorials` in `src`, and under the subfolder
populated four packages:

```bash
|src/ros_tutorials
├── roscpp_tutorials
├── rospy_tutorials
├── ros_tutorials
└── turtlesim
```

### The structure of a C++ ROS2 project.
The "turtlesim" subfolder is a C++ ROS package. `cd` into the folder to check its
structure. Note that the "turtlesim" is the package's root.

A ROS2 C++ package usually contains the following:

```bash
package.xml
(optional) colcon.pkg
CMakeList.txt
(optional) README
(optional) CHANGELOG
(optional) COLCON_IGNORE
C++ source and headers folder(s)
```

#### The manifesto file of a package.
The "package.xml" file is called the ROS package manifest of the package. It is
required that each ROS package should have exactly one of the manifest. The file
contains the following information:
* descriptive data (i.e. Package description, maintainer)
* dependencies on other ROS and system packages. There are three kinds of dependencies:
	- building dependencies.
	- executing dependencies.
	- testing dependencies.
* meta-information (i.e. the author and website)
* packaging information (i.e. the version)

Various tools rely on this file to work correctly, like `rosdep`, `colcon` or `bloom`.
The manifesto has  standardized semantics [REP 127 manifest format](https://www.ros.org/reps/rep-0127.html#motivation),
[REP 140 manifest format](https://www.ros.org/reps/rep-0140.html) and [REP 149 manifest format](https://www.ros.org/reps/rep-0149.html#id41).

#### The `colcon.pkg ` file
A `colcon.pkg` file is used in two cases:
* The package under consideration is a general-purposed package (e.g. a C++ project with a plain CMake file).
So it is not supposed to be
treated as a ROS package. You want to include the package as part of your ROS project and treat it
*as if* it is ROS package.
* The package is a ROS package and has already a manifest file. In this case, the file can be used
to source some scripts to provide autocompletion in shells. An example of this is
[ros2cli](https://github.com/ros2/ros2cli/blob/5dc4d9bfe14a2a62667977bfaa52b2288af6c2f3/ros2cli/colcon.pkg).


The documentation of the format of the file can be found [here](https://colcon.readthedocs.io/en/released/user/configuration.html#colcon-pkg-files).
Note that if this file exists, it must exist in the root of the package.
This means, either you have write access to the project, so you can add a `colcon.pkg` file to its root,
or you have to fork a project and add it to the forked project and update it when the upstream is update.
If you don't have write access to the project and you don't want to maintain a forked version of a project,
here is another way to make it work with ROS.

> Putting a package_name.meta file outside the source tree,
> and use `colcon --metas <path/to/file> <subcommand>`
> to build, test, and install the package.

#### The `CMakeList.txt` file
This file is supposed to to consumed by `ament_cmake`, which is `CMake` with
some features added to facilitate ROS package development.

#### Other files.
The "README" and "CHANGELOG" files don't need explanation. They could be plain text files,
of markdown format or of reStructureText format.

### The structure of a Python ROS2 project
A python ROS2 project contains the following files/directories:
```bash
package.xml
(optional) colcon.pkg
setup.py
setup.cfg
(optional) README
(optional) CHANGELOG
(optional) COLCON_IGNORE
/<package_name>
```
* The `package.xml` and `colcon.pkg` are like that of the C++ case.
* The `setup.py` contains instructions of installing the package.
* The `setup.cfg` is needed if one prefers putting package metadata in a `cfg` file.
In ROS's case, if one wants the package can be invoked by `ros2 run`, this file
also needs to exist. A package can contain multiple executables that can be invoked
by `ros2 run`.
* The "<package_name>" directory is of the same name as that of the package. ROS2
requires this so that ROS2 tools can find your package. Python requires that
this folder contains a file named `__init__.py`, which makes the folder a standard
python module. This "<package_name>" directory does not have to be located in the
same directory with other files. A common practice is put it in another folder,
for example, the Python package in a folder named "resource" or "src".

Note that according to [PEP 517](https://www.python.org/dev/peps/pep-0517/),
[PEP 518](https://www.python.org/dev/peps/pep-0518/#file-format)
and [PEP 621](https://www.python.org/dev/peps/pep-0621/),
future Python projects might use a `pyproject.toml` file instead of `setup.py`
and `setup.cfg`.

## Create a new package from command line

### C++ ROS2 packages
For C++ ROS2 packages, in the "src" subfolder of the workspace, run
```bash
ros2 pkg create --build-type ament_cmake <package_name>
```
For example, the following
```bash
cd ~/ros_examples/ex1/src
ros2 pkg create --build-type ament_cmake my_cpp_ros_project
```
populates an empty package with the following directory structure:
```bash
my_cpp_ros_project
├── CMakeLists.txt
├── include
│   └── my_cpp_ros_project
├── package.xml
└── src
```

### Python ROS2 packages
For Python ROS2 packages, in the "src" subfolder of the workspace, run
```bash
ros2 pkg create --build-type ament_python <package_name>
```

For example, the following
```bash
cd ~/ros_examples/ex1/src
ros2 pkg create --build-type ament_python my_python_ros_project
```
populates an empty package with the following directory structure:
```bash
my_python_ros_project
├── my_python_ros_project
│   └── __init__.py
├── package.xml
├── resource
│   └── my_python_ros_project
├── setup.cfg
├── setup.py
└── test
    ├── test_copyright.py
    ├── test_flake8.py
    └── test_pep257.py
```

### Create a package with given node name
If the package is supposed to implement a node, we can use the `--node-name`
option along with `ros2 pkg create`. For example
```bash
# create a cpp ROS2 project with node name being "my_node".
ros2 pkg create --build-type ament_cmake --node-name my_node my_cpp_ros_project_new
# create a Python ROS2 project with node name being "my_node"
ros2 pkg create --build-type ament_python --node-name my_node my_python_ros_project_new
```

## Build ROS2 packages
### Build a workspace
To build a workspace, `cd` into the corresponding directory, and run
`colcon build`.
Foe example
```bash
cd ~/ros_examples/ex1
colcon build
```
will build all the projects in workspace `ex1`.
This will create another three sub-directories:
* `build`, the sub-directory where out-of-tree building happens.
* `install`, the built projects are installed here.
* `log`, building log goes here.

To accelerating building, one can use the following to build the workspace instead:
```bash
colcon build --symlink-install
```

The `--symlink-install` option means that when installing the package, use symbolic
links when possible. This avoids the cost of file copying. Without this option,
even the projects of a workspace is only mildly modified, many files will be copied
over again with each building.

### Build only one project
Usually, when we work in a ROS2 workspace, we only work on one package at a time.
It does not make sense to re-build every package in the workspace every time. Thus
it is better to only build one (or more) selected package. Use the below command to build only
selected packages.
```bash
colcon build --packages-select my_package
```

### Source the built workspace
After successfully building a workspace, one has to make the workspace be known
by ROS. To do this, one has to source the relative "setup" scripts.
For example,
```bash
# if you want to use the "underlay" and the "overlay" at the same time
# use the *.bash version if you are using bash
source ~/ros_examples/ex1/install/setup.zsh
# if you only want to use the workspace where the script in in.
source ~/ros_examples/ex1/install/setup.zsh
```

The latter is equivalent to sourcing all the underlays of the workspace, from
the first in the chain to the current workspace where the script is in.

Another thing to know is that, just like inheritance in OOP, if there is a package
in the overlay which has the same name with a package in the underlay, when calling
the package, the one in the overlay will be used. In other words, the package in
the overlay will shadow that with the same name in the underlay.
To illustrate this
```bash
vim  ~/ros_examples/ex1/src/ros_tutorials/turtlesim/src/turtle_frame.cpp
```
and replace `setWindowTitle("TurtleSim");` in function `TurtleFrame` to
`setWindowTitle("My TurtleSim");`.
Then do the following:
```bash
cd ~/ros_examples/ex1
colcon build
source ./install/setup.zsh
ros2 run turtlesim turtlesim_node
```
You will find that the title of the TurtleSim window has changed to "My TurtleSim".
One can use
```bash
ros2 pkg list
```
to check which package has been included currently.

## Run a package with `ros2 run`
Recall that
```bash
ros2 run <package_name> <executable_name>
```
will run the executable with the given name in that package. If a package contains
multiple executables, one can run each of them by specifying its name.

If a package is created with the `--node-name`option, one can launch the node with
```bash
ros2 run <package_name> <node_name>
```

### Update relevant files if you want to publish your packages.
Always remember to update relevant files, such as the manifest file, README, CHANGELOG, etc.
If you want to publish your packages. All information should stay sync with each release.

