# 1 功能简介

Xilinx公司的收发器主要包括以下四种，四种收发器支持的最大线速率不同，每种收发器位于的器件也不同。

<img src="https://i.loli.net/2021/05/26/O7UNRF2dTqMhgtn.png" alt="image-20210526160100657" style="zoom:67%;" />

7系列FPGA GTP/GTX/GTH收发器特性如表

<img src="https://i.loli.net/2021/05/26/DUnlhujMrx1IPZq.png" alt="image-20210526160201426" style="zoom:80%;" />

支持如下物理接口：

<img src="https://i.loli.net/2021/05/26/9zGNmoasknxFLSA.png" alt="image-20210526160629740" style="zoom:80%;" />

首次使用收发器的开发者推荐阅读**《High-Speed Serial I/O Made Simple》**文献，其描述了高速串行收发器技术和应用。

https://china.xilinx.com/publications/archives/books/serialio.pdf

# 2 GTP收发器结构

A 7中的GTP收发器结构框图如下，一共有8组收发器：

<img src="https://i.loli.net/2021/05/26/w5cinlU8ECRWXtB.png" alt="image-20210526161532190" style="zoom:80%;" />

由4组GTPE2通道以及GTPE2_COMMON原语构成一个单元称为`Quad`，每个通道带有一对收发器。

<img src="https://i.loli.net/2021/05/26/ZdeUo8E647nuKy2.png" alt="image-20210526161635461" style="zoom:80%;" />

通道的内部拓扑如下

<img src="https://i.loli.net/2021/05/26/vFkf19Xr3ApiLxh.png" alt="image-20210526162156497" style="zoom:80%;" />

# 3 实现

推荐设计原则：在设计早期定义GTP收发器Quad的位置，以确保时钟资源的正确使用，并便于在电路板设计期间进行信号完整性分析。实现过程通过在XDC中使用位置约束来完成Quad位置分配验证。

该部分介绍映射7系列GTP收发器到器件资源所需的信息，主要包括三部分：

- GTP收发器Quad在可用器件封装的位置
- GTP收发器Quad外部信号的pad编号
- 如何使用XDC文件约束GTP收发器通道原句和时钟资源到可用位置

## GTP收发器Quad封装位置和pad编号

每个GTP收发器CHANNEL和COMMON原语的位置，由相对位置的XY坐标系指定，下图左边框内。目前所有的7系列器件家族中，GTP收发器Quads都位于单一的列中，并且位于芯片Die的一侧。

pad编号即图中间一列。

对于FGG484封装的FPGA来说，只有一个Quad可以进行配置：

![image-20210527155232272](ug482_7series_GTP_transceivers-0.assets/image-20210527155232272.png)

## GTP收发器约束

有两种方法可以为GTP收发器的创建XDC约束：

- 首选方法是使用7系列FPGA收发器向导。向导会自动生成XDC模板，然后可以编辑向导生成的XDC，以自定义应用程序的操作参数和放置信息。
- 第二种方法是手工创建XDC约束。使用此方法时，设计者必须输入控制收发器操作的配置属性以及位置参数。必须注意确保正确输入配置GTP收发器所需的所有参数。

## FPGA封装与GTP物理位置

该部分内容可以参照官方UG475文档，7 Series FPGAs Packaging and Pinout Specification，该文档描述了Xilinx器件管脚命名规则，器件信号布局等封装信息，对于FPGA原理图及PCB设计人员具有非常重要的作用。

![image-20210527151326462](ug482_7series_GTP_transceivers-0.assets/image-20210527151326462.png)

![image-20210527151353273](ug482_7series_GTP_transceivers-0.assets/image-20210527151353273.png)



不同封装的型号端口可使用情况不同，对于FGG484封装：

- HR I/O bank 13 部分端口可使用
- GTP Quad 213 端口不可使用，但是参考时钟是从这里引入

<img src="ug482_7series_GTP_transceivers-0.assets/image-20210527153242011.png" alt="image-20210527153242011" style="zoom:80%;" />