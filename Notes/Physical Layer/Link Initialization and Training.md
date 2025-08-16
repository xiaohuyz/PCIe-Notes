# Link Initialization and Training
在本章中，定义了PL层的控制过程，用于配置和初始化每个link，从而使其能正常运作。

该部分包含以下内容：
- link的配置和初始化
- 支持常规的packet传输，即TLP的传输
- 当从link errors中恢复时，支持state转换。其实就是切换到recovery state以重新训练link。基本的retrain流程就是重新获得bit lock / symbol lock / block align
- 从低功耗状态中重启port，例如从L1 / L2等等

在训练过程中，下列参数被发现和配置：
- Link width，即连接宽度
- Link data rate，即连接速率
- Link reversal，即是否有lane number的反转
- Lane polarity，即是否有单条lane上的电平反转

Training过程中：
- Link data rate协商
- 在每条Lane上进行bit block，即时钟锁定
- 决定是否有Lane polarity，即电平反转
- 在每个lane上进行Symbol lock或者Block alignment，其实就是确定传输的数据的块边界，symbol lock用于8b/10b编码，block alignment用于128b/130b编码和1b/1b编码
- 在一个link上决定好lane ordering，即lane号码的分配，其实只有两种分配方式，即lane reversal和正常模式
- Link width的协商
- 在一个multi-Lane Link上，进行Lane-to-Lane的de-skew，即每条lane的对齐
## Training Sequences

本节描述Training Sequences，即TS。

Training Sequences**由Ordered Sets组成**，主要用于进行bit对齐 / symbol边界的对齐 / PL层参数的交换。

对于TS是否经过scramble的判断，具体scramble规则前几章有述：
1. 当工作于2.5 GT/s或者5.0 GT/s时，Ordered Sets从来不经过scramble，总是直接进行8b/10b编码
2. 当工作于8.0 / 16.0 / 32.0 GT/s时，使用128b/130b编码，Symbols有的经过scramble，有的不用
3. 当工作于64 .0 GT/s时，使用1b/1b编码，Symbols有的经过scramble，有的不用

Training Sequences（TS0 / TS1 / TS2 / Modified TS1 / Modified TS2）都是**接连不断的进行传输**的，它们仅仅能在以下情况下被打断：
- 被SKP OS打断
- 在**除了2.5 GT/s之外**的其他速率中，被EIEOS打断

当支持8.0 GT/s或以上速率时，使用8b/10b编码的TS1 / TS2（2.5或者5.0 GT/s下）OS可能是一个标准的TS1 / TS2 OS（对TS1，其symbol 6为D10.2，对TS2，其symbol 6为D5.2），或者是一个EQ TS1 OS / EQ TS 2（Symbol 6 bit 7为1b） OS。发送EQ TS1 OS的能力由具体实现决定。对于支持8.0 GT/s及以上速率的port，在LTSSM States，它必须支持接收**任意一种TS Type**。不支持8.0 GT/s的ports允许接收EQ TS，但并不强制。EQ TS1 OS是在Redo EQ时使用的

当支持16.0 GT/s及以上速率时，使用128b/130b编码的TS2可能是一个标准的TS2 OS（Symbol 7 为45h），也有可能是一个**128b/130b EQ TS2 OS**（Symbol 7 bit 7为1b）。对于支持16.0 GT/s及以上速率的port，在LTSSM States，它必须支持接收**任意一种TS2 OS Type**。不支持16.0 GT/s的ports允许接收128b/130b TS2 OS，但并不强制。EQ TS2 OS是在8.0 GT/s向16.0 GT/s，以及16.0 GT/s向32.0 GT/s做EQ时在Recovery.RcvrCfg state所使用的

当使用8b/10b编码时，只有当当前TS1 / TS2的**symbol 6**和之前的TS1 / TS2的**symbol 6**相同时，才认为TS是**连续**的。这定义了连续TS1 / TS2 OS的概念，这对链路训练至关重要

