# PCIe Physical Layer

PCIe物理层将TL层和DL层与link data exchange所实际使用的**信号技术**隔离开来。物理层分为逻辑和电气两个子部分。此处我们仅关注逻辑子部分。

> 何谓信号技术？即具体信号传输所使用的方式，如Gen1~5所使用的NRZ编码和Gen6及以上所使用的PAM4编码

PCIe的logic sub-block同样分为两个部分，即Transmit section和Receive section。

PCIe的逻辑部分和电气部分，通过一个**status and control register interface**或者其**功能等效**，来实现各自收发部分的状态交换。PL层的逻辑部分，负责PL层功能的控制和管理。

> status and control register interface. 具体细节见Intel的PIPE SPEC

PCIe协议使用3种**编码方式**，即

- 8b/10b
- 128b/130b
- 1b/1b

同时，采用两种**数据流模式**，即

- Flit Mode，在FM下，数据被打包成**Flit**，其数据传输方式为**SDS Ordered Set -> SKP Ordered Set -> Flit -> 非SKP Ordered Set的OS（该OS使得Link退出L0 state，例如使用TS1 OS来使得link进入Recovery.RcvrLock） / Link进入Electrical Idle** 

- Non Flit Mode，其数据组成方式为**end of an Ordered Set -> 连续的TLPs / DLLPs / Logical Idle / IDL Token -> another Ordered Set / Link Electrical Idle**

  其中，编码方式由link的**data rate**决定（**实际上就是由PCIe协议的代数决定**），而数据流模式，由最初的**link training**决定，如果Flit Mode没有被disabled（通过**Flit Mode Disable bit**），并且所有port / Pseudo-port 支持Flit Mode（通过**Flit Mode supported bit**），则选择Flit Mode。否则选择Non-Flit Mode。注意Flit Mode的协商是在Configuration state完成的

所有可用的编码方式和数据流模式如图所示：

![image-20240413132141400](./assets/image-20240413132141400.png)

1. 除了64.0 GT/s速率强制使用Flit Mode外，**是否启用Flit Mode与速率无关**
2. 除了64.0 GT/s下的1b/1b编码强制使用Flit Mode外，是否启用Flit Mode与data encoding方式无关

注意，对于**同一data rate**，Ordered Sets在Flit Mode和Non Flit Mode下有所不同，如下表所示。其实主要是**SKP OS有所不同**。

![image-20240413132401096](./assets/image-20240413132401096.png)

1. 对于2.5 GT/s和5.0 GT/s，所有的OS在FM和NFM下是一样的
2. 对于8.0 GT/s，在NFM下，当需要发送SKP OS时，**仅仅只能发送Standard SKP OS**。在FM下，可以选择发送Standard SKP OS或者Control SKP OS
3. 对于16.0 GT/s和32.0 GT/s，所有OS在FM和NFM下是一样的。在FM和NFM下，当需要发送SKP OS时，可以选择发送Standard SKP OS或者Control SKP OS
4. 对于64.0 GT/s，**仅仅只能使用Flit mode**，所有的OS都必须使用1b/1b编码，采用PAM4信号传输方式。当需要发送SKP OS时，**只能发送Control SKP OS**

对于Flit Mode的支持与协商，spec中有5处定义：
1. Flit Mode Supported bit。在**8b/10b编码**或者**128b/130b编码**的**TS1和TS2**中的**Data Rate Identifier Symbol**的**Bit 0**。**该bit受两个寄存器共同控制**。当PCI Express Capabilities Register中的**Flit Mode Supported bit set**，而且Link Control Register中的**Flit Mode Disable bit clear**时，该bit set。

2. Flit_Mode_Enabled。该变量用来**指示link两端是否已经成功协商使用Flit Mode**。这并非一个PCIe规定的标准寄存器bit，而是具体逻辑定义的一个内部寄存器。

3. Flit Mode Supported。该bit位于PCI Express Capabilities Register。用于指示该port是否支持Flit Mode

4. Flit Mode Disable。该bit位于Link Control Register。用于实际控制port是否启用Flit Mode。主要用于Downstream Port

5. Flit Mode Status。这是一个状态寄存器

   * 该bit位于Link Status 2 Register，用于指示Link正工作于Flit Mode，或者将要工作于Flit Mode

   * 该bit的值必须与Flit_Mode_Enabled变量相匹配，即不可能在Flit_Mode_Enabled=0的状态下，Flit Mode Status=1。 


