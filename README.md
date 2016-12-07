# IMPORTANT

After this `kinova-ros` release, the previous ROS release, which mainly developed for jaco arm will be named as "jaco-ros"; and the previous "master" branch is renamed as "jaco-ros-master" branch. Users can keep both "jaco-ros" and new release "kinova-ros" as two parallel stacks. However, further update and support will be only available on "kinova-ros".


# KINOVA-ROS

The `kinova-ros` stack provides a ROS interface for the Kinova Robotics JACO, JACO2 and MICO robotic manipulator arms, and it is built to support further kinova products as well. Besides  widely support of Kinova products, there are many bug fixing, improvements and new features as well. The stack is developped upon the Kinova C++ API functions, which communicates with the DSP inside robot base. 

## Supported versions
The recommended configuration is ROS Indigo with 64 bit Ubuntu 14.04.

The package may work with other configurations as well but it only tested for the one recommended above. 

## file system
 - kinova_bringup: launch file to start kinova_driver and apply some configurations
 - kinova_driver: most essential files to run kinova-ros stack. Under the include folder, Kinova C++ API headers are defined in ../indlude/kinova, and ROS package header files are in kinova_driver folder. kinova_api source file is a wrap of Kinova C++ API, and kinova_comm builds up the fundamental functions. Some advanced accesses regarding to force/torque control are only provided in kinova_api.  Most parameters and topics are created in kinova_arm. A generous archeteture from low level up could be: 
    DSP --communicate--> Kinova C++ API --wrapped--> kinova_api --> kinova_comm 
    --> {kinova_arm; kinova_fingers_action; kinova_joint_angles_action; ...} --> kinova_arm_driver. **It is not recommaned to modify kinova_comm and any level below.** 

 - kinova_demo: python scripts for actionlibs in joint space and cartesian space.
 - kinova_msgs: all the messages, servers and actionlib format are defined here.
 - kinova_description: robot urdf models and meshes are stored here. display_kinova_robot.launch can run without having a robot.
 - kinova_docs: kinova_comm reference html files generated by doxygen. The comments are based on the reference of Kinova C++ API, and some additional information is provided. The documents of Kinova C++ API are automatically installed while installing Kinova SDK from Kinova website "http://www.kinovarobotics.com/service-robotics/products/software/"

## Installation
To make kinova-ros part of your workspace, follow these steps (assuming your workspace is setup following the standard conventions):

    cd ~/catkin_ws/src
    git clone https://github.com/Kinovarobotics/kinova-ros.git kinova-ros
    cd ~/catkin_ws
    catkin_make

To access the arm via usb copy the udev rule file `10-kinova-arm.rules` from `~/catkin_ws/src/kinova-ros/kinova_driver/udev` to `/etc/udev/rules.d/`:

    sudo cp kinova_driver/udev/10-kinova-arm.rules /etc/udev/rules.d/

## How to use the stack

### launch driver
`kinova_robot.launch` in kinova_bringup folder launches the essential drivers and configurations for kinova robots. kinova_robot.launch has two arguments:

**kinova_robotType** specifies which robot type is used. For better supporting wider range of robot configurations,  *robot type* is defined by a `char[8]`, in the format of: `[{j|m|r|c}{1|2}{s|n}{4|6|7}{s|a}{2|3}{0}{0}]`. *Robot category* `{j|m|r|c}` refers to *jaco*, *mico*, *roco* and *customized*, *version* is `{1|2}` for now, *wrist type* `{s|n}` can be spherical or *non-spherical*, *Degree of Freedom* is possible to be `{4|6|7}`, *robot mode* `{s|a}` can be in *service* or *assistive*, *robot hand* `{2|3}` may equipped with *2 fingers* or *3 fingers* gripper. Last two positions are *undifined* and *reserved* for further features.

*eg*: `j2n6a300` (default value) refers to *jaco v2 6DOF assistive 3 fingers*. Please be aware that not all options are valided for different robot types.

**use_urdf** specifies whether the kinematic solution is provided by the URDF model. 

When `use_urdf:=true` (default value), the kinematic solution is automatically solved by URDF model. 
The robot can be virtually presented in the Rviz and the frames in Rviz are located at each joints. 
To visulize the robot in Rviz, run `$ rosrun rviz rviz`, and select *root* as the world frame. 
The robot model will synchronize the motion with the real robot.

If `use_urdf:=false`, the kinematic solution is as ame as the DSP code inside robot. 
Node `kinova_tf_updater` will be activated to publish frames, and the frames are defined 
according the classic D-H converntion(frame may not locat at joints). Even you are not able to visulize
the robot properly in Rviz, you are able to observe the D-H frames in Rviz.

