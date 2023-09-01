# WIDCAutonomousDriving

1 快速启动
修改代码，切换成不同的任务ID，任务ID的范围是0,1,2,3,4，分别对应着五项任务：



2 控制模块
pid.py：采用PID进行控制，包括纵向控制和横向控制，其中纵向控制包括了转向角的控制和角速度的控制


另外，由于开启的初始过程的偏向角过大，因此我们需要给一个相反方向的小角度初始化，防止出现较大的超调量。

3 状态模块
state.py：包括两类数据的初始化，分别是车辆自身状态和控制器状态的初始化。
3.1 车辆自身状态
速度
当前位置
相对前车的速度
相对前车的距离
加速度
3.2 控制器状态
横向控制
转向角控制
角速度控制
纵向控制
目标速度
速度阈值
4 自动紧急制动AEB
计算紧急停车时间的关键是计算加速度，我们通过雷达的信息可以求得近似加速度。

    dt = np.round(myCar.dist/ myCar.delta_v,3)

    a = np.round(myCar.delta_v/dt,3)
4.1 碰撞时间TTC
假设前车和当前车辆在同一直线车道中，定义如下变量：

两车距离为$s$，安全停止距离为$d$，行驶时间为$t$，本车的速度为$v_1$，加速度为$a_1$，同样前车的速度为$v_2$，前车的加速度为$a_2$。在本赛题中，安全距离取$[0.5,1]$。相对车速为$v_{vel}$，相对加速度为$a_{rel}$，二者均为前车减后车。

那么可以列出公式为: $$ v_2t+\frac{1}{2}a_2t^2+(s-d)=v_1t+\frac{1}{2}a_1t^2 $$

解得: $$ TTC=t=\frac{-v_{rel}+\sqrt{v_{rel}^2+2a_{rel}(s-d)}}{a_{rel}},a_{rel}\ne0 $$

计算TTC的函数如下：

def getTTC(current_speed, current_acceleration, dist, safe_dist):

    v = current_speed
    a = current_acceleration

    if a !=0:
        ttc = ( -v + np.sqrt(v **2 + 2 * a * (dist-safe_dist)) ) / a
    else:
        ttc = 999

    return ttc
4.2 基于规则的停车
设定了DANGER的标志位，在小于绝对安全距离和大于特定加速度时候，采取强制的刹车动作
判断ttc时间，如果在安全的刹车距离外，采用PID进行控制，否则如果在安全的刹车距离内，采用线性刹车的函数进行控制。
 DANGER = False

    if myCar.dist < 1:
        DANGER = True

    if myCar.dist < 15 and a > 5.5:
        DANGER = True

    if ttc > 0.5 and not DANGER and ttc != 999:
		PidControl()

    elif ttc <= 0.5 and not DANGER:
		SoftBrake()
        
	elif DANGER:
		HardBrake()
 DANGER = False

    if myCar.dist < 1:
        DANGER = True

    if myCar.dist < 15 and a > 5.5:
        DANGER = True

    if ttc > 0.5 and not DANGER and ttc != 999:
        # 正常情况下使用PID来进行纵向控制
        print("ttc:", ttc)
        ADCPlatform.control(Controller.speedPid.thorro_, Controller.latPid.steer_, 0, 1)

    elif ttc <= 0.5 and not DANGER:
        # 如果ttc时间在一定范围了，软刹车
        print("dist:",myCar.dist,"ttc:",ttc,"a:",a)

        brake_ = 0.14 * a

        ADCPlatform.control(0, Controller.latPid.steer_, brake_, 1)
5 决赛题目
5.1 前车制动到停车
即需要完成两项任务：

在前车未停止时，实现跟车功能
在前车停止后，使用上述的AEB方法停车
5.2 行人从视觉盲区横穿马路
这项任务和初赛的任务很相似，只是增加了一辆靠边的静止汽车。

5.3 曲线行驶，前方多车障碍
5.3.1 计算加速度
我们用雷达的相对距离和相对速度来估算当前的相对加速度，具体方法如下: $$ \Delta t \approx \frac{\Delta x}{\Delta v} \ \Delta a \approx \frac{\Delta v}{\Delta t} = \frac{(\Delta v )^2}{\Delta x} $$

5.3.2 计算时间
假设前车和当前车辆在同一直线车道中，定义如下变量：

两车距离为$s$，安全停止距离为$d$，行驶时间为$t$，本车的速度为$v_1$，加速度为$a_1$，同样前车的速度为$v_2$，前车的加速度为$a_2$。在本赛题中，安全距离取$[0.5,1]$。相对车速为$v_{vel}$，相对加速度为$a_{rel}$，二者均为前车减后车。



计算TTC的函数如下：


那么跟车的时候其实也满足这个模型，停车的时候最后我们希望相对加速度为0，而在跟车的过程中相对加速度也为0，因此在跟车任务中我们依然采用了这个模型。

5.3.3 基于规则的跟车
检测到前车的速度在$20km/h$附近，因此我们设置了纵向控制的PID，为了实现跟车，我们把任务分成两个部分。
将PID的目标速度设置为$30km/h$，目的是将当前车辆和前方车辆的距离减少到$20m$附近。
通过当前速度计算出保持$20m$跟车距离的TTC，根据规则将速度逐渐降低至$20km/h$附近。
代码如下，通过时间我们来控制这个油门和刹车。如果估算时间较大，我们通过PID来控制，如果估算时间较小，说明与前车距离可能稍微近了一些，我们采用线性刹车来让距离恢复到跟车区间。

5.3.4 转向控制
通过车道线传感器，我们可以获得横向距离，那么通过增量式PID就可以控制转向角。此外，还需要设置距离参数，根据未来多远的点来计算横向距离的偏差。

