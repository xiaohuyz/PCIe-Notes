# 8b/10b encoding for 2.5 GT/s and 5.0 GT/s

本节中，阐述了PCIe Gen1/Gen2中所使用的8b/10b编码。

PCIe中的8b/10b encoding仅仅适用于**2.5 GT/s**和**5.0 GT/s** data rate。

## Symbol Encoding

在2.5 GT/s和5.0 GT/s下，PCIe使用8b/10b传输模式。在这种场景下，实际上8bit的data被分为2部分，即3bit和5bit，而后分别映射为4bit和6bit。Control bit与data character一起，决定了何时来编码8b/10b中的12个special symbols中的一个。

### Data的串并转换

如上图所示，分别展示了在x1宽度和x4宽度下的数据传输，**首先放在lane上的是**a，即**按时间顺序从左向右传输**。

其实在PCIe中，TL层的SPEC描述TL的Layout时就将Byte 0放于最左边，以说明其最先传输。在bit级也是先传输bit 0

> Symbol time，指的是将一个Symbol放到Lane上所需要花费的时钟周期
>
> UI，即Unit Interval，指的是传输1 bit数据所花费的时间（在NRZ编码下） / 传输2 bit数据所花费的时间（在PAM4编码下）

![image-20240413140638673](./assets/image-20240413140638673.png)

### 用于Framing和Link Management的Special Symbols（K codes，即Control codes） 

8b/10b编码下，提供了一些和Data Symbol不同的Special Symbols来表示characters。这些special symbols有两个功能：

- 用于**各种Link管理机制**
- 在**Non Flit Mode**下，用于**frame TLPs和DLLPs**。用这些special symbols，有助于这两种packets被快速且轻松的识别

**在Flit Mode下，Data Stream中的每个Byte仍然使用8b/10b编码**，但是**不使用Framing**，因为数据已经被打包成固定大小的Flit了，就无需进行Framing了。

注意即使启用了Flit Mode，在FM和NFM下，2.5 GT/s和5.0 GT/s所使用的所有Ordered Sets都是相同的。

下面展示了PCIe所使用的special symbols。注意，special symbols和data symbols必须通过**对整个10bit symbol的完全解析**来确认。这和使用PAM4编码时的一些Ordered Set不同，有的OS在看完某些field之后便可以进行判断，这是因为Gen6及以上采用了PAM4编码，有更高的误码率。

1. K28.5，即COM / Comma，用于**Line和Link的初始化和管理**，例如用于Rx获得symbol lock，在8b/10b编码中用于Ordered Set的开头。在FM和NFM下用法是一致的。 一般PCIe PHY的log里都直接把他标记为COM

2. K27.7，即STP，在NFM下，用于**Framing**。用于标记一个TLP的开头。

3. K28.2，即SDP，在NFM下，用于**Framing**。用于标记一个DLLP的开头。

4. K29.7，即END，在NFM下，用于**Framing**。用于标记TLP / DLLP的结尾

5. K30.7，即EDB，在NFM下，用于**Framing**。用于标记一个**无效TLP（nullified TLP）**的结尾。这个用于cut through模式的Switch。在cut through模式下，switch会直接在USP接收到TLP的时候，将其转发给DSP，而不用等待全部的传输，但是有可能当USP接收完TLP时发现TLP是有错误的，此时使用nullified机制来标记该TLP无效，从而使得DSP的link partner直接丢弃该TLP，而不触发Ack / Nak机制

6. K23.7，即PAD，在NFM下，仅仅用于**Framing。**在FM和NFM下，用于Link Width和Lane Ordering的协商。在协商过程中，发送和接收TS1 / TS2 OS时，有时候Link number / Lane number需要为PAD

7. K28.0，即SKP，在FM和NFM下用法一致。用于细微时钟差的补偿，即Rx侧内部logic时钟和Rx侧从data stream中恢复出来的时钟的差距补偿。

8. K28.1，即FTS。仅仅用于NFM。用在Ordered Set中，使device**从L0s恢复到L0**。即从低功耗状态的快速恢复

9. K28.3，即IDL。在FM和NFM下用法一致，用于Electrical Idle Ordered Sets（EIOS）中，使得link进入Electrical Idle State。

10. K28.4 / K28.6，reserved

11. K28.7，即EIE。**在2.5 GT/s下是reserved**。在其他速率下，用于
    1. Electrical Idle Exit Ordered Sets（EIEOS），用于将link返回L0 full power状态
    
    2. 在发送FTS之前发送
    
       > 为啥在2.5 GT/s下reserved？因为在2.5 GT/s下Tx侧无需使用EIEOS来使得Rx侧退出Electrical Idle

