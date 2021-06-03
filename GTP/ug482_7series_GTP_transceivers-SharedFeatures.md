

# 0 前言

这部分对应ug482第二章节部分：共享功能，主要了解以下内容：

- 收发器有哪些共享资源
- 如何使用这些共享资源

# 1 GTP共享资源概述

共享资源总共分为：参考时钟输入结构、参考时钟选择及分配网络、PLL、复位和初始化、电源关电、环回测试、动态配置端口和数字监控器总共八部分结构

<img src="https://i.loli.net/2021/05/28/hVk16PJce8Gg9My.png" alt="image-20210528112646900" style="zoom:67%;" />

上图显示了时钟相关的结构。

# 2 输入参考时钟结构

输入参考时钟必须通过`IBUFDS_GTE2`原句才能使用，这一点在图1-2所示的结构中可以看到。输入参考时钟结构如图2-1所示。

![image-20210528140102562](https://i.loli.net/2021/05/28/zamDKNnCu7wPgVe.png)

Xilinx FPGA基本都是采用端口（Port）和属性（Attribute）实现参数化组件控制。属性即图中`CLK_RCV_TRST，CLKCM_CFG，CLKSWING_CFG`。

我们可以从IP的例程中找到`IBUFDS_GTE2`的例化方法，例子中并没有显示原句的属性参数，这是因为该原句的属性均为预留，无需用户设置。

<img src="https://i.loli.net/2021/05/28/sbDPZMQmO3Up2Bu.png" alt="image-20210528140304494" style="zoom:67%;" />

此处输出作为Quad 0的参考时钟。`ODIV2`为`O`的时钟频率一半的时钟，但是他们相位并不匹配。

# 3 参考时钟选择及分配结构

图2-3显示了GTPE2_COMMON参考时钟选择器结构，图中我们可看到每路时钟的来源：

![image-20210528143127601](https://i.loli.net/2021/05/28/Fwn9hBtx6CoM7z4.png)

在IP向导中可以看到时钟选择的对应，下图为`XC7A100T-FGG484`封装，由于只有Quad 1的GTP以及Quad 0的时钟输入端口可以配置（见上一篇图1-9），在时钟选择上只有来自Quad 0的`IBUFDS_GTE2`可以选择。

![image-20210528143711372](https://i.loli.net/2021/05/28/O5lGZkitN6cWf4T.png)

同样，对于该组件Xilinx FPGA也是采用端口和属性来进行参数化控制。

我们在项目开发时通常采用Xilinx提供的收发器向导通过GUI图形界面方式来实现控制，而不是通过繁琐的的输入参数来实现。当我们通过GUI实现参数配置后，软件在生成源码文件时自动回帮我们生成好。

<img src="https://i.loli.net/2021/05/28/TXQPrO4EedjxK3R.png" alt="image-20210528150956272" style="zoom: 80%;" />

## 外部参考时钟使用模式

有4中外部参考时钟供我们选择：

**1.单个时钟驱动单个Quad**

<img src="https://i.loli.net/2021/05/28/mrZy8g9qIPUYX6S.png" alt="image-20210528152220676" style="zoom:67%;" />

**2.单个时钟驱动两个Quad**

<img src="https://i.loli.net/2021/05/28/poQG8Wr9gTqmELK.png" alt="image-20210528152411337" style="zoom:67%;" />

**3.两个时钟驱动单个Quad**

<img src="https://i.loli.net/2021/05/28/QUmHNVCfWXGd7aK.png" alt="image-20210528152538445" style="zoom:67%;" />

**4.两个时钟驱动两个Quad**

<img src="https://i.loli.net/2021/05/28/UMHbCNeWkAlDOLR.png" alt="image-20210528152703113" style="zoom:67%;" />

对于高版本的FPGA来说（例如K 7，V 7等），Quad数量相对A 7要多，在第二种和第4种方案使用时必须遵循Xilinx推荐的规则：

1）外部参考时钟输入Quad（源Quad）的上方Quad的数量不能超过1个；

2）外部参考时钟输入Quad（源Quad）的下方Quad数量不能超过1个；

3）一个外部参考差分时钟对（`MGTREFCLKN/MGTREFCLKP`）驱动Quad的数量不能超过3个（或12个收发器）。如果需要驱动多余12个收发器，需要使用多个外部参考时钟差分对，为确保时钟对之间偏移最小，必须使用同一个晶振来驱动时钟缓冲器（buffer），然后时钟缓冲器的多个输出时钟再驱动多个收发器（超过12个）。

<img src="https://pic1.zhimg.com/v2-e76c3badcd475638d34953fea18e8678_r.jpg" alt="preview" style="zoom:67%;" />

上图为外部参考时钟举例PCIe×16时钟方案。

# 4 PLL

GTP内部时钟结构如图2-9所示，主要由原语`GTPE2_COMMON`和`GTPE2_CHANNEL`搭配构成，主要分为：PLL、TX时钟分频器和RX时钟分频器。TX时钟分频器和RX时钟分频器允许收发器接收器和发送器操作在不同的线速率，使用不同的参考时钟输入。

<img src="https://i.loli.net/2021/05/28/X2jPRtzYqdC8WOb.png" alt="image-20210528153721718" style="zoom:67%;" />

PLL功能模块框图如图2-10所示。输入时钟在进入鉴频鉴相器(PFD)前首先进行M倍分频。反馈分频器N 1和N 2决定了VCO倍频比例和PLL输出频率。一个锁定指示器模块用于比较参考时钟和VCO反馈时钟频率以决定PLL输出是否锁定。

<img src="https://i.loli.net/2021/05/28/GPlORD64J9FZCi3.png" alt="image-20210528161728707" style="zoom: 50%;" />

PLL输出频率由式2-1确定，式2-2为FPGA的GTP线速率，D为TX和RX模块分频器因子，如表2-7所示。

<img src="https://i.loli.net/2021/05/28/ebqDpNwLdAs82IF.png" alt="image-20210528162129843" style="zoom: 80%;" />

<img src="https://i.loli.net/2021/05/28/hrQOBoL2zlMwdgn.png" alt="image-20210528162323582" style="zoom: 67%;" />

公式对应到IP中如下所示：

![image-20210528163331916](https://i.loli.net/2021/05/28/Dg5oGbeYpJfvm8R.png)

可以IP文件`*_multi_gt`中找到对应配置的参数

<img src="https://i.loli.net/2021/05/28/sOvZMj4Gyfz8UCr.png" alt="image-20210528164054409" style="zoom: 67%;" />

在高端的FPGA中，PLL被分为两种类型：CPLL和QPLL，当线速率大于6.6Gbps时，必须使用QPLL。

# 5 复位和初始化

GTP收发器在FPGA上电和配置后必须进行初始化，GTP收发器的发送器（TX）和接收器（RX）可以独立和并行初始化。初始化包括两个步骤：

1. 初始化TX/RX的关联PLL
2. 初始化TX和RX的数据路径（PMA+PCS）

其中，PLL和TX/RX的复位是完全独立的，TX/RX的数据路径初始化必须在PLL锁定后才能执行。

<img src="https://i.loli.net/2021/05/31/ODjwHaxsU2vugB1.png" alt="image-20210531101133911" style="zoom:67%;" />

GTP收发器使用状态机控制TX和RX初始化，允许PMA先初始化，PCS后初始化。同样，也允许正常操作时PMA、PCS和其内部的功能模块独立复位。收发器提供两种类型复位：

- 初始化复位：该服用用来执行GTP完成初始化，用在器件上电和配置时。正常操作时，GTTXRESET和GTRESET也用来复位GTP的TX和RX。
- 组件复位：该复位用来复位特殊事件或者收发器某些子部分，如PCS、PMA复位等。

## 复位模式

GTP收发器有两种复位模式：

- 顺序模式：当初始化指令或者复位输入为高时，复位状态机执行对所有状态进行复位，复位结束后拉高RESETDONE
- 单一模式：复位状态机只对请求器件进行复位，复位对象可以是PMA，PCS或者它们的内部模块，复位结束后拉高RESETDONE

**GTP的TX发送器复位必须为顺序模式，**因此推荐都使用顺序模式。

## PLL复位

**在FPGA逻辑检测到参考时钟触发沿前，PLL必须处于关电状态。而在PLL使用之前，必须进行复位。**每个`GTPE2_COMMON` 有6个端口用于PLL复位，PLL 0和PLL 1各三个，如下所示：

![image-20210531111700457](https://i.loli.net/2021/05/31/asgOMYAzkFq1UVT.png)

`PPLxRESET`的有效时间至少为1个参考时钟周期。

## TX初始化和复位

GTP收发器TX复位由如图8所示的状态机控制。通过执行一个`GTTXRESET`高开始复位TX，自动执行一个完整异步TX复位，包括PMA和PCS复位。只有当`TXUSERRDY`为高电平时，PCS才进行复位，而`TXUSERRDY`为高必须满足一下条件：

- 所有使用的时钟必须稳定或锁定

- 用户接口准备好发送数据到GTP发送器

  <img src="https://i.loli.net/2021/05/31/XYjqv91S4csGKbQ.png" alt="image-20210531133529250" style="zoom:80%;" />

对应的端口在*_gt文件中：

<img src="ug482_7series_GTP_transceivers-1.assets/image-20210531134138678.png" alt="image-20210531134138678" style="zoom:67%;" />

## GTP发射器TX复位对各种复位相应响应

### 对FPGA配置完成相应

图2-13中的TX复位不会自动开始，必须满足如下条件：

- GTRESETSEL必须为低电平（**选择顺序复位模式**）
- GTTXRESET必须使用
- 在TXRESETDONE拉高之前，TXPMARESET和TXPCSRESET信号必须一直为低
- 在使用的PLL锁定前，GTTXRESET必须一直为高

图2-14为GTP的TX在FPGA配置后的初始化时序，状态机跳转条件如图序号

![image-20210531135226663](ug482_7series_GTP_transceivers-1.assets/image-20210531135226663.png)

### 对GTTXRESET脉冲相应

![image-20210531141054605](https://i.loli.net/2021/05/31/mT4nVNQvKUieAoh.png)

### 对组件复位相应

![image-20210531141124911](https://i.loli.net/2021/05/31/cnBhgeOPaGoxwUV.png)

![image-20210531141149555](https://i.loli.net/2021/05/31/Mg2OulfDmFARxST.png)

### 推荐复位方式

<img src="https://i.loli.net/2021/05/31/1VujzrT2pOIQ4Dn.png" alt="image-20210531141448372" style="zoom:80%;" />

## RX初始化和复位

GTP收发器RX部分初始化和TX部分初始化分析方法类似，相对于TX，RX复位涉及的组件较多，复位稍微复杂，但初始化和复位过程差别不大，不再进行更详细分析。图2-18显示了GTP收发器RX复位状态机顺序图。

<img src="https://i.loli.net/2021/05/31/kQxNCgdSq6lBjtM.png" alt="image-20210531142105087" style="zoom:80%;" />

# 6 收发器环回功能

环回模式是FPGA收发器数据路径特殊配置模式，它将数据流返回数据源端。它可以在开发过程中使用，也可以在已经部署的产品中用于隔离、定位错误。

该功能可以用来检查或测试近端（本地收）收发器或者远端（其他业务板）收发器收发链路是否正常工作以及判断通信链路信号质量。

![image-20210531142720738](https://i.loli.net/2021/05/31/61yejKi5UCBwobv.png)

从图2-22中可以看到，环回测试分为两大类：

- 近端环回：发送数据在本地GTP内环回（数据路径1和2）
- 远端环回：发送数据经过远端GTP后，再回到本地GTP（数据路径3和4）

每个GTP收发器都有一个内置的PRBS（伪随机码）产生器和检查器。为了灵活的进行还回测试，每个收发器有4中还回测试模式：

**近端PCS还回测试（数据路径①）：**该模式需要将RX弹性缓冲器使能，并且RX_XCLK_SEL设置为RXREC，此时时钟有TX PMA并联时钟TX XCLK驱动。

如果在正常操作时，RXOUTCLK用于驱动FPGA逻辑，并且RXOUTCLKSEL设置为RXOUTCLKPMA，此时，必须要将RXOUTCLKSEL选择为RXOUTCLKPC或者设置RXCDRHOLD=1'b1。

**近端PMA还回测试（数据路径②）：**该模式下，GTRXRESET复位信号在进入和退出该模式时需要设置。

**远端PMA还回测试（数据路径③）：**该模式下，TX缓冲器必须使能，并且设置TX_XCLK_SEL设置为TXOUT。GTTXRESET复位信号在进入和退出该模式时需要设置。

**远端PCS还回测试（数据路径④）**：该模式下，无论时钟校准使用与否，TXUSRCLK和RXUSRCLK端口必须由同一时钟源（BUFG、BUFR或者BUFH）驱动。PCS还回不支持通道变速使能。

表2-27为环回配置方式：

![image-20210531144410412](https://i.loli.net/2021/05/31/FAlgbsHBj26aPqp.png)

# 7 动态配置功能（DRP）

通过DPR接口可以实现动态配置功能，动态配置功能允许动态的改变GTPE2_CHANNEL和GTPE2_COMMON原句参数。接口如下：

![image-20210531145853981](https://i.loli.net/2021/05/31/2w13DvtbjX6lZ7S.png)

# 8 收发器power down模式

GTP收发器支持一列关电模式，这些模式支持一般的电源管理，也支持PCIe和SATA标准支持的模式。收发器提供不同级别的电源控制。每个通道内的每个方向都可以使用TXPD和RXPD端口进行关电控制。 PLL0PD和PLL1PD分别影响PLL0 和PLL1功能。

表2-25列出了经常使用的几种基本的关电模式。

<img src="https://i.loli.net/2021/05/31/dMTp47seqUVwjaJ.png" alt="image-20210531150823865" style="zoom: 80%;" />

# 9 数字监视器功能

收发器接收均衡器的LPM（低功耗均衡）和DFE（判决反馈均衡）接收模式使用自适应算法优化收发器链路，数字监视器提供对这些自适应环路当前状态的可见性。数字监视器要求一个自由运行的时钟，可以为RXUSRCLK2。