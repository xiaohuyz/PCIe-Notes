# 128b/130b Encoding for 8.0 GT/s, 16.0 GT/s, and 32.0 GT/s Data Rates

当PCIe工作于8.0 GT/s，16.0 GT/s，32.0 GT/s时，使用128b/130b编码。同时为了前向兼容性，整个link会在最开始在2.5 GT/s的速率下，使用8b/10b编码训练到L0，而后当速率切换到8.0 GT/s，16.0 GT/s，32.0 GT/s时，使用128b/130b编码。在Non-Flit模式下，128b/130b编码使用link-wide的打包机制。而在Non Flit Mode和Flit Mode下，128b/130b编码都采用基于每条lane的编码和scramble机制。

在128b/130b编码下，data传输的基本单位是8-bit data character，也被称为**一个symbol**，如图所示。

![image-20240413153115906](./assets/image-20240413153115906.png)

![image-20240413153136074](./assets/image-20240413153136074.png)

注意，在Byte级，是Byte0最先传输，而查看TLP的定义，Byte0是MSB的byte，即最重要的Byte先传输。而在Bit级，反而是Bit0先传输，bit0是最低位的bit。

## Lane Level Encoding

PL层使用基于每个Lane的block编码方式。每个Block包含一个2bit的Sync Header和一个payload。有**两种Sync Header编码方式，即10b和01b**。Sync Header定义了该block所包含的payload种类（**是data stream还是ordered sets**）。

Sync Header为10b，指示这是一个data block。拥有128bit的payload，因此整个block大小为130bit。

Sync Header为01b，指示这是一个ordered set block。每个ordered set block都有128bit的payload，因此整个block大小为130bit。但是**SKP Ordered Set除外，其长度是可变的。**

对于一个Multi-Lane Link而言，它所有的lane必须同时传输**具有相同Sync Header的blocks**。例外情况：**在Polling.Compliance阶段传输Jitter Measurement Pattern**时

bit级的整个传输顺序如上图所示，都是每个bit的**LSB先放到lane上**。对于Sync Header，先放H0后放H1，即传输顺序是先传输bit 0，后传输bit 1。对于symbol级，则以S0开始而以S7结束。而在layout表示中，则遵循与spec其他部分一致的做法，用小端模式来表示layout。

## Ordered Set Blocks

一个Ordered Set Block包含了一个Sync Header，后面接一个Ordered Set。对于一个Multi-Lane link，它所有的lane在同一时间必须传输**相同type**的ordered set。Ordered Set的第一个symbol决定了OS的type（也就是说，**所有lane的该symbol必须相同**）。Ordered Set的**后续symbols由OS的type决定**，且**不必要在每个Lane上都相同**，这个是显而易见的，例如传输TS1 OS / TS2 OS，在表示lane number的那个symbol，每个lane应该各不相同。在Flit Mode和Non Flit Mode下，Ordered Set Blocks是相同的，除了**SKP OS的使用方法和频率不一样。**同时，在8.0 GT/s下使用Flit Mode时，同时使用了Standard SKP OS和Control SKP OS，在8.0 GT/s下的Non Flit Mode，仅仅使用Standard SKP OS

### Block Alignment，block对齐，即找到block的边界

在Link training过程中，Electrical Idle Exit Ordered Set（EIEOS）是一个独特的bit pattern，用于receiver来**找到接收到的bit流中的block Sync Header的位置，即实现block对齐**。在128b/130b编码中，EIEOS由Sync Header01b，加上16个值为66h的symbol组成。

Receiver的block alignment过程有三种状态：

- **Unaligned Phase**。receiver在一段时间的Electrical Idle之后，进入此状态，比如：
  - data rate切换到使用128b/130b编码的速率
  - receiver退出低功耗link状态
  - 被特定的机制导向Unaligned Phase。在该状态下，receiver监控接收到的bit流，**寻找EIEOS bit pattern**。当检测到一个这样的pattern时，receiver将自己的block alignment切换到该bit pattern，并且进入Aligned Phase
  
