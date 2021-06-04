# 0 概述

GTH收发器的接收器（RX）资源包括PCS和PMA组件两部分，与TX类似，可以看做是TX结构的逆向。图4-1为RX的结构框图：

![image-20210603154419307](https://i.loli.net/2021/06/03/fYeQ7WhuHpStyCj.png)

RX主要包括如下关键模块：

| 序号 | 模块                   | 序号 | 模块                |
| ---- | ---------------------- | ---- | ------------------- |
| 1    | RX模拟前端             | 8    | RX字节和字对齐      |
| 2    | RX均衡器（DFE和LPM）   | 9    | RX 8B/10B解码器     |
| 3    | RX OOB信号检测         | 10   | RX相位校准          |
| 4    | RX时钟数据恢复（CDR）  | 11   | RX状态控制          |
| 5    | RX接收串并变换（SIPO） | 12   | RX Buffer（缓冲器） |
| 6    | RX极性控制             | 13   | RX变速器（Gearbox） |
| 7    | RX PBRS检测器          | 14   | FPGA RX接口         |

# 1 RX模拟前端

RX模拟接收前端（AFE）是高速电流模式输入差分缓冲器，如图4-2所示。该缓冲器具有以下特性：

- 可配置的RX端接电压
- 校准的端接电阻

![image-20210603155132634](https://i.loli.net/2021/06/03/j5oPnmtYLCvkzed.png)

## 端口与属性声明

端口声明如表4-1：

<img src="https://i.loli.net/2021/06/03/lOgMCymBtJhE2rq.png" alt="image-20210603155410948" style="zoom:80%;" />

GTPRXN和GTPRXP是差分输入端口，在使用时**必须做时序约束**。

模拟前端属性声明如表4-2：

<img src="https://i.loli.net/2021/06/03/8lvoAOK6dtW9PEa.png" alt="image-20210603155621031" style="zoom:80%;" />

<img src="https://i.loli.net/2021/06/03/cVthnvwJ45WyiT7.png" alt="image-20210603155638592" style="zoom:80%;" />

可用于选择端接电压的输入，以及共模电压的配置。在*gt.v中可以看到例化结果：

<img src="https://i.loli.net/2021/06/03/cwZo7TqkK3MOvtj.png" alt="image-20210603160213042" style="zoom:80%;" />

默认属性配置：

<img src="https://i.loli.net/2021/06/03/7LurXJkA9n8gfeN.png" alt="image-20210603160416553" style="zoom:80%;" />

## RX模拟接收端的使用模式

RX端接针对不同的协议应用，在GTP有3种不同的使用模式（GTX/GTH有4种），我们在进行如PCIe、SRIO、SFP+、XAUI等协议时，可以选择对应的配置模式。

### 模式1

<img src="https://i.loli.net/2021/06/03/ODLosHPKzTVbg3p.png" alt="image-20210603161459271" style="zoom:80%;" />

### 模式2

<img src="https://i.loli.net/2021/06/03/tsgz9CueqJRLV4Q.png" alt="image-20210603162448409" style="zoom:80%;" />

### 模式3

<img src="https://i.loli.net/2021/06/03/3ork49A8vJhUBia.png" alt="image-20210603162232082" style="zoom:80%;" />



## RX模拟前端的IP配置界面

![image-20210603161035782](ug482_7series_GTP_transceivers-RX.assets/image-20210603161035782.png)

- voltage：即选择端接电压输入，根据不同协议选择相应输入
- trim value：即端接共模电压配置

# 2 RX均衡器

串行链路比特误码率（BER）性能是发送器、传输媒介和接收器的一个功能。传输媒介是带宽受限的，通过它的信号会衰减和失真。**均衡器主要用于补偿由于频率不同而引起的阻抗或者衰减差异**。

对于GTP收发器，提供高效的自适应连续时间线性均衡器（CTLE），以补偿物理通道中高频衰减带来的信号失真。为了与7系列中GTX/GTH收发器保持一致，CTLE也被称为LPM（低功耗模式）。

![image-20210604151411475](https://i.loli.net/2021/06/04/VeB3xC7GWPjzAm6.png)

LPM模式(如图4-15所示)具有自适应的低频和高频升压。它适用于线速率高达6.6 Gb/s的短距离应用，奈奎斯特频率的信道损耗为12 dB或更少。

在GTX/GTH中有更详细的结构描述：

![image-20210604152737825](https://i.loli.net/2021/06/04/rM9KVTZAyDlJxN2.png)

## 端口与属性声明

<img src="https://i.loli.net/2021/06/04/wOl9Hjbo36J8YQV.png" alt="image-20210604151816313" style="zoom:80%;" />

其属性Xilinx推荐使用默认值。

## RX均衡器IP配置界面

![image-20210604151626861](https://i.loli.net/2021/06/04/GZXFQNPHv2BteJc.png)

对于A7系列FPGA只有默认可选项

# 3 RX OOB信号解码

GTP接收器RX提供：

- 支持解码SATA和SCSI协议要求的OOB信令以及PCIe规范描述的信令
- 支持SATA/SAS OOB信令的GTP接收机包括解码OOB信号状态所需的模拟电路
- 支持解码SATA/SAS OOB信号突发的状态机COM序列

## 端口和属性声明

<img src="https://i.loli.net/2021/06/03/G2euoLSlxRTiEaP.png" alt="image-20210603164135058" style="zoom:80%;" />

对于有使用到OOB的协议，RXELECIDLEMODE配置为00

<img src="https://i.loli.net/2021/06/03/CIODEjgx56kpuwK.png" alt="image-20210603164816639" style="zoom:80%;" />

## OOB信号的使用模式

OOB信号使用需要满足：

- 交流耦合模式：**端接电压需要大于800mv**
- 直流耦合模式：**端接电压需要大于900mv**

- 属性PCS_RSVD_ATTR[8] 设置为1

OOB电路的时钟来源如图4-6所示：

![image-20210603165610189](https://i.loli.net/2021/06/03/Ub576dqtM2k4gSR.png)

输出时钟根据如下选择：

| RXOOB_CLK_CFG | RXSYSCLKSEL | oobclk      |
| ------------- | ----------- | ----------- |
| 0             | 0           | PLL0REFCLK  |
| 0             | 1           | PLL1REFCLK  |
| 1             | x           | SIGVALIDCLK |

![image-20210604141039872](https://i.loli.net/2021/06/04/eGrUJHYxQVPdb6T.png)

## OOB信号IP配置界面

![image-20210604152030234](ug482_7series_GTP_transceivers-RX.assets/image-20210604152030234.png)

# 4 RX时钟数据恢复(CDR)

每个GTPE2_CHANNEL包含时钟数据恢复电路（CDR）用以从数据流中恢复时钟和数据。CDR结构如图4-16所示，时钟路径为虚线：

![image-20210604154218167](https://i.loli.net/2021/06/04/6opdetlTgb5sHAx.png)

GTP收发器采用相位旋转CDR结构。输入数据首先经过接收器均衡阶段，均衡后的数据由边沿采样器和数据采样器捕获。数据采样器捕获的数据被馈送至CDR状态机和下游收发器模块。

CDR状态机使用来自边缘和数据采样器的数据来确定传入数据流的相位并控制相位插值器（PIs）。边缘采样器的相位锁定在数据流的过渡区域，而数据采样器的相位则位于数据眼的中间，如图4-17：

<img src="https://i.loli.net/2021/06/04/pGuePTo9RsUVLYm.png" alt="image-20210604154606269" style="zoom:67%;" />

PLL0或PLL1为相位插值器提供基准时钟。相位插值器依次产生精细的、均匀分布的采样相位，以允许CDR状态机具有精细的相位控制。CDR状态机可以跟踪传入的数据流，这些数据流可以与本地PLL参考时钟存在频率偏移。

## RX CDR的IP配置界面

![image-20210604160457824](https://i.loli.net/2021/06/04/HFq2E7nOzopGeSV.png)

# 5 RX时钟输出控制

RX时钟输出结构与TX时钟输出结构很相似，外围的时钟输入结构和GTPE2_COMMON是与TX共用。RX时钟分频器控制模块包括两个主要组件：**串行时钟分频器**和**并行时钟分频器**及其选择器控制，如图4-18所示：

![image-20210604162431390](https://i.loli.net/2021/06/04/rGXgPuUIDqOYWyk.png)

同样，在使用时有如下注意点：

1. RXOUTCLKPCS和RXOUTCLKFABRIC是**冗余输出**。**RXOUTCLK时钟一般用于FPGA内部逻辑设计**。
2. REFCLK_CTRL选项由软件自动控制的，用户不可选择。用户只能使用使用IBUFDS_GTE2中的O或者ODIV2通过CMT、BUFH或者BUFG输出到FPGA逻辑资源。
3. IBUFDS_GTE2可以看做**冗余时钟**，增加了收发器时钟方案的灵活性。
4. /4或者/5分频器模块由GTPE2_CHANNEL的TX_DATA_WIDTH属性控制。TX_DATA_WIDTH = 16，32时，选择/4分频器；TX_DATA_WIDTH = 20，40时，选择/5分频器。

## 串行时钟分频器

每个接收器的PMA模块有一个D分频器用来分频来自PLL的时钟，以产生所需的线速率时钟。该D分频器可以设置为用于固定线速率的静态配置或者用于变化线速率的动态配置。

![image-20210604163028538](https://i.loli.net/2021/06/04/MIDYAfLGTN1yRal.png)

## 并行时钟分频器和选择器

来自RX时钟输出模块的并行时钟可以用于FPGA逻辑设计，推荐RXOUTCLK用于FPGA内部逻辑设计，该时钟输出延迟可控。也可以将MGT管脚MGTREFCLK时钟直接输到FPGA内部作为逻辑时钟。

![image-20210604165615998](https://i.loli.net/2021/06/04/ayZCxntBXSp62Yr.png)

**RXOUTCLKPCS不推荐使用**，它在PCS模块中引入了额外的延时。

**RXOUTCLKPMA为恢复时钟**，可以提供给FPGA逻辑使用，用于同步接收到的数据。**（默认值）**

RXPLLREFCLK来自PLL0/PLL1，不与接收时钟同步。



## 时钟输出IP配置界面

![image-20210604163850497](https://i.loli.net/2021/06/04/PDLVweGyrOZo2CI.png)

- RXOUTCLK source：对于不要求输出恢复时钟的应用，可以选择PLL时钟作为系统时钟。但是通常应选择TXOUTCLK作为系统时钟（默认）。



