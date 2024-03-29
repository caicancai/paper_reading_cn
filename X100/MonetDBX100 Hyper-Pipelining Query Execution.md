# MonetDB/X100: Hyper-Pipelining Query Execution

## 摘要
在决策支持、OLAP和多媒体检索等计算密集型应用领域，数据库系统在现代CPU上往往只能实现较低的IPC（每周期执行次数）效率。本文首先深入研究了这种情况发生的原因，重点是TPC-H基准测试。我们对各种关系系统和MonetDB的分析使我们获得了一套设计查询处理器的新准则。本文的第二部分描述了我们的新的X100查询引擎的MonetDB系统，遵循这些准则的体系结构。从表面上看，它类似于一个经典的Volcano风格的引擎，但关键的区别是所有执行都基于向量处理的概念，这使得它具有很高的CPU效率。我们在100 GB版本的TPC-H上评估了MonetDB/X100的性能，显示其原始执行能力比以前的技术高出一到两个数量级。

## 1. 引言
现代CPU每秒可以执行大量的计算，但前提是它们可以找到足够的独立工作来利用它们的并行执行能力。在过去的十年中，硬件的发展已经显著地增加了CPU以全吞吐量运行和最小吞吐量运行之间的速度差异，现在可以很容易地达到一个数量级。

人们会期望，查询密集型数据库工作负载（如决策支持、OLAP、数据挖掘以及多媒体检索）都需要许多独立的计算，应该为现代CPU提供接近最佳IPC（每周期执行次数）效率的机会。

然而，研究表明，在这些应用领域中，数据库系统在现代CPU上的IPC效率往往很低。我们怀疑是否真的应该这样。超越（重要的）主题的缓存意识查询处理，我们详细研究了关系数据库系统如何与现代超标量CPU在查询密集型工作负载，特别是TPC-H决策支持基准。

我们从这次调查得出的主要结论是，大多数DBMS所采用的体系结构抑制编译器使用最performancecritical优化技术，导致低CPU效率。特别是，实现流行的Volcano iterator model用于流水线处理的常见方法导致每次执行元组，这导致高解释开销，并从而导致编译器隐藏CPU并行的机会。

我们还分析了我们小组开发的内存数据库系统MonetDB 1及其MIL查询语言的性能。MonetDB/MIL使用一次列的执行模型，因此不会受到一次元组解释产生的问题的影响。但是，它的全列具体化策略导致它在查询执行期间生成大型数据流。在我们的决策支持工作负载中，我们发现MonetDB/MIL受到内存带宽的严重限制，导致其CPU效率急剧下降。

因此，我们主张将the column-wise execution of MonetDB与Volcano风格的流水线提供的增量实现相结合。

我们为MonetDB系统设计并实现了一个新的查询引擎，称为X100，它采用向量化查询处理模型。除了实现高CPU效率外，MonetDB/X100还旨在向非主存（基于磁盘）数据集扩展。本文的第二部分致力于描述MonetDB/X100的体系结构，并在100 GB大小的完整TPC-H基准测试中评估其性能。

### 1.1 纲要
本文的组织结构如下。第2节介绍了现代超标量（或超流水线）CPU，涵盖了与查询评估性能最相关的问题。在第3节中，我们将TPC-H Query 1作为CPU效率的微基准进行研究，首先用于标准关系数据库系统，然后用于MonetDB，最后我们将此查询的独立手工编码实现降至最低，以获得最大可实现原始性能的基线。

第4节描述了我们新的MonetDB X100查询处理器的架构，重点是查询执行，但也概述了数据布局，索引和更新等主题。

在第5节中，我们在TPCH基准测试中对Monet系统中的MIL和X100进行了性能比较。在第7节结束之前，我们将在第6节讨论相关工作。

## 2 How CPUs Work
图1显示了过去十年中每年最快的CPU（以MHz计），以及最高的性能（两者不一定等同），以及当年生产的最先进的芯片制造技术。

