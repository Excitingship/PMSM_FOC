# PMSM_FOC

*PMSM_FOC_CompVer为2017b以及以上兼容版本*

https://blog.csdn.net/qlexcel/article/details/98534608

https://blog.csdn.net/qlexcel/article/details/74787619

https://zhuanlan.zhihu.com/p/147659820

上文中有对FOC,SVPWM算法的具体解释，本文主要对PMSM_FOC模型进行介绍

![9C04E0C2C206C623C2F7C0E392F2254C](../asset/9C04E0C2C206C623C2F7C0E392F2254C.png)

## PMSM电机及其控制电路模型

在仿真中暂时不用考虑具体的电路细节，三相PMSM应该用三个逆变器来控制，而Simulink中都有现成的模块，直接拖出来使用即可。

首先我们使用Simscape->Specialized Power System->Fundamental Blocks->Machines->Permanent Magnet Synchronous Machine永磁同步电机模块（Simscape内部Electrical里面也有PMSM的模型，不过在同样的仿真步长下比前者要慢很多，具体原因未知，用其搭建的电机模型也在此模型下部）。**所有的模型都要右键点开help看一眼，往往会节省很多时间**。打开help可以看见永磁同步电机的介绍，输入输出的含义以及d轴与q轴的定义。我们这时双击模块设置参数：

![WSFCCJBQG7FVBR0](../asset/WSFCCJBQG7FVBR0.png)

基础设置不用改动

![NOG3BNLQZFF167FP0](../asset/NOG3BNLQZFF167FP0.png)

第二页Parameter中最重要的是**最下面的Rotor flux position when theta = 0**，由于Park变换需要电机角度，所以需要着重强调，正常来说，当电机角度为0时，**d轴（磁极）应与a相对齐**，所以像图中一样设置。

输入为abc三相电流和外加扭矩（可选），输出是一个总线Bus，可以用Bus Selector将其中有用的信息挑选出来。我们需要**abc三相电流（采样电流），电机扭矩，速度以及角度**。

接下来是逆变器，我们选择Simscape->Specialized Power System->Fundamental Blocks->Power Electronics->Universal Bridge(通用桥)，将其拖入模型并查看help，可以看到这个模块可以设置为三相逆变器，输入是正负极和**门控制信号g，是一个6个元素的向量**，双击点开设置，![JIUUYCPQMZZWUWHD](../asset/JIUUYCPQMZZWUWHD.png)

将桥臂设置为3，设备其实都可以，可以选择IGBT或者MOSFET。

所以用一个直流电源供电，然后将输出的三相电连接电机即可。

## SVPWM

https://blog.csdn.net/qlexcel/article/details/98534608

https://blog.csdn.net/qlexcel/article/details/74787619

主要是根据这两篇文章搭建，也来源于《现代永磁同步电机控制原理及MATLAB仿真》，SVPWM子系统内部首先根据Park to Clarke变换出的α和β判断扇区（Sector Judgement）

根据计算发现各个矢量作用时间的计算公式是相同的，可以用三个公式进行排列组合，也就是XYZ，，根据α和β算出XYZ，然后再根据扇区查表算出当前扇区两个矢量的作用时间

![img](../asset/20190805234658855.png)

![img](../asset/20190806220841569.png)

其中由于七段式SVPWM每次只变换一项，我们需要计算换相的时间例如T0 T4和T6

![img](../asset/20190806230508854.png)
![img](../asset/2019080523362985.png)

根据上述公式，我们查表算出Tcm，最后与三角载波比较形成中心对齐的PWM波形

![img](../asset/20190806231245635.png)

最后将每个桥的0和1状态转换为每个门的控制信号（3转6）输出到逆变器模块即可。

## 数学变换

用了自带的Park和Clarke变换模块	（Simscape->Electrical->Mathmatical Transform），简单的输入abc三相电流和机械角度即可进行Park变换，简单易用。模块的Help里面有对两种变换的详细解释，**关键是模块参数一定要设置成a相对其d轴**。

## 电流环&速度环

两个PI控制器，其中位置环的主要任务是控制Id为0，Iq为想要的电流值，这样效率最高，所以把Id的期望值设成0即可。

*数学变换和双环其中使用的角度和角速度经过平均滤波处理*。