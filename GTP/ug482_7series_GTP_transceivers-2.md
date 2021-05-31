# 0 前言

如图1-2所示，Xilinx的收发器以quad为单位，一个quad包括4组收发器，一对PLL。每组收发器内由一个发送通道（TX）和接收通道（RX）组成。本节用于介绍TX的结构和功能，结合IP配置应用于设计。

<img src="https://i.loli.net/2021/05/28/hVk16PJce8Gg9My.png" alt="image-20210528112646900" style="zoom:67%;" />

TX的内部结构框图如下所示

<img src="https://i.loli.net/2021/05/31/jnopr7wYW6lE8mc.png" alt="image-20210531155714133" style="zoom:80%;" />

内部主要分为11种资源：

| 序号 | 名称            | 序号 | 名称            |
| ---- | --------------- | ---- | --------------- |
| 1    | FPGA TX接口     | 7    | TX 极性控制     |
| 2    | TX 8B/10B编码器 | 8    | TX 时钟输出控制 |
| 3    | TX 变速器       | 9    | TX 驱动器       |
| 4    | TX Buffer       | 10   | TX PCIe检测支持 |
| 5    | TX Buffer 旁路  | 11   | TX OOB信号      |
| 6    | TX PRBS产生器   |      |                 |

# 1 FPGA TX接口

## 接口位宽配置

用户通过FPGA TX接口在TXUSRCLK2的上升沿将数据写到TXDATA端口。TXDATA端口可以配置为2或者4字节宽度。TXDATA端口字节宽度由TX_DATA_WIDTH和TX_INT_DATAWIDTH属性以及TX8B10BEN端口决定。

<img src="https://i.loli.net/2021/05/31/HUNEcKTXPomY3s5.png" alt="image-20210531155946578" style="zoom:80%;" />

当不使用8B/10B编码器时，TXDATA端口位宽需要做位宽拓展，比如TXDATA端口16bit扩展为20bits，32扩展为40bit，展宽格式如表3-2所示。

![image-20210531161312148](https://i.loli.net/2021/05/31/amSd6kbRPnG79AB.png)

GTP的TX接口配置界面如下：

![image-20210531161903260](https://i.loli.net/2021/05/31/MXW1zvyatCITobA.png)

## TXUSRCLK和TXUSRCLK2时钟产生

FPGA TX接口包括两个并行时钟：TXUSRCLK和TXUSRCLK2。TXUSRCLK为GTP内部TX PCS逻辑时钟。TXUSRCLK时钟的计算如下

![image-20210531162208485](https://i.loli.net/2021/05/31/eO5n4Hqz9KQM8ih.png)

TXUSRCLK2为进入TX侧所有信号的采样时钟，大部分信号在TXUSRCLK2的上升沿采样。表3-3表示TXUSRCLK2与TXUSRCLK以及TX_DATA_WIDTH的关系

<img src="https://i.loli.net/2021/05/31/JNlCOQqAv9nBpWZ.png" alt="image-20210531161456767" style="zoom:80%;" />

## TX 端口定义

<img src="https://i.loli.net/2021/05/31/Byo7VO8CxWvFkeE.png" alt="image-20210531162447083" style="zoom:80%;" />

## TX 接口时钟设计

FPGA TX接口时钟TXUSRCLK2有4中时钟设计方案，这些时钟方案中，TXOUTCLK时钟来自MGTREFCLK0[P/N]或者MGTREFCLK1[P/N]，并且设置TXOUTCLKSEL=3'b011选择TXPLLREFCLK_DIV1路径。

### 2字节TXDATA位宽下TXOUTCLK驱动FPGA TX接口（单个Lane）

<img src="https://i.loli.net/2021/05/31/mdlLFiKUkcZj3aN.png" alt="image-20210531162758394" style="zoom:67%;" />

这种情况下TXUSRCLK时钟和TXUSRCLK2时钟频率相同。

### 2字节TXDATA位宽下TXOUTCLK驱动FPGA TX接口（多个Lane）

<img src="https://i.loli.net/2021/05/31/d1apMbBZynURWsi.png" alt="image-20210531163137196" style="zoom:67%;" />

这种情况下TXUSRCLK时钟和TXUSRCLK2时钟频率相同。

### 4字节TXDATA位宽下TXOUTCLK驱动FPGA TX接口（单个Lane）

<img src="https://i.loli.net/2021/05/31/cVWRpQzAkDKHfF7.png" alt="image-20210531163246783" style="zoom:67%;" />

这种情况下TXUSRCLK时钟为TXUSRCLK2时钟频率的**两倍**。

### 4字节TXDATA位宽下TXOUTCLK驱动FPGA TX接口（多个Lane）

<img src="ug482_7series_GTP_transceivers-2.assets/image-20210531163442181.png" alt="image-20210531163442181" style="zoom:67%;" />

这种情况下TXUSRCLK时钟为TXUSRCLK2时钟频率的**两倍**。

### TX 时钟配置界面

<img src="https://i.loli.net/2021/05/31/xk2OQi3r8nMKXVE.png" alt="image-20210531163813135" style="zoom:80%;" />

