# 01. RasperryPi3 Ros系统安装\(Debian\)

﻿01. RasperryPi3 Ros系统安装\(Debian\)

今天在树莓派3上安装ROS的系统，使用版本indigo，首先安装ROS源，

```text
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu wheezy main" > /etc/apt/sources.list.d/ros-latest.list'
```

然后获取密钥：

```text
wget https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -O - | sudo apt-key add -
```

```text
wget https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -O - | sudo apt-key add ---2016-06-02 03:33:28--  https://raw.githubusercontent.com/ros/rosdistro/master/ros.keyResolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.31.18.133Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.31.18.133|:443... connected.HTTP request sent, awaiting response... 200 OKLength: 1162 (1.1K) [application/octet-stream]Saving to: ‘STDOUT’-                                               100%[======================================================================================================>]   1.13K  --.-KB/s   in 0s     2016-06-02 03:33:29 (8.82 MB/s) - written to stdout [1162/1162]OK
```

然后更新源：

```text
 sudo apt-get update sudo apt-get upgrade
```

接下来安装软件依赖：

```text
$ sudo apt-get install python-setuptools python-pip python-yaml python-distribute python-docutils python-dateutil python-setuptools python-six$ sudo pip install rosdep rosinstall_generator wstool rosinstall
```

python-argparse依赖python2.6，所以先不安装了。

配置rosdep：

```text
sudo rosdep initrosdep update
```

```text
pi@raspberrypi:~/indigo $ rosdep updatereading in sources list data from /etc/ros/rosdep/sources.list.dHit https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/osx-homebrew.yamlHit https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/base.yamlHit https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/python.yamlHit https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/ruby.yamlHit https://raw.githubusercontent.com/ros/rosdistro/master/releases/fuerte.yamlQuery rosdistro index https://raw.githubusercontent.com/ros/rosdistro/master/index.yamlAdd distro "groovy"Add distro "hydro"Add distro "indigo"Add distro "jade"Add distro "kinetic"updated cache in /home/pi/.ros/rosdep/sources.cache
```

下载代码：

```text
$ rosinstall_generator ros_comm --rosdistro indigo --deps --wet-only --exclude roslisp --tar > indigo-ros_comm-wet.rosinstall$ wstool init src indigo-ros_comm-wet.rosinstall
```

这一步会下载ros的源码下来。如果下载过程中失败了，那么就用**wstool update -j 4 -t src**命令更新代码。

下载外部代码：

```text
$ mkdir ~/ros_catkin_ws/external_src$ sudo apt-get install checkinstall cmake$ sudo sh -c 'echo "deb-src http://mirrordirector.raspbian.org/raspbian/ testing main contrib non-free rpi" >> /etc/apt/sources.list'$ sudo apt-get update
```

```text
$ cd ~/ros_catkin_ws/external_src$ sudo apt-get build-dep console-bridge$ apt-get source -b console-bridge$ sudo dpkg -i libconsole-bridge0.2_*.deb libconsole-bridge-dev_*.deb
```

下载liblz4代码：

```text
$ cd ~/ros_catkin_ws/external_src$ apt-get source -b lz4$ sudo dpkg -i liblz4-*.deb
```

编译代码：

```text
 sudo ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release --install-space /opt/ros/indigo
```

========================================================================================

编译失败：

```text
/opt/ros/indigo/include/ros/serialization.h:223:5: note: in expansion of macro ‘ROS_CREATE_SIMPLE_SERIALIZER_ARM’     ROS_CREATE_SIMPLE_SERIALIZER_ARM(double);     ^CMakeFiles/Makefile2:317: recipe for target 'CMakeFiles/roscpp.dir/all' failedmake[1]: *** [CMakeFiles/roscpp.dir/all] Error 2Makefile:127: recipe for target 'all' failedmake: *** [all] Error 2<== Failed to process package 'roscpp':   Command '['/opt/ros/indigo/env.sh', 'make', '-j4', '-l4']' returned non-zero exit status 2Reproduce this error by running:==> cd /home/pi/indigo/build_isolated/roscpp && /opt/ros/indigo/env.sh make -j4 -l4Command failed, exiting.
```