- **Aligned Phase**。Receiver**继续监控EIEOS bit pattern**，同时**检测接收到的blocks**，**寻找Start of Data Stream（SDS）Ordered Set**。即在此Phase下，既能重新调整Block Alignment，也能通过接收到SDS而进入Lock Phase。如果在该阶段，接收到了一个EIEOS，且该EIEOS的pattern不符合当前对齐状态，则调整当前对齐状态到**新接收到的EIEOS pattern**。如果接收到一个SDS OS，则receiver进入Locked Phase。如果**接收到一个未定义的Sync Header（00b / 11b）**，则**允许**receiver切换回Unaligned Phase。接受到undefined Sync Header，意味着在离开Unaligned Phase时获得的Block Alignment根本不对。

- **Locked Phase**。在该phase下，Alignment已经锁定了，不允许改变。因为在SDS OS后就是Data的传输，改变对齐状态会影响到这些data的传输。如果接收到一个未定义的Sync Header（00b / 11b），则receiver**必须**切换回Unaligned Phase / Aligned Phase

在链路训练过程中，如果receiver是misaligned的，那么EIEOS和TS Ordered Set会使得receiver接收到一个非法的Sync Header。

对于Block Alignment的additional requirements：

- 当在Aligned / Locked Phase时，当需要的时候，receiver必须根据接收到的SKP OS调整它的alignment。其实就是根据接收到的SKP OS来微调自己的Block Alignment
- 在LTSSM切换到Recovery后，Receiver必须丢弃所有接收到的TS OS，直到它接收到一个EIEOS（因为在没有进行Block Alignment的情况下，无法判断接受到的OS是否有效）。其实就是在Recovery.RcvrLock state中，必须首先使用EIEOS来重新获得Block Alignment，而后才能有效地接收TS OS。**接收到一个EIEOS，意味着receiver的alignment有效**，从而允许进行TS OS的处理。但是，如果**正是一个EIEOS将receiver的LTSSM从L0切到了Recovery**，那么receiver允许处理该EIEOS后接收到的所有TS（**因为已经有一个EIEOS了**，已经根据该EIEOS来进行了Block Alignment），或者丢弃TS直到接收到下一个EIEOS（**这种情况就是前面描述的common scenario**）

- 只要Data Stream传输一停止，则允许receiver从locked phase切换到unaligned phase / aligned phase（**切换回这两个phase以方便调整block alignment**）

- 对于Loopback Leads：当LTSSM处于Loopback.Entry时，leads必须能够根据**接收到的EIEOS**来调整自己**receiver的block alignment**。当LTSSM处于Loopback.Active时，leads允许**发送EIEOS**，而后**根据looped back bit stream（绕回来的bit流）来调整自己receiver的block alignment**

- 对于Loopback Follower：当LTSSM处于Loopback.Entry时，follower必须能够**根据接收到的EIEOS来调整自己receiver的block alignment**。当LTSSM处于Loopback.Active时，follower不允许调整自己receiver的block alignment。因此，当follower开始loop back接收到的bit stream时，它的receiver会直接被切换到Lock Phase
## Data Blocks
Data Blocks的payload称为data stream。在Non Flit Mode下，data stream由**Framing Tokens / TLPs / DLLPs组成**。Data Stream的每一个symbol都被分配到link的一个lane上（即**对于multi-lane link，按symbol为单位将数据分到每个lane上**）。

对于一个Data Stream：

- 从紧跟着SDS OS后的Data Block的第一个symbol开始
- **检测到一个Framing Error**时结束 / 以Ordered Set（**SKP OS除外**）之前的Data Block中的最后一个symbol结束
### Framing Tokens in Non-Flit-Mode
Non Flit Mode下使用的Framing Tokens如下所述。每个Framing Token都描述或者暗示了与该Token有关的symbol的数目，**因此暗示了下一个Framing Token的位置**。
- IDL，即Logical Idle。长度为1 symbol。当没有TLPs / DLLPs / 其他Framing Tokens传输的时候，传输该Framing Token
- SDP，即start of DLLP。长度为2 symbol。而后跟着的是DLLP信息
- STP，即start of TLP。长度为4 symbols，而且包含一个12 bit的TLP Sequence Number。后面跟着的是TLP的信息。STP Token中有2个symbols在Data Link层就已经添加上了，因为TLP Sequence Number是在DL层添加的
- EDB，即End Bad。长度为4 symbols。用于确认前面的一个TLP是无效的
- EDS，即End of Data Stream。长度为4 symbols。用于指示Data Stream已经结束了，下一个block是OS block

Framing Tokens的细节如图所示：

![image-20240413155600333](./assets/image-20240413155600333.png)

注意，Data Stream的第一个Framing Token永远在Data Stream的第一个Data block的Symbol 0 上。 