*eg*: `roslaunch kinova_bringup kinova_robot.launch kinova_robotType:=m1n4a200 use_urdf:=true`

If the robot is not able to move after boot, please try to home the arm by either pressing *home* button on the joystick or calling rosservice in the **ROS service commands** below.

### Joint position control
Joint position control can be realized by calling KinovaComm::setJointAngles() in customized node, or you may simply call the node `joints_action_client.py` in the kinova_demo package. Help information is availabe with `-h` option. The joint position can be commanded by `{degree | radian}`, relative or absolute value by option `-r`. The following code will drive the 4th joint of a 4DOF mico robot rotate +10 degree (not to 10 degree), and print additional information about the joint position.

*eg*: `rosrun kinova_demo joints_action_client.py -v -r m1n4a200 degree -- 0 0 0 10`

Joint position can be observed by echoing two topics:
`/'${kinova_robotType}_driver'/out/joint_angles` (in degree) and 
`/'${kinova_robotType}_driver'/out/state/position` (in radians including finger information)

 *eg*: `rostopic echo -c /m1n4a200_driver/out/joint_state` will print out joint names, velocity and effort information. However, the effort is a place holder for further verstion.


 Another way to control joint position is to use interactive markers in Rviz. Please follow the steps below to active interactive control:
  - launch the drivers: roslaunch kinova_bringup kinova_robot.launch kinova_robotType:=m1n4a200
  - start the node of interactive conrol: rosrun kinova_driver kinova_interactive_control m1n4a200
  - open Rviz: rosrun rviz rviz
  - On left plane of Rviz, *Add* *InteractiveMarkers*, click on the right of *Updated Topic* of added interactive marker, and select the topic */m1n4a200_interactive_control_Joint/update*
  - Now a ring should appear at each joint location, and you can move the robot by drag the rings.


### Cartesian position control
Cartesian position control can be realized by calling KinovaComm::setCartesianPosition() in customized node, or you may simply call the node `pose_action_client.py` in the kinova_demo package. Help information is availabe with `-h` option. The unit of position command can be by `{mq | mdeg | mrad}`, which refers to meter&Quaternion, meter&degree and meter&radian. The unit of position is always meter, and the unit of orientation is different. Degree and radian are regarding to Euler Angles in XYZ order. Please be aware that the length of parameters are different when use Quaternion and Euler Angles. With the option `-v` on, positions in other unit format are printed for convenience. The following code will drive a mico robot to move along +x axis for 1cm and rotate hand for +10 degree along hand axis. The last second *10* will be ignored since 4DOF robot cannot rotate along y axis.

*eg*: `rosrun kinova_demo pose_action_client.py -v -r m1n4a200 mdeg -- 0.01 0 0 0 10 10`

The Cartesian coordinate of robot root frame is defined by the following rules:
- origin is the intersection point of the bottom plane of the base and cylinder center line.    
- +x axis is directing to the left when facing the base panel (where power switch and cable socket locate).
- +y axis is towards to user when facing the base panel.
- +z axis is upwards when robot is standing on a flat surface.

The current Cartesian position is published via topic: `/'${kinova_robotType}_driver'/out/tool_pose`
In addition, the wrench of end-effector is published via topic: `/'${kinova_robotType}_driver'/out/tool_wrench`


 Another way to control Cartesian position is to use interactive markers in Rviz. Please follow the steps below to active interactive control:
  - launch the drivers: roslaunch kinova_bringup kinova_robot.launch kinova_robotType:=m1n4a200
  - start the node of interactive conrol: rosrun kinova_driver kinova_interactive_control m1n4a200
  - open Rviz: rosrun rviz rviz
  - On left plane of Rviz, *Add* *InteractiveMarkers*, click on the right of *Updated Topic* of added interactive marker, and select the topic */m1n4a200_interactive_control_Cart/update*
  - Now a cubic with 3 axis (translation) and 3 rings(rotation) should appear at the end-effector, and you can move the robot by drag the axis or rings.


### Finger position control
Cartesian position control can be realized by calling KinovaComm::setFingerPositions() in customized node, or you may simply call the node `fingers_action_client.py` in the kinova_demo package. Help information is availabe with `-h` option. The unit of finger command can be by `{turn | mm | percent}`, which refers to turn of motor, milimeter and percentage. The finger is essentially controlled by `turn`, and the rest units are propotional to `turn` for convenience. The value 0 indicates fully open, while *finger_maxTurn* represents a fully close. The value of *finger_maxTurn* may vary due to many factors. A proper reference value for finger turn will be 0 (fully-open) to 6400 (fully-close)  If necessary, please modify this variable in the code. With the option `-v` on, positions in other unit format are printed for convenience. The following code fully close the fingers.