CPU MHz改进的根本原因是芯片制造工艺规模的进步，通常每18个月缩小1.4倍（也就是说，摩尔定律[13]）。每缩小一个制造规模，就意味着两倍（1.4的平方）的晶体管数量和两倍的晶体管尺寸，以及1.4倍的导线距离和信号延迟。因此，人们会期望CPU MHz随着反相信号的延迟而增加，但图1显示时钟速度甚至进一步增加。这主要是通过流水线来完成的：将CPU指令的工作划分为更多的阶段。每级工作量更少意味着CPU频率可以提高。虽然1988年的Intel 80386 CPU在一个（或多个）周期内执行一条指令，但1993年的Pentium已经有了5级流水线，在1999年的Pentium III中增加到14级，而2004年的Pentium 4有31级流水线。

CPU MHz改进的根本原因是芯片制造工艺规模的进步，通常每18个月缩小1.4倍（也就是说，摩尔定律[13]）。每缩小一个制造规模，就意味着两倍（1.4的平方）的晶体管数量和两倍的晶体管尺寸，以及1.4倍的导线距离和信号延迟。因此，人们会期望CPU MHz随着反相信号的延迟而增加，但图1显示时钟速度甚至进一步增加。这主要是通过流水线来完成的：将CPU指令的工作划分为更多的阶段。每级工作量更少意味着CPU频率可以提高。虽然1988年的Intel 80386 CPU在一个（或多个）周期内执行一条指令，但1993年的Pentium已经有了5级流水线，在1999年的Pentium III中增加到14级，而2004年的Pentium 4有31级流水线。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/e46e34e0-59f7-412d-ad90-a01bfda84102)


Piplines会带来两种危险：（i）如果一个指令需要前一指令的结果，则不能将其推入紧随其后的流水线中，而必须等待直到第一指令已经通过流水线（或其相当大的一部分），以及（ii）在IF-a-THEN-b-ELSE-c分支的情况下，CPU必须预测a的计算结果是true还是false。它可能会猜测后者，并将c放入管道中，就在a之后。更进一步，当a的评估结束时，它可能会确定它猜错了（即误预测了分支），然后必须刷新流水线（丢弃其中的所有指令）并重新开始b。显然，流水线越长，被清除的指令就越多，性能损失也就越大。在数据库系统中，依赖于数据的分支，例如在选择操作符中发现的那些选择性既不是很高也不是很低的数据，是不可能预测的，并且可以显着降低查询执行速度。

此外，超标量CPU 2提供了并行执行多个指令的可能性，如果它们是独立的。也就是说，CPU不是一个，而是多个管道。每个周期，一条新的指令可以被推入每个流水线，只要它们再次独立于已经在执行的所有指令。超标量CPU可以达到> 1的IPC（每周期指令数）。图1显示，这使得实际CPU性能的增长速度快于CPU频率。

现代CPU以不同的方式进行平衡。Intel Itanium 2处理器是一个VLIW（超大指令字）处理器，具有许多并行流水线（每周期最多可执行6条指令），只有很少（7）个阶段，因此时钟速度相对较低，为1.5GHz。相比之下，Pentium 4拥有非常长的31级流水线，允许3.6GHz的时钟速度，但每个周期只能执行3条指令。无论哪种方式，要达到理论上的最大吞吐量，Itanium 2在任何时候都需要7x6 = 42个独立指令，而Pentium 4需要31x3 = 93个。这种并行性并不总是能找到的，因此许多程序使用Itanium 2的资源要比Pentium 4好得多，这就解释了为什么在基准测试中，两种CPU的性能相似，尽管时钟速度相差很大。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/18243bc2-0ad3-47bb-9f6f-503a038bf49c)


大多数编程语言不要求程序员在程序中明确指定哪些指令（或表达式）是独立的，因此，编译器优化对于实现良好的CPU利用率至关重要。最重要的技术是循环流水线，其中由数组A的所有n个独立元素上的多个相关操作F()，G()组成的操作被转换为：

F(A[0]),G(A[0]), F(A[1]),G(A[1]),.. F(A[n]),G(A[n])
into:
F(A[0]),F(A[1]),F(A[2]), G(A[0]),G(A[1]),G(A[2]), F(A[3]),..
假设F()的流水线依赖延迟是2个周期，当G(A[0])被执行时，F（A[0]）的结果刚刚变得可用。