TLP和DLLP在PL层的layout如图所示：

![image-20240413155638436](./assets/image-20240413155638436.png)

其中symbol 0和symbol 1是SDP，后面的是DLLP的信息

对STP Token的解析：使用STP Token中的11 bit field来描述TLP length。注意，TLP Length包括Framing Token，TLP Prefixes，TLP Header，TLP data payload，TLP digest，TLP PCRC，TLP MAC，TLP LCRC。注意，**当TLP是nullified时，EDB Token不计算在TLP length中**。因为在传送STP Token时，根本被不可能知道TLP是否Nullified

整个TLP Length field由一个4 bit的CRC保护，即FCRC。而后使用一个奇校验来保护TLP Length和计算而来的CRC，即FP。

注意，STP Token中，TLP Length不可能为1。如果**接收到一个形似STP Token的symbol，但TLP Length field为1，则需要考虑这是不是EDS Token**

对于TLP而言，DL层将TLP Sequence number field前面的4个bit，即后来的FCRC在的field，预先假设为0000b，而后计算LCRC（原因很明显，因为FCRC是后来在PL层才计算的，这在计算LCRC后面）。同理，receiver在计算LCRC时，也**将这4bit假设为0000b，而不看其实际值**。但实际上这4个bit最后在传输时当然不是传输的0000b，而是在PL层将它们赋值为FCRC

TLP Length field 大于1535，则为PMUX Packets。对于PMUX Packets，其真实Packet长度另有计算方法，而STP Token中的TLP Sequence Number field也并另有含义。同时LCRC也使用另外的方法进行计算。

TLP Length field在1152～1535的，是reserved。

对于TLP而言，Transmitter必须发送STP Framing Token中的TLP length field中描述的TLP的**所有DW**。即使该TLP是nullified。如图所示。发完LCRC之后才抛出EDB

![image-20240413160310800](./assets/image-20240413160310800.png)

### Transmitter Framing Requirements in Non-Flit Mode
下列requirements对transmitted Data Stream适用：
- 发送一个TLP：
  
  - 发送一个STP Token，而后**紧跟着从DL层传下的**整个TLP Information（即紧跟着**经过DL层处理的TLP**）
  - Tranmitter必须发送**STP Framing Token中的TLP length field**中描述的TLP的所有DW。即使该TLP是nullified的
  - 如果TLP是nullified，那么在TLP之后必须紧跟着一个EDB Token。即在TLP的最后一个symbol和EDB的第一个symbol之间，**不允许有任何symbol**。同时注意，STP Token中的TLP Length**不包括EDB的内容**。注意EDB Token和EDS Token在此处有不同。在EDS Token中，如果EDS Token占据的不是这条Link的最后四个Lane的内容，则先在TLP后面插入IDL Token，最后再放EDS Token，而EDB Token不是这样做的，它是先直接放EDB Token，剩下的lane用IDL Token填充
  - 注意在一个symbol time内，STP Token不能被发送超过1次，即不能同时在一个Link上传输两个不同的TLP
  
- 发送一个DLLP：
  - 发送一个SDP Token，而后**紧跟着**DL层提供的整个DLLP Information
  - DLLP的所有6个symbol都必须被发送。注意，DLLP的长度是固定的，为6 bytes
  - 注意在一个symbol time内，SDP Token不能被发送超过1次，即不能同时在一个Link上传输两个不同的DLLP

- 在Data Stream中间发送一个SKP OS，这是用于Tx和Rx之间细微时钟差的补偿：
  - 在现存的Data Block的最后一个DW之后，发送一个4 symbol的EDS Token。例如，在x16的Link中，EDS Token必须占据Lane12~15。在EDS Token中，如果EDS Token占据的不是这条Link的最后四个Lane的内容，则先在TLP后面插入IDL Token，最后再放EDS Token。
  - 而后发送SKP OS
  - 在SKP OS后发送Data Block。Data Block的恢复以发送出Data Block的第一个Symbol为标志
  - 如果安排发送了多个SKP OS，则**每个SKP OS之前都必须有一个Data Block + EDS Token（实际上就是不允许一次发送多个SKP OS）**