### 8b/10b编码和解码规范

8b/10b编码根据现在的正负极性，**一个8b character能够被编为2种不同的10b symbol**。Link上不能够连续传输同样极性的symbol，因为一直让1的个数大于0，或者相反，会使得lane上的电容充电，从而影响信号稳定性。

如图所示

![image-20240413141858557](./assets/image-20240413141858557.png)

在离开Electrical Idle state后，第一次发送差分数据时，transmitter允许选择任意极性。如果Rx侧此时选择了不同的极性，那么不会认为是一个Error，而是直接将Rx的极性修正为Tx侧的即可。而后，直到下次进入Electrical Idle State，transmitter必须始终遵循8b/10b编码规范。

对于Receiver而言，它将自己的极性设置为用于**获得symbol lock**的第一笔symbol的极性。如果symbol lock丢失，则允许极性被重新初始化，而后由后续的差分数据传输中重新获得。在初始disparity设置之后，所有**接收到的symbol都必须按照表格中的合适的列进行编码**。

如果接收到的symbol不正确，即Receiver的Current RD为-，但是收到的10bit symbol却处于RD+的那一列，则PL层必须告知DL层接收到的symbol是invalid的。在Non Flit Mode下，这是一个receiver error，并且由相关的port进行报错。在Flit Mode下，这个symbol会被**送往FEC来进行修复**。如果在Flit边界里探测到了8b/10b error或者K-character，则receiver允许发送任意8-bit value给FEC logic。

## Framing and Application of Symbols to Lanes

对于lane而言，有两种**Framing and Application of Symbols**。第一种由Ordered Sets组成，而第二种由Data Stream中的TLPs和DLLPs组成。Ordered Sets总是在所有lane上串行传输的，也就是在一个Multi-Lane Link上，**Ordered Set同时出现在所有lane上**（**每个Lane上传输一样的OS**）。接下来描述在**Non Flit Mode**下Data Stream的传输。注意在使用**8b/10b编码传输**，且使用**Flit Mode**的情况下，没有framing-related errors（因为**在Flit Mode下，根本就没有Framing的概念**）

### Framing and Application of Symbols to Lanes for TLPs and DLLPs in Non-Flit Mode

整个Framing机制，使用Special Symbol **K28.2 "SDP"**来开始一个DLLP，Special Symbol **K27.7 "STP"**来开始一个TLP，使用Special Symbol **K29.7 ”END"**来标记一个TLP或者DLLP的结束。

整个数据流，**以symbol为单位**映射到所有Lane上。例如第一个symbol被映射到lane 0，第二个symbol被映射到lane 1，等等。

当没有**Packet information（即data）**或者**special Ordered Sets**正在被传输的时候，transmitter处于**logical Idle**状态。**在此时，必须transmit idle data**。Idle data实际上就是00h data，而后像一个常规的Data character一样，**经过scramble和8b/10b编码**，而后发送。同理，当receiver没有接收任何packet information或者special ordered sets的时候，它**处于logical Idle状态**，需要像上面一样接收Idle data。注意，在logic idle状态下，发送idle data时，**SKP OS必须仍然继续被传输**，因为此时仍然需要进行时钟补偿。这是为了在Logical Idle state下，使得Rx侧的PLL仍然保持锁定状态，与Tx侧时钟同步。

接下来描述Data Stream中TLP和DLLP的**framing规则**，以及它们是如何被**分配到一个link的每个lane上**的，注意，place，即放置，**意思是transmitter必须将这个symbol放在link的合适的lane上**：

- TLP的framing：在TLP的开头处放置一个STP，在TLP的结尾处放置一个END或者EDB
- 一个正常的TLP，在STP和END/EDB symbol之间，至少有**18个symbol**（minimum 12 bytes Header + 2 bytes sequence number + 4 bytes LCRC，注意这是对于Non Flit Mode TLP而言的），其中**sequence number和LCRC是经过DL层时加上的**。
- DLLP的framing：在DLLP的开头放置一个SDP，结尾处放置一个END，如图所示。注意DLLP的**长度总是固定的**。正好是**8 symbol**
- Logical Idle：定义为这样一段**symbol times**，在此时，没有information（TLPs，DLLPs，Special Symbols）正在被传输/接收。与**Electrical Idle**不同，在Logical Idle状态下，发送和接收**idle data symbol （00h）**
  - 当transmitter在logical idle状态下时，idle data（00h）应该**在所有lane上同时传输**。注意idle data也**需要被scramble**
  - receiver必须丢弃接收到的idle data symbols。