在Itanium 2处理器的情况下，编译器的重要性甚至更强，因为编译器必须找到可以进入不同管道的指令（其他CPU在运行时使用乱序执行）。由于Itanium 2芯片不需要任何复杂的逻辑来寻找乱序执行机会，因此它可以包含更多的管道来执行真实的工作。Itanium 2还具有一个称为分支预测的功能，用于消除分支预测错误，允许并行执行THEN和ELSE块，并在条件的结果变得已知时立即丢弃其中一个结果。检测分支预测的机会也是编译器的任务。

图2显示了选择查询SELECT oid FROM table WHERE col < X ，其中X均匀随机分布在[0：100]上，我们在0和100之间改变选择性X。由于分支预测错误，像AthlonMP这样的普通CPU显示出大约50%的最坏情况行为。正如中所建议的，通过巧妙地重写代码，我们可以将分支转换为布尔计算（“谓词”变体）。这种重写变体的性能与选择性无关，但会导致更高的平均成本。有趣的是，Itanium2上的“分支”变体非常高效，而且与选择性无关，因为编译器将分支转换为硬件断言的代码。

最后，我们应该提到片上缓存对CPU吞吐量的重要性。CPU执行的所有指令中约有30%是内存加载和存储，它们访问DRAM芯片上的数据，这些芯片位于主板上离CPU几英寸远的地方。这对大约50 ns的存储器延迟施加了物理下限。对于3.6GHz的CPU来说，这个（理想的）50ns的最小延迟已经转化为180个等待周期。因此，只有当程序访问的绝大多数内存都可以在片上缓存中找到时，现代CPU才有机会以最大吞吐量运行。最近的数据库研究表明，DBMS的性能受到内存访问成本（“缓存未命中”）的严重损害，如果使用缓存敏感的数据结构，例如缓存对齐的B树[16，7]或列式数据布局，如PAX 和DSM（如MonetDB），则可以显着改善。此外，查询处理算法将其随机内存访问模式限制在适合CPU缓存的区域，例如基数分区的散列连接[18，11]，大大提高了性能。

总而言之，CPU已经成为高度复杂的设备，其中处理器的指令吞吐量可以按数量级（！）这取决于存储器加载和存储的该高速缓存命中率、分支的数量和它们是否可以被预测/断言、以及编译器和CPU平均可以检测到的独立指令的量。已经表明，商业DBMS系统中的查询执行得到的IPC仅为0.7 [6]，因此每个周期执行不到一条指令。相比之下，科学计算（例如矩阵乘法）或多媒体处理确实从现代CPU中提取了平均高达2的IPC。我们认为，数据库系统不需要执行得那么糟糕，特别是不是在大规模的分析任务，数百万的元组需要检查和表达式进行计算。这种丰富的工作包含大量的独立性，应该能够填充CPU可以提供的所有管道。因此，我们的目标是调整数据库架构，以便在可能的情况下将其暴露给编译器和CPU，从而显着提高查询处理吞吐量。

## 3 Microbenchmark: TPC-H Query 1
虽然我们通常以查询处理的CPU效率为目标，但我们首先关注表达式计算，放弃更复杂的关系操作（如join）以简化分析。我们选择TPC-H基准测试的查询1，如图3所示，这个查询是受CPU限制的，因为在我们测试的所有RDBMS上。此外，这个查询几乎不需要优化或花哨的连接实现，因为它的计划非常简单。因此，所有数据库系统都在一个公平的竞争环境中运行，并且主要暴露它们的表达式计算效率。

TPC-H基准测试运行在数据仓库上。1GB的房子，其大小可以通过缩放因子（SF）来增加。查询1是对SF* 6 M元组的行项目表的扫描，其选择几乎所有元组（SF*5.9M），并计算许多定点十进制表达式：两个列到常数减法，一个列到常数加法，三个列到列乘法，以及八个聚合（四个SUM（），三个AVG（）和一个SUM（））。聚合分组是在两个单字符列上，并且仅产生4个唯一组合，使得它可以用小哈希表高效地完成，不需要额外的I/O，甚至不需要CPU缓存未命中（用于访问哈希表）。

在下文中，我们首先在关系数据库系统上分析Query 1的性能，然后在MonetDB/MIL上分析，最后在手工编码的程序中分析。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/af844b4c-0443-4648-834d-f8979b1f3521)



