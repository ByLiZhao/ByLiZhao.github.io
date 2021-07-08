---
layout: post
mathjax: true
comments: true
title:  "All about Navigation2 maps"
author: John Z. Li
date:   2021-03-10 19:00:18 +0800
categories: robotics
tags: ROS, Navigation2
---

# All about Navigation2 maps
## Maps
### Portable GrayMap
This kind of files are ended with `.pgm`.
- Each `pgm` file starts with a two-byte "magical number" to specify its type.
A value of "P5" means the data of the image is stored in binary format,
while a value of "P2" means the data of the image is stored in human readable ASCII format.
- In raw-data format, aka, magical number being "P5", each pixel uses 8-bit or 16-bit space.
- After the line of the magical number, one can add comments lines starting with `#`.
- After comment lines, there is a dimension line, consisting of a pair of integer numbers,
first number is the X-dimension (number of columns), and the second number is the Y-dimension (number of rows).
- After the dimension line, there is a line of a integer number, indicating the value corresponding "white",
that is the largest value possible.
- The image data is stored in rows. And rows are separated by a newline character if
the data of the image is stored in human readable ASCII format. (No newline character
needed if the data of the image is stored in binary format.)

Below is an example of `pgm` image in ASCII format:

```txt
P2
# Shows the word "FEEP" (example from Netpbm man page on PGM)
24 7
15
0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
0  3  3  3  3  0  0  7  7  7  7  0  0 11 11 11 11  0  0 15 15 15 15  0
0  3  0  0  0  0  0  7  0  0  0  0  0 11  0  0  0  0  0 15  0  0 15  0
0  3  3  3  0  0  0  7  7  7  0  0  0 11 11 11  0  0  0 15 15 15 15  0
0  3  0  0  0  0  0  7  0  0  0  0  0 11  0  0  0  0  0 15  0  0  0  0
0  3  0  0  0  0  0  7  7  7  7  0  0 11 11 11 11  0  0 15  0  0  0  0
0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
```

Navigation2 can use other image formats like `PGN` files. Eventually, Navigation2
needs only the images "width", "depth", "bit depth" and image data as a matrix of
integers.

### The `map.yaml` file
A `yaml` files needs to exist to specify the metadata of a map associated with an image.

The `yaml` file has to specify the following:

```yaml
image: <path-to-image: string>
resolution: <resolution: float>
origin: [<X0: float>, <Y0: float>, <Yaw: float>]
occupied_thresh: <o-thresh: float>
free_thresh: <f-thresh: float>
negate: <0 or 1>
mode (optional): <trinary or scale or raw>
```

In the `yaml` file:
- `image` is the path to the image file, either being relative or absolute.
- `resolution` is the resolution of the map, in meters. For example, `resolution: 0.05` means
each pixel of the image stands for a 5cm-by-5cm cube in the map.
- `origin` is the pose of the pixel at the left-down corner of the image in the map. Yaw is measured counterclockwise.
- `occupied_thresh`, a value that is between (0.0, 1.0). A probability larger than this value is interpreted as the pixel is occupied.
- `free_thresh`, a value that that is between (0.0, 1.0). A probability smaller than this value is interpreted as the pixel is free.
- `negate`, default value is `0`, meaning a larger value of a pixel means the pixel is more likely to be free.
(whiter pixels stand for "freer" regions.)
If `negate` is `1`, it means a larger value of a pixel means  the pixel is more likely to be occupied.
- `mode` affects how a probability is transformed into an integer value of a pixel.

Each pixel value has a corresponding probability, which stands for the probability
that the grid is occupied. While the map server loads a map, it converts pixel values
to a series of values of type `uint_8`. Let us call such a value the grid value of
the corresponding pixel.
Assuming that the maximal value of the image pixel is 255.
1. Transformation from a pixel value to a probability.
	- If `negate` is `0`, letting `x` denotes a pixel value, the probability can be computed via `p = (255 - x)/255`.
	The result is a floating number that is between [0.0, 1.0].
	- If `negate` is `1`, p is calculated by `p = x/255`.
2. Transformation from a probability to a grid value.
	- If `mode` is `trinary` (default).
		* if `p < free_thresh`, the pixel value is `0'.
		* if `p > occupied_thresh`, the pixel value is `100`.
		* Otherwise,  the pixel value is `255`. Notice that with signed integer being represented with 2-complement format, `255` when interpreted as `int8_t`, is of the value `-1`.
	- If `mode` is `scale`.
		* if `p < free_thresh`, the pixel value is `0`.
		* if `p > occupied_thresh`, the pixel value is `100`.
		* Otherwise, the pixel value is calculated via ` x = (p - free_thresh) * 99 / (occupied_thresh - free_thresh).
		* `255` does not mean "unknown`, one has to use a `PNG` image format for that purpose, where any non-zero alpha channel means "unknown".
	- If `mode` is `raw`
		* the pixel value is calculated by `x = p * 255`.

## Map server
On starting up, Navigation2 launches a node called `map_server` which is implemented in package `nav2_map_server`,
which is part of the `Navigation2` repository.

The `nav2_map_server` contains three parts:
1. `map_server`, for loading map from a `map_name.yaml` file. The map file can be specified as command line arguments,
or provided by service request.
2. `map_saver`, for saving an existing map as a map image file and an accompanying `yaml` file.
3. `map_io`, a library used by `map_server` and `map_saver`. This is an abstraction layer to hide implementation details of different image formats.

The `map_server` takes parameters which specifies the node of the name, and the path to the map `yaml` file.
For example, with the following configuration `yaml` file

```yaml
# map_server_params.yaml
map_server:
    ros__parameters:
        yaml_filename: "map.yaml"
```

This configuration file says that the name of the node is "map_server" and the node will load a map specified by the file `map.yaml`.
The map server can be launched by

```bash
# for ROS2-dashing
ros2 run nav2_map_server map_server __params:=map_server_params.yaml
# for ROS version newer than dashing
ros2 run nav2_map_server map_server --ros-args --params-file map_server_params.yaml
```

Besides providing a topic named `map`, a `map_server` instance also provide a service also called `map`,
which is of the type `nav_msgs/srv/GetMap`. By subscribing the the `map` topic or request the `GetMap` service,
one will get an object, which is of the type `nav_msgs/msg/OccupancyGrid`, which is defined as

```bash
# This represents a 2-D grid map, in which each cell represents the probability of occupancy.

std_msgs/Header header

# MetaData for the map
MapMetaData info

# The map data, in row-major order, starting with (0,0).  Occupancy
# probabilities are in the range [0,100].  Unknown is -1.
int8[] data
```

Recall that `std_msgs/Header` contains a time stamp and a frame ID, while
`MapMetaData` is defined as

```bash
# This hold basic information about the characterists of the OccupancyGrid

# The time at which the map was loaded
builtin_interfaces/Time map_load_time

# The map resolution [m/cell]
float32 resolution

# Map width [cells]
uint32 width

# Map height [cells]
uint32 height

# The origin of the map [m, m, rad].  This is the real-world pose of the
# cell (0,0) in the map.
geometry_msgs/Pose origin
```

Notice that **not all informatoin in the map.yaml file is included in the message.**
This means, there is no reliable method to know from a message of the type `OccupancyGrid`
whether the mode of the map is `trinary`, or `scale` or `raw`, or whether the `negate` field
of the `map.yaml` file has been set to `1`.
