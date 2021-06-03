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

<img src="https://i.loli.net/2021/06/01/dMmhaolXQ2xnuHv.png" alt="image-20210531163442181" style="zoom:67%;" />

这种情况下TXUSRCLK时钟为TXUSRCLK2时钟频率的**两倍**。

### TX 时钟IP配置界面

![image-20210601113050312](https://i.loli.net/2021/06/01/1wlM5Qj2rnoAEIY.png)

# 2 TX 8B/10B编码器

PCIe、SRIO、STAT等高速串行协议数据发送都采用了8B/10B编码方案，它是一种行业标准编码方案。8B/10B以每字节（8bits）两比特的开销来**换取DC直流平衡**，来确保时钟可以从数据流中正确恢复。GTP收发器内置8B/10B TX路径实现TX数据编码，无需消耗FPGA逻辑资源。

**使能8B/10B编码会增加TX路径上数据延迟，如果不需要，可以旁路8B/10B编码，以减少TX路径延迟。**具体见IP配置界面。

## 8B/10B比特和字节顺序

8B/10B编码器要求a0比特先发送，GTP收发器优先发送最右侧比特。为了匹配8B/10B编码，GTP收发器内部的8B/10B编码器自动翻转数据比特顺序。

![image-20210601110138389](ug482_7series_GTP_transceivers-2.assets/image-20210601110138389.png)

## K字符和差异控制

8B/10B编码表使用特殊的字符（K字符）用作控制功能。TXCHARISK端口用来指示TXDATA端口的数据是否为K字符。8B/10B编码器检查接收到的TXDATA字节，如果匹配为K字符，则TXCHARISK比特为置为高。

<img src="https://i.loli.net/2021/06/01/QqpKk6nZlf4RBNP.png" alt="image-20210601111235902" style="zoom:67%;" />

由于8B/10B是直流平衡的，所以发送“1”和“0”的个数比例应该为1:1。为了实现这一要求，编码器总是计算发送“1”和“0”之间的差值。每发送一个字符后，这种差值要么为“+1”，要么为“-1”，这种不同被称为 running disparity （运行差异）。

差异控制由表3-6控制：

<img src="https://i.loli.net/2021/06/01/sKftNI1ZTyaj6mE.png" alt="image-20210601110739422" style="zoom:67%;" />

## 编码方式IP配置界面

![image-20210601112606548](ug482_7series_GTP_transceivers-2.assets/image-20210601112606548.png)

# 3 TX Gearbox（变速器）

一些高速数据速率协议使用64B/66B编码来减少8B/10B编码的开销，同时保留编码方案的优点。TX Gearbox提供了对64B/66B和64B/67B编码支持。常见的高速协议Interlaken就采用了64B/67B编码方案。TX Gearbox支持2字节和4字节接口定义，数据加扰是在FPGA逻辑内实现的。

# 4 TX Buffer与TX Buffer Bypass

图3-11可以看到TX发送器的时钟域，在PCS组件中有两个并行的时钟域：

- PMA组件并行时钟（XCLK）
- PCS组件并行时钟（TXUSRCLK）

为了正确发送数据，XCLK速率必须匹配TXUSRCLK速率，另外，这两个时钟域之间的相位误差必须解决。

![image-20210601135352459](https://i.loli.net/2021/06/01/L14MbfUYPJqgWtu.png)

## TX Bufffer与TX Buffer Bypass对比

GTP收发器提供**两种方法解决XCLK和TXUSRCLK跨时钟域问题**：

- **TX Bufffer**
- **TX相位对齐电路**（TX Buffer Bypass应用时）

当TX Buffer旁路时，TX相位对齐电路被使用解决跨时钟域问题。也就是说，所有的**TX数据路径必须要么使用TX Bufffer，要么使用TX相位对齐电路**。表3-12给出了这两种方法在选取时的权衡。

<img src="https://i.loli.net/2021/06/01/AMTqInuD7PgJCYz.png" alt="image-20210601140707188" style="zoom: 80%;" />

TX Buffer的优势在于易使用，缺点在于会引入额外的延时。是**推荐的默认方式**。

TX 相位对齐电路的优势在于延时小，但是需要消耗额外的逻辑资源并且需要对时钟进行约束。另外当多个GTP发送器的线速率相同时，可以同来减少各个发送器之间的相位偏差。

## TX Buffer Bypass使用方法

旁路TX Buffer是7系列GTP收发器的**高级特性**，此时TX相位对齐电路用来实现XCLK和TXUSRCLK时钟域之间的相位差异，也可以实现TX延迟对齐调整。TX Buffer Bypass可以在以下两种模式使用：

- one channel (single lane) use TXOUTCLK
- a group of channels (multi-lane) sharing a single TXOUTCLK

需要配置属性：

<img src="https://i.loli.net/2021/06/01/mX68wiIRp1WAj4k.png" alt="image-20210601144912992" style="zoom:80%;" />

## TX Buffer IP配置界面

**当TXBUFSTATUS指示溢出时应该复位TX Buffer。**GTTXRESET、TXPCSRESET或者GTP收发器内部产生的TX Buffer复位都可以复位TX Buffer。为了使能TX Buffer，需要设置以下选项：

- TXBUF_EN = TRUE ：界面上勾选 `enable TX Buffer` 
- TX_XCLK_SEL = TXOUT：配置PMA时钟为TXOUTCLK

<img src="https://i.loli.net/2021/06/01/6y1hu5Mzaiowvgq.png" alt="image-20210601142615389" style="zoom:80%;" />

# 6 TX PRBS（伪随机序列）产生器

TX PRBS通常**用来测试高速链路的信号完整性**。GTP可以产生几种工业级PRBS，如表3-18：

<img src="https://i.loli.net/2021/06/01/TaiNSeXm2knb89W.png" alt="image-20210601150431418" style="zoom:80%;" />

PRBS-7用于测试8B/10B编码通道。

图3-17显示了TX PRBS序列产生器模块图。图中，错误插入模块用于检测链路连通性和抖动容限性测试，采用TXPOLARITY信号支持PRBS翻转。

<img src="https://i.loli.net/2021/06/01/4f3B271DQWchuvH.png" alt="image-20210601150721842" style="zoom: 67%;" />

## TX PRBS IP配置界面



![image-20210601151210325](https://i.loli.net/2021/06/01/SjXbiY8QsNz9nFR.png)

- PRBS Detecor：PRBS功能
- TXPRBSSEL：选择PRBS模式：

<img src="https://i.loli.net/2021/06/01/AF9Lnd2KheNUOPo.png" alt="image-20210601151922206" style="zoom:80%;" />

- TXPRBSFORCEERR：当该端口拉高，PRBS强制发送错误数据。当TXPRBSSEL=000时，该端口不影响输出数据。
- RXPRBS_ERR_LOOPBACK：
  - 当设置为1时，RXPRBSERR bit会从内部传输给同一个GTP的TXPRBSFORCEERR，可以避免抖动测试中的数据跨时钟域。
  - 当设置为0时，TXPRBSFORCEERR会强制发给TX PRBS。

### 链路测试模式

图 3-18显示了使用PRBS-7进行链路测试的示意图，PRBS配置如蓝框所示，只有PRBS模式的数据流才能被RX接收端的PRBS检测器接收。

<img src="https://i.loli.net/2021/06/01/ZSlnRV3j5fH4coB.png" alt="image-20210601153109892" style="zoom:80%;" />

### 抖动容忍测试

图3-19显示了使用PRBS-7模式进行抖动容忍测试。为了精确的计算器接收器的BER（比特错误率），可以采用外部抖动容忍检测器，PRBS配置如蓝框所示。

<img src="https://i.loli.net/2021/06/01/m6BsYHkebaqVRjv.png" alt="image-20210601153424006" style="zoom:80%;" />

# 7 TX管脚极性控制

如果GTP收发器TXP和TXN差分管脚在PCB布线时进行了交换，差分对发送输出的比特流会取反。有两种方式可以解决：

- 在并串转换之前对发送的数据位逐位取反。
- 通过TX极性控制，实现TXP和TXN极性交换。

表3-22给出了TX极性控制端口操作：

<img src="https://i.loli.net/2021/06/01/tsGB9Owvby6RgHf.png" alt="image-20210601154124401" style="zoom: 80%;" />

## TX 管脚极性IP配置界面

<img src="https://i.loli.net/2021/06/01/Y4h1kVqT92Sneu7.png" alt="image-20210601154611645" style="zoom: 67%;" />

选中可以配置极性端口

# 8 TX 时钟输出控制

TX时钟分频器控制模块有两个主要的组件：

- 串行时钟分频器控制模块（针对PMA）
- 并行时钟分频器及选择器控制（针对PCS）

图3-20给出了时钟分频器和选择器详细的结构。

![image-20210601163941671](https://i.loli.net/2021/06/01/6MrpQHGViez1xtg.png)

需要有几个注意点：

1. TXOUTCLKPCS和TXOUTCLKFABRIC是**冗余输出**。**TXOUTCLK时钟一般用于FPGA内部逻辑设计**。
2. REFCLK_CTRL选项由软件自动控制的，用户不可选择。用户只能使用使用IBUFDS_GTE2中的O或者ODIV2通过CMT、BUFH或者BUFG输出到FPGA逻辑资源。
3. IBUFDS_GTE2可以看做**冗余时钟**，增加了收发器时钟方案的灵活性。
4. /4或者/5分频器模块由GTPE2_CHANNEL的TX_DATA_WIDTH属性控制。TX_DATA_WIDTH = 16，32时，选择/4分频器；TX_DATA_WIDTH = 20，40时，选择/5分频器。

## 串行时钟分频器

每个发送器PMA模块有一个D分频器，用来将PLL时钟分频为较低的线速率要求的时钟。该分频器可以用于设置为静态线速率或者动态线速率。静态线速率和动态线速率配置如表3-23所示。

<img src="https://i.loli.net/2021/06/01/BN4pZjJhtMIPeTw.png" alt="image-20210601164239161" style="zoom:80%;" />

## 并行时钟分频器和选择器

**从TX时钟分频器模块输出的并行时钟可以用于FPGA逻辑时钟**，Xilinx推荐的FPGA逻辑时钟为TXOUTCLK或者使用MGTREFCLK管脚输入时钟直接作为FPGA逻辑资源时钟。TX时钟输出控制端口定义如表3-24所示。

![image-20210603095603888](https://i.loli.net/2021/06/03/PeQlM9BkhqsxuY3.png)

![image-20210603102131807](https://i.loli.net/2021/06/03/Skrmvz1AJjHfCNd.png)

这部分端口对应图3-20时钟结构以及注意点。TXRATE在表3-23中有对应。

# 9 TX 配置驱动器

GTP收发器的TX驱动器是一个高速电流模式差分输出缓冲器。为了最大信号完整性，它包括以下特性：

- 差分电压控制
- Pre-cursor和Post-cursor发送器预加重（pre-emphasis）
- 校准端接电阻

<img src="https://i.loli.net/2021/06/03/TvioRZU5247WNYf.png" alt="image-20210603101814968" style="zoom:80%;" />

TX配置驱动器部分端口如表3-28所示：

<img src="https://i.loli.net/2021/06/03/QhUgBaSl7d3Rw15.png" alt="image-20210603111527746" style="zoom: 80%;" />

## 预加重

千兆位级驱动器最重要的特性可能就是预加重的能力。

**预加重是为了解决码间干扰（ISI）问题**：如果串行流包含多个比特位时间的相同数值数据，而其后跟着短比特位（1或2）时间的 相反数据数值时，会发生符号间干扰。介质（传输通道电容）在短位时间过程中没有足够的 充电时间，因此产生了较低的幅度。

具体如下所示：

<img src="https://i.loli.net/2021/06/03/gbQBm2GSR46eHao.png" alt="image-20210603112317029" style="zoom:67%;" />

在符号间干扰的情况下，长时间的恒定值将通道中的等效电容完全的充电，在紧接着的相反数据数值位时间内无法反相补偿。所以，**相反数据的电压值有可能不会被检测到**。

这个问题的解决方法是：转变开始时加入过量驱动，而在任意的连续相同数值时间内减少驱动量。**预加重是在转变开始前的有意过量驱动。**

对应到IP中也有相关配置，如表3-28：

![image-20210603135824224](https://i.loli.net/2021/06/03/dUFJSCQjT8i7fzv.png)

![image-20210603135836582](https://i.loli.net/2021/06/03/Lw5vuzSlJay63WO.png)

## TX 配置驱动器IP配置界面

![image-20210603141049368](https://i.loli.net/2021/06/03/Cpf1bk7MBqOnsXK.png)

- TXINHIBIT：置1，此信号阻止TXDATA传输，并强制GTPTXP和GTPTXN为1
- TXMAINCURSOR：可直接设定信号的主要窗口，范围如下

$$
51 – TXPOSTCURSOR~coefficient units – TXPRECURSOR~coefficient units \\< TXMAINCURSOR~ coefficient units\\< 80 –TXPOSTCURSOR~coefficient units – TXPRECURSOR~coefficient units.
$$

# 11 TX OOB信号

每个GTP发送器都支持生成OOB序列用于SATA、SAS或PCIE。其端口如表3-32所示：

<img src="https://i.loli.net/2021/06/03/r7CBgeLGoyODwEU.png" alt="image-20210603144310932" style="zoom:80%;" />

<img src="https://i.loli.net/2021/06/03/S4wEAn6mqgcGUCX.png" alt="image-20210603144322606" style="zoom:80%;" />

对应的配置参数如表3-33：

<img src="https://i.loli.net/2021/06/03/hFjV52SkwH8Ly7c.png" alt="image-20210603150657529" style="zoom:80%;" />

## TX OOB IP配置界面

![image-20210603150548691](https://i.loli.net/2021/06/03/3LKFQPbCU6Ogdxf.png)