### 3.1 Query 1 on Relational Database Systems
从RDBMS的早期开始，查询执行功能是通过实现物理关系代数来提供的，通常遵循火山模型的流水线处理。然而，关系代数在其参数方面具有高度的自由度。例如，即使是一个简单的ScanSelect（R，B，P）也只在查询时接收输入关系R的格式（列的数量、它们的类型和记录偏移）、布尔选择表达式B（可以是任何形式）和定义输出关系的投影表达式P（每个具有任意复杂度）的列表的全部知识。为了处理所有可能的R、b和P，DBMS实现者实际上必须实现一个表达式解释器，它可以处理任意复杂度的表达式。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/df2cb6b4-42b5-478d-b416-7999259557dd)



这种解释器的危险之一，特别是如果解释的粒度是元组，是“真实的工作”（即执行查询中找到的表达式）的成本只是总查询执行成本的一小部分。我们可以在表2中看到这种情况，表2显示了在SF=1的数据库上TPC-H Query 1的MySQL 4.1的gprof跟踪。第二列显示例程花费的总执行时间的百分比，不包括它调用的例程花费的时间（不包括）。第一列是第二列的累计和（cum）。第三列列出例程被调用的次数，而第四列和第五列显示每次调用执行的平均指令数以及实现的IPC。

第二个观察结果是与查询的计算“工作”相对应的Item操作的成本。例如，Item func plus：：val的每次加法开销为38条指令。该性能跟踪是在具有MIPS R12000 CPU 3的SGI机器上进行的，该机器每个周期可以执行三个整数或浮点指令和一个加载/存储，平均操作延迟约为5个周期。一个简单的算术运算+（double src 1，double src 2）：double在RISC指令中看起来像这样：

LOAD src1,reg1
LOAD src2,reg2
ADD reg1,reg2,reg3
STOR dst,reg3
这段代码中的限制因素是三个加载/存储指令，因此MIPS处理器每3个周期可以执行一个（double，double）。这与MySQL的#ins/Instruction-PerCycle（IPC）= 38/0.8 = 49周期的成本形成鲜明对比！这种高成本的一个解释是缺乏循环流水线。由于MySQL调用的例程每次调用只计算一个加法，而不是一个加法数组，因此编译器无法执行循环流水线。因此，加法由必须彼此等待的四个依赖指令组成。在平均指令延迟为5个周期的情况下，这解释了大约20个周期的成本。剩下的49个周期用于跳转到例程，以及推入和弹出堆栈。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/8b46fe94-e46a-47a6-b4c5-2a9834bca656)


MySQL每次执行表达式元组的策略的后果是双重的：

Item func plus::val只执行一次加法，防止编译器创建流水线循环。由于用于一个操作的指令是高度依赖的，因此必须生成空的流水线槽（暂停）以等待指令延迟，使得循环的成本变为20个周期而不是3个周期。

例程调用的成本（在20个周期的大致范围内）必须仅在一个操作上摊销，这实际上使操作成本加倍。

我们还在一个著名的商业RDBMS上测试了相同的查询（参见表1的第一行）。由于我们显然缺乏这个产品的源代码，我们不能产生一个gprof跟踪。然而，这个DBMS上的查询评估成本与MySQL非常相似。

表1的下半部分包括来自TPC网站的一些官方TPC-H查询1结果。查询1由全扫描中的计算主导，并且这与表大小成线性比例。该查询也是使用水平并行性的“并行并行”，使得并行系统上的TPC-H结果最有可能实现线性加速。因此，我们可以通过将所有时间标准化为SF=1和单个CPU来比较不同系统的吞吐量。我们还提供了所使用的各种硬件平台的SPECcpu int/float分数。我们这样做主要是为了检查我们获得的关系DBMS结果与TPC发布的结果大致相同。这使我们相信，我们在MySQL跟踪中看到的可能代表了商业RDBMS实现中发生的情况。

### 3.2 Query 1 on MonetDB/MIL
由我们小组开发的MonetDB系统以其垂直碎片的使用而闻名，按列存储表，每个列都在包含[oid，value]组合的二进制关联表（BAT）中。BAT是一个2列表，其中左列称为头，右列称为尾。MonetDB的代数查询语言是一种称为MIL的列代数.

