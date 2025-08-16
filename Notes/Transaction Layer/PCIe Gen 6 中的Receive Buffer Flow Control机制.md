# PCIe Gen 6 中的Receive Buffer Flow Control机制
本文主要基于PCIe 6.1 spec中的2.6节，记录一下对PCIe 6.1的Transaction Layer Flow Control机制的理解。尤其是PCIe 6.1中新增的shared credits / credits block / merged credits部分的理解。由于所做的方向及时间限制，本文主要集中于PCIe EP(End Point)中的Flow Control机制，对RC / SW则不涉及。

Flow Control机制在PCIe中，被用于防止接收方的buffer溢出，即只有接受方有足够的buffer来接收这一笔TLP时，发送方才允许发送该笔TLP出去，否则就会阻塞住。为了防止死锁的出现，Flow Control机制经常与2.4节中描述的ordering rules联合使用。注意，Flow Control机制仅仅用于发送方，即requester来追踪接收方的queue / buffer的情况，**不能被用于request的完成与否的判断**。

在Non-Flit Mode下，每个virtual channel都有一个**独立**的Flow Control credit pool。

在Flit Mode下，为了满足更高速率的传输要求，同时节约硬件资源，使用Shared FC机制。对于每个VC而言，它都有**两种**Flow Control credit pool。
1. 一个通常较小的FC credit pool，是该VC专用的credit资源。该pool存在的作用是防止死锁，允许transmitter至少使用接收方的专用credit pool中的资源，至少发送一笔TLP给接收方。要指定一笔TLP使用接收方的该FC credit pool中的资源进行传输，需要在TLP中添加Flit Mode Local TLP Prefix，且将Byte 1，bit 0 置1来硬性指定。 	
2. 使用一个较大的，shared flow control credit pool中的一部分资源。

PCIe 6.1中提供了一个**transmitter gate function， 它使用的是receiver的所有VC返回的shared FC之和**。同时，transmitter gate function还提供了一种机制，来防止被阻塞住的VC使用过多的shared FC资源。该机制通过软件配置，而且默认关闭。link两端的FC信息是通过DLLP来传输的。在DLLP中，包含了VC ID field，从而能正确指示是哪个VC发出的TLP占用/释放了FC。正是通过DLLP中包含的VC ID信息，才得以实现Usage Limit功能。此外，引入了Merged FC机制，该机制允许**Posted Requests和Completions**共享buffer，从而进一步降低硬件资源的消耗。

整个Flow Control机制由Transaction Layer和Data Link Layer共同完成。其中，TL层的接收端负责接收到的TLP的accounting，而TL层的发送端，则会根据link对面的credit pool情况来决定自己的TLP能否发送出去。

Flow Control机制虽然由TL层和DL层同时完成，但它是属于TL层的function。这就意味着它只负责TLP的追踪。对于DLLP，还有PL层有关的信息传输，Flow Control机制不负责追踪。实际上，其他层的这些packet和信息传输都是不被缓存的，在到达的同时就必须被receiver来处理。同时，由于Flow Control机制位于TL层，它对DL层的retry机制所触发的TLP重传也无法感知，且不关心。

## Flow Control Rules
Spec在本章中，首先描述了Flow Control的一些共性的规则。注意规则分为Non-Flit Mode与Flit Mode。
- 如前文所述，link两端的Flow Control信息主要是通过DLLP来传输的，准确的说是两种DLLP，即 
  - Flow Control Packets(FCPs)
  - Optimized_Update_FC
- 在Flow Control信息的传输中，描述flow control credit的单位并非Byte或者PCIe中常用的DW，而是FC Unit Size。对于不同的Data / Header，以及在NFM与FM下，每个FC Unit Size的大小实际上是不同的。 
  - 对于Data，FC Unit Size的大小为4DW
  - 对于Header，FC Unit Size的大小实际上是Header所允许的最长值
    -在Non-Flit Mode下： 		
      -Receiver不支持TLP Prefix，则FC Unit Size是Header所允许的最长值 + TLP Digest的长度
      -Receiver支持TLP Prefix，则FC Unit Size是Header所允许的最长值 + TLP Digest的长度 ＋最大数目的TLP Prefix的长度之和
    -在Flit Mode下，FC Unit Size是支持的最大Header长度 + 所有支持的OHC长度 + 支持的最大TLP Tailer长度。
- 在NFM下，或者在FM中使用dedicated credits时，每个VC都有自己独立的Flow Control机制
- 在FM下，每个VC都有自己的独立Flow Control Credits，同时拥有一部分shared Flow Control Credits 
  - **特殊情况：当function仅仅启用了一个VC（即VC0），且没有启用Merge Credit机制，则该VC没有dedicated credits，它的所有flow control机制都使用shared credits**