想要协商一个可选协议，或者传输Training Set Message的components，必须在它们的**Configuration.Lanenum.Wait**, **Configuration.Lanenum.Accept**, **Configuration.Complete**这三个substates使用**Modified TS1 / TS2 OS**，即在涉及lane number确定的state就开始用Modified TS了。为了能够在上述3个substate中发送Modified TS1 / TS2 OS，components必须在一开始就设置TS OS，即将在**Polling.Active, Polling.Configuration, Configuration.Linkwidth.Start, Configuration.Linkwidth.Accept**这四个substates中交换的的TS1 / TS2 OS中的**Enhanced Link behavior Control bits** (bit 7:6 of Symbol 5)设置为11b，同时当LinkUp=0b时，遵循切换到**Configuration.Lanenum.Wait** substate的步骤。如果link partner并不支持Modified TS1 / TS2，则从**Configuration.Lanenum.Wait** substate开始，标准的TS应该停止在**Enhanced Link behavior Control bits**发送11b，并且切换到合适的编码（其实就是一方支持发送Modified TS，而另一方不支持）。

接下来描述何谓连续的TS1/TS2，连续的TS常常用于链路训练的过程 

当使用8b/10b编码时，对于Modified TS，仅仅当**所有symbols都与前一个Modified TS1 / TS2相同**，且**symbol 15的parity与预期值相同**时，才认为modified TS1 / TS2是连续的。对于link上的所有lane，symbols 8～14必须相同。

当使用128b/130b编码时，对于标准TS OS，当**symbol 6～9与前一个TS1 / TS2相同**，才认为TS1 / TS2是连续的。对于Reserved field的处理，遵循下方的要求。

当使用1b/1b编码时，TS0 / TS1 / TS2 OS中，它们的前8个symbol和后8个symbol完全相同。在使用1b/1b编码时，在Recovery.Equalization阶段，使用TS0 OS。对于TS0 OS，每个UI发送的电平要么是0，要么是3，即encoding为00b或者11b。因此，TS0中scramble的symbols被称为**Half Scrambled**，即奇数bit经过scramble，而偶数bit直接与奇数bit相同。对于TS0 OS，在transmitter和receiver侧都跳过precoding。

明显，由于在1b/1b编码中，由于TS0 / TS1 / TS2 OS的前8个symbol和后8个symbol完全相同，因此只要它们任意一个half是valid的，则认为整个TS是valid的：
- 如果symbol 0是一个valid TS0 / TS1 / TS2 encoding，且在接收方经过格雷码解码和解扰之后，symbol 0～7通过奇偶校验检查（在symbol7是奇偶校验位，而非DC balance pattern的情形下），则认为该half是valid的
- 同理，如果symbol 8是一个valid TS0 / TS1 / TS2 encoding，且在接收方经过格雷码解码和解扰之后，symbol 8～15通过奇偶校验检查（在symbol15是奇偶校验位，而非DC balance pattern的情形下），则认为该half是valid的
- 在symbol7和symbol15都为DC balance，而非奇偶校验位的情况下，如果symbol 0～6与symbol 8～14相同，则这两个half都是valid
- 其中，具体的TS种类，由valid half的第一个symbol决定
- 对于Half Scrambled的TS，当经过格雷码解码和解扰之后，需要**直接丢弃偶数bits**。因为Half Scramble的机制就决定了经过格雷码解码和解扰之后恢复出的bit流中的偶数bits不一定等于发送的值。

在1b/1b编码下，两个TS0 / TS1 / TS2编码在满足以下情形时，被认为是连续的：
- 这两个TS是同一种类
- 这两个TS是接连不断的到达的（注意这两个TS可能被SKP OS隔开）
- 每个OS都有一个Valid Half，且在Valid Half中，first 7 symbols相同。而reserved fields照以下规则处理

