# 背景

本文关注简单axi-interconnect应用：2slave-1master

IP核配置情况如下：

![image-20210223153755202](AXI-interconnect.assets/image-20210223153755202.png)

![image-20210223153816828](AXI-interconnect.assets/image-20210223153816828.png)

![image-20210223153834132](AXI-interconnect.assets/image-20210223153834132.png)

------

# AXI 基础IP核

通过以上IP配置，可以得到如下IP核的展开图

![image-20210223154058353](AXI-interconnect.assets/image-20210223154058353.png)

其中couplers所代表为AXI总线直连，那么在上述配置产生的AXI-interconnect主要由两个子IP核构成：

- AXI MMU

  > AXI MMU provides address range decoding and remapping services for AXI Interconnect

  

- AXI Crossbar

  > AXI Crossbar connects one or more similar AXI memory-mapped masters to one or more similar memory-mapped slaves.

  AXI交叉开关（crossbar）为一个或多个类似内存映射的主机到从机提供连接服务。

  
  
  ## AXI Crossbar
  
  ​	每个AXI-interconnect都会配置有一个AXI crossbar，其具有如下一些特性：
  
  - 