重新编译能过这个文件，但是会卡在其他文件，似乎是树莓派的运算能力有限，在编译ros源码时总是会卡住，CPU灯都不再闪烁了。

再次编译还是会报错：

```text
c++: internal compiler error: Killed (program cc1plus)Please submit a full bug report,with preprocessed source if appropriate.See <file:///usr/share/doc/gcc-4.9/README.Bugs> for instructions.CMakeFiles/roscpp.dir/build.make:629: recipe for target 'CMakeFiles/roscpp.dir/src/libros/service_client.cpp.o' failedmake[2]: *** [CMakeFiles/roscpp.dir/src/libros/service_client.cpp.o] Error 4CMakeFiles/Makefile2:317: recipe for target 'CMakeFiles/roscpp.dir/all' failedmake[1]: *** [CMakeFiles/roscpp.dir/all] Error 2Makefile:127: recipe for target 'all' failedmake: *** [all] Error 2<== Failed to process package 'roscpp':   Command '['/opt/ros/indigo/env.sh', 'make', '-j4', '-l4']' returned non-zero exit status 2Reproduce this error by running:==> cd /home/pi/indigo/build_isolated/roscpp && /opt/ros/indigo/env.sh make -j4 -l4Command failed, exiting.
```

网上搜到一个答案：

```text
g++: internal compiler error: Killed (program cc1plus)
Please submit a full bug report,
主要原因大体上是因为内存不足,有点坑 临时使用交换分区来解决吧sudo dd if=/dev/zero of=/swapfile bs=64M count=16sudo mkswap /swapfilesudo swapon /swapfile
After compiling, you may wish to
Code:sudo swapoff /swapfilesudo rm /swapfile
```

那么如何添加树莓派的交换空间呢？

现在的交换空间大小：

```text
pi@raspberrypi:~/indigo $ free -h             total       used       free     shared    buffers     cachedMem:          925M       109M       815M       1.1M        17M        48M-/+ buffers/cache:        43M       881MSwap:          99M        49M        50M
```

100M，那么修改文件/etc/dphys-swapfile可以增加空间，然后重启即可。

增加到500，上面的可以过了，不过还是非常卡，经常停住不动。

编译过程非常吃内存，所以增加到1000M。

这次安装成功，然后source一下执行路径：

source /opt/ros/indigo/setup.bash  


然后就可以使用了。

===================================================================================================================================================

上面的编译没有把全部的东西编译进去，我们重新整理一下。

首先添加源和key：

```text
pi@raspberrypi:~ $ sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'pi@raspberrypi:~ $ sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net --recv-key 0xB01FA116Executing: gpg --ignore-time-conflict --no-options --no-default-keyring --homedir /tmp/tmp.qYVzkTZHsn --no-auto-check-trustdb --trust-model always --keyring /etc/apt/trusted.gpg --primary-keyring /etc/apt/trusted.gpg --keyserver hkp://ha.pool.sks-keyservers.net --recv-key 0xB01FA116gpg: requesting key B01FA116 from hkp server ha.pool.sks-keyservers.netgpg: key B01FA116: public key "ROS Builder <rosbuild@ros.org>" importedgpg: Total number processed: 1gpg:               imported: 1
```

然后update源。

下载工具依赖：

```text
pi@raspberrypi:~ $ sudo apt-get install python-rosdep python-rosinstall-generator python-wstool python-rosinstall build-essential
```

初始化源码环境：

```text
$ sudo rosdep init$ rosdep update
```

```text
$ mkdir ~/ros_catkin_ws$ cd ~/ros_catkin_ws
```

```text
$ rosinstall_generator desktop_full --rosdistro indigo --deps --wet-only --tar > indigo-desktop-full-wet.rosinstall$ wstool init -j8 src indigo-desktop-full-wet.rosinstall
```

安装源码依赖：

```text
$ rosdep install --from-paths src --ignore-src --rosdistro indigo -y
```