对于TS0 / TS1 / TS2 / Modified TS1 / Modified TS2 OS，其中的reserved bits必须按照如下规则来处理：
- 对于transmitter，对reserved bit，必须设置为0
- 对于receiver
  - 不允许根据reserved bit来判断 TS OS是否为valid
  - 如果reserved bits被包含在奇偶校验中，则receiver计算奇偶校验时，必须如实使用接收到的reserved bits值
  - 可选的，如果确实需要这么做的话，比较symbols中的reserved bits，来决定两个TS是否为连续的
  - 除此之外，不允许根据reserved bits来进行任何操作

下面描述有关DC Balance的规则：

当使用128b/130b编码或者1b/1b编码时，transmitter必须追踪此时wire上正在传输的**组成TS0 / TS1 / TS2 OS的bit流**的DC balance（该bit流经过1b/1b下的格雷码编码，加扰，precoding（如果开启））。

对于8.0 / 16.0 / 32.0 GT/s的传输速率，running DC balance指的是已经传输的0和已经传输的1的个数之差

对于64 GT/s的传输速率，running DC balance指的是已经传输的所有电平的DC balance值之和。如图所示：

![image-20240525222902164](./assets/image-20240525222902164.png)



每一条lane都必须能够独立追踪自己的DC balance，且要能够追踪至少+/- 511bits的0和1个数之差。注意，对于这些计数器，允许计满，但**不允许roll-over**。例如计数器的最大值为511bits，如果探测到了一个513的difference，则它记为511，如果在未来difference减少2，则计数器变为509.

在以下两种情况下，DC balance被重置为0：
1. Transmitter退出Electrical Idle
2. 在Data block后发送一个EIEOS

对于使用128b/130b编码的TS1和TS2 OS，transmitter必须评估正在运行的DC balance，而且按照下列算法来传输symbol 14和symbol15：
- 如果1的数量需要被减少，则symbol 14为20h，symbol 15为08h
- 如果0的数量需要被减少，则symbol 14为DFh，symbol 15为F7h
- 如果无需变化，则symbol 14和symbol 15传输合适的TS1 / TS2 Identifier
- 对于symbol 14和symbol 15，如果他们是DC balance symbol，则它们跳过scrambling，如果是TS1 / TS2 Identifier，则遵循常规的scrambling rules
- **注意，**如果**在TS的symbol11结尾处**，**对于DC Balance > 31，则symbol 14和15都使用DC Balance symbol。而DC Balance > 15，则仅仅symbol 15使用DC Balance Symbol，而symbol 14仍然使用TS1 / TS2 Identifier。对于DC Balance在0～15，则认为无需变化。其实就是描述如何平衡DC balance**

对于Receiver而言，允许他们检测symbol 14和15，以确定该TS1 / TS2 OS是否为一个Valid TS。
- 在解扰之后，该symbol是一个合适的TS1 / TS2 Identifier
- 解扰之前，symbol 14为DFh或者20h
- 解扰之前，symbol 15为F7h或者08h
- 注意，如果在symbol 14处得到的是一个DC Balance pattern，针对symbol 15，仍然要保持解扰和de-precoding逻辑运作，因为这种情况下，symbol 15仍然可能是一个TS Identifier。其实就是针对上述的DC Balance在0~15的情况

注意，Block Sync Header以及TS1 / TS2 OS的第一个symbol对DC balance没有影响，因为他们中的0和1的数量相等。

当使用1b/1b编码传输TS0 / TS1 / TS2 OS时，transmitter必须评估正在运行的DC balance，而且按照下列算法来传输symbol 7和symbol15。注意在symbol 7和15传输的DC balance symbols是跳过scrambling的：
- 如果**在TS OS开始之前**，DC Balance > 47，则在symbol 7和symbol 15处发送00h，且不加扰
- 如果**在TS OS开始之前，**DC Balance < -47，则在symbol 7和symbol 15处发送FFh，且不加扰
- 否则，设置symbol 7和symbol 15，使得扰码前的symbol 0～7，以及symbol 8～15满足偶校验，注意，**此时symbol 7和symbol 15经过扰码**。其实这就是不调整DC balance的情况

对于TS中的Hot Reset bit, Disable Link bit, and Loopback bit这三个training control bits，**他们是互斥的**。在一个configured link（all lanes in L0）或者possible link（all lanes in configuration）上传输TS时，Hot Reset bit, Disable Link bit, and Loopback bit，这三个bit不能超过一个为1.