- 结束一个Data Stream：
  - 在现存的Data Block的**最后一个DW处发送一个EDS Token**（前文已讲述）。而后紧跟着一个EIOS或者EIEOS。
    - EIOS，即Electrical Idle Sequence，标志着整个Link进入Electrical Idle状态了，用于LTSSM power management state transitions
    - EIEOS用于其它情况，例如让LTSSM进入Recovery

- 当没有发送TLP / DLLP / other Framing Token时，在所有lane上发送IDL Token。其实这就是Logical Idle的情况，这是为了防止接收方的PLL和发送方失调，从而丢失时钟锁定。

- 对于Multi-Lane Links：
  - 在发送一个IDL Token后，下一个STP / SDP Token的第一个symbol必须在下一个Symbol time开始传输，**且从lane 0 开始**。**而对于EDS Token，它能在同一个Symbol Time，紧跟着IDL Token传输。（因为它必须在Block的最后一个DW传输）**
  - 对于xN links，且x>=8。如果EDB Token / TLP / DLLP在lane K结束，而K不等于N-1，而且后面也没有紧跟着下一个STP / SDP / EDB Token，即后面没有东西来填充了。则**使用IDL Token来填充lane K ~ lane N-1**。注意EDS Token是一个例外，它能紧跟着IDL Token传输
  - Tokens / TLPs / DLLPs允许接连不断的传输。因此，只要传输符合spec的规范，**同一个symbol time内能传输超过一个Token**。其实就是在一个Link上，能同时传输一个TLP和一个DLLP

- 超过x4宽度的links，**Tokens必须从Lane 4 × N开始传输**

### Receiver Framing Requirements in Non-Flit Mode

  下列requirements对receiver适用，而且也适用于在Data Stream的开头和结尾出现的Block type transactions：

- 当处理期待是Framing Token的symbols时，接收到一个或者一系列**与Framing Token定义不同的symbols**。这是一个Framing Error。强烈建议对于Multi-lane link而言，使用接收到预期之外的第一个symbol的lane的lane error status register来汇报错误
- 所有的error check / report都是可选的
- 当接收到一个STP Token：
  - receiver必须计算出received TLP Length field的Frame CRC和Frame Parity。而后将其与接收到的Frame CRC和Frame Parity比较。如果Frame CRC或者Frame Parity不一致，则为Framing Error
    - 当一个STP有Framing Error时，为了方便向DL层report，不将其视为TLP的一部分
  - 当STP的TLP Length field为1时，该symbol**不是STP Token**，需要评估该symbol是不是**EDS Token**
  - receiver**允许检查**TLP Length field是否为0。如果检查，则接收到一个值为0的TLP Length field为framing error
  - receiver**允许检查**TLP Length field是否为2 / 3 / 4。如果检查，则接收到一个值为2 / 3 / 4的TLP Length field为framing error（即**TLP Length永远是大于4DW的**）
  - receiver**允许检查**TLP Length field是否为1152-1535（包括）。如果检查，则接收到一个值为1152-1535（包括）的TLP Length field为framing error
  - 如果ports**不支持Protocol Multiplexing**，则receiver**允许检查**TLP Length field是否超过1535。如果检查，则接收到一个值大于1535的TLP Length field为framing error
  - 如果ports支持Protocol Multiplexing，则receiver接收到TLP Length field大于1535的STP Tokens时，需要将其处理为PMUX Packet的开头。
  - 下一个被处理的Token是TLP的最后一个DW后紧跟着的那个symbol。这个Token的位置是由TLP Length field决定的
    - receiver必须评估该symbol**以确定它是不是EDB的第一个symbol**，如果是的话，该TLP是nullified的（就是**评估TLP之后有没有接着EDB Token**）。因为EDB是在STP Token规定的TLP Length预期之外的
  - receiver允许检查在一个symbol time内是否接收到超过一个STP Token。如果开启了该检查，在一单独的symbol time内接收到超过一个STP Token是Framing Error。
- 当接收到一个EDB Token：
  - 如果在一个TLP之后紧跟着就接收到了一个EDB Token（在TLP的last symbol和EDB的first symbol之间没有symbols，这是合法的传输情况），receivers在**接收到EDB Token的第一个symbol**或者**接收到EDB的任何或者全部剩下的symbol**后，通知DL层它接收到了一个EDB Token。无论是否通知DL层，receivers必须检查EDB Token的所有symbols。接收到任何一个不符合EDB Token的symbol，则为Framing Error（其实就是检查接收到的EDB Token的所有bit是否都是对的）
  - 除了上述时间点，在其他任何时间接收到一个EDB Token都是Framing Error
  - 下一个被处理的Token从EDB Token结束后的第一个Symbol开始
