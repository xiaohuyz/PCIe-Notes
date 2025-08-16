# PCIe 6.1中的Flow Control Credit Count与Transmit Gating机制
与PCIe 5.0相比，PCIe Gen6中由于shared FC / merged FC / credit block等新概念的引入，整个的Credit count机制与transmit gating机制复杂了许多。

整个Flow Control机制需要结合2.6节与Chapter 3来分析。

首先我们解释一下在credit count中出现的各个elements

## FC Information Tracked by Transmitter
**CREDITS_CONSUMED (per VC, all modes)**
这是transmit侧统计的数据，缩写为CC，包含了该transaction pending buffer所发送出去的所有TLP使用的credit数量。该指针永远是单调增加的。
- 在NFM下，所有TLP都会更新CREDIT_CONSUMED
- 在FM下，只有使用dedicated credits的TLP才会更新CREDIT_CONSUMED。其实在FM下，使用dedicated credits，即消耗VC自己独有的credits来传输TLP，这种传输行为就和NFM下的常规传输一样。
- CC统计了自FC初始化以来，所有发送出去的TLP所消耗的credit数量。注意，由于DLLP中的[Field Size]影响，该值需要，即当单调增加的指针达到顶端后再绕下来。
- 当FC初始化时，CC为0。这是因为刚初始化时，没有TLP被发送，自然也就没有credits被consumed
- 当某个VC的VC Enable被cleared，即该VC被disable时，将其CC重置为0
- CC的更新：当TL层允许某个TLP通过Flow Control Gate而发送出去时，CC更新如下。其中Increment就是这被TLP所消耗的FC credits的值，同样注意最后的

**SHARED_CREDITS_CONSUMED(per VC, Flit Mode only)**
- 在NFM下，SHARED_CREDITS_CONSUMED没有使用
- 在FM下，使用shared credits的TLP更新SHARED_CREDITS_CONSUMED
- 同样的，SCC统计了自FC初始化以来，所有发送出去的TLP所消耗的credit数量。注意，由于DLLP中的[Field Size]影响，该值需要，即当单调增加的指针达到顶端后再绕下来。
- 当FC初始化时，SCC为0。这是因为刚初始化时，没有TLP被发送，自然也就没有shared credits被consumed
- SCC的更新与CC类似：
- 对于没有启用或者没有实现的VC，SCC为0
- 当VC1-7的VC enable被clear之后，SCC处于保留状态，没有意义。**这意味着仅仅启用VC0的时候，该VC没有dedicated credits，它的所有flow control机制都使用shared credits，但是gate机制不使用SCC来计算？**
- 对于[Merged] FC而言，Posted credits和Completion credits的SCC是单独统计和计算的

**SUM_SHARED_CREDITS_CONSUMED(per VC, Flit Mode only)**
- 我们知道，shared credits是在所有VC之间共享的，因此定义了该变量，SSCC，来统计自FC初始化以来，所有VC使用shared credits发送出去的TLP，所消耗的credits数量之和。公式如下：

**SHARED_CREDITS_CONSUMED_CURRENTLY(per VC, Flit Mode only)**
- 在NFM下，SCCC没有使用
- 在FM下，使用shared credits发送TLP时，更新SCCC。**同时，在transmitter接收到receiver返回的UpdateFCs时，也会更新SCCC。注意，不同于前文的elements，SCCC的指针不是单调递增的**
- 当FC初始化时，SCCC为0。
- 当TL层允许一个使用shared credits的TLP通过Flow Control Gate发送出去时，SCCC进行如下更新，Increment就是该笔TLP所消耗的credits数量
- 当transmitter接收到一个UpdateFC时，它按照如下的规则更新SCCC：

**TOTAL_SHARED_CREDITS_AVAILABLE**

**CREDIT_LIMIT(per VC, all modes)**
- 在NFM下，CL反映了所有的credit flow control更新。其实就是receiver方已经处理完成了TLP，而后发送DLLP来告知transmitter，transmitter会更新CL
- 与CC一样，在FM下，CL仅反映了dedicated credit flow control的更新情况。
- CL包含了**receiver最近一次advertise的FC units的数目**。其意义是**自从初始化以来，Receiver通过处理TLP，释放出的所有FC credits之和。**注意，由于DLLP中的[Field Size]影响，该值需要，即当单调增加的指针达到顶端后再绕下来。
- 在interface初始化时，CL的值未定义。因为此时receiver没有advertise FC units。
- 对于infinite credits，在FC初始化时，直接将CL设置为0
- 每次当transmitter接收到一个Update FC时，将Transmitter侧的CL值更新到Update FC中的value即可

**SHARED_CREDIT_LIMIT(per VC, Flit Mode only)**
- 在NFM下，SHARED_CREDIT_LIMIT没有使用
- 在FM下，SHARED_CREDIT_LIMIT反映了shared credit flow control updates
- SCL包含了**receiver最近一次advertise的FC units的数目**。其意义是**自从初始化以来，Receiver通过处理TLP，释放出的所有shared** **FC credits之和。**注意，由于DLLP中的[Field Size]影响，该值需要，即当单调增加的指针达到顶端后再绕下来。
- 同样的，在initerface初始化时，SCL的值没有定义
- 在Flow Control初始化时，根据以下情况进行初始化：

**SUM_SHARED_CREDIT_LIMIT**
## FC Information Tracked by Reciver