当Upstream port或者Downstream port传输TS时，TS1 OS中的Retimer Equalization Extend bit总是为0。这个bit是由retimer来负责设置的

下表展示了8b/10b编码和128b/130b编码下使用的TS1 OS。注意TS OS为16 symbol长度。

| Symbol Number | Description                                                  |
| :-----------: | :----------------------------------------------------------- |
|       0       | **TS1 Identifier**<br>1. 当工作与2.5 GT/s或者5.0 GT/s下时，为COM（K28.5）,目的是为了实现Symbol alignment<br>2. 当工作于8.0 GT/s及以上速率时，编码为1Eh，表明这是一个TS1 OS |
|       1       | **Link Number**<br />1. 对于不支持8.0 GT/s及以上速率的Port，该symbol为PAD或者0-255<br />2. 对于支持8.0 GT/s及以上速率的Downstream Ports，该symbol为PAD或者0-31<br />3. 对于支持8.0 GT/s及以上速率的Upstream Ports，该symbol为PAD或者0-255<br />4. 当工作于2.5 GT/s或者5.0 GT/s时，PAD为K23.7<br />5. 当工作于8.0 GT/s及以上速率时，PAD为F7h |
|       2       | **Lane Number within Link**<br />1. 当工作于2.5 GT/s或者5.0 GT/s时，该symbol为PAD或者0-31，其中PAD编码为K23.7<br />2. 当工作于8.0 GT/s或者以上速率时，该symbol为PAD或者0-31，其中PAD编码为F7h |
|       3       | **N_FTS**-Receiver所需要的Fast Training Sequence的数量，该symbol为0-255。**当成功协商了Flit Mode之后，该symbol为rsvd** |
|       4       | **Data Rate Identifier**<br />Bit[0] - Flit Mode Supported bit:<br />    **0b**: Flit Mode is not Supported<br />    **1b**: Flit Mode is Supported<br />Bit[5:1] - Supported Link Speeds<br />    Non-Flit Mode Encodings，在Flit Mode协商过程中，或者Flit Mode没有被协商时有效<br />        0_0001b: 仅支持2.5 GT/s<br />        0_0011b: 仅支持2.5 GT/s和5.0 GT/s<br />        0_0111b: 仅支持2.5 GT/s，5.0 GT/s和8.0 GT/s<br />        0_1111b: 仅支持2.5 GT/s，5.0 GT/s，8.0 GT/s和16.0 GT/s<br />        1_1111b: 仅支持2.5 GT/s，5.0 GT/s，8.0 GT/s，16.0 GT/s和32.0 GT/s<br />        Others: Reserved in Non-Flit Mode<br />    额外的Encodings仅仅在以下情况下允许：<br />        1. |
|       5       |                                                              |
|       6       |                                                              |
|       7       |                                                              |
|       8       |                                                              |
|       9       |                                                              |
|      10       |                                                              |
|      11       |                                                              |
|      12       |                                                              |
|      13       |                                                              |
|      14       |                                                              |
|      15       |                                                              |

Expected usage of the 64.0 GT/s supported link speeds encoding：只有在以下情况下，使用8b/10b编码或者128b/130b编码的TS1和TS2 OS才被允许将**Supported Link Speeds field**编码为1 0111b（指示支持2.5GT/s～64.0 GT/s）：

