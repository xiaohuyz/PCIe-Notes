# Clock Tolerance Compensation

SKP Ordered Sets用于补偿link两端port之间的bit rates差距。执行补偿的Receiver端的物理层的logical sub-block必须包含一个elastic buffering（这个用于出现bit rates差距，但尚未来得及补偿时的数据存储），该elastic buffer在Original PIPE架构中位于PHY中，而在SerDes架构中位于MAC Physical Layer中。Transmitter发送SKP OS的间隔由Transmit，Receiver和Refclk决定。

SPEC中，支持shared reference clocking architecture（即common Refclk），在这种情况下，Rx和Tx的时钟速率没有任何差别。同时，SPEC也支持另外两种参考时钟架构，在这两种时钟架构下，Rx和Tx的时钟是不一样的。即Separate Reference Clocks With No SSC-SRNS和Separate Reference Clocks With Independent SSC-SRIS。在SRNS下，最大的时钟差为600ppm，因此**最短每1666个时钟周期，Tx和Rx会有一个时钟周期的差距**。而在SRIS下，最大时钟差为5600ppm，即**最短每178个时钟周期，Tx和Rx会有一个时钟周期的差距**。注意，启用SSC，即扩频时钟的情况下，Tx和Rx的时钟偏差会更大，因此在SRIS下Tx和Rx出现时钟周期差距的时间间隔更短

具体的form factor SPEC会要求仅仅使用SRIS，或者仅仅使用SRNS，或者提供一种机制来选择时钟架构。对于Upstream Port，允许实现SRIS和SRNS的任意组合，或者一个都不实现（即USP仅仅支持Common Refclk），但是要符合具体的form factor SPEC的要求。对于Downstream Port，**如果支持SRIS，则必须同时支持SRNS**，除非该Downstream Port仅仅与一个特定的form factor相连接，form factor spec对此另有要求。

如果Receiver即使工作于SRIS下，也能以SRNS下的SKP OS频率工作，则该Port允许将**Link Capabilities 2 Register**中的**Lower SKP OS Reception Supported Speeds Vector**中的相应bit置为1，从而告知link对端的transmitter，允许它以SRIS下的发送间隔来发送SKP OS。同理，如果一个Transmitter即使工作于SRIS下，也能以SRNS下的SKP OS产生频率工作，则Port允许将**Link Capabilities 2 Register** 中的**Lower SKP OS Generation Supported Speeds Vector field**置为1。对于软件而言，必须首先检查Link一侧的Lower SKP OS Reception Supported Speeds Vector，而后才设置link对侧的Link Control 3 Register中的Enable Lower SKP OS Generation Vector field（其实就是，software必须在确保Rx侧支持**在SRIS下使用SRNS的SKP OS频率工作**的前提下，才能够使能Tx侧的该功能）。同时对于软件而言，如果要将Link Control 3 Register中的Enable Lower SKP OS Generation Vector field设置为1，则整个link上的所有software transparent Extension Devices，比如repeater，必须也支持lower SKP OS generation（这个应该由具体实现决定，因为这些device对software是不可见的）。当在**某个速率**下的Enable Lower SKP OS Generation Vector field为1时，transmitter在L0下，按照SRNS要求的频率来安排SKP OS的发送，而不管link实际工作的时钟架构。而在其他LTSSM下，需要按照具体的时钟架构的要求来安排SKP OS的发送频率。

> 注意，Lower SKP OS Reception / Generation机制，仅仅适用于L0下的数据传输，而不适用于Link Training过程，因为这会引起Link Training过程中的timeout等问题。因此，在Link Training过程中，SKP OS的发送频率和期待的接收频率还是按照实际的时钟架构而定。

支持SRIS的components可能会需要更大的elastic buffer，因为在SRIS下，在很短的时间间隔内，Tx和Rx就会出现时钟周期的差距

## SKP Ordered Set for 8b/10b Encoding

当使用8b/10b编码时，**发送出去**的一个SKP OS由一个COM Symbol，和3个SKP Symbols组成。一个例外是在Loopback.Active下的Loopback Follower，在此LTSSM state下，一个接收到的SKP OS由一个COM Symbol和1-5个SKP OS组成，因为它在把数据Loopback回去的时候，通过加减SKP OS中的SKP Symbols的数量，完成了时钟补偿。

具体表格如图所示

| Symbol Number | Symbol |
| :-----------: | :----: |
|       0       |  COM   |
|       1       |  SKP   |
|       2       |  SKP   |
|       3       |  SKP   |