*eg*: `rosrun kinova_demo fingers_action_client.py m1n4a200 percent -- 100 100 `

The finger position is published via topic: `/'${kinova_robotType}_driver'/out/finger_position`

### Velocity Control (joint space and Cartesian space)
The user have access to both joint velocity and Cartesian velocity (linear velocity and angular velocity). The joint velocity control can be realized by publishing to topic  `/'${kinova_robotType}_driver'/in/joint_velocity`. The following command can move the 4th joint of a mico robot at a rate of approximate 10 degree/second. Please be aware that the publishing rate **dose** effect the motion speed much.

*eg*: `rostopic pub -r 100 /m1n4a200_driver/in/joint_velocity kinova_msgs/JointVelocity "{joint1: 0.0, joint2: 0.0, joint3: 0.0, joint4: 10.0}" ` 

For Cartesian linear velocity, the unit is meter/second. Definition of angular velocity "Omega" is based on the skew-symmetric matrices "S = R*R^(-1)", where "R" is the rotation matrix. angular velocity vector "Omega = [S(3,2); S(1,3); S(2,1)]". The unit is radian/second.  An example is given below:

*eg*: `rostopic pub -r 100 /m1n4a200_driver/in/cartesian_velocity kinova_msgs/PoseVelocity "{twist_linear_x: 0.0, twist_linear_y: 0.0, twist_linear_z: 0.0, twist_angular_x: 0.0, twist_angular_y: 0.0, twist_angular_z: 10.0}" `

The motion will stop once the publish on the topic is finished. Please be caution when use velocity control as it is a continuous motion unless you stopped it.

** Note on publish frequency **
The joint velocity is set to publish at a frequency of 100Hz, due to the DSP inside the robot loops each 10ms. Any higher frequency will not have any influence to the speed. However, it will fills up a buffer (size of 2000) and the robot may continue to move a bit even stops receiving velocity topic. For a frequency lower than 100Hz, the robot will not able to achieve the requested velocity.

Therefore, the publishing rate at 100Hz is not an optional argument, but a requirement.

### ROS service commands
User can home the robot by the command below. It takes no argument and bring robot to pre-defined home position. The command support customized home position that user defined by SDK or JacoSoft as well.
`/'${kinova_robotType}_driver'/in/home_arm`

User can also enable and disable the ROS motion command via rosservice 
`/'${kinova_robotType}_driver'/in/start`
and `/'${kinova_robotType}_driver'/in/stop`. When `stop` is called, robot command from ROS will not able to drive robot until `start` is called. However, joystick still has the control during this phase.

### Cartesian Admittance mode (User can control the robot by manually guiding it by hand) 
The admittance force control can be actived by command 
`rosservice call /'${kinova_robotType}_driver'/in/start_force_control` and disabled by `rosservice call /'${kinova_robotType}_driver'/in/stop_force_control`. The user is able to move the robot by appling force/torque to the end-effector/joints. When there is a Cartesian/joint position command, the result motion will be a combination of both force and position command.

#### Re-calibrate torque sensors
Over time it is possible that the torque sensors develop offsets in reporting absolute torque. For this they need to be re-calibrated. The calibration process is very simple -   
1. Move the robot to candle like pose (all joints 180 deg, robot links points straight up), this configuration ensures zero torques at joints.  
2. Call the service 'rosservice call /'${kinova_robotType}_driver'/in/set_zero_torques'

### Support for 7 dof spherical wrist robot
#### new in release 1.1 
Support for the 7 dof robot has been added in this new release. All of the previous control methods can be used on a 7 dof Kinova robot.

##### Inverse Kinematics for 7 dof robot
The inverse kinematics of the 7 dof robot results infinite possible solutions for a give pose command. The choice of the best solution (redundancy resolution) is done in the base of the robot considering criteria such as joint limits, closeness to singularities.

##### Move robot in Null space
To see the full set of solutions, a new fuction is introduced in KinovaAPI - StartRedundantJointNullSpaceMotion(). When in this mode the Kinova joystick can be used to move the robot in null space while keeping the end-effector maintaining its pose.

The mode can be activated by calling the service SetNullSpaceModeState - ${kinova_robotType}_driver'/in/set_null_space_mode_state. 
Pass 1 to service to enable and 0 to disable.