1. Initial Link Training to L0（在Configuration.Complete及以上）：
- Link被训练到L0，且在训练过程中成功协商了Flit Mode。在这种情况下，只要link一进入Configuration.Complete，则允许使用1 0111b编码，其实就是第一次在Gen1速率下进L0的训练过程中已经协商了Flit Mode，而后返回Recovery进行重新训练以切速率时，允许使用64.0 GT/s
2. Configuration.Linkwidth.Start（**仅仅作为receiver**）：
- 当LinkUp=0b时，link从Configuration.Linkwidth.Start转到Loopback。如果Loopback Lead提前得知Loopback Follower（此时它还处于Configuration.Linkwidth.Start）支持64.0 GT/s，则Loopback Lead允许transmit TS1 OS，且其中的Supported Link Speeds为1 0111b。Loopback Lead怎么得知Follower支持的速率，由具体实现决定。因为这样就直接在Loopback state中使用64.0 GT/s的速率了。
3. Recovery.Equalization when LinkUp=0b
- 当执行Equalization for Loopback时，在Recovery.Equalization的phase来advertise 1 0111b encoding。更精确的说，这是一种测试场景，在该场景下，LTSSM从Loopback进入Recovery.Equalization，而后Link在当前速率下进行链路均衡，而后Link回到Loopback。在这种情形下，如果工作的data rate比实际支持的更低（例如link在32.0 GT/s下进行链路均衡，但是实际上link支持64.0 GT/s且64.0 GT/s需要进行链路均衡），则使用1 0111b来指示link partner，在重新进入Loopback时它有能力转换到64.0 GT/s，而后在64.0 GT/s下进行链路均衡，而后重新进入Loopback。
4. Loopback
  - 当通过Configuration.Linkwidth.Start进入Loopback且LinkUp=0b时，Loopback Lead允许在LTSSM切换到Loopback.Entry的TS1 OS中使用1 0111b encoding。
5. Polling.Active（**仅仅作为receiver**）：
  - 对于一个测试装置，如果它知道port的capabilities，例如正在被测试的port支持64.0 GT/s操作，则测试装置可能会发送TS1 OS，且Supported Link Speeds field设置为64.0 GT/s supported encoding（1 0111b），并且在Polling.Active state中，当TS1 OS同时将Compliance Receive bit设置为1b且Loopback bit设置为0b时，将Flit Mode Supported bit设置为1b。如果一个port支持64.0 GT/s的操作，则当它因为在Polling.Active state接收到合适的TS1 OS，而从Polling.Active切换到Polling.Compliance时，它将会认为1 0111b是一个valid Supported Link Speeds encoding，并且认为这是link所要协商的最终速率。

下面是8b/10b编码和128b/130b编码中所使用的TS2 OS

下图是8b/10b编码中所使用的Modified TS1/TS2 OS

注意，Modified TS1/TS2 OS中，跨symbols的数据存储采用小端存储方式。例如symbol 8和symbol 9，Modified TS Usage存储于Symbol 8[2:0]，而Modified TS information 1存储于Symbol 9[7:0]和Symbol 8[7:3]

下表展示了1b/1b encoding中使用的TS1/TS2 OS，注意这些OS的长度也是16 bytes，但是其前一半和后一半的内容完全相同。

下表展示了1b/1b encoding中使用的TS0 OS，TS0 OS仅仅适用于1b/1b encoding

## Alternate Protocol Negotiation
除了skip equalization之外，alternate protocol也能在LinkUp=0b时，在Configuration.Lanenum.Wait, Configuration.Lanenum.Accept, and Configuration.Complete substates中，通过交换8b/10b编码的Modified TS1/TS2来协商。强烈建议在**整个Alternate Protocol Negotiation过程中**，在所有以上的3个substates中都advertise data rate为32.0 GT/s及以上。

当使用128b/130b或者1b/1b的PCIe PHY时，允许支持Alternate protocol。Alternate protocol定义为使用PCIe PHY的非PCIe协议，如CXL。在alternate protocol mode下，允许同时运行PCIe协议和一个或者多个alternate协议。Ordered Set blocks像往常一样工作，包括SKP OS的插入频率，OS和Data Block之间的转换等等。但是，根据alternate protocol的rules，data blocks的内容有可能与PCIe协议不同。注意，在alternate protocol的data block中，必须有类似EDS Token的机制，用于将Data block切换到Order set block。这防止了2bit block header中的随机bit错误不会被误翻译为data stream的结尾。

下图展示了alternate protocol协商以及equalization bypass的过程：

![image-20240526143118028](./assets/image-20240526143118028.png)