## SKP Ordered Set for 128b/130b Encoding

当使用128b/130b encoding时，发送出去的SKP OS为16 Symbols，而接收到的SKP OS可能为8，12，16，20或者24 Symbols，这是因为receiver通过加减SKP OS中的SKP Symbols的数量，完成了时钟补偿。

在128b/130b编码下，定义了两种类型的SKP OS，即Standard SKP OS和Control SKP OS。

其中，Standard SKP OS如下表所示

|         Symbol Number         |                            Value                             |                         Description                          |
| :---------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 0 through (4*N-1)，其中N为1-5 | 1. AAh for 8.0 GT/s and 16.0 GT/s data rates. <br>2. 99h for 32.0 GT/s and higher data rates. | SKP Symbol. Symbol 0 is the SKP Ordered Set identifier. 其实就是1-5组SKP Symbol，每组由4个SKP Symbol组成 |
|              4*N              |                             E1h                              | SKP_END Symbol。标志着在接下来的3个Symbol之后，SKP OS就结束了。实际上就是SKP OS中最后一组Symbol的开头（我们定义SKP OS中一组Symbol为4个） |
|            4*N + 1            |                            00-FFh                            | 1. 如果LTSSM处于Polling.Compliance，则该Symbol为固定值AAh<br>2. 如果该SKP OS之前，是一个Data Block，则Bit[7] = Data Parity, Bit[6:0] = LFSR[22:16].<br>3. 否则，Bit[7] = ~LFSR[22], Bit[6:0] = LFSR[22:16]. |
|            4*N + 2            |                            00-FFh                            | 1. 如果LTSSM处于Polling.Compliance，则该Symbol为固定值Error_Status[7:0]<br/>2. 否则，该Symbol为LFSR[15:8] |
|            4*N + 3            |                            00-FFh                            | 1. 如果LTSSM处于Polling.Compliance，则该Symbol为固定值~Error_Status[7:0]<br/>2. 否则，该Symbol为LFSR[7:0] |

注意Control SKP OS和Standard SKP OS仅仅在最后一组Symbol，即最后4个Symbol处不同。Control SKP OS用于link两端交流parity bits的值，包括DSP / USP计算出的parity bits，以及每一个Retimer计算出的parity bits。同时，Control SKP OS也被用于Retimer Receiver的Lane Margining

Control SKP OS如下表所示

|         Symbol Number         |                            Value                             |                         Description                          |
| :---------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 0 through (4*N-1)，其中N为1-5 | 1. AAh for 8.0 GT/s and 16.0 GT/s data rates. <br>2. 99h for 32.0 GT/s and higher data rates. | SKP Symbol. Symbol 0 is the SKP Ordered Set identifier. 其实就是1-5组SKP Symbol，每组由4个SKP Symbol组成 |
|              4*N              |                  78h，注意和SKP OS中不一样                   | SKP_END_CTL Symbol。标志着在接下来的3个Symbol之后，Control SKP OS就结束了。实际上就是Control SKP OS中最后一组Symbol的开头（我们定义Control SKP OS中一组Symbol为4个） |
|            4*N + 1            |                            00-FFh                            | Bit[7]: Data Parity<br>Bit[6]: First Retimer Data Parity<BR>Bit[5]: Second Retimer Parity<br>Bits[4:0]: Margin CRC[4:0] |
|            4*N + 2            |                            00-FFh                            |    Bit[7]: Margin Parity<br>Bit[6:0] 参见Lane Margin部分     |
|            4*N + 3            |                            00-FFh                            |                 Bit[7:0] 参见Lane Margin部分                 |

Margin CRC的算法和Margin Parity略过

每种SKP OS都包含了1-5组SKP Symbol，其中每组SKP Symbol中含有4个SKP Symbol。同时，每种SKP OS，最后都有一个由SKP_END或者SKP_END_CTL标注的4 Symbol组来结束。其中，SKP_END和SKP_END_CTL表示的是在接下来3个Symbol之后，整个SKP OS结束。

* 当在Non Flit Mode下，工作于8.0 GT/s时，仅仅使用Standard SKP OS。

* 当在Non Flit Mode下，工作于16.0 GT/s或者32.0 GT/s，Standard SKP OS和Control SKP OS都会使用。

* 当在Flit Mode下，工作于8.0 GT/s，16.0 GT/s和32.0 GT/s下，Standard SKP OS和Control SKP OS都会使用。