- 即使shared credits是充足的，也允许特别指定一笔TLP使用dedicated credits，通过TLP Prefix指定即可，如前文所述。
- 整个Flow Control机制使用6种Flow Control Credits，对这6种Credits执行独立的追踪。即3种TLP类型，每种类型的Header和Data独立追踪。
- 对于每种类型的TLP所消耗的FC credits数量，如图所示。注意，在计算data所需要的credits数量时，直接用data length（注意以DW为单位）除以FC unit size，而后向上取整。
- 仅仅对于默认的VC，即VC0，执行基于硬件的自动初始化。
  - 当reset之后，进行VC0的初始化，确切的说是在DL层状态机的DL_Init state来执行VC0的初始化
- 对于其他VC，软件通过set **link两端**的device的VC Enable bit，来开启该VC。而后根据Flow Control的初始化协议进行初始化。协议允许多个VC同时执行初始化过程，它们是相互独立的进程。
- 同样的，软件通过clear link两端的device的VC Enable bit，来disable该VC。 
  - Disable某个VC，意味着重置该VC的Flow Control tracking机制
  - 在FM下，disable某个VC，仅仅会重置该VC的dedicated credits的Flow Control tracking，而**对shared credits没有影响**
  - 在FM下，当link仍然有效时，disabled某个VC，而后又启用该VC，device的行为是未定义的。

对于传递credits信息的DLLP：
1. InitFC1和InitFC2 FCP仅仅用于Flow Control初始化过程
2. 任何描述**未启用的VC**的DLLP，包括InitFC1 / Init FC2 / UpdateFC FCP / Optimized_Update_FC，都需要被直接丢弃，不报错。

在FC初始化过程中，下表展示了所能广播的VC credit的最小值：
对上表的一些解析：
1. 当link两端都启用了Scaled Flow Control机制时，整个link启用该机制，VC Credit的最小值参考对应的Scale Factor那一列
2. 不启用Scaled Flow Control机制时，Scale Factor为1
3. 对于MFD，且不同Function拥有不同的Rx_MPS_Limit，则使用**其中的最大值**
4. 在Flit Mode下的初始化过程中，对于每一种Credit Type，shared credit advertisement只能执行以下操作之一： 
   1. 在所有VC上都advertise [Infinite.3]
   2. 在有的VC上advertise [Zero]，而有的VC上advertise non-[Zero] 	
      1. 当一个device支持多个VC时，允许多有VC都advertise [Zero] shared credits，这时候针对这种credit type，**所有VC都只能使用dedicated credits**
      2. 所有advertise non-[Zero] shared credits的VC，都必须使用**同样的Scale Factor**
5. 如果有VC advertise non-[Zero] shared credits，则以下要求必须满足： 
   1. 对于所有的其他**advertise [Zero] shared credits**的VC，它们的UpdateFC中的scale factor也必须使用该VC的Scale Factor
   2. 所有VC advertise的**某一类credits**数量之和，必须大于等于上表描述的**credits最小值 × VC数量**。同时，对于单独的VC的credits数量不作要求
   3. 在使用Merged Flow Control机制时，所有VC advertise的PH或者PD的数量之和，必须大于等于上表描述的**credits的最小值 × VC数量 × 2**
6. 在Flit Mode下，dedicated credits允许使用任何scale factor
7. 对于minimum credits的解析： 
   1. 在NFM下： 	
      1. Posted Header的最小值就是credit value为01h，有scale factor注意乘上scale factor即可
      2. Posted Data 		
         1. 在scale factor为1时，规定为最少能接受一笔最大length的TLP。即Rx_MPS_Limit / FC Unit Size，其中Rx_MPS_Limit在Flit Mode下其实就等于Max_Payload_Size
         2. 在scale factor为4时
         3. 在scale factor为16时
      3. Non Posted Header的最小值就是credit value为01h，有scale factor注意乘上scale factor即可
      4. Non Posted Data 		
         1. 在scale factor为1时，规定为Rx_NP_MPS_Limit。其中Rx_NP_MPS_Limit是receiver所能接受的non-posted request的data payload的最大值。
         2. 在scale factor为4时
         3. 在scale factor为16时
      5. CplH，对于EP，为无限大
      6. CplD，对于EP，为无限大
   2. 在FM下：
- 对于receiver而言，它能向transmitter通报的unused credits数量是有上限的，这是由于UpdateFC的field size限制决定的。如图 

  1. 对于scale factor为1，则data credits的上限为2047，header credits的上限为127，**由于credits计数的指针是单向增加的，为了credits的追踪，最大值限定为此**。
  2. 对于其他scale factor，见下表 	
  3. 这是一个可选检查，报错为Flow Control Protocol Error

