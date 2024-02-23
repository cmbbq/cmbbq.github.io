+++
title = "Tech Talk: Wall is Coming"
date = "2024-02-22"
tags = ["sys", "talk"]
description = "Tech Talk文稿，梳理内存墙问题的历史渊源，尝试给出对优化空间的理解，推导出相应的启发性策略，并列举一些访存优化技术。"
showFullContent = false
+++

# 内存墙和登纳德定律失效
“内存墙（the Memory Wall）”和“登纳德定律失效（the break down of Dennard Scaling）”是计算生态演化中的两个核心矛盾。
- 为了对抗Dennard Scaling的失效，计算硬件的架构从单核向多核、众核突围。
- 为了掩盖内存墙问题，内存层级（memory hierarchy）变得越来越深，off-chip互连带宽不得不迅速提高。

起初单核到多核是巨变，逼迫软件架构进行痛苦重构，并发编程问题木秀于林，因此被近20年的工业界实践+学术界研究集火秒了——如今我们有多线程编程范式、异步回调范式、goroutine式的有栈协程、C++/Rust的async/await无栈协程、lockfree/waitfree数据结构等诸多工具。
相较而言，这几十年来内存墙问题由于其隐蔽性、不紧急性、棘手性，不仅没得到妥善解决，反而根深蒂固，愈发遮掩不住，暴露在软件工程师面前，因此这次分享的重点是内存墙。

不过在进入正题之前，还需先介绍一下数据中心硬件和微处理器架构演化的些许背景。

# 21世纪数据中心硬件的演化简史
## 00s，Commodity Computing时代
远古时期我们并不会说“数据中心硬件”，因为还不存在现代意义上的互联网产业，也就没有现代意义上的数据中心。自然也就没有“for 大数据/云/Edge/AI”的营销话术，更多的是强调用廉价、不可靠、大规模的commodity hardware搭建分布式有状态系统，Google在这方面做了开创性的尝试。

> 相对IBM mainframe、super computer而言，当年x86的廉价是一目了然的。Google草创之初用的是奔腾2。
> 银行业、DAPRA成就了IBM power系列，如今Power9/10机器虽然被Xeon/EPYC完爆，仍然可以靠银行软件的祖宗之法不可变和政府订单苟延残喘，保有一份niche市场。