- 当一个EDS Token作为Data Block的最后4个symbols被接收到：
  - receivers必须停止处理Data Stream
  - 在EDS Token后，接收到**除了 SKP / EIOS / EIEOS 之外的OS**，都是Framing Error
    - SKP OS用于时钟补偿，其实这时候数据传输并没有结束，在SKP OS结束之后会继续传输数据的
    - EIOS用于让Link进入Electrical Idle，从而进入其他power states
    - EIEOS用于其他情况，例如进入Recovery
  - 如果在EDS Token之后的Block中接收到了SKP OS，则receivers必须从SKP OS之后的data block的第一个symbol开始恢复data stream的处理
- 当接收到一个SDP Token：
  - 下一个被处理的Token紧跟着DLLP的最后一个symbol
  - receiver允许检查在一个symbol time内，是否有超过一个SDP Token被接收到。这是一个Framing Error（即同一个Symbol time内不允许Link上出现2个及以上的DLLP）
- 当接收到一个IDL Token：
  - 对于x1 link，下一个被处理的Token以下一个接收到的symbol开始
  - 对于x2 link，下一个被处理的Token以**下一个symbol time中Lane 0接收到的symbol开始。**强烈建议在Lane0接收到IDL Token时，在同一个symbol time内**对Lane 1进行检查**。如果lane 1没有接收到IDL Token，则这是一个Framing Error，且使用lane 1对应的lane error status register进行汇报
  - 对于x4 link，下一个被处理的Token以**下一个symbol time中Lane 0接收到的symbol开始。**强烈建议在Lane0接收到IDL Token时，在同一个symbol time内**对Lane 1-3进行检查**。如果lane 1-3没有接收到IDL Token，则这是一个Framing Error，且使用lane 1-3的lane error status register进行汇报
  - 对于x8和x16 link，下一个被处理的Token以IDL Token之后，**下一个DW对齐的lane接收到的symbol开始**。例如，对于一个x16的link，如果在Lane 4接收到一个IDL Token，则下一个Token从**同一个symbol time**的Lane 8开始。强烈建议在接收到IDL Token之后，检查IDL Token和下一个Token之间的symbol。如果不是IDL Token，则这是一个Framing Error，且使用对应Lane的lane error status register进行汇报。
  - 其实就是IDL Token用于填充Lane时，必须填充至x4边界处
  - 注意在同一个Symbol time内，与IDL Token同时出现的Token只有可能是EDS Token
- 当处理Data Stream时，receivers必须同时检查每一个lane接收到的Block Type
  - 在任意lane上，在SDS OS之后立即接收到一个OS block，这是一个Framing Error。因为SDS OS后面只能接着Data Block
  - 接收到一个block，且使用未定义的block type（Sync Header为00b或者11b），这是一个Framing Error。强烈建议multi-lane link中，receiver在接收到这个Sync Header的lane的link error status register中汇报对应的错误。
  - 在任何一个lane上，当在前面一个block中**没有接收到EDS Token**，**而后却接收到了一个Ordered Set，这是一个Framing Error**。例如，在没有前置的EDS Token的情况下，接收到一个SKP OS，这是Framing Error。此外，在Data Stream内，在一个OS block后立即接收到一个SKP OS也是Framing Error。强烈建议当OS的第一个Symbol表明该OS为SKP OS，对于multi-lane link，检查symbol number 1～4N，如果不符合SKP OS的内容，则为Framing Error
  - 在任意lane上，在EDS Token之后，接收到一个Data Block，这是一个Framing Error。强烈建议对于一个Multi-Lane Link，如果有Lane接收到了Data Block，则使用Lane Error Status Register来汇报这个错误
  - Receiver允许在不同的lane上检查**不同种类**的OS。如果开启该检查，则在不同Lane上接收到不同种类的Ordered Sets，是Framing Error。**注意，OS的种类相同即可，OS内容不一定要求完全相同。**