与关系代数相比，MIL代数没有任何自由度。它的代数运算符有固定格式的固定数量的参数（所有两列表或常数）。运算符计算的表达式以及结果的形状都是固定的。例如，MIL联接（BAT[tl，te] A，BAT[te，tr] B）：BAT[tl，tr]是A的尾列和B的头列之间的等联接，对于元组的每个匹配组合，返回来自A的头值和来自B的尾值。MIL中连接A的另一列（即头部，而不是尾部）的机制是使用MIL reverse（A）操作符返回A上的视图，其中列已交换：BAT[te，tl]。这种反向操作是MonetDB中的零成本操作，它只是交换BAT内部表示中的一些指针。在MIL中，复杂表达式必须使用多个语句来执行。例如，extprice （1 - tax）变为tmp 1：= [-]（1，tax）; tmp 2：= []（extprice，tmp 1），其中[*]（）和[-]（）是将函数“映射”到整个BAT（列）的多路复用运算符。MIL以列方式执行，因为它的操作符总是消耗许多具体化的输入BAT并具体化单个输出BAT。

我们使用MonetDB/MIL SQL前端将TPC-H Query 1转换为MIL并运行它。表3显示了所有20个MIL调用，它们总共占用了超过99%的查询时间。在TPC-H查询1上，MonetDB/MIL明显比MySQL和同一台机器上的商业DBMS快，并且与公布的TPC-H分数也有竞争力（见表1）。然而，仔细检查表3可以发现，几乎所有的MIL操作符都是内存限制的，而不是CPU限制的！这是通过在SF=0.001的TPC-H数据集上运行相同的查询计划来建立的，使得行项表的所有使用的列以及所有中间结果都适合CPU缓存，从而消除任何内存流量。MonetDB/MIL的速度几乎是原来的两倍。第2列和第4列列出了各个MIL操作实现的带宽（BW）（MB/s），包括输入BAT的大小和产生的输出BAT。在SF=1时，MonetDB会停留在500 MB/s，这是该硬件上可持续的最大带宽。当纯粹在SF=0.001的CPU缓存中运行时，带宽可以达到1.5GB/s以上。对于多路复用乘法[*]（），只有500 MB/s的带宽意味着每秒20 M元组（16字节输入，8字节输出），因此在我们的1533 MHz CPU上每次乘法75个周期，这甚至比MySQL更糟糕。

因此，MIL中的一次列策略是一把双刃剑。它的优点是，MonetDB不容易出现MySQL的问题，即花费90%的查询执行时间在tupleat-a-time解释“开销”上。由于执行表达式计算的多路复用操作对整个BAT（基本上是在编译时已知布局的数组）起作用，因此编译器能够采用循环流水线，使得这些运算符实现高CPU效率，具体表现为SF=0.001的结果。

然而，我们发现以下问题与充分实现。首先，包含许多元组的复杂计算表达式的查询将为表达式中的每个函数具体化整个结果列。通常，这样的函数结果在查询结果中不是必需的，而只是作为表达式中其他函数的输入。例如，如果聚合是查询计划中最顶层的操作符，则最终的结果大小甚至可以忽略不计（例如在查询1中）。在这种情况下，MIL实现了比严格需要的更多的数据，导致其高带宽消耗。

此外，查询1以6M元组表的98%选择开始，并对剩余的590万元组执行聚合。同样，MonetDB使用六个位置连接（）来具体化select（）的相关结果列。在类似Volcano的流水线执行模型中不需要这些连接。它可以在一次通过中完成选择，计算和聚合，而不具体化任何数据。

虽然在本文中，我们专注于主存场景中的CPU效率，但我们指出，MonetDB/MIL产生的“人为”高带宽使系统更难有效地扩展到基于磁盘的问题，这仅仅是因为内存带宽往往比I/O带宽大得多（也更便宜）。维持例如1.5GB/s的数据传输将需要具有大量磁盘的真正高端RAID系统。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/4b4e34b2-0d2c-4694-ac58-2c95581f48dd)


### 3.3 Query 1: Baseline Performance
为了了解现代硬件在处理类似Query 1这样的问题时可以做些什么，我们在MonetDB中将其实现为单个用户定义函数（UDF），如图4所示。UDF仅在查询涉及的那些列中传递。在MonetDB中，这些列存储为BAT[void，T]s中的数组。也就是说，head列中的oid值从0开始密集递增。在这种情况下，MonetDB使用不存储的void（“virtual-oids”）。然后BAT采用数组的形式。我们将这些数组作为限制指针传递，这样C编译器就知道它们是不重叠的。只有这样，它才能应用循环流水线。