![commodity_computing](https://cmbbq.github.io/img/commodity_computing.png)

10s之前是x86微处理器完全不具备多核可扩展性的年代。
- `FSB`(front-side bus)是罪魁祸首。下图架构中，内存总线和PCIe总线需共享一个`FSB`才能与CPU相连，导致`FSB`成为瓶颈，CPU数量不具备横向扩展性。
- 当年的`PCIe`还是1.0（03年初代`PCIe`），带宽和lane数都相当有限，即使强上多核，网络IO、磁盘IO也跟不上。
![fsb](https://cmbbq.github.io/img/fsb.png)

零零年代恰逢摩尔定律逐渐在单核语境失效，专注提升芯片性能在2004年后不再可行，硬件厂商不得不在架构上向多核方向突围。
![clock](https://cmbbq.github.io/img/clock.png)

## 一零年代初，多核时代
2012年的Intel Xeon E5-2600 V1 32nm Sandy Bridge是里程碑式的服务器产品，移除`FSB`（这个实际上在09年的Nehalem机器上就已经做了）、引入`QPI`/`DMI`取代`FSB`、PCIe2.0，使微架构获取多核可扩展性。
> E5 family算是耳熟能详，虽然普遍要到寿命极限，但至今应该仍有很多公司在用。
> 当年买电脑时，所谓二代i3/i5/i7就是`SandyBridge`，和一代i5/i7的`Nehalem`有代差。

`Sandy Bridge`之后是`Haswell`，变化不大。`Haswell`之后是`Broadwell`，环状拓扑`Broadwell Ring`即得名于此。
![clock](https://cmbbq.github.io/img/broadwell_ring.png)
再后面就是Skylake，开始冠上Scalable之名了，从多核走向众核，从近代走到现代。

## 一零年代末~二零年代初，众核时代
17年Intel推出了1st gen Xeon Scalable，`Skylake`，采用了Mesh Architecture。见Things are getting meshy
同时期AMD也推出了ENYC 7001，算是打破了Xeon的垄断局面。在US-TTP机房我们就有不少AMD机器。

`Skylake`的升级版`Ice Lake`并未顺利孵化，因为18年出了Meltdown/Spectre的大新闻（speculative execution的安全漏洞），于是在`Skylake`上修了漏洞，19年推出`Cascade Lake`作为2nd-Gen Xeon。原本的顺位继承者`Ice Lake`在21年姗姗来迟，变成了第三代。

`Cooper Lake`和`Ice Lake`同代，都被称为第三代，但实际上架构和`Ice Lake`不同，是基于`Skylake`改的，专为多socket(4~8s)设计，相比general-purpose的`Ice Lake`， `Cooper Lake`稍稍超出commodity hardware范畴，估计是想卖给特定的专用计算领域，用来替代老旧的UNIX系统，比如`Oracle Solaris`，`IBM AIX`。互联网场景下我们还是倾向于横向扩容而不是纵向扩容。

目前数据中心应用的主力机型是`Ice Lake`、`Cascade Lake`机器，23~24年起计算/访存密集的场景则会逐步用到第四代Xeon：`Sapphire Rapids`机器。
19年20年我们还零零散散有一些1st gen Xeon Scalable Gold机器，后来很快就汰换掉了。
![clock](https://cmbbq.github.io/img/broadwell_ring.png)

`Ice Lake`和`Cascade Lake`都是monolithic mesh设计。
- 这里monolithic是相对于chiplet/tile-based而言的单个huge die承载many-core的范式；
- 这里mesh是相对于此前E5时代Broadwell Ring环状拓扑而言的网状拓扑。

二者差异主要在制程（10nm vs 14nm）、最大核心数、PCIe路数、内存通道数（单socket支持的DIMM[^1]数 16 vs 12）。

24年起开始交付的4th-Gen `Sapphire Rapids`的芯片架构从mono-die转型为更类似AMD的multi-die(Intel自称是tile-based），微架构从sunny cove更新到golden cove（tpause指令可用于优化spinlock），配置相当华丽，支持先进互连协议（`PCIe5`/`CXL`），支持DDR5，新指令集`AMX`，顶配还有3D堆叠的on-package HBM[^2]。

## 一些总结
### 硬件生态里，生命和环境也是相互塑造的
PC用户成就了繁荣、易获得、标准化的x86 商用硬件市场。商用硬件集群刚好又适应互联网workloads（没有大量浮点数计算，integer server为主，但数据量极为庞大），才有了分布式系统支撑的现代数据中心，再然后才有硬件厂商为数据中心定制优化的服务器硬件，比如Scalable Xeon。

PC玩家成就了Nvidia GPU，GPU恰好适应AI workloads，于是有了各种MLSys和GPGPU应用。然后才有Nvidia在加速计算方向上的投入。有了A100之后，ChatGPT训练就是在A100+IB network基础上量体裁衣做的大规模模型并行。随后又因为ChatGPT的轰动效应，反哺了高端GPU产业。

生态位很难被人为设计、凭空创造，比如Intel Optane PMem，占据了非常合理的生态位，很多系统方向的研究都是靠PMem发的论文，因为它太合理了，弥补了磁盘和内存之间的空缺。但还是因为需求侧跟不上，在22年被砍掉了。究其根本，PMem在AI训推场景下没啥用，在主流的搜广推应用上不能带来显著成本优势，在交易场景和分析型场景要么比不上磁盘+内存，要么不值得投入人力去大改架构。
### 现代硬件特征
- 核心数众多
- 设法缓解Memory Wall问题——近20年来DRAM cycle time每年缩减的速率与摩尔定律对比呈相对停滞，cache miss的后果愈发严重。
  - Memory hierarchy变深：cache变多，最新的顶配SPR机器还增加了on-package HBM，本地内存之外还有远端NUMA node，内存下面还可以有PMem、SSD，本地节点之外还有局域网/云端节点。总之，是通过增加层级隐藏Memory Wall问题。
  - Off-package/chip-to-chip互连带宽提升：跟上核心数增多带来的IO需求，缓解批量读写场景下的Memory Wall问题。
### 黑盒化和白盒化同时发生
现代硬件对工程能力提出了更苛刻的要求——通过off-CPU analysis、data-oriented cache-friendly设计、手动内存管理、甚至手动prefetching才能真正释放其性能潜力。

但这和现代软件工程愈发简化的发展方向背道而驰——通过runtime、虚拟机、动态语言、微服务范式的职责划分、基于hypervisor的虚拟化和容器化等手段解放程序员心智。

这种背道而驰使现代软件实践走向两条岔路，一个是以白盒化为手段的基础设施建设：更好的hypervisor、更好的mlsys、高性能检索、高性能存储、高性能网络，另一个是以黑盒化为目标的上层应用：利用虚拟化、容器化、微服务化、动态语言、runtime语言的便捷性提升productivity。

# 现代硬件上的性能工程实践
## 对优化空间的理解
性能工程即有系统方法论指导的软件优化实践。
可以从两个视角分解“优化任务”：
- 优化 = 减少算法总工作量 + 硬件使能
- 优化 = 减少运行时间 = 减少CPU时间 + 减少阻塞时间
“算法改良”和“硬件使能”接近正交，“on-CPU time”和“off-CPU time”又大体互补，因此可以为优化空间$W$构造正交基{硬件使能$x$，算法优化$y$，算术密度$z$}，则$W = \{[x,y,z] \in R^3 | 0 \le z \le 1 \}$。

<div id="damn"><svg width="960" height="500"></svg></div>

## 推导出的一些Heuristics
基于对优化空间完整图景的理解，能推导出一些性能工程的Heuristics：

- 应首先确定程序的算术密度
  - 为何区分on-CPU vs off-CPU至关重要？因为你的100%忙碌的CPU可能并不忙碌。CPU profiling仅描述完整图景的一部分，甚至一小部分。常用的CPU利用率这一指标具有欺骗性和迷惑性，实则既包括on-CPU计算也包括off-CPU阻塞。如果存在严重访存瓶颈，100%的CPU的superscalar pipeline里全是stall的空泡，CPU的各类算术逻辑单元、SIMD/AMX等专用计算硬件都在空等。
  - 当然off-CPU过高也可能是磁盘/网络IO密集导致的，但这类IO-bound应用往往不是成本大头，还不足以兴师动众地做优化，除非是专门做存储或专门搞网络的infra部门才需要关心。此外还有可能是代码写得有问题，锁粒度太大，过度持有锁，同样造成off-CPU 比例过高。
- 如今的内存应被视为外设
  - 曾经为慢速外设准备的数据结构，如今适合内存场景：
    - C++的有序map是用红黑树实现的，Rust选择了B+树，以便利用其更好的locality，因为如今的内存已经和几十年前的磁盘一样慢得不可容忍了。
    - DashTable这种原本用在PMem（非易适性内存，内存带宽远低于DRAM）上的数据结构，如今被DragonFly拿来用在内存数据库上，性能远超Redis/Memcached。
  - 因此需将内存层级白盒化，充分发挥硬件潜力。
- 避免和编译器优化撞车
  - 不需要用移位操作或其他汇编指令优化乘除法。乘以8必然会被自动优化成左移动3。除法也会被优化成乘加。下面的例子是非常古老的编译器对 volatile int y = x / 71的处理。编译器优化乘法还会用LEA指令，LEA原本旨在加速小结构体数组的成员地址计算，但实际上也能用来加速乘法，比如乘以5可以写成lea eax, [eax*4 + eax]，利用现成的电路就比较快，算是硬件使能领域里编译器帮你做好的工作。
    ```assembly
    // volatile int y = x / 71;8b 0c 24        mov ecx, DWORD PTR _x$[esp+8] ; load x into ecx
    mov eax, -423447479 ; magic happens starting here...
    imul ecx            ; edx:eax = x * 0xe6c2b44903 d1           add edx, ecx        ; edx = x + edx
    sar edx, 6          ; edx >>= 6 (with sign fill)

    mov eax, edx        ; eax = edx
    shr eax, 31         ; eax >>= 31 (no sign fill)
    add eax, edx        ; eax += edx

    mov DWORD PTR _y$[esp+8], eax
    ```
  
  - 可以放心去做手动SIMD。手动SIMD和LLVM自动向量化优化位于不同抽象层次，基本上也不必对LLVM的自动向量化抱有期望。短期内见不到程序语言和库层面对SIMD进行良好抽象的希望。
    - 手动SIMD需要程序员根据目标机器微架构型号选择合适的指令（不同微架构的各种simd指令的latency各不相同）；
    - 选择恰当的simd size(并非越大越好，而且不同size对应不同的shuffle)；
    - 还要处理最后不满一个batch的数据导致的各种各样的corner cases；
    - 对数据地址进行对齐或对不对齐数据进行容忍。
  - 用最新的编译器，用O3，很多细节就不必手动优化，如Copy Ellison，Tail-recursion Elimination，甚至Mutually recursion Elimination，Inlining，多数Loop优化——Loop unrolling(+步长)/fission(逆fusion)/tiling(cache blocking)/unswitching(内层去分支化)/自动向量化/interchange。
  - 涉及到逻辑和具体应用场景，没有标准优化策略的优化仍需自己做，比如Loop fusion，调整递归粒度，调整编码策略（紧凑程度和编解码开销的tradeoff）。


## 访存优化的一些Tricks
- 如何度量算术密度或访存密度?
  - perf 或 perf_event_open: cache_miss/instructions, 或ipc
  - Intel PCM
  - eBPF tools，比如https://github.com/iovisor/bcc
  - 静态分析：load/store指令占比可粗略翻译算术/访存密度，但受缓存命中率影响较大。
- 如何根据内存配置设置合适的object padding？
  - 内存配置中有两个影响应用层的重要概念：内存通道（memory channels）、内存列（memory ranks）。x86架构下内存通道和内存列在内存地址上interleaving，即均匀分布且递增。因此RAM可视为$n_chan \times n_rank$个block组成。其DIMM架构如下图。
[图片]
  - 具体的object padding方式可以参考下面这段伪码，其中64B是cache line size，也恰好是一个block的size。大体思路是先保证内存池里的地址都是64B的整数倍，再保证下一个对象的block id和n_chan*n_rank互质。
  - 这个padding不让对象的起始地址反复命中同一个channel或同一个rank，令下一个对象起始地址落入不同的channel/rank，充分利用不同内存通道、不同内存列，避免通道、列之间的负载不均，提升访存带宽。
    ```C  
    static unsigned int object_align(unsigned int obj_size)
    {
            unsigned nchan = get_nchannel();
            unsigned nrank = get_nrank();
            unsigned new_obj_size = (obj_size + 63) / 64; 
            while (get_gcd(new_obj_size, nrank * nchan) != 1)
                    new_obj_size++;
            return new_obj_size * 64; 
    }
    ```
- Cache优化：Cache line对齐避免false sharing；利用cache blocking。
- 规避锁瓶颈
  - 基于静态锁分析或ebpf off-CPU分析，找到过分粗粒度的锁和过度超期持有的锁。
  - 寻找更好的并发数据结构：无锁实现良莠不齐；Many-core scalability最好的永远是array。
  - Kernel bypassing：规避某些带锁的内核实现，如用户态网络协议栈替代内核栈。
  - Share-nothing：最极端的做法可以效仿seastar，每个核心只在自己的专用内存上执行单线程代码，尽量避免CPU-to-CPU traffic，将锁彻底从代码里消除。
- In-register storage：尽量保证简单函数用到的参数、存放中间产物的容器足够小，可完全用寄存器存下，编译器自动就会把所有load/store优化掉。
- 考虑使用预分配内存和栈上静态结构：不得不访存时，生命周期较长的对象可考虑使用预分配内存，临时容器可以考虑根据线上数据分布设计成按某种规则切分的静态结构，并放在栈上（栈内存分配只是栈指针移动，而malloc复杂的多，如果说默认版本的malloc还有全局锁，不够scalable）。
- 考虑使用编译期evaluation和全局静态内存区。
- 利用1GB huge pages存数据，相比4KB默认页，hugepage所需的page table entry总数大大减少，可显著减小page table size和tlb size、降低tlb miss和page table walk开销，提升内存分配的连续性和内存访问的局部性，这些都有助于提升内存带宽。
- 利用4MB large pages存代码，.text segment也可以用更大的页（不过最大只支持4MB），ITLB和DTLB一样，miss也会造成stall。ITLB问题的诊断可借助https://github.com/intel/iodlr，解决方法把.text移动到已有的large pages里[^3]或静态链接并使用libhugetlbfs库。目前尚未看到该优化在真实项目中落地，但看起来相当promising，可参考[Runtime Performance Optimization Blueprint: Large Code Pages](https://www.intel.com/content/dam/develop/external/us/en/documents/runtimeperformanceoptimizationblueprint-largecodepages-q1update.pdf)。
- 尊重NUMA拓扑，避免远端内存访问，即UPI traffic。
![numa](https://cmbbq.github.io/img/NUMA.png)
- 利用新架构特性和新的指令集扩展，如基于AMX实现GEMM的精度和性能优化广泛用在各种训推框架，AVX512_IFMA指令扩展做大数乘法已被用在新版本的OpenSSL中，以及基于QAT（QuickAssist）对AES、RSA、ECC等密码学应用做硬件加速。
- 利用prefetching让cache变得更聪明。
  - 所谓hardware prefetching就是从内存预取数据到cache（通常是LLC）。Hardware prefetcher有简单的stride pattern识别逻辑，比如a,a+2,a+4,a+6这种loop是可以被识别的。没必要刻意触发hardware prefetching，正常的代码都可以触发。但需避免误触发hardware prefetching——比如一个横跨多个cache line的大结构体其实只需要访问它的前几个field，但hardware prefetcher误以为还要接着读，就会造成cache pollution。稍微调整下这几个fields的访问顺序，破坏掉constant stride pattern即可。
  - 找到合适的timing使用software prefetching，预取数据会带来巨大吞吐性能提升，也能用来隐藏latency，但究竟何时预取，预取哪些数据只能靠反复尝试。而一旦timing选错了反而造成cache污染，导致性能下降。
    - 尽量让prefetch指令分散（最好和load也分散），夹杂在计算指令中间。如果连续prefetch，那和一堆load一样，也会造成空泡。
    - 选择合适的PSD(prefetch scheduling distance)，即提前几个iteration预取。对计算量大的loop，可以提前1个iteration，对于计算量小的，可能要提前多个iteration。下面这个例子PSD=3。
    ```assembly    
    top_loop:
    prefetchnta [edx + esi + 128*3]
    prefetchnta [edx*4 + esi + 128*3]
    movaps xmm1, [edx + esi] 
    movaps xmm2, [edx*4 + esi]
    movaps xmm3, [edx + esi + 16] 
    movaps xmm4, [edx*4 + esi + 16]
    add esi, 128 
    cmp esi, ecx 
    jl top_loop
    ```

    - 双层循环时，需注意弥补内外循环切换时的空泡，为外层循环也做prefetch。

    ```C    
    for (i = 0; i < 100; i++) {  
    for (j = 0; j < 32; j+=8) { 
        prefetch a[i][j+8]  // 最后一次iteration，不需要prefetch
        computation a[i][j] // 第一次a[i+1][j]未预取，会miss
    } 
    }
    // 优化后
    for (i = 0; i < 100; i++) {  
    for (j = 0; j < 24; j+=8) {
        prefetch a[i][j+8]  
        computation a[i][j]  
    }  
    prefetch a[i+1][0]  // 提前准备好 a[i][0]，否则会在a[i][j+8]阻塞
    computation a[i][j] // 最后一个iteration单独处理，因为它不需要prefetch。
    }
    ```
- 利用先进互连技术，如`CXL`。

![cxl](https://cmbbq.github.io/img/spr-cxl.png)


[^1]: DIMM(dual in-line memory module)，即ram stick，内存条，DDR(Double Data Rate)技术的物理具现。
[^2]: https://www.intel.com/content/www/us/en/products/sku/232592/intel-xeon-cpu-max-9480-processor-112-5m-cache-1-90-ghz/specifications.html
[^3]: https://github.com/intel/iodlr/blob/master/large_page-c/large_page.c

<style type="text/css">
svg {
    box-shadow: 0 0 10px #999;
    border-radius: 5px;
}
</style>
<script type="module">
import {
  drag,
  color,
  select,
  range,
  randomUniform,
  randomNormal,
  scaleOrdinal,
  selectAll,
  schemePastel1,
} from "https://cdn.skypack.dev/d3@7.8.5";
import {
    gridPlanes3D,
    points3D,
    lineStrips3D,
} from "https://cdn.skypack.dev/d3-3d@1.0.0";
document.addEventListener("DOMContentLoaded", () => {
    console.log("dom loaded, starts to draw svg ...");
    const origin = { x: 480, y: 250 };
    const j = 10;
    const scale = 20;
    const key = (d) => d.id;
    const startAngle = Math.PI/2;
    // const startAngle = 0;
    const colorScale = scaleOrdinal(schemePastel1);
    let scatter = [];
    let yLine = [];
    let xLine = [];
    let zLine = [];
    let xGrid = [];
    let beta = 0;
    let alpha = 0;
    let mx, my, mouseX = 0, mouseY = 0;
    const svg = select("svg")
        .call(
          drag()
            .on("drag", dragged)
            .on("start", dragStart)
            .on("end", dragEnd)
        )
        .append("g");
    const grid3d = gridPlanes3D()
        .rows(20)
        .origin(origin)
        .rotateY(startAngle)
        .rotateX(-startAngle)
        .scale(scale);
  const points3d = points3D()
    .origin(origin)
    .rotateY(startAngle)
    .rotateX(-startAngle)
    .scale(scale);
  const yScale3d = lineStrips3D()
      .origin(origin)
      .rotateY(startAngle)
      .rotateX(-startAngle)
      .scale(scale);
  const xScale3d = lineStrips3D()
      .origin(origin)
      .rotateY(startAngle)
      .rotateX(-startAngle)
      .scale(scale);
  const zScale3d = lineStrips3D()
      .origin(origin)
      .rotateY(startAngle)
      .rotateX(-startAngle)
      .scale(scale);
  function processData(data, tt, recolor) {
    /* ----------- GRID ----------- */
    const xGrid = svg.selectAll("path.grid").data(data[0], key);
    xGrid
      .enter()
      .append("path")
      .attr("class", "d3-3d grid")
      .merge(xGrid)
      .attr("stroke", "black")
      .attr("stroke-width", 0.3)
      .attr("fill", (d) => (d.ccw ? "#eee" : "#aaa"))
      .attr("fill-opacity", 0.7)
      .attr("d", grid3d.draw);
    xGrid.exit().remove();
    /* ----------- POINTS ----------- */
    const points = svg.selectAll("circle").data(data[1], key);
    function GetColor(x, y){
      // console.log("x: %d, y: %d", x, y);
      // return (x > 0) ? 5 : -5 + (y > 0) ? 3 : -3;
      if (x >= 0 && y >= 0) return schemePastel1[0];
      if (x < 0 && y >= 0) return schemePastel1[1];
      if (x < 0 && y < 0) return schemePastel1[2];
      if (x >= 0 && y < 0) return schemePastel1[3];
    }
    if(recolor){
      points
      .enter()
      .append("circle")
      .attr("class", "d3-3d")
      .attr("opacity", 0)
      .attr("cx", posPointX)
      .attr("cy", posPointY)
      .merge(points)
      .transition()
      .duration(tt)
      .attr("r", 3)
      .attr("stroke", (d) => color(colorScale(d.id)).darker(3))
      .attr("fill", (d) => GetColor(d.projected.x - 480, d.projected.y - 250))
      .attr("opacity", 1)
      .attr("cx", posPointX)
      .attr("cy", posPointY);
    }else{
      points
      .enter()
      .append("circle")
      .attr("class", "d3-3d")
      .attr("opacity", 0)
      .attr("cx", posPointX)
      .attr("cy", posPointY)
      .merge(points)
      .transition()
      .duration(tt)
      .attr("r", 3)
      .attr("stroke", (d) => color(colorScale(d.id)).darker(3))
      .attr("opacity", 1)
      .attr("cx", posPointX)
      .attr("cy", posPointY);
    }
    points.exit().remove();
    /* ----------- x-Scale ----------- */
    const xScale = svg.selectAll("path.xScale").data(data[3]);
    xScale
      .enter()
      .append("path")
      .attr("class", "d3-3d xScale")
      .merge(xScale)
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", xScale3d.draw);
    xScale.exit().remove();
    /* ----------- y-Scale ----------- */
    const yScale = svg.selectAll("path.yScale").data(data[2]);
    yScale
      .enter()
      .append("path")
      .attr("class", "d3-3d yScale")
      .merge(yScale)
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", yScale3d.draw);
    yScale.exit().remove();
    /* ----------- z-Scale ----------- */
    const zScale = svg.selectAll("path.zScale").data(data[4]);
    zScale
      .enter()
      .append("path")
      .attr("class", "d3-3d zScale")
      .merge(zScale)
      .attr("stroke", "black")
      .attr("stroke-width", 1.5)
      .attr("d", zScale3d.draw);
    zScale.exit().remove();
    /* ----------- y-Scale Text ----------- */
    const yText = svg.selectAll("text.yText").data(data[2][0]);
    function GetYText(y){
      if (y==-11){
        return "[Arithmetic Intensity]";
      }else{
        return  (-y*10 + 100)/2+"%";
      }
    }
    function GetYWeight(y){
      if (y==-11){
        return 700;
      }else{
        return  350;
      }
    }
    yText
      .enter()
      .append("text")
      .attr("class", "d3-3d yText")
      .attr("font-family", "system-ui, sans-serif")
      .merge(yText)
      .each(function (d) {
        d.centroid = { x: d.rotated.x, y: d.rotated.y, z: d.rotated.z };
      })
      .attr("x", (d) => d.projected.x)
      .attr("y", (d) => d.projected.y)
      .style("font-weight", (d) => GetYWeight(d.y))
      .text((d) => GetYText(d.y))
      .attr("fill", "#78E2A0");
    yText.exit().remove();
    /* ----------- x-Scale Text ----------- */
    const xText = svg.selectAll("text.xText").data(data[3][0]);
    xText
      .enter()
      .append("text")
      .attr("class", "d3-3d xText")
      .attr("font-family", "system-ui, sans-serif")
      .merge(xText)
      .each(function (d) {
        d.centroid = { x: d.rotated.x, y: d.rotated.y, z: d.rotated.z };
      })
      .attr("x", (d) => d.projected.x)
      .attr("y", (d) => d.projected.y)
      .attr("z", (d) => d.projected.z)
      .text((d) =>  d.x == 10 ? "[Hardware Enablement]" : "")
      .style("font-weight", 700)
      .attr("fill", "#78E2A0");
    xText.exit().remove();
    /* ----------- x-Scale Text ----------- */
    const zText = svg.selectAll("text.zText").data(data[4][0]);
    zText
      .enter()
      .append("text")
      .attr("class", "d3-3d zText")
      .attr("font-family", "system-ui, sans-serif")
      .merge(zText)
      .each(function (d) {
        d.centroid = { x: d.rotated.x, y: d.rotated.y, z: d.rotated.z };
      })
      .attr("x", (d) => d.projected.x)
      .attr("y", (d) => d.projected.y)
      .attr("z", (d) => d.projected.z)
      .text((d) =>  d.z == 10 ? "[Work Reduction]" : "")
      .style("font-weight", 700)
      .attr("fill", "#78E2A0");
    zText.exit().remove(); 
    selectAll(".d3-3d").sort(points3d.sort);
  }
  function posPointX(d) {
    return d.projected.x;
  }
  function posPointY(d) {
    return d.projected.y;
  }
  function init() {
    xGrid = [];
    scatter = [];
    yLine = [];
    xLine = [];
    zLine = [];
    let cnt = 0; 
    for (let z = -j; z < j; z++) {
      for (let x = -j; x < j; x++) {
        xGrid.push({ x: x, y: 0, z: z}); // grid position
        scatter.push({
          x: x,
          y: randomNormal(0, 0.8)()*3,
          // y: randomUniform(9, -9)(),
          z: z,
          id: "point-" + cnt++,
        });
      }
    }
    range(-10, 12, 1).forEach((d) => {
      yLine.push({ x: 0, y: -d, z: 0 });
      xLine.push({ x: -d, y: 0, z: 0 });
      zLine.push({ x: 0, y: 0, z: -d });
    });
    const data = [
      grid3d(xGrid),
      points3d(scatter),
      yScale3d([yLine]),
      xScale3d([xLine]),
      zScale3d([zLine]),
    ];
    processData(data, 1000, true);
  }
  function dragStart(event) {
    mx = event.x;
    my = event.y;
  }
  function dragged(event) {
    beta = (event.x - mx + mouseX) * (Math.PI / 230);
    alpha = (event.y - my + mouseY) * (Math.PI / 230) * -1;
    const data = [
      grid3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)(xGrid),
      points3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)(scatter),
      yScale3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)([yLine]),
      xScale3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)([xLine]),
      zScale3d.rotateY(beta + startAngle).rotateX(alpha - startAngle)([zLine]),
    ];
    processData(data, 0, false);
  }
  function dragEnd(event) {
    mouseX = event.x - mx + mouseX;
    mouseY = event.y - my + mouseY;
  }
  selectAll("button").on("click", init);
  init();
});
</script>