### Receiver Framing Requirments in Flit Mode
当处理Data Stream时，receiver必须检查每个lane接收到的block type。
- 在SDS OS之后，没有立即接收到一个SKP OS + Data Block，则是Framing Error。这是Flit Mode下的Data Block传输要求，注意和Non Flit Mode不一样
- 接收到一个block，且使用未定义的block type（Sync Header为00b或者11b），这是一个Framing Error。强烈建议multi-lane link中，接收到的Lane receiver在link error status register中汇报对应的错误
- 对于**没有正在进入或者退出electrical idle state的lane**（注意，在此处的语义下，进入或者退出electrical idle state是进出L0p state的一部分），下面的情形是Framing Error。强烈建议对于一个multi-lane link而言，receiver在link error status register中汇报对应的错误
  - 在不是规定好的block boundary处，接收到一个Ordered Set Block
  - 在规定好的block boundary处，接收到以下三个OS，但长度不对
    - EIEOS
    - SKP OS
    - EIOS
  - Receiver允许在不同的lane上检查**不同种类**的OS。如果开启该检查，则在不同Lane上接收到不同种类的Ordered Sets，是Framing Error。注意，**在L0p width transition过程中，这是允许的。**
### Recovery from Framing Errors in Non-Flit Mode and Flit Mode
如果一个receiver在处理Data Stream的过程中，探测到一个Framing Error，则必须：
- report a receiver error
- 停止处理Data Stream。当接收到下一个SDS OS时，开始处理一个新的Data Stream
- 开启错误恢复进程。
  - 如果LTSSM此时处于L0，则将其导向Recovery state
  - 如果LTSSM此时处于Configuration.Complete / Configuration.Idle，则作如下**任意**处理
    - 由于特定的timeout，LTSSM从Configuration.Idle切换到Reovery.RcvrLock
    - 先进入L0，而后再进入进入Recovery
  - 如果LTSSM此时处于Recovery.RcvrCfg / Recovery.Idle，则作如下**任意**处理
    - 由于特定的timeout，LTSSM从Recovery.Idle直接切换到Reovery.RcvrLock
    - 通过L0进入Recovery
  - 如果LTSSM此时正处于Recovery.RcvrLock / Configuration.LinkWidth.Start，则直接退出这些sub states，不用将LTSSM导向Recovery
  - 注意，Framing Error recovery不会由DL层的recovery机制，如NAK来触发。Framing Error都是由PL层检测并且触发Recovery机制的
- 当使用128b/130b编码时，**所有的Framing Errors都需要link recovery**。从link两端都进入recovery state，到从Framing Error中恢复，时间低于1ms

## Scrambling in Non-Flit Mode and Flit Mode

对于一个Multi-Lane Link而言，transmitter的每一个lane都可以实现一个单独的LSFR来做scrambling，而receiver的每一个lane都可以实现一个单独的LSFR来做de-scrambling。具体实现可以选择使用更少的LSFR，但必须获得和独立的LSFR一样的效果。

scrambling规则如下所述：

- 对于 8.0 / 16.0 / 32.0 GT/s，**Sync Header的2 bit不用scrambling，且他们也不会使得LSFR前进**
- **EIEOS的所有16 symbols都不scrambling**。除了**在L0p状态下**，对于transmitter，当EIEOS的**最后一个symbol被发送出去之后**，scrambling LSFR重置。而对于receiver，当**接收到EIEOS的最后一个symbol时**，de-scrambling LSFR重置。其实EIEOS就是用来重置Link两端的LSFR的，与8b/10b编码中的COM编码作用一样，用来解决link两端LSFR失配的问题
- 对于 8.0 / 16.0 / 32.0 GT/s，TS1和TS2 OS：
  - TS1和TS2 OS的Symbol 0 跳过scrambling
  - Symbol1～13经过scrambling
  - 如果DC Balance需要，则Symbol14 / 15跳过scrambling，否则它们经过scrambling
- 对于 8.0 / 16.0 / 32.0 GT/s，FTS的所有16 symbols都不经过scrambling
- SDS的所有16 symbols都不经过scrambling
- EIOS的所有16 symbols都不经过scrambling
- SKP的所有symbols都不经过scrambling
- 除了SKP OS之外，对于所有其他OS，Transmitter都使它们的LSFR shift。注意，有的OS虽然不经过scrambling，但是LSFR仍然shift
- receivers通过评估接收到的OS来判断是否shift它们的LSFR。
  - 如果接收到的OS是SKP OS，或者是在data stream中的带着L0p的EIOS，则LSFR不shift
  - 对于其他OS，对他们的所有symbol，LSFR都shift