对于一个Downstream port，当LinkUp为0时，它在Configuration.Lanenum.Wait, Configuration.Lanenum.Accept, and Configuration.Complete substates中，根据发送的Modified TS OS中的 **Modified TS Usage Mode Selected field**值来管理alternate protocol协商和training set messages的发送。

对于一个Upstream port，如果它不支持收到的Modified TS Usage值，则它发送Modified TS Usage 000b

对于Modified TS Usage Mode Selected的值，规则如下：
- 000b：没有发生Alternate Protocol Negotiation / Training Set Message。link将工作于PCIe模式
- 001b：启用了Training Set Message。Modified TS Information 1和Modified TS Information 2这两个field，携带了Training Set Message Vendor ID field中携带的vendor specific messages。
- 010b：启用了Alternate Protocol Negotiation。Modified TS Information 1和Modified TS Information 2这两个field，携带了由Alternate Protocol Vendor ID field所定义的alternate protocol细节。使用Alternate Protocol Negotiation Status field来指示协议协商的进度
- others：reserved

对于一个Downstream Port，如果支持Alternate Protocol Negotiation，则当他第一次进入Configuration.Lanenum.Wait，且LinkUp=0b，Modified TS Usage Mode Selected field为010b时，它将开始negotiation过程。开始的步骤包括发送Modified TS1/TS2 OS，且Modified TS Usage为010b。

下表展示了如果Modified TS Usage为010b，则Modified TS Information 1 field所包含的内容

![image-20240526143818416](./assets/image-20240526143818416.png)

如果Modified TS Usage为001b，则Modified TS Information 1和Modified TS Information 2这两个field，携带了training set message中的细节。

注意，Alternate Protocol Negotiation必须与Lane number negotiation同时进行。Downstream必须确保在切换到Configutation.Complete substate之前，完成Alternate Protocol Negotiation。如果没有完成协商，则允许link退回纯PCIe模式。一旦成功协商了Alternate Protocol，link在2.5 GT/s中进入L0，而后切入最高的支持速率，进行链路均衡（如果需要），而后在最高的支持速率下进入L0。在最高速率下，且经过链路均衡之后，在SDS OS发送之后，开始发送携带alternate protocol的数据，且整个link将处于alternate protocol的控制之下。
## Electrical Idle Sequences（EIOS and EIEOS）
当Transmitter进入Electrical Idle状态之前，它总是发送一个Electrical Idle Ordered Set Sequence（EIESQ）。如果当前速率为2.5 / 8.0 / 16.0 / 32.0 / 64.0 GT/s，则一个EIESQ被定义为一个EIOS，而对于5.0 GT/s的速率，一个EIESQ被定义为2个连续的EIOS.

当使用8b/10b编码时，EIOS编码为K28.5（COM） + 3个K28.3（IDL）Symbols，即一共4个symbols。对于发送方而言，它必须发送EIOS中的所有内容。而对于接收方而言，当接收到COM和3个IDL中的2个的时候，便认为已经接收到了EIOS。EIOS如图所示：

![image-20240526144835339](./assets/image-20240526144835339.png)

当使用128b/130b编码时，一个EIOS是一个Ordered Set block，如图所示

![image-20240526144852259](./assets/image-20240526144852259.png)

当使用1b/1b编码时， 一个EIOS是一个Ordered Set block，如图所示

![image-20240526144916405](./assets/image-20240526144916405.png)

对于使用1b/1b编码的情况，如果紧接着还有EIOS要发送，则transmitter必须发送EIOS的所有内容，否则，允许transmitter仅仅发送symbol 0～13，而在symbol 14或者15的任意处中断，而后进入Electrical Idle state。

当速率为64.0 GT/s以下时，当Ordered Set block中的symbol 0～3符合EIOS的定义时，则认为收到了一个EIOS。在64.0 GT/s时的OS判断，见Flit Mode Operation（因为更高的误码率，FM下的OS判断有自己的要求）。