安装的依赖有：

```text
libqt4-core libcurl4-openssl-dev libltdl-dev python-wxgtk3.0 python-matplotlib tango-icon-theme libcollada-dom2.4-dp-dev uuid-dev python-sip-dev shiboken graphviz libgtest-dev libpyside-dev libpcl1.7 python-paramiko liblog4cxx10-dev libapr1-dev cmake python-qt4-gl python-empy python-netifaces libtinyxml-dev liburdfdom-headers-dev libqt4-dev qt4-qmake liburdfdom-dev libvtk-java sbcl python-pyside python-opengl python-opencv python-qwt5-qt4  libeigen3-dev gazebo7  liblz4-dev libbz2-dev libyaml-cpp-dev gazebo7 libogg-dev hddtemp libshiboken-dev libogre-1.9-dev libcppunit-dev libassimp-dev libgtk2.0-dev libtheora-dev python-psutil libpcl-dev libpoco-dev libfltk1.1-dev python-pygraphviz libqhull-dev python-pydot python-vtk 
```

在安装某个库时出错：

```text
pi@raspberrypi:~/ros_catkin_ws $ sudo -H apt-get install -y libtbb2Reading package lists... DoneBuilding dependency tree       Reading state information... DonePackage libtbb2 is not available, but is referred to by another package.This may mean that the package is missing, has been obsoleted, oris only available from another sourceE: Package 'libtbb2' has no installation candidate
```

使用下面指令跳过不能安装的依赖：

```text
$ rosdep install --from-paths src --ignore-src --rosdistro indigo -ry
```

下面是安装失败的包：

```text
ERROR: the following rosdeps failed to install  apt: command [sudo -H apt-get install -y gazebo7] failed  apt: command [sudo -H apt-get install -y libyaml-cpp-dev] failed  apt: command [sudo -H apt-get install -y libbz2-dev] failed  apt: command [sudo -H apt-get install -y python-coverage] failed  apt: command [sudo -H apt-get install -y python-qt4-dev] failed  apt: command [sudo -H apt-get install -y libopencv-dev] failed  apt: command [sudo -H apt-get install -y libshiboken-dev] failed  apt: command [sudo -H apt-get install -y libogre-1.9-dev] failed  apt: command [sudo -H apt-get install -y libcppunit-dev] failed  apt: command [sudo -H apt-get install -y libassimp-dev] failed  apt: command [sudo -H apt-get install -y libgtk2.0-dev] failed  apt: command [sudo -H apt-get install -y libtheora-dev] failed  apt: command [sudo -H apt-get install -y python-psutil] failed  apt: command [sudo -H apt-get install -y libpcl-dev] failed  apt: command [sudo -H apt-get install -y libpoco-dev] failed  apt: command [sudo -H apt-get install -y libfltk1.1-dev] failed  apt: command [sudo -H apt-get install -y python-pygraphviz] failed  apt: command [sudo -H apt-get install -y libqhull-dev] failed  apt: command [sudo -H apt-get install -y python-pydot] failed  apt: command [sudo -H apt-get install -y libjpeg62-turbo-dev] failed  apt: command [sudo -H apt-get install -y libtool-bin] failed  apt: command [sudo -H apt-get install -y libboost-all-dev] failed  apt: Failed to detect successful installation of [gazebo7]  apt: Failed to detect successful installation of [libyaml-cpp-dev]  apt: Failed to detect successful installation of [libpcl-dev]  apt: Failed to detect successful installation of [libbz2-dev]  apt: Failed to detect successful installation of [python-coverage]  apt: Failed to detect successful installation of [python-qt4-dev]  apt: Failed to detect successful installation of [libopencv-dev]  apt: Failed to detect successful installation of [libshiboken-dev]  apt: Failed to detect successful installation of [libogre-1.9-dev]  apt: Failed to detect successful installation of [libcppunit-dev]  apt: Failed to detect successful installation of [libassimp-dev]  apt: Failed to detect successful installation of [libgtk2.0-dev]  apt: Failed to detect successful installation of [libtheora-dev]  apt: Failed to detect successful installation of [python-psutil]  apt: Failed to detect successful installation of [libboost-all-dev]  apt: Failed to detect successful installation of [libpoco-dev]  apt: Failed to detect successful installation of [libfltk1.1-dev]  apt: Failed to detect successful installation of [python-pygraphviz]  apt: Failed to detect successful installation of [libqhull-dev]  apt: Failed to detect successful installation of [python-pydot]  apt: Failed to detect successful installation of [libjpeg62-turbo-dev]  apt: Failed to detect successful installation of [libtool-bin]
```