这个实现利用了这样一个事实，即两个单字节字符上的GROUP BY永远不会产生超过65536个组合，因此它们的组合位表示可以直接用作具有聚合结果的表的数组索引。像在MonetDB/MIL中一样，我们执行了一些常见的子表达式消除，以便可以省略一个减和三个AVG聚合。

表1显示了这个UDF实现（标记为“手工编码”）将查询评估成本降低到惊人的0.22秒。从同一个表中，您会注意到我们新的X100查询处理器（这是本文剩余部分的主题）能够达到这个手工编码实现的2倍。

## 4 X100: A Vectorized Query Processor
X100的目标是

（i）以高CPU效率执行大容量查询，
（ii）可扩展到其他应用程序领域，如数据挖掘和多媒体检索，并在可扩展性代码上实现同样的高效率
（iii）随最低存储层次结构（磁盘）的大小而扩展。
为了实现我们的目标，X100必须克服整个计算机架构中的瓶颈：

磁盘X100的ColumnBM I/O子系统面向高效的顺序数据访问。为了降低带宽需求，它使用垂直分段的数据布局，在某些情况下，通过轻量级数据压缩进行增强。

Disk X100的ColumnBM I/O子系统面向高效的顺序数据访问。为了降低带宽需求，它使用垂直分段的数据布局，在某些情况下，通过轻量级数据压缩进行增强。
RAM 与I/O类似，RAM访问通过显式的存储器到缓存和缓存到存储器例程（其包含平台特定的优化，有时包括例如SSE预取和数据移动汇编指令）来执行。在RAM中使用相同的垂直分区和均匀压缩的磁盘数据布局，以节省保存空间和带宽
Cache 我们使用基于向量化处理模型的类似火山的执行流水线。被称为“向量”的高速缓存驻留数据项的小（例如1000个值）垂直块是X100执行原语的操作单元。CPU缓存是唯一一个带宽无关紧要的地方，因此压缩（解压缩）发生在RAM和缓存之间的边界上。X100查询处理操作符应该是缓存意识的，并将巨大的数据集有效地分割成缓存块，并仅在那里执行随机数据访问。
CPU 向量化原语向编译器暴露处理元组独立于前一元组和下一元组。用于投影（表达式计算）的矢量化原语很容易做到这一点，但我们也试图为其他查询处理操作符（例如聚合）实现同样的目标。这允许编译器产生高效的循环流水线代码。为了进一步提高CPU吞吐量（主要是通过减少指令组合中的加载/存储数量），X100包含了为整个表达式子树而不是单个函数编译矢量化原语的工具。目前，这种编译是静态引导的，但它最终可能会成为优化器强制执行的运行时活动。
为了保持本文的重点，我们只概括性地描述磁盘存储问题，这也是因为ColumnBM缓冲区管理器仍在开发中。在我们的所有实验中，X100使用MonetDB作为其存储管理器（如图5所示），在这里它对内存中的BAT进行操作

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/81b1d203-edad-40de-81cf-bcbaf31e41ad)


### 4.1 Query Language
X100使用一种相当标准的关系代数作为查询语言。我们脱离了一次列MIL语言，以便关系运算符可以同时处理多个列（的向量），允许使用一个表达式产生的向量作为另一个表达式的输入，而数据在CPU缓存中。

Aggr(
Project(
Select(
    Table(lineitem),
    < (shipdate, date(’1998-09-03’))),
[ discountprice = *( -( flt(’1.0’), discount),
extendedprice) ]),
[ returnflag ],
[ sum_disc_price = sum(discountprice) ])
X100使用一种相当标准的关系代数作为查询语言。我们脱离了一次列MIL语言，以便关系运算符可以同时处理多个列（的向量），允许使用一个表达式产生的向量作为另一个表达式的输入，而数据在CPU缓存中。

第二步是Select操作符，它创建一个选择向量，填充与谓词匹配的元组的位置。然后执行Project运算符来计算最终聚合所需的表达式。请注意，“discount”和“extendedprice”列在选择期间不会修改。相反，选择向量被映射基元考虑，仅对相关元组执行计算，将结果写入输出向量中与输入向量相同的位置。此行为需要将选择向量传播到最终Aggr。在那里，计算每个元组在哈希表中的位置，然后使用此数据更新聚合结果。此外，对于散列表中的新元素，分组属性的值被保存。一旦底层操作符耗尽并且不能产生更多向量，哈希表的内容就可以作为查询结果。