- 在**L0p状态下**，对于transmitter和receiver，其他lane的LSFR跟随lane 0 的LSFR一起shift。因此，在L0p下，非0的lane，即使收到EIEOS，也不会重置它的LSFR
- Data Block的所有16 symbols都经过scrambling，且会使得LSFR shift
- 对于需要经过scramble的symbols，他的**LSB最先scramble，MSB最后scramble**
- LSFR的种子与Lane number有关。注意这里使用的是**default lane number，这与device的默认设置有关**，而**与link width / lane reversal协商无关**。这些default lane number是link从Detect，经过Polling，从而第一次进入Configuration.Idle时被分配的
- 当LinkUp=1时，LSFR的seed不会改变。只要LinkUp始终为1，通过LTSSM的configuration state的link reconfiguration也不会改变最初的lane number分配，即Link变宽也不影响LSFR的seed。
- 当使用128b/130b编码时，scrambling不能在Configuration.Complete状态被disabled
- 一个Loopback Follower不能descramble或者scramble looped-back bit stream
## Precoding
预编码技术，用于减少传输过程中出现的突发错误（即连续的bit错误）。适用于32.0 GT/s和64.0 GT/s以及以上速率。

当工作于32.0 GT/s及以上速率时，一个**receiver可以请求transmitter开启precoding**。Precoding同时适用于Flit Mode和Non Flit Mode。其rules如下：

- 对于一个port / pseudo-port，必须**在link的所有configured lanes上request precoding**，如果一个Port在一些Configured Lanes上请求了Precoding而另一些lanes上没有，则行为未定。
- 一个port / pseud-port可能不管其他port / pseud-port的状态，只自己请求precoding。
- 当LTSSM处于**Detect State**下，对于所有速率都关闭precoding
- 如果需要发出针对某一个data rate的precoding request的话，该request必须**在进入该data rate之前发出**。
  - 发出一个precoding request的方法：在进入需要开启precoding的data rate的Recovery.Speed之前，将**EQ TS2**或者**128b/130b EQ TS2**中的**transmitter precode request bit**设置为1
  - 在detect之后，第一次进入32.0 GT/s之前，当EQ TS2或者128b/130b EQ TS2 Ordered Sets中的Supported Link Speeds field是32.0 GT/s或者更高时，Transmitter Precode Request bit代表的是**precoding request for 32.0 GT/s**。因为既然要执行Precoding Request，则必须首先对32.0 GT/s执行Precoding Request
  - 如果自detect状态以来，link speed已经被训练到32.0 GT/s，Transmitter Precode Request bit代表的是precoding request for **the same data rate that the values in the Transmitter Preset field apply to**。即针对Transmitter Preset field所述的data rate的precoding request
  - 对于32.0 GT/s及以上的**每一个data rate**，precoding request都必须被独立配置，即每个速率都要重新发起Precoding Request
- 每个port都必须保存每个lane上最近一次链路均衡过程中使用的precoding request和Tx Eq values。如果link工作于32.0 GT/s及以上速率，且不使用链路均衡（这是通过TS1/TS2 OS或者modified TS1/TS2 OS协商好了No Equalization Needed Mechanism），或者link工作于32.0 GT/s及者以上速率的Polling.Compliance / Loopback state（这两个LTSSM states不是常规传输用的states），则必须使用最近一次链路均衡过程中的precoding requests。如果在当前linkup之前，没有执行过链路均衡过程，则precoding不会被开启。
- 如果在进入Recovery.Speed之前的Recovery.RcvrCfg阶段，接收到的8个连续不断的EQ TS2或者128b/130b EQ TS2 OS，它们的Transmitter Precode Request bit被设置为1b，则transmitter必须在退出Recovery.Speed时，对link将要工作的speed开启precoding（如果该speed为32.0 GT/s或者更高，且precoding request是为了该data rate的）。如果Precoding Request适用于一个Speed，但是该Speed不是link退出Recovery.Speed后将要工作的Speed，则接收到的Transmitter Precode Request必须被ignore，且任何data rate的Precoding Setting都不能被更新。一旦开启precoding，则它将一直生效，直到对于同一速率，在进入Recovery.Speed之前的Recovery.RcvrCfg states中，transmitter接收到另外8个连续不断的EQ TS2或者128b/130b EQ TS2 OS，它们的Transmitter Precode Request bit被设置为0b
- 对于32.0 GT/s以下的速率，Transmitter不允许开启Precoding
- 对于32.0 GT/s及以上的速率，如果在当前速率下开启了precoding，则在Recovery state中，transmitter发送的TS1 OS必须**set Transmitter Precoding On bit**。否则该bit应该被设置为0b
- 对于transmitter，如果它启用了precoding for the 32GT/s data rate on Lane 0，则它必须设置32.0 GT/s Status Register中的**32.0 GT/s Transmitter Precoding On bit**为1b。否则应该将该bit设置为0b。对于receiver，如果它**已经requested**，或者**将要request它的link partner在32.0 GT/s下去开启precoding**，那它必须设置32.0 GT/s Status Register中的**32.0 GT/s Transmitter Precode Request bit**为1b。否则应该将该bit设置为0。
- 对于transmitter，如果它启用了precoding for the 64.0 GT/s data rate on Lane 0，则它必须设置64.0 GT/s Status Register中的**64.0 GT/s Transmitter Precoding On bit**为1b。否则应该将该bit设置为0b。对于receiver，如果它**已经requested**，或者**将要request它的link partner在64.0 GT/s下去开启precoding**，那它必须设置64.0 GT/s Status Register中的**64.0 GT/s Transmitter Precode Request bit**为1b。否则应该将该bit设置为0。
### Precoding at 32.0 GT/s Data Rate
当在32.0GT/s下开启precoding时，需要遵循以下的rules：
- 仅仅**scrambd的bit**会被precoding，即symbol要先经过扰码，后来才能precoding