- 如果在初始化时，advertise了[Infinite.1] / [Infinite.2] / [Infinite.3]，则在初始化之后就不需要发送Flow Control Updates了。 
  - 如果在这种情况下，初始化之后仍然发送了UpdateFC DLLP或者Optimized_Update_FC，**那么credit value field必须为0**。这是一个可选检查，报错为Flow Control Protocol Error
- 如果在初始化过程中，启用了Scaled Flow Control，那么后续的UpdateFCs中的HdrScale和DataScale的值必须保持不变，和初始化过程中的一致。除了以下例外： 
  - 在Flit Mode下，当启用了多个VC的时候，允许某个VC在初始化时advertise [Zero] shared credits。对于这种VC，它的shared credit UpdateFCs中的HdrScale和DataScale必须和其他的VC使用的non-[Zero] HdrScale和DataScale值一致。
  - 如果所有VC在初始化时都advertise [Zero] shared credits，那么它的shared credit UpdateFCs中的HdrScale和DataScale的值是未定义的。
  - 在Flit Mode下，在初始化时，允许advertise [Merged] shared completion credits。在这种情况下，shared completion credits UpdateFCs中的HdrScale和DataScale必须**和初始化时shared posted credits中的一致**。
  - 当[Merged] completion credits被advertise时，**至少要有一个VC advertise non-[Zero] shared posted completion credits**.（要不然completion就没有credits了）
  - 以上是可选的检查，报错为Flow Control Protocol Error
- 使用没有启用的VC的TLP是一个malformed TLP 
  - VC0总是enabled的
  - 对于VC1-7，当corresponding VC Enable bit 被Set，且那个VC的FC negotiation已经退出FC_INIT1，进入FC_INIT2，则将其视为enabled
- 直到DL层的状态机退出FC_INIT2，才允许发送对应的TLP
- 对于VCs 1-7而言，采用的是软件驱动的初始化过程。因此如软件使用corresponding VC Resource Status Register中的VC Negotiaton Pending bit来确保直到协商完成后，才使用该VC
- 在Flit Mode下，如果软件disable了VC 1-7： 
  - 对应VC的dedicated credit counts被清除
  - shared credit counts不受影响
  - 如果软件后续又re-enable了VC 1-7，那么device行为未定
- Field Size的定义：其实就是DLLP中的field长度。如下图所示 

在Flit Mode下，以下规则对[Merged] FC credits适用：
- receivers可以支持[Merged] FC，而transmitter则必须支持[Merged] FC
- 当真的启用[Merged] FC时： 
  - shared completion header credits与shared posted header credits merged 	
    - 在FC初始化时，使用**shared Posted Header credits**来表示总的merged shard header credit pool
    - **FC updates**必须根据receiver free的TLP类型来指明shared posted header credits或者shared completer header credits，即虽然两种credit已经shared了，但是在UpdateFC DLLPs中还是必须区分到底是Posted TLP被处理，从而释放出的credits，还是Completion TLP被处理从而释放出的credits
  - shared completion data credits与shared posted data credits merged 	
    - 在FC初始化时，使用**shared Posted Data credits**来表示总的merged shard data credit pool
    - FC updates必须根据receiver free的TLP类型来指明shared posted data credits或者shared completer date credits
  - Dedicated Header / Data credits不允许merged
  - 对于一个link而言，两边的device是否启用Merged FC是独立的。
  - [Merged] shared credits这一特性在所有VC中必须一致，即**如果一个VC启用了[Merged] shared credits，那么所有VC都必须启用[Merged] shared credits**。
  - [Merged] shared credits这一特性的启用**必须对Hdr和Data一致**，即如果一个VC的Hdr启用了[Merged] shared credits，那么其Data必须启用[Merged] shared credits。
  - 当启用了[Merged] FC时，必须确保在Flit Mode下，receiver必须返回credits来指示释放的buffer对应的VC。如果启用了[Merged] FC，**那么返回的credits必须还要指示释放的Buffer所对应的FC种类**。

对于VC所使用的Scale Factor，spec也作出了限制。即**使用该Scale Factor要能够表示所有已经分配的credits**。例如同时启用了VC0和VC1，它们都advertise了120个header credits，虽然不使用scale factor，或者使用scale factor = 01b都能分别表示VC0或者VC1中advertise的header credits数量。但是VC0和VC1所advertise的header credits之和为240，已经超出了127，因此必须使用不为01b的scale factor。

对于一个给定的VC内的某种FC Type而言，如果这种FC Type的shared credits为infinite，那么对于这种FC Type，它在**所有VC中的shard credits和dedicated credits都必须为infinite**。Spec中举了一个例子：
- function中同时启用了VC0和VC1，如果VC0中的shared completion credits为infinite，那么在VC0和VC1中的shard completion credits和dedicated completion credits都必须为infinite。