### 4.1.2 X100 Algebra
图7列出了当前支持的X100代数运算符。在X100代数中，Table是一个物化的关系，而Dataflow只是由流经管道的元组组成。

Order、TopN和Select返回一个与其输入形状相同的数据流。其他操作符使用新形状定义数据流。这个代数的一些特性是Project只用于表达式计算;它不消除重复。可以使用仅具有group-by列的Aggr执行重复消除。Array运算符生成一个Dataflow，它将一个N维数组表示为一个N元关系，其中包含以列为主的维度顺序的所有有效数组索引坐标。它被MonetDB系统的RAM数组操作前端使用。

聚合由三个物理运算符支持：（i）直接聚合，（ii）散列聚合和（iii）有序聚合。如果所有组成员在源数据流中一个接一个地到达，则选择后者。直接聚合可用于小数据库，其中位表示限于已知（小）域，类似于“手工编码”解决方案中处理聚合的方式（第3.3节）。在所有其他情况下，使用哈希聚合。

X100目前只支持左深连接。默认的物理实现是一个CartProd操作符，顶部有一个Select（即嵌套循环连接）。如果X100在连接条件中检测到外键条件，并且join-index可用，则它会使用Fetch 1 Join或FetchNJoin来利用它。

在X100中包含这些fetch-joins并不是巧合。在MIL中，oid到空列的“位置连接”已被证明对存储在密集列中的垂直碎片数据很有价值。位置连接允许以高效的方式处理垂直碎片所需的“额外”连接[4]。就像MonetDB中的void类型一样，X100为每个表提供了一个虚拟的#rowId列，它只是一个从0开始的密集递增数字。Fetch 1 Join允许通过#rowId定位地获取列值。

### 4.2 Vectorized Primitives
使用按列向量布局的主要原因不是为了优化该高速缓存中的内存布局（X100无论如何都应该对缓存的数据进行操作）。相反，向量化的执行原语具有自由度低的优点（如3.2节所述）。在垂直分段数据模型中，执行原语只知道它们操作的列，而不必知道整个表的布局（例如记录偏移量）。当编译X100时，C编译器看到X100矢量化的基元对固定形状的受限（独立）数组进行操作。这允许它应用积极的循环流水线，这对现代CPU性能至关重要（见第2节）。作为一个例子，我们展示了矢量化浮点加法的（生成的）代码：

map_plus_double_col_double_col(int n,
double*__restrict__ res,
double*__restrict__ col1, double*__restrict__ col2,
int*__restrict__ sel)
{
if (sel) {
for(int j=0;j<n; j++) {
int i = sel[j];
res[i] = col1[i] + col2[i];
}
} else {
for(int i=0;i<n; i++)
res[i] = col1[i] + col2[i];
} }
sel参数可以是NULL或指向n个选定数组位置的数组（即图6中的“selectionvector”）。所有X100矢量化原语都允许传递这样的选择矢量。基本原理是，在选择之后，保持子操作符提供的向量不变通常比将所有选定数据复制到新的（连续的）向量中更快

X100包含数百个矢量化图元。这些不是手工编写（和维护）的，而是从原始模式生成的。加法的基本模式是

any::1 +(any::1 x,any::1 y) plus = x + y
此模式声明了两个相同类型的值的加法（但没有任何类型限制）在C中通过中缀运算符+实现。它产生相同类型的结果，名称标识符应该是加号。规范文件中稍后的类型特定模式可能会覆盖此模式（例如str +（str x，str y）concat = str concat（x，y））

The other part of primitive generation is a file with

map signature requests:
+(double, double) +(double, double) +(double, double) +(double, double)
这要求生成单个值和列之间的所有可能的加法组合（后者用一个额外的 * 标识）。其他可扩展的RDBMS通常只允许具有单值参数的UDF [19]。这抑制了循环流水线，降低了性能（见3.1节）。

