---
layout: post
mathjax: true
comments: true
title:  "Beware of ABI compatibility of OpenCV and Eigen"
author: John Z. Li
date:   2021-03-18 19:00:18 +0800
categories: c++ programming
tags: ABI
---

# Beware of ABI compatibility of OpenCV and Eigen

## OpenCV compatibility.

According to [the docker file] of the OpenVSLAM project](https://github.com/xdspacelab/openvslam/blob/master/Dockerfile.desktop),
OpenVSLAM requires OpenCV version 4.1.0.


**Note, the official [installation guide](https://openvslam.readthedocs.io/en/master/installation.html) says it needs OpenCV version 3.4.0.
And the CMakeList file of the project looks for version 4.0 first, if not found, resorting to version 3.31.
I think it is because the author forgot to update its webpage. We should go with dependency requirements stated in the docker file.**


The OpenCV version going with Ubuntu 18 is 3.2.0. That is the version if you install "ibopencv-dev" using `apt` or `apt-get`.
This is NOT the version you will get if you use `find_package(OpenCV)` in your project CMake configurations if you have installed
OpenCV 4.1.0 to /usr/local, because the version (3.2.0) shipped with official distribution gets shadowed by the user-installed one.


According to [this page](https://abi-laboratory.pro/?view=timeline&l=opencv),
OpenCV constantly breaks backward compatibility with each new version. It also
does not follow semantic versioning rules. This means, version 4.1.0 is not compatible with
version 4.0.0.

To avoid possible problems, always specify the correct OpenCV version in your CMake configurations.


## Eigen compatibility
The ABI of Eigen can change based on

1. the version of  Eigen (version 3.3 or later, memory alignment to 32-bit is default).
2. certain compiler options (compiling with AVX or AVX2 support with `-mavx` or
`-mavx2`).
3. and certain Eigen-specific macros that user can define `EIGEN_DONT_ALIGN, EIGEN_DONT_ALIGN_STATICALLY, EIGEN_MALLOC_ALREADY_ALIGNED, EIGEN_MAX_STATIC_ALIGN_BYTES, EIGEN_MAX_ALIGN_BYTES, EIGEN_DEFAULT_DENSE_INDEX_TYPE, EIGEN_DEFAULT_TO_ROW_MAJOR, etc.`.

If your PC has a CPU which is of type x64 and  not too old which should already support AVX2 instructions, in this case,
you should
1. use a Eigen library whose version number is larger than 3.3.
2. When compiling, specify the C++ standard to be at least C++11. (Considering the fact that ROS2-dashing packages are compiled with C++14,
you might want also use C++14.)
3. Don't define any of the macro mentioned before (let Eigen auto-detect and use default values.)

If you are not sure about the CPU you are using, use

```bash
grep flags /proc/cpuinfo
```

to check.


If your CPU doesn't support AVX, consider define `EIGEN_MAX_ALIGN_BYTES` to be 32.

Since OpenVSLAM uses Eigen 3.3.7, and the one that installed with `apt` form official repository is
of version 3.3.4, always specify the version you want to use in the CMake configurations of your project.
If two binary objects used different versions of Eigen, don't pass pointers between them.