- 对于宽度超过x1的link，当从**Logical Idle状态恢复**后重新发送TLPs时，STP必须位于Lane 0
- 对于宽度超过x1的link，当从**Logical Idle状态恢复**后重新发送DLLPs时，SDP必须位于Lane 0
- 在一个symbol time内，STP不能被放在Link上超过一次**（理论上这是不可能出现的，因为PCIe Gen 6的最大宽度是x16，而一个完整的TLP至少有20 symbol）**
- 在一个symbol time内，SDP不能被放在Link上超过一次，也就是说**不能连续不断传送2个DLLP**，因为在PCIe Gen 6中，最大宽度为x16，这种情况只可能出现在连续不断传输2个DLLP时
- 在上述条件满足的情况下，TLPs和DLLPs能够接连传输（其实意思就是**在同一个symbol time内可以同时传输1个 TLP和1个 DLLP**）
- 在同一个symbol time内，一个STP和一个SDP能同时放在link上
  - 对于宽度大于x4的link，**STP/STD允许被放在4n的lane上**。比如对于x16的link，STP/SDP可能出现在lane 0，4，8，12上
- 对于**x8及以上的link**，有可能出现END/EDB并**没有出现在Lane n-1**上，而且END/EDB后面也**没有STP/STD的情况（即后面没有TLP/DLLP要传输了）**。此时，该symbol time内，link上剩余的Lane必须**用PAD Symbol来填充**。注意不能使用IDL来填充，因为IDL必须同时出现在所有Lane上
- EDB用于标记一个**无效的TLP**的结尾
- 以上检查是可选的

##  Data Scrambling

为了提升link的电气性能，数据一般是**加扰**的。在2.5 GT/s和5.0 GT/s（即**使用8b/10b编码**）下，无论工作于Flit Mode还是Non Flit Mode，Scramble都适用。所谓加扰，就是**将data stream与LSFR产生的一个pattern按位异或**。对于transmitter，scramble步骤在**8b/10b编码之前**。对于reciever，de-scramble步骤在**8b/10b解码之后**。scramble模式是可以预测的，并非真的完全随机（因为LSFR的行为实际上是可以预测的）。

对于一个multi-Lane Link，可以采用**一个或者多个**LSFR来实现scramble。当采用多个LSFR时，它们必须步调一致，即在每个LSFR中维持相同大小的Lane-to-Lane output skew value。无论实现了多少LSFR，它与data symbol的交互都是以lane为基础的，就像每个lane上有一个独立的LSFR一样。

LSFR如图所示：

![image-20240413143037330](./assets/image-20240413143037330.png)

Scrambling和unscrambling都是**串行进行**的，将**8bit data与16bit的LSFR输出进行异或**。比如，D15与D0进行异或，而后，**LSFR和data register都线性前进一位**，而后则是D14和D1的异或。

即G(X) = X^16^+X^5^+X^4^+X^3^+1

这是一个伽罗瓦LSFR，其抽头作用于bit 3，4，5，16

Disabling scrambling有助于简化测试和debug。DL层用来告知PL层**停止scramble**的机制和接口由具体实现决定。

scramble rules：

- 使用COM Symbol来**周期性重置link两端的LSFR**，从而解决link两端的LSFR可能出现的失调
- 除了**SKP Symbol**之外，对于每个symbol，LSFR**移位8次。因为每个symbol为8bit**（注意，加扰在8b/10b编码之前执行）
  - SKP OS的发送和接收端，LSFR不能移位，原因时SKP OS在Tx和Rx端，其中含有的SKP Symbol数量是不一样多的，如果LSFR移位了，则会造成link两侧LSFR的失调

- 需要被scramble的data symbols：
  - 不在Ordered Sets中的所有data symbols，注意OS中有data symbols，用来指示某些数字，这些data symbols时不经过scramble的
  - Compliance Pattern
  - Modified Compliance Pattern
- **所有special symbols（K codes）都不经过scramble**
- LSFR最初的种子为FFFFh。当COM**一离开**Transmit LSFR，transmitter的LSFR就进行重置。当COM**进入receiver的任何lane**时，receive side的所有lane的LSFR进行重置
- Scramble仅仅只能在**end of configuration**阶段被disable
- Scrambling不适用于**Loopback Follower**
- Scrambing默认情况下，在**detect**阶段总是启用的