在文档中，当没有特殊指定某一种SKP OS时，则指的是任意种类的SKP OS。当一个SKP OS被发送出去，则link上所有的Lane必须发送同种类型的SKP OS。即所有lanes都必须发送Standard SKP OS或者Control SKP OS。

接下来描述SKP_END / SKP_END_CTL symbol之后的3个Symbol的具体内容。

对于一个Standard SKP OS，它在SKP_END Symbol之后的3个Symbol中包含的信息，由LTSSM state和Block sequence决定。当LTSSM处于Polling.Compliance state中时，这3个Symbols都包含了Lane的error status information（由上表可知，Symbol 4n+1为固定值AAh，4n+2为error_status，4n+3为error status取反）。如果前一个Block为data block，则Symbol 4n+1的bit[7]为Data Parity，其余Symbol构成了lane的scrambling LSFR的值。在其他情况下，symbol也是lane的scrambling LSFR值，只不过symbol 4n+1的bit[7]为LSFR[22]取反。具体参见上表

对于Control SKP OS，它包含了3个Data Parity，以及其他数据。

当在Non Flit Mode下，工作于8.0 GT/s时，Standard SKP OS中的Data Parity bit是Data Payload的偶校验数值，而且每条lane独立进行计算。对于Upstream和Downstream Ports，按照如下方法进行计算：

- 当发送出去一个SDS OS之后，Parity被重置，即每一次单独的data block传输都重置Parity计算
- Parity随着经过scramble的每一个data payload的bit的发出而变化，即实时计算和更新Parity
- 在Data Block之后，发送Standard SKP OS时，其中的Data Parity bit就是这个Data block最终计算出来的Data Parity bit
- 在发送出去一个Standard SKP OS之后，Parity被重置，从而开始计算下一个Data Block的Parity

对于Upstream和Downstream Port的receiver，按照如下方法计算parity，同时与接收到的实际Parity值进行比对

- 当接收到一个SDS OS之后，Parity被重置，即开始计算接下来一个Data Block的data parity
- Parity随着接收到Data block的每一个bit的处理而更新，注意Parity计算使用的是de-scramble之前的data。因为在Tx侧的Parity计算使用的是Tx scramble之后的data
- 当在一个Data Block之后，接收到一个Standard SKP OS，每个Lane将接收到的Standard SKP OS中的Data Parity和自己计算出的Parity相比较。如果不匹配，则receiver必须将Lane default number所对应的Lane Error Status Register中的对应bit置1。注意使用的是**Lane default number**对应的Lane Error Status Register，不受lane reverse影响。注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当接收到一个Standard SKP OS时，Parity被重置。因为Data Parity已经计算并且比对完成了，可以开始下一次计算和比对了。

当在Non Flit Mode下，且工作于16.0 GT/s或者32.0 GT/s，或者在Flit Mode下，且工作于8.0 GT/s，16.0 GT/s或者32.0 GT/s，发送的可能是Standard SKP OS或者Control SKP OS。在Non-flit mode下的Standard SKP OS，以及Control SKP OS，其中的Data Parity值都是lane上发送的Data block计算出的偶校验值，并且每个lane独立计算。对于Port上的Transmitter，parity按照如下规则进行计算：

- 当LTSSM处于Recovery.Speed state时，Parity被重置
- 当一个SDS OS被发送出去，则Parity被重置，即重新开始计算Data Parity
- Parity随着经过scramble的每一个data payload的bit的发出而更新，其实就是随着数据每一个bit的发送而实时更新。
- 因此，在Flit Mode下，Parity是在FEC之后进行计算的
- 在Data Block之后发送Standard SKP OS时，其中的Data Parity bit就是这个Data block最终计算出来的Data Parity bit
- 同理，对于Control SKP OS的data parity，First Retimer Data Parity，Second Retimer Data Parity，被设置为现在的parity值
- 当发送出去一个Control SKP OS之后，重置Parity。但是，当发送出去一个Standard SKP OS之后，不重置Parity（为啥？）

Upstream Port和Downstream Port Receiver按照如下规则计算Parity