在128b/130b编码下，时钟周期的边界不一定和symbol的边界符合。因此，允许最后一个EIOS进行截断。这不会影响传输，因为receiver仅仅看symbol 0～3就能判断该Ordered Set是否为EIOS

在发送了最后一个EIOS的最后一个Symbol之后，Transmitter将处于valid Electrical Idle状态。

仅仅在除了2.5 GT/s之外的其他速率中，传输EIEOS。这是一个周期性发送的低频率pattern，用来确认receiver的Electrical Idle exit探测电路能够正常探测到link partner到从Electrical Idle退出。在128b/130b编码下，EIEOS还用于block的边界对齐。

在32.0 GT/s下，Electrical Idle Exit Ordered Set Sequence（EIEOSQ）定义为2个连续的EIEOS，而在5.0 / 8.0 / 16.0 GT/s下，EIEOSQ定义为1个EIEOS。在32.0 GT/s下，两个EIEOS必须背靠背传输，中间不能被中断，从而成功构成一个EIEOSQ。不管EIEOSQ的长度，block对齐发生于一个**单个的EIEOS**.

在64.0 GT/s下，一个EIEOSQ遵循如下定义：
- 无论是从Recovery.Speed还是从L1进入Recovery.RcvrLock，直到receiver在所有lane上检测到exit Electrical Idle，或者receiver在任意lane上接收到2个连续的valid的TS1 OS
  - 4个连续的EIEOS，不被其他Ordered Set打断，包括Control SKP OS
- 当使用L0p状态来升lane，直到receiver**在所有需要被激活lane上**检测到exit Electrical Idle，或者receiver在**任意需要被激活lane上**接收到2个连续的valid的TS1
  - 4个连续的EIEOS，不被其他Ordered Set打断，包括Control SKP OS
- 当从electrical idle进入Loopback state，直到receiver**在所有需要被激活lane上**检测到exit Electrical Idle，或者receiver在**任意lane上**接收到2个连续的valid的TS1
  - 4个连续的EIEOS，不被其他Ordered Set打断，包括Control SKP OS
- 当传输Compliance Patterns或者Modified Compliance Patterns：
  - 1个EIEOS
- 其他情况下，1个EIEOS

当在5.0 GT/s下，使用8b/10b编码时，在以下场景中传输EIEOSQ：

- 在进入LTSSM Configuration.Linkwidth.Start state之后，在第一个TS1 OS之前
- 在进入LTSSM Recovery.RcvrLock state之后，在第一个TS1 OS之前
- 在LTSSM Configuration.Linkwidth.Start, Recovery.RcvrLock, and Recovery.RcvrCfg states中，当每32个TS1或者TS2 OS被传输之后，通过以下步骤将TS1 / TS2 count清零： 

  - 发送一个EIEOS
  - 当LTSSM处于Recovery.RcvrCfg中，接收到第一个TS2 OS

当使用128b/130b编码时，在以下场景中传输EIEOSQ：
- 在进入LTSSM Configuration.Linkwidth.Start state之后，在第一个TS1 OS之前
- 在进入LTSSM Recovery.RcvrLock state之后，在第一个TS1 OS之前
- 在Non Flit Mode下，紧跟着EDS Token传输，这种情形用于结束一个Data Stream，并且不发送一个EIOS，且不进入LTSSM Recovery.RcvrLock substate
- 在Flit Mode下，作为预先安排好的Ordered Set interval，来结束一个Data Stream
- 在所有需要传输TS1 / TS2 OS的LTSSM states中，

* 当每32个TS1或者TS2 OS被传输之后，当以下条件之一满足时，将TS1 / TS2 count清零： 

    - 发送一个EIEOS

    - 当LTSSM处于Recovery.RcvrCfg中，接收到第一个TS2 OS

    - 当LTSSM处于Configuration.Complete中，接收到第一个TS2 OS

    - 一个Downstream Port处于LTSSM Recovery.Equalization state的phase 2，并且在任意lane上接收到了2个连续的TS1 OS，且Reset EIEOS Interval Count bit为1

    - 一个Upstream Port处于LTSSM Recovery.Equalization state的phase 3，并且在任意lane上接收到了2个连续的TS1 OS，且Reset EIEOS Interval Count bit为1

