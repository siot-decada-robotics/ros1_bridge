# Bridge communication between ROS 1 and ROS 2

This package provides a network bridge which enables the exchange of messages between ROS 1 and ROS 2.

The bridge is currently implemented in C++ as at the time the Python API for ROS 2 had not been developed.
Because of this its support is limited to only the message/service types available at compile time of the bridge.
The bridge provided with the prebuilt ROS 2 binaries includes support for common ROS interfaces (messages/services), such as the interface packages listed in the [ros2/common_interfaces repository](https://github.com/ros2/common_interfaces) and `tf2_msgs`.
See [the documentation](doc/index.rst) for more details on how ROS 1 and ROS 2 interfaces are associated with each other.
If you would like to use a bridge with other interfaces (including your own custom types), you will have to build the bridge from source (instructions below), after building and sourcing your custom types in separate ROS 1 and ROS 2 workspaces.
See [the documentation](doc/index.rst) for an example setup.

For efficiency reasons, topics will only be bridged when matching publisher-subscriber pairs are active for a topic on either side of the bridge.
As a result using `ros2 topic echo <topic-name>` doesn't work but fails with an error message `Could not determine the type for the passed topic` if no other subscribers are present since the dynamic bridge hasn't bridged the topic yet.
As a workaround the topic type can be specified explicitly `ros2 topic echo <topic-name> <topic-type>` which triggers the bridging of the topic since the `echo` command represents the necessary subscriber.
On the ROS 1 side `rostopic echo` doesn't have an option to specify the topic type explicitly.
Therefore it can't be used with the dynamic bridge if no other subscribers are present.
As an alternative you can use the `--bridge-all-2to1-topics` option to bridge all ROS 2 topics to ROS 1 so that tools such as `rostopic echo`, `rostopic list` and `rqt` will see the topics even if there are no matching ROS 1 subscribers.
Run `ros2 run ros1_bridge dynamic_bridge -- --help` for more options.

## Prerequisites

In order to run the bridge you need to either:

* get [prebuilt binaries](https://github.com/ros2/ros2/releases) or
* build the bridge as well as the other ROS 2 packages from source.

After that you can run both examples described below.

For all examples you need to source the environment of the install space where the bridge was built or unpacked to.
Additionally you will need to either source the ROS 1 environment or at least set the `ROS_MASTER_URI` and run a `roscore`.

The following ROS 1 packages are required to build and use the bridge:
* `catkin`
* `roscpp`
* `roslaunch` (for `roscore` executable)
* `rosmsg`
* `std_msgs`
* as well as the Python package `rospkg`

To run the following examples you will also need these ROS 1 packages:
* `rosbash` (for `rosrun` executable)
* `roscpp_tutorials`
* `rospy_tutorials`
* `rostopic`
* `rqt_image_view`

You will also have to clone the updated ros perception plugins from the 

### Building the bridge from source

Before continuing you should have the prerequisites for building ROS 2 from source installed following [these instructions](https://github.com/ros2/ros2/wiki/Installation).

In the past, building this package required patches to ROS 1, but in the latest releases that is no longer the case.
If you run into trouble first make sure you have at least version `1.11.16` of `ros_comm` and `rosbag`.

The bridge uses `pkg-config` to find ROS 1 packages.
ROS 2 packages are found through CMake using `find_package()`.
Therefore the `CMAKE_PREFIX_PATH` must not contain paths from ROS 1 which would overlay ROS 2 packages.

Here are the steps for Linux and OSX.

First clone the ros_perception_plugins into your src folder in your bridge workspace and checkout the the following commit:

```
git clone -b ros2 https://github.com/ros-perception/image_transport_plugins.git
cd image_transport_plugins
git checkout 7cedbf04d5deb3b183d92756b912816ce7618aca
```

After which build the workspace with foxy.

You should first build everything but the ROS 1 bridge with normal colcon arguments.
We don't recommend having your ROS 1 environment sourced during this step as it can add other libraries to the path.

```
cd ~/ros_bridge_ws
source /opt/ros/foxy/setup.bash
colcon build --symlink-install --packages-skip ros1_bridge
```
Next, you need to source the workspace that you just built in which you skipped the ros1_bridge:
```
cd ~/ros_bridge_ws
source install/setup.bash
```
Next you need to source the ROS 1 environment, for Linux and ROS Melodic that would be:

```
source /opt/ros/melodic/setup.bash
```

Then build just the ROS 1 bridge:

```
cd ~/ros_bridge_ws/src
git clone https://github.com/siot-decada-robotics/ros1_bridge.git 
cd ~/ros_bridge_ws
colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
```

*Note:* If you are building on a memory constrained system you might want to limit the number of parallel jobs by setting e.g. the environment variable `MAKEFLAGS=-j1`.


## Example 1: run the bridge and the example talker and listener

The talker and listener can be either a ROS 1 or a ROS 2 node.
The bridge will pass the message along transparently.

*Note:* When you are running these demos make sure to only source the indicated workspaces.
You will get errors from most tools if they have both workspaces in their environment.


### Example 1a: ROS 1 talker and ROS 2 listener

First we start a ROS 1 `roscore`:

```
# Shell A (ROS 1 only):
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
roscore
```

---

Then we start the dynamic bridge which will watch the available ROS 1 and ROS 2 topics.
Once a *matching* topic has been detected it starts to bridge the messages on this topic.

```
# Shell B (ROS 1 + ROS 2):
# Source ROS 1 first:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
# Source ROS 2 next:
. <install-space-with-bridge>/setup.bash
# For example:
# . /opt/ros/dashing/setup.bash
export ROS_MASTER_URI=http://localhost:11311
ros2 run ros1_bridge dynamic_bridge
```

The program will start outputting the currently available topics in ROS 1 and ROS 2 in a regular interval.

---

Now we start the ROS 1 talker.

```
# Shell C:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
rosrun rospy_tutorials talker
```

The ROS 1 node will start printing the published messages to the console.

---

Now we start the ROS 2 listener from the `demo_nodes_cpp` ROS 2 package.

```
# Shell D:
. <install-space-with-ros2>/setup.bash
ros2 run demo_nodes_cpp listener
```
(btw samuel says this might not work -  some error about symbol lookup error)

The ROS 2 node will start printing the received messages to the console.

When looking at the output in *shell B* there will be a line stating that the bridge for this topic has been created:

```
created 1to2 bridge for topic '/chatter' with ROS 1 type 'std_msgs/String' and ROS 2 type 'std_msgs/String'
```

At the end stop all programs with `Ctrl-C`.
Once you stop either the talker or the listener in *shell B* a line will be stating that the bridge has been torn down:

```
removed 1to2 bridge for topic '/chatter'
```

The screenshot shows all the shell windows and their expected content:

![ROS 1 talker and ROS 2 listener](doc/ros1_talker_ros2_listener.png)


### Example 1b: ROS 2 talker and ROS 1 listener

The steps are very similar to the previous example and therefore only the commands are described.

```
# Shell A:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
roscore
```

---

```
# Shell B:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
. <install-space-with-bridge>/setup.bash
export ROS_MASTER_URI=http://localhost:11311
ros2 run ros1_bridge dynamic_bridge
```

---

Now we start the ROS 2 talker from the `demo_nodes_py` ROS 2 package.

```
# Shell C:
. <install-space-with-ros2>/setup.bash
ros2 run demo_nodes_py talker
```

---

Now we start the ROS 1 listener.

```
# Shell D:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
rosrun roscpp_tutorials listener
```


## Example 2: run the bridge and exchange images

The second example will demonstrate the bridge passing along bigger and more complicated messages.
A ROS 2 node is publishing images retrieved from a camera and on the ROS 1 side we use `rqt_image_view` to render the images in a GUI.
And a ROS 1 publisher can send a message to toggle an option in the ROS 2 node.

First we start a ROS 1 `roscore` and the bridge:

```
# Shell A:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
roscore
```

```
# Shell B:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
. <workspace-with-bridge>/install/setup.bash
export ROS_MASTER_URI=http://localhost:11311
ros2 run ros1_bridge dynamic_bridge
```

---

Now we start the ROS 1 GUI:

```
# Shell C:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
rqt_image_view /image
```

--

Now we start the ROS 2 image publisher from the `image_tools` ROS 2 package:
```
# Shell D:
. <workspace-with-ros2>/install/setup.bash
ros2 run image_tools cam2image
```

You should see the current images in `rqt_image_view` which are coming from the ROS 2 node `cam2image` and are being passed along by the bridge.

--

To exercise the bridge in the opposite direction at the same time you can publish a message to the ROS 2 node from ROS 1.
By publishing either `true` or `false` to the `flip_image` topic, the camera node will conditionally flip the image before sending it.
You can either use the `Message Publisher` plugin in `rqt` to publish a `std_msgs/Bool` message on the topic `flip_image`, or run one of the two following `rostopic` commands:

```
# Shell E:
. /opt/ros/melodic/setup.bash
# Or, on OSX, something like:
# . ~/ros_catkin_ws/install_isolated/setup.bash
rostopic pub -r 1 /flip_image std_msgs/Bool "{data: true}"
rostopic pub -r 1 /flip_image std_msgs/Bool "{data: false}"
```

The screenshot shows all the shell windows and their expected content (it was taken when Indigo was supported - you should use Melodic):

![ROS 2 camera and ROS 1 rqt](doc/ros2_camera_ros1_rqt.png)

## Example 3: run the bridge for AddTwoInts service

In this example we will bridge a service TwoInts from
[ros/roscpp_tutorials](https://github.com/ros/ros_tutorials) and AddTwoInts from
[ros2/roscpp_examples](https://github.com/ros2/examples).

While building, ros1_bridge looks for all installed ROS and ROS2 services.
Found services are matched by comparing package name, service name and fields in a request and a response.
If all names are the same in ROS and ROS2 service, the bridge will be created.
It is also possible to pair services manually by creating a yaml file that will include names of corresponding services.
You can find more information [here](doc/index.rst).

So to make this example work, please make sure that the roscpp_tutorials package
is installed on your system and the environment is set up correctly while you build ros1_bridge.

Launch ROS master

```
# Shell A:
. <ros-install-dir>/setup.bash
roscore -p 11311
```

Launch dynamic_bridge:

```
# Shell B:
. <ros-install-dir>/setup.bash
. <ros2-install-dir>/setup.bash
export ROS_MASTER_URI=http://localhost:11311
ros2 run ros1_bridge dynamic_bridge
```

Launch TwoInts server:

```
# Shell C:
. <ros-install-dir>/setup.bash
export ROS_MASTER_URI=http://localhost:11311
rosrun roscpp_tutorials add_two_ints_server
```

Launch AddTwoInts client:

```
# Shell D:
. <ros2-install-dir>/setup.bash
ros2 run demo_nodes_cpp add_two_ints_client
```

## Example 3: run the bridge for AddTwoInts service

To get the rmf demonstrations running with a free fleet server and client in simulation in ros1, you have to start the simulations in ros1 (i.e. turtlebot3 office simulation or caato mbc simulation).

Then source your ros installations:
```
source /opt/ros/foxy/setup.bash
source /opt/ros/noetic/setup.bash
```

Then, source your workspace

```
cd ~/rosbridge_ws
source install/setup.bash
```

Then run the clock bridge

```
ros2 run ros1_bridge clock_bridge
```