- 在**block boundary**处，**“previous bit”设置为1b，即在边界处与1进行异或**

- 对于经过scrambled的bit，首先先进行de-precoding，而后de-scramble

  发送和接收方的逻辑如图：

  ![image-20240414135408657](./assets/image-20240414135408657.png)

  当precoding开启的时候，在transmit端，必须**先计算出SKP OS的parity，而后再进行precoding**。即顺序如下：

1. scrambling
2. parity bit calculation
3. precoding for scrambled bits

而接收方顺序与之相反。

采用这种顺序的原因如下：如果一个link有1个或者2个retimer，那么**不同的link segments可能有不同的precoding开关状态**。例如retimer和RC间的通路开启了precoding而retimer和EP间的通路关闭了precoding，如果parity bit calculation在precoding之后进行的话，那对于整个end2end传输，奇偶校验可能失败。

对于TS1 / TS2而言，由于**symbol 0 是没有scrambled的**，所以对于symbol 1 的bit0，和他异或的那个bit直接设置为**1b**即可

## Loopback with 128b/130b Code in Non-Flit Mode and Flit Mode
当使用128b/130b编码时，Loopback Leads必须传送值为01b或者10b的sync header。但是，当从发送Ordered Set Blocks转为发送Data Block时，它**无需发送SDS OS**，同理，当从发送Data Block转为发送Ordered Set Blocks时，也**无需发送EDS Token**。Leads必须周期性的发送SKP OS，而且必须能够处理loopback回来的，**长度各异的SKP OS**（这是为了时钟补偿用的）。Leads允许发送EIEOS。Leads允许在Ordered Set Blocks和Data Blocks中传输任何payload。如果Loopback lead发送了一个Ordered Set block，且他的第一个symbol符合SKP / EIEOS / EIOS的第一个symbol，则该Ordered Set Block必须是一个完整且有效的SKP OS / EIEOS / EIOS

当使用128b/130b编码时，Loopback Followers必须将它接收到的所有bit不经过任何改变而重新发出去（SKP OS除外，其长度可以根据时钟补偿的结果改变）。如果需要时钟补偿，则Followers必须对每个OS增加或者减少4个SKP Symbol。但是注意改变后的SKP OS必须还要符合SKP OS的一般要求，比如SKP OS由4-20个SKP Symbols和1个SKP_END Symbol组成，在SKP_END Symbol后面还有3个Symbol，如SPEC Table 4-52所示。如果Follower不能获得Block对齐，或者错误对齐，那么它有可能不能执行时钟补偿，因此不能loopback回它收到的bits。在这种情况下，允许follower按需要增加或者减少symbols，以继续运行。当检测到Ordered Set Blocks转为Data Block时，Followers**不允许检查SDS OS**，同理当检测到Data Block转为Ordered Set Blocks时，**不允许检查EDS Token**。（因为Transmitter发送时就没有发送SDS / EDS Token出去）