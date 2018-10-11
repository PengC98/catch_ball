# ReadMe

+ by pc 2018.10.11 694977655@qq.com

# 摘要

本项目使用python完成，运行在ROS（机器人操作系统下） 用于控制bebop2扎红色气球的任务

# 如何使用

使用本代码只需要调整几个参数用于适应不同飞机，分别是P I D P1 I1 DI control

P I D 用于控制飞机左右旋转，用于纠正飞机相对气球的左右偏移

P1  I1 D1用于控制飞机上下移动，用于纠正飞机相对气球的上下偏移

P2 I2 D2 用于控制飞机前后移动，用于飞机靠近和远离气球

control 是飞机前部碳杆的长度

# 代码逻辑

1. 计算飞机摄像头焦距

   ```python
   image = cv2.imread("1.jpg") 
   #cv2.imshow("1",image)
   x,y,marker,center = find_ball(image) 
   
   global focalLength          
   focalLength = (2*marker * KNOWN_DISTANCE) / KNOWN_WIDTH 
   ```

2. 获取飞机图像话题消息并转换为opencv格式

   ```python
   cv_image=self.bridge.imgmsg_to_cv2(data,"bgr8")
   ```

3. 搜索气球并返回气球在图像上的大小

   ```python
   x,y,marker,center=find_ball(cv_image)
   ```

4. 计算气球与飞机间的直线距离

   ```python
   inches = distance_to_camera(KNOWN_WIDTH, focalLength, 2*marker)
   ```

5. 卡尔曼滤波器（滤波器参数也可自行调整 ）

   ```python
   r,d=KF(marker,inches)
   
   
   #卡尔曼滤波器参数 测量矩阵 转移矩阵 噪声协方差矩阵 测量误差矩阵
   kalman=cv2.KalmanFilter(4,2)
   kalman.measurementMatrix=np.array([[1,0,0,0],[0,1,0,0]],np.float32)
   kalman.transitionMatrix=np.array([[1,0,1,0],[0,1,0,1],[0,0,1,0],[0,0,0,1]],np.float32)
   kalman.processNoiseCov=np.array([[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]],np.float32)*0.03
   kalman.measurementNoiseCov=np.array([[1,0],[0,1]],np.float32)
   ```

6. PID计算（pid实现在代码src文件夹` PID.py`中）

   ```
   pid1.update(center[0]-cv_image.shape[1]/2)
   #print(center[1]-cv_image.shape[0]/2)
   pid2.update(center[1]-cv_image.shape[0]/2)
   pid3.update(control-inches)
   output_y=pid1.output
   output_z=pid2.output
   output_x=pid3.output
   ```

7. 向bebop2发送控制命令

   ```python
   twist = Twist()
   twist.linear.x = output_x; twist.linear.y = 0; twist.linear.z = output_z;
   twist.angular.x = 0; twist.angular.y = 0; twist.angular.z = output_y
   pub.publish(twist)
   ```

8. 控制逻辑 如果距离检测已经小于初始设置杆长，那么飞机后退至离气球2米的位置，然后再次前进，此时前进距离气球（初始杆长-5cm）位置，并修改杆长为（初始杆长-5cm）

   ```python
   if (inches<control and i==0) :
   	control_last=control
       control=200
       i=1
       
       if (inches>=200 and i==1) :
       	control=control_last-5
           i=0
   ```

   

# 不是开源协议的开源协议

代码还有很多不足，欢迎大家一起来改进，但改进后请一定记得再次完全开源哦