下一步就是编译源码了：

在catkin目录下执行：

```text
./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release
```

编译报错：

```text
==> Processing catkin package: 'gazebo_ros'==> Building with env: '/home/pi/ros_catkin_ws/install_isolated/env.sh'==> cmake /home/pi/ros_catkin_ws/src/gazebo_ros_pkgs/gazebo_ros -DCATKIN_DEVEL_PREFIX=/home/pi/ros_catkin_ws/devel_isolated/gazebo_ros -DCMAKE_INSTALL_PREFIX=/home/pi/ros_catkin_ws/install_isolated -DCMAKE_BUILD_TYPE=Release -G Unix Makefiles in '/home/pi/ros_catkin_ws/build_isolated/gazebo_ros'-- Using CATKIN_DEVEL_PREFIX: /home/pi/ros_catkin_ws/devel_isolated/gazebo_ros-- Using CMAKE_PREFIX_PATH: /home/pi/ros_catkin_ws/install_isolated-- This workspace overlays: /home/pi/ros_catkin_ws/install_isolated-- Using PYTHON_EXECUTABLE: /usr/bin/python-- Using Debian Python package layout-- Using empy: /usr/bin/empy-- Using CATKIN_ENABLE_TESTING: ON-- Call enable_testing()-- Using CATKIN_TEST_RESULTS_DIR: /home/pi/ros_catkin_ws/build_isolated/gazebo_ros/test_results-- Found gtest sources under '/usr/src/gtest': gtests will be built-- Using Python nosetests: /usr/bin/nosetests-2.7-- catkin 0.6.18-- Using these message generators: gencpp;genlisp;genpyCMake Error at CMakeLists.txt:26 (find_package):  By not providing "Findgazebo.cmake" in CMAKE_MODULE_PATH this project has  asked CMake to find a package configuration file provided by "gazebo", but  CMake did not find one.  Could not find a package configuration file provided by "gazebo" with any  of the following names:    gazeboConfig.cmake    gazebo-config.cmake  Add the installation prefix of "gazebo" to CMAKE_PREFIX_PATH or set  "gazebo_DIR" to a directory containing one of the above files.  If "gazebo"  provides a separate development package or SDK, be sure it has been  installed.-- Configuring incomplete, errors occurred!See also "/home/pi/ros_catkin_ws/build_isolated/gazebo_ros/CMakeFiles/CMakeOutput.log".See also "/home/pi/ros_catkin_ws/build_isolated/gazebo_ros/CMakeFiles/CMakeError.log".<== Failed to process package 'gazebo_ros':   Command '['/home/pi/ros_catkin_ws/install_isolated/env.sh', 'cmake', '/home/pi/ros_catkin_ws/src/gazebo_ros_pkgs/gazebo_ros', '-DCATKIN_DEVEL_PREFIX=/home/pi/ros_catkin_ws/devel_isolated/gazebo_ros', '-DCMAKE_INSTALL_PREFIX=/home/pi/ros_catkin_ws/install_isolated', '-DCMAKE_BUILD_TYPE=Release', '-G', 'Unix Makefiles']' returned non-zero exit status 1Reproduce this error by running:==> cd /home/pi/ros_catkin_ws/build_isolated/gazebo_ros && /home/pi/ros_catkin_ws/install_isolated/env.sh cmake /home/pi/ros_catkin_ws/src/gazebo_ros_pkgs/gazebo_ros -DCATKIN_DEVEL_PREFIX=/home/pi/ros_catkin_ws/devel_isolated/gazebo_ros -DCMAKE_INSTALL_PREFIX=/home/pi/ros_catkin_ws/install_isolated -DCMAKE_BUILD_TYPE=Release -G 'Unix Makefiles'Command failed, exiting.
```

