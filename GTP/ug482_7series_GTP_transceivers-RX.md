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

# 6 RX PRBS（伪随机码）检查器

GTP收发器内置一个PRBS检查器，用来测试通道信号完整性。该检查器的结构如图4-24所示

![image-20210607102204931](https://i.loli.net/2021/06/07/vlwf1eCcqhJ8QG4.png)

## 端口与属性声明

![image-20210607102325123](https://i.loli.net/2021/06/07/eN2FdpEacv5k1TO.png)

![image-20210607102346308](https://i.loli.net/2021/06/07/fiClAcN6LzEp3xb.png)

## PRBS检查器IP配置界面

![image-20210607102954682](https://i.loli.net/2021/06/07/jfydsIinSHFu91P.png)

# 7 RX 8B/10B解码器

如果GTP收发器接收到的数据是8B/10B编码的，它必须在接收时解码。GTP收发器内置解码器，该解码

- 支持2字节、4字节数据路径操作
- 为正确的视差提供运行视差的菊花链接
- 产生K码字符和状态信息输出
- 当收到非表内数据时，输出10位字面编码值
- 如果接收数据没有进行8B/10B编码，则可以旁路该解码器。

8B/10B解码器Bit和Byte顺序如图4-33所示

![image-20210607104643652](https://i.loli.net/2021/06/07/tOWgUeDwBLYF7Qb.png)

## RX运行不一致

两种运行不一致情况：

- 当检测到RXDATA出现运行不一致错误时，RXDISPERR为高电平；

- 当检测到非法的8B/10B字符时，RXNOTINTABLE端口输出为高电平。

图4-34显示了RX数据接口解码数据波形，图中A数据为正确数据，B、C和D为错误数据。

![image-20210607105126499](https://i.loli.net/2021/06/07/H2Bz8hx9OMigKWL.png)

## 特殊字符

8B/10B解码器包括特殊字符（K字符）经常用于控制功能。当RXDATA检测到位K字符时，解码器驱动RXCHARISK为高电平。

如果DEC_PCOMMA_DETECT被使能，当RXDATA接收到正8B/10B comma，RXCHARISCOMMA为高电平。

如果DEC_MCOMMA_DETECT被使能，当RXDATA接收到负8B/10B comma，RXCHARISCOMMA为高电平。

![image-20210607110313925](https://i.loli.net/2021/06/07/XWS3bfrLjpsZKdP.png)

## 端口与属性声明

![image-20210607110454928](https://i.loli.net/2021/06/07/xc8DZHnTCfO3SQh.png)

## 8B/10B解码器IP配置界面

![image-20210607110142286](https://i.loli.net/2021/06/07/rtgJyi6wnSs5uOh.png)

# 8 RX字节和字对齐

输入到FPGA收发器的串行数据在解串（串并转换）之前必须进行符号边界对齐。为了保证数据对齐，发送器发送一个通常称为comma码（K码）的字符，接收器在输入的数据里查找comma码。当发下comma码后，则将comma移动到字符边界，这样使得接收到的并行数据匹配发送的并行数据。

图4-25显示了10bit comma对齐过程。数据流由左向右流动，RX接收到没有对齐的数据。图中虚线为查找到的comma码，标志查找到字节边界，后续comma码之后每10bit自动划分为一个字，自此完成数据字对齐。

![image-20210607110926762](https://i.loli.net/2021/06/07/qViBRM1lxyXcDWZ.png)

图4-26显示左侧显示了TX发送并行数据，右侧显示了RX在comma对齐后识别到了正确的并行数据。

![image-20210607111839649](https://i.loli.net/2021/06/07/YsvSZG1dwx2bocB.png)

## 使能Comma对齐

RXCOMMADETEN端口设置为高，使能Comma对齐模块。

RXCOMMADETEN端口设置为低，会旁路该模块，可以减少路径延迟。

![image-20210607112314752](https://i.loli.net/2021/06/07/26n7l4yYXSRto3M.png)

## 配置Comma参数

为了设置comma参数，需要配置ALIGN_MCOMMA_VALUE，ALGN_PCOMMA_VALUE和ALIGN_COMMA_ENABLE属性。

comma的长度和RX_DATA_WIDTH有关。图4-27显示了comma匹配掩码模式

![image-20210607133334391](https://i.loli.net/2021/06/11/SYeXnGbuAv3kagm.png)

扩展的comma匹配掩码模式与之类似。

![image-20210607133356793](https://i.loli.net/2021/06/11/a7Uhsl6xTVIkE2D.png)

## comma状态指示

当MCOMMA或者PCOMMA对齐被激活后，任何匹配的comma模式与最近的边界重新对齐。comma对齐后，RXBYTEISALIGNED信号置为高。

此时可以将RXENMCOMMAALIGN和RXENPCOMMAALIGN置为0，关闭comma对齐功能，使comma对齐模块保持当前对齐位置。

当RXBYTEISALIGNED置为高时，表明已经检测到comma与字节边界对齐。若后续到达的comma都可以对齐，则RXBYTEISALIGNED继续为高，否则RXBYTEISALIGNED为低。

![image-20210607134238777](https://i.loli.net/2021/06/11/Iru1ZVdA9p8Qbce.png)

##  comma对齐边界设定

ALIGN_COMMA_WORD属性用于定义对齐边界。边界空白区域长度由RX_DATA_WIDTH属性决定，边界位置的数量由RX用户接口RXDATA的字节数决定。图4-30为可选择的字节边界：

![image-20210607134706838](https://i.loli.net/2021/06/11/ECyWxYhmBeJn4wj.png)

对于ALIGN_COMMA_WORD定义边界为2，comma位置为灰色位置。

## comma手动对齐

通过RXSLIDE信号可以设置手动comma对齐。手动comma对齐时，RXENMCOMMAALIGN和RXENPCOMMAALIGN信号输入为0。

RXSLIDE信号每次置高一个RXUSRCLK2时钟周期，并行数据向左移动一位。

图4-31为手动comma码对齐时序图。

![image-20210607140221784](https://i.loli.net/2021/06/11/OvHsF38mEpI4d9X.png)

两个RXSLIDE信号之间最少为32个RXUSRCLK2时钟。

## 端口与属性声明

![image-20210607160431945](https://i.loli.net/2021/06/11/naKP3uEpyR8lYcq.png)

![image-20210607160446922](https://i.loli.net/2021/06/07/YyKoCWaB7sfPZuD.png)

![image-20210607160503485](https://i.loli.net/2021/06/07/DN3ZLG6IulO97qx.png)

![image-20210607160406955](https://i.loli.net/2021/06/11/TUgPs9uJ2Cvy8ae.png)

## RX对齐IP配置界面

![image-20210607160204591](https://i.loli.net/2021/06/07/YQzKyNtp6nqmT2u.png)

# 9 RX弹性buffer

GTP收发器内部包括两个内部并行时钟域：PMA并行时钟域XCLK和RXUSRCLK时钟域。为了正确接收数据，PMA并行速率必须匹配RXUSRCLK时钟速率，并且解决跨时钟域问题。图4-43显示了XCLK和RXUSRCLK时钟域。

![image-20210607161051406](https://i.loli.net/2021/06/07/lQd43DAnmfVLKve.png)

GTP提供了两种方式解决跨时钟域问题：

- RX弹性缓冲器
- RX相位对齐电路

这两种方法的特性对比如表4-32所示：

![image-20210607161343350](https://i.loli.net/2021/06/07/1nxJud7GsVjzaYi.png)

## RX buffer 旁路

旁路RX弹性缓冲区是7系列GTP收发器的高级特性。**RX相位对齐电路用来调整SIPO并行时钟和XCLK时钟域相位差异，以保证数据从SIPO可靠的传输到PCS组件。**它也通过RXUSRCLK来调整RX延迟，以补偿由于温度或者电压变化引起的延迟。RX相位延迟和对齐可以通过GTP收发器自动调整或者手动调整。图4-35显示了XCLK和RXUSRCLK时钟域。

![image-20210608133807719](https://i.loli.net/2021/06/11/AKDtJLrioByxUuh.png)

## 端口和属性声明

![image-20210608135911563](https://i.loli.net/2021/06/11/oqlb3sznZ2KjmSX.png)

## RX buffer的IP配置界面

![image-20210608135840945](https://i.loli.net/2021/06/08/WDXFCA86Lx4RZkj.png)

# 10 RX接口

FPGA RX接口是GTX/GTH收发器并行接口，实现收发器并行数据输出到FPGA内部逻辑。**FPGA在RXUSRCLK2时钟的上升沿读取RXDATA端口数据**，该端口可以配置为2字节或者4字节。

RXDATA宽度和RX_DATA_WIDTH和RX_INT_DATAWIDTH属性以及RX8B10BEN有关。并行时钟RXUSRCLK2速率由RX线速率、RXDATA宽度以及8B10B编码属性决定。RXUSRCLK时钟可以提供给PCS内部逻辑使用。

## RX接口配置

7系列GTP收发器包含2字节内部数据路径，通过RX_INT_DATAWIDTH属性配置。RX接口配置如表4-43所示。

<img src="https://i.loli.net/2021/06/08/eICtME8PqB45Kju.png" alt="image-20210608162106402" style="zoom:67%;" />

当8B/10B解码器**旁路**时，RXDISPERR和RXCHARISK端口用来扩展RXDATA端口。

![image-20210608162312202](https://i.loli.net/2021/06/08/UyShVLlu8FrtHR3.png)

## RXUSRCLK和RXUSRCLK2时钟产生

RX接口包括两个并行时钟：RXUSRCLK和RXUSRCLK2。**RXUSRCLK用于收发器PCS内部逻辑资源使用，RXUSRCLK2用于FPGA RX接口所有信号同步时钟。**RXUSRCLK时钟产生方程如下

<img src="https://i.loli.net/2021/06/08/diwnCfHrLaXpP3D.png" alt="image-20210608162510828" style="zoom:67%;" />

RXUSRCLK和RXUSRCLK2时钟之间关系如表4-45所示。

<img src="https://i.loli.net/2021/06/08/3IwN8cfYl1bWuGC.png" alt="image-20210608162640059" style="zoom:80%;" />

使用时需要注意：

- RXUSRCLK和RXUSRCLK2必须上升沿对齐，通常需要BUFG或BUFH来驱动以减少偏移（skew）

- 如果通道的TX和RX使用相同的参考时钟源驱动，那么可以选择TXOUTCLK来驱动RXUSRCLK和RXUSRCLK2。当时钟校准关闭或者RX buffer旁路时，RX相位对齐电路必须用来对齐串行时钟和并行时钟。
- 如果通道的TX和RX使用不同参考时钟，并且时钟校准没有使用，那么必须由RXOUTCLK来驱动RXUSRCLK和RXUSRCLK2，并且相位对齐电路必须使用

## 端口和属性声明

![image-20210608163452337](https://i.loli.net/2021/06/08/wztP4TShUbAEBDn.png)

![image-20210608163509233](https://i.loli.net/2021/06/08/2CcpVYjM51WSKXQ.png)

## RX接口IP配置界面

![image-20210609094007593](https://i.loli.net/2021/06/09/IhZdCkEDVvHle4G.png)

当选择8B/10B或其他编码时，RXCHARISK默认使用，而选择none时为可选。

# 11 RX时钟校准

RX弹性缓冲器用来设计桥接RXUSRCLK和XCLK时钟域。理想情况下这两个时钟应该频率和相位相同，实际应用中两者在频率和相位上会存在一定偏移。RX弹性缓冲器可以实现两个时钟域数据稳定传输。

**RX时钟校准功能通过删除或者复制特定的空闲字符来防止RX弹性缓冲器上溢出或者下溢出。**

![image-20210609100808007](https://i.loli.net/2021/06/09/lqwHMZTDapEg8KX.png)

图4-44举例RXUSRCLK和XCLK时钟的三种应用场景：

- 正常情况下，读时钟RXUSRCLK和XCLK频率相同，此时RX弹性缓冲器数据量保持稳定
- 当读时钟XUSRCLK快于写时钟XCLK时，为避免出现读空RX弹性缓冲器，通过插值实现重复读或空读
- 当读时钟XUSRCLK慢于写时钟XCLK时，为避免出现RX弹性缓冲器溢出，通过移除字符减少数据

## 端口和属性声明

![image-20210609102550972](https://i.loli.net/2021/06/09/e3Di1CLxX6UKwO2.png)

## RX时钟校准IP配置界面

**使用RX时钟校准的前提是使用RX弹性buffer**

![image-20210610105419854](https://i.loli.net/2021/06/10/5zaZ98Wn4uxiVrC.png)

用于校准的序列可以是K字符，或者反向字符（负字符）或者正常字符，在表格中勾选，对应实际序列如图4-45所示：

<img src="https://i.loli.net/2021/06/10/NVxAyLCamrERcUi.png" alt="image-20210610110049857" style="zoom:80%;" />

当使用多字节序列时，对应的映射如图4-46所示：

<img src="https://i.loli.net/2021/06/10/dx3D2PCeubqrRVL.png" alt="image-20210610110211036" style="zoom:67%;" />

同时，时钟校准还支持添加一条辅助序列用于校准，选择【use two clock correction sequences】

时钟校准电路的状态可由【RXCLKCORCNT】和【RXBUFSTATUS】进行监控。默认的配置可在*_gt.v中看到：

<img src="https://i.loli.net/2021/06/10/VOgMLDkaqCe57uU.png" alt="image-20210610111647860" style="zoom:67%;" />

在手册中提到：

> The 7 Series FPGAs Transceivers Wizard chooses an optimal setting for CLK_COR_MIN_LAT and CLK_COR_MAX_LAT based on application requirements. The values selected by the Wizard must be followed to maintain optimal performance and must not be overridden.

最大延时`CLK_COR_MAX_LAT`和最小延时`CLK_COR_MIN_LAT`将由收发器向导设置，并且不要去手动修改。

# 12 RX通道绑定

XAUI和PCIe等协议使用多个串行收发器以产生更高的数据速率。由于每个收发器所在的通道延迟可能存在差异，这会导致通道间数据会存在“错位”现象，RX通道绑定功能就解决此问题。

<img src="https://i.loli.net/2021/06/10/XJkghi4PNYKRmEM.png" alt="image-20210610112218897" style="zoom:80%;" />

通常在收发器TX发送端发送一串特殊字符，称为通道绑定序列。每个收发器接收到特殊字符后，GTP接收器可以决定每个通道之间的偏移，通过RX弹性缓冲器调整延迟，保证用户接口可以无偏移接收。

**RX通道绑定支持8B/10B编码，但是不支持：64B/66B，64B/67B，128B/130B，Scrambled data**

## 端口和属性声明

![image-20210611151133510](https://i.loli.net/2021/06/11/iaWGrD8AI1coELx.png)

![image-20210611151154735](https://i.loli.net/2021/06/11/s8blwB1TEmDWAOU.png)

![image-20210611151221020](https://i.loli.net/2021/06/11/F4OUsXeZ1Y2yxQo.png)

## RX通道绑定IP配置界面

**用RX时钟校准的前提是使用RX弹性buffer**

![image-20210610112836434](https://i.loli.net/2021/06/10/lJhdzQ4wHmVNr5M.png)

RX通道绑定的IP界面与RX时钟校准很相似，在配置上也基本相同。使用流程如下：

1. 使能通道绑定模式
2. 将主收发器的RXCHBONDMASTER置高
3. 将从收发器的RXCHBONDSLAVE置高
4. 使用直连或者菊花链的方式连接主从收发器对应通道绑定端口：主（RXCHBONDO）- 从（RXCHBONDI）
5. 设定通道绑定序列和探测参数

当主从设备距离较远时，采用直连方式容易有时序问题，解决方式之一为采用菊花链连接：

<img src="https://i.loli.net/2021/06/11/D8gCAM7uosaGyVi.png" alt="image-20210611105644934" style="zoom:67%;" />

**由【RXCHBONDLEVEL】端口设置主发送器经过多少周期后进行通道绑定操作**

### 通道绑定序列

<img src="https://i.loli.net/2021/06/11/9heLTHS1cb5DiMW.png" alt="image-20210611111121182" style="zoom: 67%;" />

<img src="https://i.loli.net/2021/06/11/rhCic5gjN2Ja4Y7.png" alt="image-20210611111133253" style="zoom:67%;" />

### 设定最大偏移

当主机接收到通道绑定序列时，它不会立即触发通道绑定。如果从服务器有更多的延迟，则必须多到达几个字节。这个等待时间成为RX弹性缓冲区可以处理的最大偏差。

如果偏差大于这个等待时间，那么在主服务器触发通道绑定时（**利用RXCHBONDO端口**），从服务器可能无法接收到序列。

![image-20210611150712189](https://i.loli.net/2021/06/11/YxoKhJzQwq72bOL.png)

> 这部分可能理解有偏差

由于通道绑定和时钟校准都会控制从弹性buffer指针，因此需要对两个操作进行优先级设定：

- CLK_COR_PRECEDENCE = 1 优先时钟校准
- CLK_COR_PRECEDENCE = 0 优先通道绑定