- 当LTSSM处于Recovery.Speed state时，Parity被重置
- 当接收到一个SDS OS之后，Parity被重置
- 在de-scramble之前，Parity随着Data block的每一个bit的接收而更新
- 因此，在Flit Mode下，在FEC decoding之前计算Parity
- 当接收到一个Control SKP OS之后，每一条lane都将接收到的Data Parity和自己计算出的Parity相比较。强烈建议这一检查仅仅针对于紧跟着Data Block的Control SKP OS执行。如果不匹配，则receiver必须将Lane default number所对应的Lane Error Status Register中的对应bit置1.注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当接收到一个Control SKP OS之后，每一条lane都将接收到的First Retimer Data Parity和自己计算出的Parity相比较。强烈建议这一检查仅仅针对于紧跟着Data Block的Control SKP OS执行。如果不匹配，则receiver必须将Lane default number所对应的Lane Error Status Register中的对应bit置1.注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当接收到一个Control SKP OS之后，每一条lane都将接收到的Second Retimer Data Parity和自己计算出的Parity相比较。强烈建议这一检查仅仅针对于紧跟着Data Block的Control SKP OS执行。如果不匹配，则receiver必须将Lane default number所对应的Lane Error Status Register中的对应bit置1.注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当在Data Block之后，立即接收到一个Standard SKP OS，receiver允许将接收到的Data Parity bit和自己计算出的相比较。但是，比较的结果不允许影响Lane Error Status Register
- 当接收到一个Control SKP Ordered Set，Partiy会被初始化。但是，当接收到一个Standard SKP Ordered Set，parity不会被初始化。
- 其实就是Standard SKP OS的发送和接收条件与在8.0 GT/s下不一样

Standard Ordered Set中的LFSR：

Control SKP OS和Standard SKP OS仅仅最后4个Symbol不相同。这4个Symbol用于交流每个Retimer计算出的Parity bits，加上Upstream Port / Downstream Port计算出的Data Parity bit。同时，也会用于Retimer Receiver处的Lane Margining。其实就是为了方便16.0 GT/s及以上速率中加入的Retimer而设计的

## SKP Ordered Set for 1b/1b Encoding

- 在1b/1b编码下，仅仅传输Control SKP Ordered Sets
- 发送出去的SKP OS的大小为40 Symbols（40B），接收到的SKP OS的大小可能是24，32，40，48或者56 Symbols（24，32，40，48，56B），其实就是Rx侧已经通过增减SKP OS中的SKP Symbol数量完成了时钟补偿
- 发送出去的SKP OS由SKP OS（F00F_F00F）,SKP_END（在表中为28, 29, 30, 31）, PHY Payload组成，如图所示