因为没有安装gazebo的库，所以无法编译通过。这个库又因为认证原因无法安装，所以我们先把这个包取出工程外，再次编译试试。

还有其他依赖问题。。。。

===========================================================================================================

在PC上安装，遇到下面包无法编译问题：

```text
==> Processing catkin package: 'pcl_conversions'==> Building with env: '/home/pct/ros_catkin_ws/install_isolated/env.sh'==> cmake /home/pct/ros_catkin_ws/src/pcl_conversions -DCATKIN_DEVEL_PREFIX=/home/pct/ros_catkin_ws/devel_isolated/pcl_conversions -DCMAKE_INSTALL_PREFIX=/home/pct/ros_catkin_ws/install_isolated -G Unix Makefiles in '/home/pct/ros_catkin_ws/build_isolated/pcl_conversions'-- Using CATKIN_DEVEL_PREFIX: /home/pct/ros_catkin_ws/devel_isolated/pcl_conversions-- Using CMAKE_PREFIX_PATH: /home/pct/ros_catkin_ws/install_isolated-- This workspace overlays: /home/pct/ros_catkin_ws/install_isolated-- Using PYTHON_EXECUTABLE: /usr/bin/python-- Using Debian Python package layout-- Using empy: /usr/bin/empy-- Using CATKIN_ENABLE_TESTING: ON-- Call enable_testing()-- Using CATKIN_TEST_RESULTS_DIR: /home/pct/ros_catkin_ws/build_isolated/pcl_conversions/test_results-- Found gtest sources under '/usr/src/gtest': gtests will be built-- Using Python nosetests: /usr/bin/nosetests-2.7-- catkin 0.6.18CMake Error at CMakeLists.txt:6 (find_package):  By not providing "FindPCL.cmake" in CMAKE_MODULE_PATH this project has  asked CMake to find a package configuration file provided by "PCL", but  CMake did not find one.  Could not find a package configuration file provided by "PCL" with any of  the following names:    PCLConfig.cmake    pcl-config.cmake  Add the installation prefix of "PCL" to CMAKE_PREFIX_PATH or set "PCL_DIR"  to a directory containing one of the above files.  If "PCL" provides a  separate development package or SDK, be sure it has been installed.-- Configuring incomplete, errors occurred!See also "/home/pct/ros_catkin_ws/build_isolated/pcl_conversions/CMakeFiles/CMakeOutput.log".See also "/home/pct/ros_catkin_ws/build_isolated/pcl_conversions/CMakeFiles/CMakeError.log".<== Failed to process package 'pcl_conversions':   Command '['/home/pct/ros_catkin_ws/install_isolated/env.sh', 'cmake', '/home/pct/ros_catkin_ws/src/pcl_conversions', '-DCATKIN_DEVEL_PREFIX=/home/pct/ros_catkin_ws/devel_isolated/pcl_conversions', '-DCMAKE_INSTALL_PREFIX=/home/pct/ros_catkin_ws/install_isolated', '-G', 'Unix Makefiles']' returned non-zero exit status 1Reproduce this error by running:==> cd /home/pct/ros_catkin_ws/build_isolated/pcl_conversions && /home/pct/ros_catkin_ws/install_isolated/env.sh cmake /home/pct/ros_catkin_ws/src/pcl_conversions -DCATKIN_DEVEL_PREFIX=/home/pct/ros_catkin_ws/devel_isolated/pcl_conversions -DCMAKE_INSTALL_PREFIX=/home/pct/ros_catkin_ws/install_isolated -G 'Unix Makefiles'Command failed, exiting.
```

有四个包无法编译通过：

gazebo\_plugins gazebo\_ros pcl\_ros pcl\_conversions   