上面的签名是Mahanalobis距离，这是一些多媒体检索任务的性能关键操作[9]。我们发现复合基元的执行速度通常是单一功能矢量化基元的两倍。请注意，这个系数2类似于MonetDB/X100和表1中TPC-H查询的手工编码实现之间的差异。复合原语更高效的原因是更好的指令组合。与3.1节中MIPS处理器上的加法示例类似，向量化执行经常成为加载/存储限制，因为对于简单的二元计算，每个向量化指令需要加载两个参数并存储一个结果（1个工作指令，3个内存指令）。现代CPU通常每个周期只能执行1或2个加载/存储操作。在复合原语中，一个计算的结果通过CPU寄存器传递到下一个计算，加载/存储只发生在表达式图的边缘。

目前，原语生成器只不过是X100系统的make序列中的一个宏扩展脚本。然而，我们打算按照优化器的要求实现复合原语的动态编译。

map基元的一个细微变化是select * 基元（参见图2）。这些仅存在于返回布尔值的代码模式中。选择基元不是生成一个完整的布尔结果向量（如map所做的那样），而是填充一个选定向量位置（整数）的结果数组，并返回选定元组的总数

类似地，还有aggr * 基元用于计算聚合，如count、sum、min和max。对于每一个，需要指定初始化、更新和结尾模式。然后，原语生成器为X100中聚合的各种实现生成相关例程

X100允许数据库扩展开发人员提供（源）代码模式而不是编译代码的机制，允许所有ADT在查询执行期间获得头等公民待遇。这也是MIL（以及大多数可扩展的DBMS [19]）的一个弱点，因为它的主要代数运算符只针对内置类型进行了优化

Data Storage
MonetDB/X100以垂直分段的形式存储所有表。无论使用新的ColumnBM缓冲区管理器还是MonetDB BAT[void，T]存储，存储方案都是相同的。虽然MonetDB将每个BAT存储在单个连续文件中，但ColumnBM将这些文件分区为大（> 1 MB）块。

垂直存储的一个缺点是增加了更新成本：单个行的更新或删除必须为每列执行一次I/O。MonetDB/X100通过将垂直片段视为不可变对象来规避这一点。更新将转到delta结构。图8显示了通过将元组ID添加到删除列表来处理删除，插入导致在单独的增量列中追加。ColumnBM实际上将所有delta列一起存储在一个块中，相当于PAX [2]。因此，这两个操作都只产生一个I/O。更新只是一个简单的删除后插入。更新使增量列增长，使得每当它们的大小超过总表大小的（小）百分位时，数据存储应该被重新组织，使得垂直存储再次更新并且增量列为空。

垂直存储的一个优点是，访问许多元组但不是所有列的查询可以节省保存带宽（这对RAM带宽和I/O带宽都适用）。我们使用轻量级压缩进一步降低带宽要求。MonetDB/X100支持枚举类型，它有效地将列存储为单字节或双字节整数。此整数表示映射表的#rowId。MonetDB/X100自动添加Fetch 1 Join操作，以便在查询中使用此类列时使用小整数检索未压缩的值。请注意，由于垂直片段是不可变的，更新只会转到delta列（从不压缩），不会使压缩方案复杂化。

MonetDB/X100还支持简单的“汇总”索引，类似于[12]，如果列是集群（几乎排序），则使用该索引。这些汇总索引包含一个#rowId，即在基表中该点之前该列的运行最大值，以及一个非常粗粒度的最小值（默认大小为1000个条目，从基表中以固定间隔获取#rowids）。这些汇总索引可用于快速导出范围谓词的#rowId边界。再次注意，由于垂直片段是不可变的，因此它们上的索引实际上不需要维护。delta列应该很小并且在内存中，但没有索引，必须始终访问。



MonetDB/X100还支持简单的“汇总”索引，类似于[12]，如果列是集群（几乎排序），则使用该索引。这些汇总索引包含一个#rowId，即在基表中该点之前该列的运行最大值，以及一个非常粗粒度的最小值（默认大小为1000个条目，从基表中以固定间隔获取#rowids）。这些汇总索引可用于快速导出范围谓词的#rowId边界。再次注意，由于垂直片段是不可变的，因此它们上的索引实际上不需要维护。delta列应该很小并且在内存中，但没有索引，必须始终访问。

![image](https://github.com/caicancai/paper_reading_cn/assets/77189278/a0a02958-dfa9-4278-b646-145ecd9385cd)

