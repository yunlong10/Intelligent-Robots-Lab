# LAB7

## PART 1 Obstacle avoidance for robots

step 1:

```
$cd ~/smartcar_ws/src
$catkin_create_pkg smartcar_demo actionlib actionlib_msgs geometry_msgs interactive_markers nav_msgs rospy sensor_msgs std_msgs visualization_msgs
```

step 2:

```
$cd smartcar_demo
$mkdir scripts
$cd scripts
$touch smartcar_obstacle.py
```



smartcar_obstacle.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*
import rospy
import math
from sensor_msgs.msg import LaserScan
from geometry_msgs.msg import Twist
 
LINEAR_VEL = 0.20
STOP_DISTANCE = 2
LIDAR_ERROR = 0.6
SAFE_STOP_DISTANCE = STOP_DISTANCE + LIDAR_ERROR
 
class Obstacle():
    def __init__(self):
        self._cmd_pub = rospy.Publisher('cmd_vel', Twist, queue_size=1)
        self.obstacle()
        
    def get_scan(self):
        scan = rospy.wait_for_message('scan', LaserScan)   #从‘scan’ 画图读取消息类型laserscan 
        scan_filter = []
       
        samples = len(scan.ranges)  # The number of samples is defined in     返回列表元素个数
                                    # turtlebot3_<model>.gazebo.xacro file,
                                    # the default is 360.
 
        samples_view = 1            # 1 <= samples_view <= samples
        
        if samples_view > samples:  #samples 小于等于1
            samples_view = samples  
 
        if samples_view == 1:       
            scan_filter.append(scan.ranges[0])  #将雷达距离数据range【0】，添加到列表scanfilter中
 
        else:   #samples_view不等于1 ，大于1   ；小于1
            left_lidar_samples_ranges = -(samples_view//2 + samples_view % 2)
            right_lidar_samples_ranges = samples_view//2
            
            left_lidar_samples = scan.ranges[left_lidar_samples_ranges:]  #[m : ] 代表列表中的第m+1项到最后一项
            right_lidar_samples = scan.ranges[:right_lidar_samples_ranges] #[ : n] 代表列表中的第一项到第n项
            scan_filter.extend(left_lidar_samples + right_lidar_samples)  #extend() 函数用于在列表末尾一次性追加另一个序列中的多个值
 
        for i in range(samples_view):   # Python for循环可以遍历任何序列的项目，如一个列表或者一个字符串。
            if scan_filter[i] == float('Inf'):  #正负无穷
                scan_filter[i] = 3.5
            elif math.isnan(scan_filter[i]):   #是不是非数字浮点数
                scan_filter[i] = 0
        
        return scan_filter
 
    def obstacle(self):
        twist = Twist()      
        turtlebot_moving = True
 
        while not rospy.is_shutdown():
            lidar_distances = self.get_scan()
            min_distance = min(lidar_distances) 
            if min_distance < SAFE_STOP_DISTANCE:
                if turtlebot_moving:
                    twist.linear.x = 0.0
                    twist.angular.z = 0.0
                    self._cmd_pub.publish(twist)
                    turtlebot_moving = False
                    rospy.loginfo('Stop!')
            else:
                twist.linear.x = LINEAR_VEL
                twist.angular.z = 0.0
                self._cmd_pub.publish(twist)
                turtlebot_moving = True
                rospy.loginfo('Distance of the obstacle : %f', min_distance)
 
def main():
    rospy.init_node('smartcar_obstacle')
    try:
        obstacle = Obstacle()
    except rospy.ROSInterruptException:
        pass
 
if __name__ == '__main__':
    main()
 
```



1. In terminal

```
$roslaunch smartcar_gazebo smartcar_with_laser_nav.launch
```



2. In new terminal

```
$rosrun smartcar_demo smartcar_obstacle.py
```



### smartcar explore maze

```
$cd ~/smartcar_ws/src/smartcar_demo/scripts
$touch maze_explorer.py
```

maze_explorer.py

```python
#!/usr/bin/env python3

import rospy, time
from geometry_msgs.msg import Twist
from sensor_msgs.msg import LaserScan

def scan_callback(msg):
    global range_front
    global range_right
    global range_left
    global ranges
    global min_front,i_front, min_right,i_right, min_left ,i_left
    
    ranges = msg.ranges
    # get the range of a few points
    # in front of the robot (between 5 to -5 degrees)
    range_front[:5] = msg.ranges[5:0:-1]  
    range_front[5:] = msg.ranges[-1:-5:-1]
    # to the right (between 300 to 345 degrees)
    range_right = msg.ranges[300:345]
    # to the left (between 15 to 60 degrees)
    range_left = msg.ranges[60:15:-1]
    # get the minimum values of each range 
    # minimum value means the shortest obstacle from the robot
    min_range,i_range = min( (ranges[i_range],i_range) for i_range in range(len(ranges)) )
    min_front,i_front = min( (range_front[i_front],i_front) for i_front in range(len(range_front)) )
    min_right,i_right = min( (range_right[i_right],i_right) for i_right in range(len(range_right)) )
    min_left ,i_left  = min( (range_left [i_left ],i_left ) for i_left  in range(len(range_left )) )
    

# Initialize all variables
range_front = []
range_right = []
range_left  = []
min_front = 0
i_front = 0
min_right = 0
i_right = 0
min_left = 0
i_left = 0

# Create the node
cmd_vel_pub = rospy.Publisher('cmd_vel', Twist, queue_size = 1) # to move the robot
scan_sub = rospy.Subscriber('scan', LaserScan, scan_callback)   # to read the laser scanner
rospy.init_node('maze_explorer')

command = Twist()
command.linear.x = 0.0
command.angular.z = 0.0
        
rate = rospy.Rate(10)
time.sleep(1) # wait for node to initialize

near_wall = 0 # start with 0, when we get to a wall, change to 1

# Turn the robot right at the start
# to avoid the 'looping wall'
print("Turning...")
command.angular.z = -0.5
command.linear.x = 0.1
cmd_vel_pub.publish(command)
time.sleep(2)
       
while not rospy.is_shutdown():

    # The algorithm:
    # 1. Robot moves forward to be close to a wall
    # 2. Start following left wall.
    # 3. If too close to the left wall, reverse a bit to get away
    # 4. Otherwise, follow wall by zig-zagging along the wall
    # 5. If front is close to a wall, turn until clear
    while(near_wall == 0 and not rospy.is_shutdown()): #1
        print("Moving towards a wall.")
        if(min_front > 0.2 and min_right > 0.2 and min_left > 0.2):    
            command.angular.z = -0.1    # if nothing near, go forward
            command.linear.x = 0.15
            print ("C")
        elif(min_left < 0.2):           # if wall on left, start tracking
            near_wall = 1       
            print ("A")            
        else:
            command.angular.z = -0.25   # if not on left, turn right 
            command.linear.x = 0.0

        cmd_vel_pub.publish(command)
        
    else:   # left wall detected
        if(min_front > 0.2): #2
            if(min_left < 0.12):    #3
                print("Range: {:.2f}m - Too close. Backing up.".format(min_left))
                command.angular.z = -1.5
                command.linear.x = -0.07
            elif(min_left > 0.15):  #4
                print("Range: {:.2f}m - Wall-following; turn left.".format(min_left))
                command.angular.z = 1.5
                command.linear.x = 0.07
            else:
                print("Range: {:.2f}m - Wall-following; turn right.".format(min_left))
                command.angular.z = -1.5
                command.linear.x = 0.07
                
        else:   #5
            print("Front obstacle detected. Turning away.")
            command.angular.z = -1.0
            command.linear.x = 0.0
            cmd_vel_pub.publish(command)
            while(min_front < 0.3 and not rospy.is_shutdown()):      
                pass
        # publish command 
        cmd_vel_pub.publish(command)
    # wait for the loop
    rate.sleep()
```

1. In terminal

```
$roslaunch smartcar_gazebo smartcar_with_laser_nav.launch
```



2. In new terminal

```
$rosrun smartcar_demo maze_explorer.py
```

 

## PART 2 ROS+ OPENCV

Install Opencv

```
$sudo apt-get install ros-noetic-vision-opencv libopencv-dev python-opencv
```



A simple example

```
$sudo apt-get install ros-melodic-usb-cam # 安装摄像头功能包
$roslaunch usb_cam usb_cam-test.launch # 启动功能包
$rqt_image_view # 可视化工具
```



step1:In your package.xml and CMakeLists.txt (or when you use catkin_create_pkg), add the
following dependencies:

```
$cd ~/smartcar_ws/src
$catkin_create_pkg learning_opencv sensor_msgs roscpp std_msgs image_transport
```

step2:Modify CMakeLists.txt
Add one line in CMakeLists.txt:

```
find_package(OpenCV REQUIRED)
```

step3: Create a node(write a cpp file)

```c++
#include <ros/ros.h>
#include <image_transport/image_transport.h>
#include <cv_bridge/cv_bridge.h>
#include <sensor_msgs/image_encodings.h>
#ifdef ROS_NOETIC
#include <opencv4/opencv2/imgproc/imgproc.hpp>
#include <opencv4/opencv2/highgui/highgui.hpp>
#else 
#include <opencv2/imgproc/imgproc.hpp>
#include <opencv2/highgui/highgui.hpp>
#endif
// Add vision object here
using namespace std;
using namespace cv;
static const std::string OPENCV_WINDOW="Image window";

class ImageConverter
{
  ros::NodeHandle nh_;
  image_transport::ImageTransport it_;
  image_transport::Subscriber image_sub_;
  image_transport::Publisher image_pub_;

public:
  ImageConverter()
    : it_(nh_)
  {
    // Subscrive to input video feed and publish output video feed
    image_sub_ = it_.subscribe("/usb_cam/image_raw", 1,
      &ImageConverter::imageCb, this);
    image_pub_ = it_.advertise("/image_converter/output_video", 1);

    cv::namedWindow(OPENCV_WINDOW);
    
  }

  ~ImageConverter()
  {
    cv::destroyWindow(OPENCV_WINDOW);
  }

  void imageCb(const sensor_msgs::ImageConstPtr& msg)
  {
    cv_bridge::CvImagePtr cv_ptr;
    try
    {
      cv_ptr = cv_bridge::toCvCopy(msg, sensor_msgs::image_encodings::BGR8);
    }
    catch (cv_bridge::Exception& e)
    {
      ROS_ERROR("cv_bridge exception: %s", e.what());
      return;
    }

    if (cv_ptr->image.rows >60&& cv_ptr->image.cols>60){
        cv::circle(cv_ptr->image,cv::Point(50,50),10,CV_RGB(255,0,0));
    }
  
    cv::imshow(OPENCV_WINDOW,cv_ptr->image);
    cv::waitKey(3);

    
    // Output modified video stream
    image_pub_.publish(cv_ptr->toImageMsg());
  }
};


int main(int argc, char** argv)
{
  ros::init(argc, argv, "image_converter");
  ImageConverter ic;
  ros::spin();
  return 0;
}

```

step4: Building node

Add two lines to the bottom of the CMakeLists.txt:

```
add_executable(image_converter src/image_converter.cpp)
target_link_libraries(image_converter ${catkin_LIBRARIES} ${OpenCV_LIBRARIES})
```

step5: Run to see the results

```
$source devel/setup.bash
$rosrun learning_opencv image_converter
```



## PART3 YOLO ROS

step1: Download code

```
cd ~/smartcar_ws/src
git clone --recursive https://github.com/leggedrobotics/darknet_ros
```

step2: Download weights

```
cd ~/smartcar_ws/src/darknet_ros/darknet_ros/yolo_network_config/weights/
wget http://pjreddie.com/media/files/yolov2.weights
wget http://pjreddie.com/media/files/yolov2-tiny.weights
wget https://pjreddie.com/media/files/yolov3.weights
wget https://pjreddie.com/media/files/yolov3-tiny.weights
```

step3: Building

```
cd ~/smartcar_ws
catkin_make -DCMAKE_BUILD_TYPE=Release
```

step4: Test

1. 发布图像话题, 直接将电脑自带摄像头或连接电脑的USB摄像头采集的图像发布为ROS图像话题并使用。

```
sudo apt-get install ros-noetic-usb-cam
roslaunch usb_cam usb_cam-test.launch
```

2. 运行darknet_ros

   执行darknet_ros进行检测，在运行检测之前需要更改一下文件，使得darknet_ros订阅的话题与usb_cam发布的图片话题对应。打开 darknet_ros/config/ros.yaml文件，找到

```
camera_reading:
	topic:/camera/rgb/image_raw
```

convert to

```
camera_reading:
	topic:/usb_cam/image_raw
```

3. 回到smartcar_ws目录，执行：

```
source devel/setup.bash
roslaunch darknet_ros darknet_ros.launch
```



### Yolo for smartcar

/smartcar_gazebo/smartcar_with_laser_nav.launch

```
<launch>
    <!-- 设置launch文件的参数 -->
    <arg name="world_name" value="$(find smartcar_gazebo)/world/pokemon_maze_cs405.world"/>
    <arg name="paused" default="false"/>
    <arg name="use_sim_time" default="true"/>
    <arg name="gui" default="true"/>
    <arg name="headless" default="false"/>
    <arg name="debug" default="false"/>
    <param name="use_gui" value="false"/>

    <!-- 运行gazebo仿真环境 -->
    <include file="$(find gazebo_ros)/launch/empty_world.launch">
        <arg name="world_name" value="$(arg world_name)"/>
        <arg name="debug" value="$(arg debug)" />
        <arg name="gui" value="$(arg gui)" />
        <arg name="paused" value="$(arg paused)"/>
        <arg name="use_sim_time" value="$(arg use_sim_time)"/>
        <arg name="headless" value="$(arg headless)"/>
    </include>

    <!-- 加载机器人模型描述参数 -->
    <arg name="model" default="$(find smartcar_description)/urdf/smartcar_with_laser_and_camera.gazebo.xacro"/> 
    <!-- <arg name="model" default="$(find smartcar_description)/urdf/test.xacro"/>  -->
    <param name="robot_description" command="$(find xacro)/xacro $(arg model)" /> 

    <!-- 运行joint_state_publisher节点，发布机器人的关节状态  -->
    <node name="joint_state_publisher_gui" pkg="joint_state_publisher_gui" type="joint_state_publisher_gui" />

    <!-- 运行robot_state_publisher节点，发布tf  -->
    <node name="robot_state_publisher" pkg="robot_state_publisher" type="robot_state_publisher"  output="screen" >
        <param name="publish_frequency" type="double" value="50.0" />
    </node>

    <!-- 在gazebo中加载机器人模型-->
    <node name="urdf_spawner" pkg="gazebo_ros" type="spawn_model" respawn="false" output="screen"
          args="-urdf -model mrobot -param robot_description"/> 

    <node name="rviz" pkg="rviz" type="rviz" args="-d $(find smartcar_gazebo)/rivz_config/smartcar_with_laser_nav.rviz" required="true" />
</launch>
```



```
camera_reading:
	topic:/usb_cam/image_raw
```

convert to

```
camera_reading:
	topic:/camera/image_raw
```

回到smartcar_ws目录，执行：

```
source devel/setup.bash
roslaunch smartcar_gazebo smartcar_with_laser_nav.launch
```

另开teminal:

```
source devel/setup.bash
roslaunch darknet_ros darknet_ros.launch
```