* 当LTSSM处于Recovery.Equalization state，每65536个TS1 OS发送出去之后//TODO

* 作为FTS OS的一部分，Compliance Pattern或者Modified Compliance Pattern在对应的section讲解

 当使用1b/1b编码时，在以下场景中传输EIESQ：

## Inferring Electrical Idle
一个device，允许在所有工作的速率下推断出Electrical Idle，而非使用模拟电路来探测出Electrical Idle。下面描述了在不同的states下如何推断出Electrical Idle。
- L0：在所有速率下，在128微秒的时间窗口内，**以下有任意一个缺失**：
  - an UpdateFC DLLP / an Optimized_Update_FC（Flit mode）
  - SKP OS
- Recovery.RcvrCfg
  - 在2.5 GT/s和5.0 GT/s下，在1280 UI内，没有TS1或者TS2传输
  - 在8.0 GT/s及更高速率下，在4ms的时间窗口内，没有TS1或者TS2传输
- Recovery.Speed when successful_speed_negotiation = 1b
  - 在2.5 GT/s和5.0 GT/s下，在1280 UI内，没有TS1或者TS2传输
  - 在8.0 GT/s及更高速率下，在4ms的时间窗口内，没有TS1或者TS2传输
- Recovery.Speed when successful_speed_negotiation = 0b
  - 在2.5 GT/s下，在2000 UI的间隔内，没有退出Electrical Idle
  - 在5.0 GT/s以及更高速率下，在16000 UI的间隔内，没有退出Electrical Idle
- Loopback.Active (as follower)
  - 在2.5GT/s下，在128微秒的时间窗口内，没有退出Electrical Idle

Electrical Idle exit condition不允许基于Electrical Idle condition的推断来决定。为了节省电路面积，允许对一个LTSSM仅实现一个计时器，而后使用同一个时间窗口对electrical idle进行推断。
## Lane Polarity Inversion
在polling阶段的training sequence中，receiver查看TS1和TS2的symbol 6～15，来检查lane polarity inversion（即D+和D-颠倒）。如果发生了lane polarity inversion，那么收到的TS1的symbol 6～15将会是D21.5，而非D10.2，收到的TS2的symbol 6～15将会是D26.5，而非D5.2。这两类TS中的这些symbol清楚的表示了Lane polarity inversion。

如果检测到polarity inversion，则receiver必须反转接收到的数据。注意，**transmitter****不允许反转发送出去的数据。**所有的PCIe receiver都必须支持Lane polarity Inversion
## Fast Training Sequence（FTS）

## Start of Data Stream Ordered Set（SDS Ordered Set）

## Link Error Recovery

## Reset
从系统的角度对reset的描述，见第六章。
### Fundamental Reset
### Hot Reset
## Link Data Rate Negotiation
所有的device都必须在每条lane上使用2.5 GT/s开始链路初始化。Training Sequence的Ordered Set中的一个field被用来advertise所有支持的data rates。Link首先在2.5 GT/s下被训练到L0状态，而后通过进入recovery state来进行速率切换。
## Link Width and Lane Sequence Negotiation

### Required and Optional Port Behavior

## Lane-to-Lane De-skew
## Lane vs. Link Training
Link初始化的过程，会将port中不相关的lanes聚合成相关的lanes，从而生成一个Link。对于那些要聚合成一个link的lanes，TS1和TS2 OS必须将所有lane上的合适的field（Symbol 3，4 and 5）设置为同样的值。

作为Configuration的结果，形成了Link。
- 如果实现了可选的功能，即一个Port能够配置多条link，则必须注意以下规则：
  - 对于任意port，对于每一个单独的需要被配置的link，都必须实现一个单独的LTSSM
  - LTSSM Rules是为了配置一个单独的Link而使用的。对于实现了多条link的情况，它们的初始化是并行的还是串行的，由具体实现决定。 