![img](https://i-blog.csdnimg.cn/direct/561f6a87f49c4a24bea9d74c29ac2950.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

![img](https://i-blog.csdnimg.cn/direct/0cd3dc470aae4c5b99574075b84e0cd5.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)编辑

- 其中，PHY Payload的含义如图所示
- Payload Type和Parity fields重复四次，且它们的间隔超过16bits
  - 如果3个或者更多的sets match，则receiver可以使用一个多数投票来修正这些fields。如果2个sets的value为0b，而另外2个sets的value为1b，则认为是一个平局。关于Parity，tie可以直接被视为mismatch，而对于Payload Type，tie导致ignore PHY payload field的值。
- Payload field被重复2次
- 对于拥有自己的CRC的Margin field，Receiver能选择通过CRC检查的copy

Data Parity bit是一条lane上所有Data Blocks的payload的偶校验数值，且每条lane都是独立进行计算的。Upstream Port和Downstream Port的transmitters按照如下方法计算parity：

- 当LTSSM处于Recovery.Speed state时，Parity被重置
- 当一个SDS OS被发送出去，则Parity被重置
- 当一个SKP OS被发送出去，则Parity被重置
- Parity随着经过scramble的每一个data payload的bit而重新计算。因此，存在的gray coding和precoding不会影响Parity计算。并且，parity计算是在FEC之后进行的
- 在Data Block的parity确认之后，立即发送Standard SKP OS的data parity bit
- 同理，对于Control SKP OS的data parity，First Retimer Data Parity，Second Retimer Data Parity，也是在那时进行发送

对于Upstream Port和Downstream Port的Receiver，按照如下方式来计算：

- 当LTSSM处于Recovery.Speed state时，Parity被重置
- 当接收到一个SDS OS时，Parity被重置
- 在de-scrambling进行之前，Parity随着Data Block的每个bit而更新。因此，parity的计算会在gray coding和precoding之后进行。而且，parity的计算和比较是在FEC之前进行的。
- 当接收到一个Control SKP OS之后，每一条lane都将接收到的Data Parity和自己计算出的Parity相比较。强烈建议这一检查仅仅针对于紧跟着Data Block的Control SKP OS执行。如果不匹配，则receiver必须将Lane default number所对应的Local Data Parity Mismatch Status Register中的对应bit置1.注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当接收到一个Control SKP OS之后，每一条lane都将接收到的First Retimer Data Parity和自己计算出的Parity相比较。强烈建议这一检查仅仅针对于紧跟着Data Block的Control SKP OS执行。如果不匹配，则receiver必须将Lane default number所对应的First Retimer Data Parity Mismatch Status Register中的对应bit置1.注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当接收到一个Control SKP OS之后，每一条lane都将接收到的Second Retimer Data Parity和自己计算出的Parity相比较。强烈建议这一检查仅仅针对于紧跟着Data Block的Control SKP OS执行。如果不匹配，则receiver必须将Lane default number所对应的Second Retimer Data Parity Mismatch Status Register中的对应bit置1.注意，该mismatch不是一个Receiver Error，且不会导致link retrain。
- 当接收到一个SKP Ordered Set之后，初始化Parity

下面描述SKP OS的发送间隔问题

## Rules for Transmitters

- 所有的Lanes都必须以相同的速率传输Symbols（对于all multi-Lane Links，bit rates之间的difference为0 ppm）
- 当传输的时候，对于一个multi lane link，所有的lanes都必须同时发送SKP OS，且长度和Type（Control or Standard）相同。唯一的例外是Loopback.Active LTSSM State下的Loopback Follower。
- 当使用8b/10b编码时，发送的SKP OS必须符合编码规范
- 当使用128b/130b编码时，发送的SKP OS必须符合编码规范
- 当使用1b/1b编码时，发送的SKP OS必须符合编码规范
- 当使用8b/10b编码时：
  - 如果link没有工作于SRIS模式，或者LTSSM处于L0且当前速率下的Enable Lower SKP OS Generation Vector field为1，则必须按照1180-1538 symbol times的间隔来安排SKP OS的发送，其实就是按照SRNS模式下要求的间隔进行
  - 如果link工作于SRIS模式，或者LTSSM没有处于L0，或者当前速率下的Enable Lower SKP OS Generation Vector field为0，则必须按照小于154 symbol times的间隔来安排SKP OS的发送，其实就是按SRIS下要求的间隔进行
- 当使用128b/130b编码时：
  - 如果link没有工作于SRIS模式，或者LTSSM处于L0且当前速率下的Enable Lower SKP OS Generation Vector field为1，在Non Flit Mode下或者在没有发送Data Stream的Flit Mode下，必须按照370-375 Blocks的间隔来安排SKP OS的发送。Loopback Followers必须满足这一要求，直到它们开始looping back incoming bit stream
  - 如果link工作于SRIS模式，或者LTSSM没有处于L0，或者当前速率下的Enable Lower SKP OS Generation Vector field为0，在Non Flit Mode下或者在没有发送Data Stream的Flit Mode下，必须按照小于38 Blocks的间隔来安排SKP OS的发送。Loopback Followers必须满足这一要求，直到它们开始looping back incoming bit stream
  - 当LTSSM处于Loopback state，而且Link没有工作于SRIS模式，Loopback Lead必须安排两个SKP OS的传输，保证它们之间最多间隔2个Blocks，且间隔为370-375 Blocks。对于32.0 GT/s，允许Loopback Lead在2个SKP OS之间插入一个EIESQ
  - 当LTSSM处于Loopback state，且Link工作于SRIS模式，Loopback Lead必须安排两个SKP OS的传输，保证它们之间最多间隔2个Blocks，且间隔小于38 Blocks。对于32.0 GT/s，允许Loopback Lead在2个SKP OS之间插入一个EIESQ
  - 当工作于Flit Mode，且Data stream正在传输，或者正在转变为传输Data Stream，则按照64.0 GT/s的要求发送SKP OS
  - 仅仅在以下情形下，发送Control SKP OS，在其余情况下均应使用Standard SKP OS： 	
    - 在Non Flit Mode下，且速率为16.0 GT/s或者32.0 GT/s。在Flit Mode下，且速率为8.0，16.0，32.0 GT/s。并且以下条件有一个满足： 		
      - 正在发送一个Data stream。除非正在进行L0p upsize，否则Data Stream中发送的SKP OS必须区分Standard SKP OS和Control SKP OS。在L0p下，当Link中的inactive lanes正在变为active lanes，在Transitioning lanes上发送Data Stream之前，必须在所有的Lanes上发送Control SKP OS。在L0p upsize过程中，允许active lanes发送Control SKP OS来代替Standard SKP OS，从而减少L0p upsize的latency
      - LTSSM进入Configuration.Idle state或者 Recovery.Idle state。在这种情形下发送的Control SKP OS不必遵循任何最小发送间隔的限制
      - 正在发送L0p upsizing之后的第一个SKP OS。必须在所有的lanes上发送Control SKP OS-active lanes以及将要被active的lanes
  - 当使用1b/1b编码时：仅仅发送Control SKP OS，必须遵循以下rules，除非正在发送Compliance Modified Compliance patterns； 	
    - 如果link没有工作于SRIS模式，或者LTSSM处于L0且当前速率下的Enable Lower SKP OS Generation Vector field为1，在没有发送Data Stream的Flit Mode下，必须按照740-750 Blocks的间隔来安排SKP OS的发送。Loopback Followers必须满足这一要求，直到它们开始looping back incoming bit stream
    - 在Flit Mode下，在2.5 GT/s之外的速率下，改变Enable Lower SKP OS Generation Vector的设置，则link的行为未定
    - 如果link工作于SRIS模式，或者LTSSM没有处于L0，或者当前速率下的Enable Lower SKP OS Generation Vector field为0，在没有发送Data Stream的Flit Mode下，必须按照小于76 Blocks的间隔来安排SKP OS的发送。Loopback Followers必须满足这一要求，直到它们开始looping back incoming bit stream
    - 当LTSSM处于Loopback模式，且Link没有工作于SRIS模式，Loopback Lead必须安排2个背靠背的SKP OS发送。当没有处于Data stream中时，SKP OS必须以低于76 blocks的间隔发送
    - 在Data stream中，或者正在转变为发送Data stream之前，或者正好在data stream结束之后，则按照对应要求发送SKP OS
    - Phy Payload Type必须按照以下要求来发送： 		
      - 如果LTSSM处于Polling.Compliance，则Phy Payload Type为1b
      - 否则，对于Lane0，在连续的SKP OS中，在0b和1b之间选择。且从electrical idle退出之后，随便选择一个初始值。其他的任意lane都必须发送和lane0一样的数值
  - 当没有packet或者Ordered sets在传输时，安排好的SKP OS应该被发送出去，否则，将它们积累下来，而后在下一个packet或者Ordered sets边界处插入。当使用128b/130b编码或者1b/1b编码时，在Data Stream中，不允许连续传输SKP OS
  - 当计算连续的Symbols或者Ordered sets时，SKP OS不应该被视为打断
  - 当使用8b/10b编码时，如果Link Control 2 register中的Compliance SOS bit为0b，则在Polling.Compliance中，当Compliance Pattern或者Modified Compliance Pattern正在传输时，不允许发送SKP OS。如果Link Control 2 register中的Compliance SOS bit为1b，则当Compliance pattern或者Modified Compliance Pattern正在进行时，在每个安排好的SKP OS发送处，必须发送2个连续的SKP OS，而不是1个
  - 当使用128b/130b编码或者1b/1b编码时：Compliance SOS register没有作用。当处于Polling.Compliance时，Transmitter不会发送任何SKP OS，除了那些被认为是Modified Compliance Pattern的一部分的
  - 在安排SKP OS发送的时候，transmitter的任意electrical idle都不会计算在时间间隔内
  - 建议任意与安排SKP OS发送相关的counters或者其他机制，在transmitter处于electrical idle时都进行重置。

## Rules for Receivers

- Receivers必须按照不同编码的要求来区分和辨别SKP OS
  - 在一个Multi-Lane Link中，接收到的SKP OS不应该有lane to lane的不同，除了在Loopback.Active
- Receiver必须能处理接收到的SKP OS
  - 因为Transmitter处于Electrical时，不一定要重置他们的计时器，因此Receiver要允许在Electrical idle之后接收到一个SKP OS，其时间间隔小于SKP OS的要求
- 对于8.0，16.0，32.0 GT/s，在L0 state下，Receiver必须检查每个SKP OS之前都必须有一个Data block，且带着EDS Token
- 在2.5 GT/s和5.0 GT/s下，Receiver必须能够接收连续的SKP OS
- 在1b/1b编码中，receiver允许在边界处，插入或者删除8 bytes SKP，因此，接收到的SKP OS的长度可能是24，32，40，48或者56 bytes