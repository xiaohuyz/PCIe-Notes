# PCIe Gen 6 中的credit block概念
在Flit Mode下，Header和Data所使用的Shared Credits是通过credit block来组织起来的。注意，credit blocks对dedicated credits不适用。
- credit blocks由4个对应type的credits组成
- credit blocks同时也不受scale factor影响

使用credit block的原因：
- shared FC机制允许多个VC之间共享同一片buffer资源，因此减少了实际资源的消耗。但是在这种结构下，TLP存储于共享的buffer中，因此追踪TLP的复杂度较高，所需的资源也比较多。例如，来自不同VC的TLP也可能被存储于同一片FC中，但实际上这些TLP是没有order关系的，它们的具体行为也有可能根本不相同。传统的resource tracking机制，采用链表来追踪每个VC中的TLP，这使得TLP的存储变得支离破碎。
- 因此我们采用基于block的TLP存储机制。在这种基于block的存储机制中，为了实现高效的block管理，同一个给定的VC中的TLP必须被打包在同一个block中。
- 如果不同VC中的TLP被打包进了同一个block，那么在block内的TLP乱序发送（因为来自不同VC的TLP之间没有order关系），以及对应space的乱序释放会变得十分困难。

因此对credit block进行如下限制：
- 在每个VC中，对于每一种FC Type，所有的shared credits都必须以credit blocks为单位进行消费和释放。
- 当一个TLP并没有消耗完一个credit block中的所有资源时，该credit block中的其他credit只能被后续来自同一个VC的，使用同一种FC Type的TLP消耗 
  - 无论是否启用[Merged] FC，credit block的分配必须区分Posted FC Type和Completion FC Type
  - 当一个credit block被分配使用时，该block的分配状态必须保持打开，直到： 	
    - 后续的，来自同一VC，使用同一FC Type的TLP完全消耗量该credit block中的资源
    - 该credit block所对应的VC被软件disable
    - 整个device进入DL_Down状态
  - 当一块credit block被分配使用时，它必须仅仅能被来自同一VC，使用同一FC Type的TLP所消耗
- receiver必须以credit block为单位来advertise它的credits

Spec中对credit block的分配举了如下的例子：如图所示，根据发送时间，先后到达6笔TLP，即A-Z
1. A来自VCx，且有PH和PD，因此分配两个credit block来保存
2. B来自VCx，但是其中是CplH和CplD，因此分配两个新的credit block来保存
3. C来自VCx，且同样是PH和PD，因此它可以使用一开始为A分配的，没使用完的credit block来保存
4. D虽然也是PH和PD，但它来自VCy，因此分配两个新的credit block来保存
5. E虽然也是CplH和CplD，但它来自VCy，因此分配两个新的credit block来保存
6. F来自VCy，且同样是CplH和CplD，因此它使用一开始为E分配的，没有使用完的credit block来保存