### Torque control 
#### new in release 1.1 
Torque control has been made more accessible. Now you can publish torque/force commands just like joint/cartesian velocity. To do this you need to :

1. Optional - Set torque parameters  
Usually default parameters should work for most applications. But if you need to change some torque parameters, you can set parameters (listed at the end of page) and then call the service -   
SetTorqueControlParameters '${kinova_robotType}_driver/in/set_torque_control_parameters'

2. Switch to torque control from position control  
You can do this using the service  - SetTorqueControlMode '${kinova_robotType}_driver'/in/set_torque_control_mode'

3. Publish torque commands rostopic pub -r 100 /j2n6s300_driver/in/joint_torque kinova_msgs/JointTorque "{joint1: 0.0, joint2: 0.0, joint3: 0.0, joint4: 0.0, joint5: 0.0, joint6: 1.0}"

#### Torque inactivity
If not torque command is sent after a given
time (250ms by default), the controller will take an action: (0): The robot will return in position
mode (1): The torque commands will be set to zero. By default, option (1) is set for Kinova classic robots
(Jaco2 and Mico) while option (0) is set for generic mode.

## Ethernet connection
#### new in release 1.1 
Support for Ethernet connection has been added. All functionalities available in USB are available in Ethernet. 
To use ethernet just set the parameter 

connection_type: ethernet

## Parameters
#### new in release 1.1 
##### General parameters
* serial_number: PJ00000001030703130  
  leave commented out if you want to control the first robot found connected.  
* jointSpeedLimitParameter1: 10  
  Joint speed limit for joints 1, 2, 3 in deg/s
* jointSpeedLimitParameter2: 20  
  Joint speed limit for joints 4, 5, 6 in deg/s
* payload: [0, 0, 0, 0]  
  payload: [COM COMx COMy COMz] in [kg m m m]  
* connection_type: USB  
  ethernet or USB
##### Ethernet connection parameters
ethernet:

* local_machine_IP: 192.168.100.21,  
* subnet_mask: 255.255.255.0,  
* local_cmd_port: 25000,  
* local_broadcast_port: 25025  


##### Torque control parameters
Comment these out to use default values.

torque_parameters:

* torque_min: [1, 0, 0, 0, 0, 0, 0]  
* torque_max: [50, 0, 0, 0, 0, 0, 0]  
  If one torque min/max value is sepecified, all min/max values need to be specified  
* safety_factor: 1  
  Decides velocity threshold at which robot switches torque to position control (between 0 and 1)  
* com_parameters: [0,0,0,0,0,0,0, 0,0,0,0,0,0,0, 0,0,0,0,0,0,0, 0,0,0,0,0,0,0]  
  COM parameters, order [m1,m2,...,m7,x1,x2,...,x7,y1,y2,...y7,z1,z2,...z7]
  
## What's new comparison to JACO-ROS

- migrate from jaco to kinova in the scope of: file names, class names, function names, data type, node, topic, etc.
- apply kinova_RobotType for widely support
- Re-define JointAngles for consistence
- updated API version with new features
- Create transform between different Euler Angle definitions in DSP and ROS
- Criteron check when if reach the goal
- URDF models for all robotTypes
- Interactive for joint control
- New message for KinovaPose
- More options for actionlibs arguments, etc.
- Relative motion control
- Kinematic solution to be consistant with robot base code.
- Fix joint offset bug for joint2 and joint6
- Fix joint velocity control and position velocity control


## Notes and Limitations
1. Force/torque control is only for advanced users. Please be caution when using force/torque control api functions.

2. The ``joint_state`` topic currently reports only the arm position and
velocity. Effort is a placeholder for future compatibility. Depending on your
firmware version velocity values can be wrong. 

3. When updating the firmware on the arm (e.g., using Jacosoft) the serial number will be set to "Not set" which will cause multiple arms to be unusable. The solution is to make sure that the serial number is reset after updating the arm firmware.

4. Some virtualization software products are known to work well with this package, while others do not.  The issue appears to be related to proper handover of access to the USB port to the API.  Parallels and VMWare are able to do this properly, while VirtualBox causes the API to fail with a "1015" error.

5. Previously, files under ``kinova-ros/kinova_driver/lib/i386-linux-gnu`` had a bug which required uses 32-bit systems to manually copy them into devel or install to work. This package has not been tested with 32-bit systems and this workaround may still be required. 64-bit versions seem to be unaffected.


## Report a Bug
Any bugs, issues or suggestions may be sent to ros@kinovarobotics.com.


