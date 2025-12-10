---
title: 阿里云多模型混合服务系统Aegaeon分析
date: 2025-11-10
draft: false
type: posts
math: true
---
今年10月，阿里云联合北京大学在ACM计算机操作系统顶会SOSP发布了多模型混合服务系统 [Aegaeon: Effective GPU Pooling for Concurrent LLM Serving on the Market](https://link.zhihu.com/?target=https%3A//ennanzhai.github.io/pub/sosp25-aegaeon.pdf)，在阿里云的MaaS服务业务中，实现节省GPU达82%，兼具学术创新和工程成熟度。本文对Aegaeon进行了详细的技术解读与分析。认为其在模型长尾问题严重的场景，高端卡和低[SLO](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=SLO&zhida_source=entity)要求的场景可以通过[GPU pooling](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=GPU+pooling&zhida_source=entity)显著降低GPU卡量，对MaaS服务具有实用价值。  
  
阿里云的官方微信发布：[显著提升GPU利用率！阿里云AI基础设施成果入选顶会](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/3zoZY0DHzsr-tV0dJy0Vng)  
  
**Aegaeon简介**  
大模型推理MaaS服务平台往往需要为每个支持的模型至少保留一个实例，大量的小众模型往往调用量很低，带来极大资源浪费。以阿里云为例，779个在线模型，调用量最低的94%的模型，占用17.7%的GPU，服务1.35%的请求。热门模型又会突发超配额，资源不够。保留专机导致大量浪费。  
  
Aegaeon的目标是要让一块GPU尽可能的同时服务更多的不同模型，且不违反SLO（[TTFT](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=TTFT&zhida_source=entity)和[TBT](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=TBT&zhida_source=entity)）。为此构建了一个**多模型并发推理的 GPU 调度系统，通过token 粒度的抢占调度，prefill 与 decoding 分离，配合模型切换时延优化，把模型切换开销（原文叫“模型扩缩”）降低97%，达到亚秒级**。实现支持单卡最多 7 个模型并发，实测把线上所需 GPU 从 1,192 降到 213（节省 82%）。  
  
主流的推理框架，比如[vLLM](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=vLLM&zhida_source=entity)，也实现了抢占调度。区别在于，**vLLM是任务的抢占，而Aegaeon是模型的抢占**。在KV Cache吃紧或者有更高优先级的请求时，vLLM在decode阶段也有抢占机制，在decode期间之间会决策是否要逐出一些请求，当有KV Cache资源可用或者有请求结束后，再把逐出的任务调度回来，且所有的任务都是一个模型。**而Aegaeon的抢占会把一个请求的decoding分段调度，在不同的模型之间进行抢占，这两点是其和主流推理框架的区别**。  
  
下图介绍了其调度的基本逻辑，在一张GPU上运行3个模型推理，模型B的prefill不会等到模型A的decode完成才开始计算，而是会打断模型A的decode，先运行B的prefill，再把A的decode调度回来，保障SLO达标

**为什么有收益？比如一个token要求100ms的TBT，而GPU只花了25ms就完成了，空出来的75秒休闲时间，但是我不想提前交卷，而是利用这个时间再处理下一个任务，只要任务切换的时间小于75ms，就会有收益。**  
  
**总体架构解读**  
下图是Aegaeon 系统总览（论文 Figure 5）。核心的三层：  
- 代理层：请求入口和负载均衡，通过[redis](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=redis&zhida_source=entity)共享内存与推理实例同步元信息，实现负载均衡并提升容错  
- 推理实例层：PD分离，降低 HOL 阻塞风险。每个GPU上会同时加载多个模型  
- 内存管理层：统一CPU KV Cache、[VRAM Buffer](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=VRAM+Buffer&zhida_source=entity)、自管理内存与预抢占自动扩缩协同运行全链路协同  
  
当请求到达一个实例后，会通过实例上的token level scheduler进行调度。当需要切换到“非当前活跃模型”时触发扩缩容。④ Request B的Prefill被调度到Prefill Instance 1，其需要的蓝色模型没有加载，使用步骤④加载，即“模型扩容”。整个系统可用的核心在于模型切换的时间成本要足够低。

![](https://pic2.zhimg.com/v2-156ddd67c58fb1cdbccf649a152d83c1_1440w.jpg)

**关键机制一：模型切换加速（模型扩缩）**  
论文指出若不优化，缩容和再扩容一次 13B vLLM 实例可能要 几十秒，基本无法用于token级决策和模型切换。亚秒级模型切换是后续调度方案可行的前提。下图是文中的一个例子，vLLM的切换成本是26.9s，因为vLLM是为了提升单模型的运行效率优化的，因此模型切换的优化有限，可以看到，比较大的几块：模型的加载，分布式框架初始化，profiling（图右侧）。其中模型加载需要从DRAM到VRAM的数据传输，vLLM仅能实现2.83GB/s的实际带宽，远低于[PCIE-4](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=PCIE-4&zhida_source=entity)提供的32GB单向带宽。

![](https://pic4.zhimg.com/v2-0b2e126849cfb8f20f7f6a13553e8671_1440w.jpg)

**Aegaeon模型切换加速的核心机制：**  
1. **切换模型并不需要重新初始化推理框架**，省掉分布式框架初始化（[DistExec](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=DistExec&zhida_source=entity)）和profiling，以及初始化的Misc，得到了下图中的时间T1。  
2. **为了提升从DRAM到VRAM的加载速度，在DRAM中预先分配了一个pinned & page locked专用缓冲区（stage buffer），配合分块、多线程、流水化与独立 CUDA 流，能让 H2D 传输真正跑满 PCIE的带宽并与计算部分重叠**，从而显著快于“直接从paged memory随用随拷”的做法，得到下图中的T2。致此，切换时间已经降低了80%。  
3. 显式内存管理（VRAM + DRAM），自管分配（[bump allocator](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=bump+allocator&zhida_source=entity) / [slab](https://zhida.zhihu.com/search?content_id=265429614&content_type=Article&match_order=1&q=slab&zhida_source=entity)）去掉gc（garbage collection），预分配/消碎片，给权重与 KV cache 预留统一池，并预取/直加载，减少 page-lock等。结合KV cache 细粒度同步，与计算重叠，合计使模型切换开销下降 97%，从而使 token 级调度可行。  

![](https://pic1.zhimg.com/v2-b6559556c7b2a23d098b9f71ae070f08_1440w.jpg)

**模型切换速度效果实测**  
下图左，使用了模型预取之后，超过50%的模型切换可以做到几乎实时切换，scaling的时间没有被完全掩盖的情况下也以在1s内完成。KV Cache加载的开销也控制在了1s以内。

![](https://picx.zhimg.com/v2-c087911704d45c77b60546ad585686a5_1440w.jpg)

**关键机制二：Token 级自动扩缩 + 拆分调度（PD分离）**  
**为什么一定要token级而不是请求级？**  
请求级无法细腻地权衡 TTFT 与 TBT 的截止期，也无法频繁在模型间腾挪资源；长尾+突发场景下，请求级会出现严重 HOL 阻塞。  
**为什么一定要PD分离？**  
这样做可以简化调度算法。在调度模型的同时，考虑调度P和D两种任务，导致调度优化比较复杂，导致系统稳定性不足，调度开销提升，影响实际运行效果。（当然，PD分离本身也有推理性能的收益）  
  
系统会预先分配好prefill节点和decode节点，将调度问题从拆分为独立的两个子问题：prefill的调度和decode的调度。  
  
**Prefill的调度**  
基本是一个分组FCFS的策略。每个实例的任务列表是多个group，一个group中的请求都是同一个模型的请求，每个group会设置请求数量上限（一般是8个）。新来一个请求，优先放在实例最后一个当前活跃模型的group中，减少模型切换次数。如果放不进去，比如group满了，或者没有对应模型的group，就在负载最低的实例的任务列表末尾新建一个group，并把请求放入。实例的负载采用其任务列表运行总时间进行衡量。  
  
文中特意强调，采用batch size=1，因为prefill时延和处理的tokens数量线性增长。  
  
由于模型切换仍然需要0.5s左右，和小一点的prefill时间相当，所以尽量不对prefill进行驱逐，这也是为什么要使用分组FCFS策略的原因。  
  
**Decode的调度**  
**采用加权轮询（WRR），最小化违反 TBT 的 token 数**。采用和Prefill相同的方式构建任务列表。这里group里放的是同模型的推理的一些batch，batch size尽量大以用满kv cache缓存。为了减少Head-of-line时延，需要给每个batch设置单次执行时间，到时间就切换为下一个batch，就这样一直切换执行整个任务列表中的所有batch。每个batch的执行时间就是这个“加权轮询”的“权”。如果TBT是d，而实际decode的时间是t，则decode n次就可以有n(d-t)额外时延容忍额度，结合这个时间额度计算batch的运行时间片长度，尽量确保最小化违反TBT的token数量（文中提供了详细的公式）。  
  
**Token级调度的效果实测**  
下图是不同总模型数和请求速率（RPS）情况下的时延分解，基本可以看到prefill的时延控制的很好，decode的调度可以在同时服务多个模型的场景平衡等待时间和运行时间，保持SLO不超限。

![](https://pic1.zhimg.com/v2-283c95a1eece31022e77ec3589427516_1440w.jpg)

**端到端效果分析**  
在7B，32B，72B的模型混合服务场景，实现支持单卡最多 7 个模型并发，比SOTA提升1倍，实际业务中，把线上所需 GPU 从 1,192 降到 213（**节省 82%**）。 GPU利用率从13.3%（低负载）∼33.9%（高负载） 提升到48.1%。Goodput提升1.5∼9×。  
  
**支持的模型数量**：**SLO达成率对模型数量不敏感**，当一个GPU上运行的模型数量增加超过一个临界值后，每个模型等待其他模型运行的时间会增加，导致整体推理时延提升，SLO达成率开始降低。因为使用了CPU的内存缓存模型，减少GPU内存占用，可支持的最大模型数量仍然大幅优于已有的SOTA，包括MuxServer，ServerlessLLM（下图a，b）。

![](https://pic3.zhimg.com/v2-29710981015508e6c54f5ced1779f536_1440w.jpg)

**请求速率**：**请求速率更高，效果更好**。因为这时候HOL堵塞更加严重，导致SLO频繁超限，而此方法没有此问题，因此可以相比ServerlessLLM有2.5X的RPS提升（上图c）。  
  
整体上，系统对模型的数量以及请求速率都有比较高的容忍度，适用的场景比以往的算法更加宽泛。  
  
**收益随硬件与 SLO 而变**  
**类似H20的高端卡和更宽松的TBT 更易出大收益**，因为高端卡的计算资源高，更容易出现利用率不足的问题，而更宽松的TBT也给GPU pooling提供了更多的操作空间，来提升资源利用率。

![](https://pic1.zhimg.com/v2-3363b638fdef87e317df071ad900ad64_1440w.jpg)

**在低端卡或极紧 SLO 下收益会打折，但是整体上仍然可能通过提升资源利用率降低服务成本，仍然具备积极的价值。其价值断裂点在于模型切换时间是否能小于GPU总体的空闲时间。**

![](https://pic2.zhimg.com/v2-5753eeeefafa28506cad944e4d659575_1440w.jpg)

**思考**  
**Aegaeon的核心思想是GPU的分时复用**。相当于不同模型的请求合并，变成对一个虚拟模型的请求，相比独立为每个模型部署资源效率高出很多。这种思路和CPU的分时复用有些类似，需要更高的H2D带宽以及利用率，以及AI原生的内存管理替代CPU时代的自动内存管理。当前已经做到50%的模型切换几乎无时延，通过提升H2D带宽，分层传输和计算，甚至模型压缩和解压等方式，可以进一步减少切换时延。  
  
**GPU或者NPU的虚拟化可能没有必要**。CPU时代的进程隔离和任务抢占是在上下文切换极快，内存虚拟化，任务难以batching的条件下演进的，而AI任务切换太重，需要batch合并加速计算，而SLO作为外部调度约束，OS级调度不可见。**多模型切换是为了解决HBM容量不足的问题，虚拟化只能恶化HBM容量不足的问题**。GPU/NPU的虚拟化切分会将资源切分的更碎，减少了可调度的任务类型，整体资源利用率反而会降低。如果把顺丰的卡车车厢用小格子格成一堆小隔间 vs 保持一个车厢空间使用装箱算法调度箱子的放置，显然后者更具普适性，以及更高的性能上限。而Aegaeon走的是后面这条路。没有虚拟化可能会带来安全性的问题，需要后续研究跟进。  
  
**Aegaeon提供了一个普适的大量模型混合服务方法，个人模型托管成为了可能。基于这个思路构建一个个人模型托管服务可能会是一个不错的idea**。  
  
**Aegaeon适用于MaaS生态构建场景**。Aegaeon适合对SLO宽松（TBT>50ms），且模型长尾现象严重的场景，这对应的是大量微调后的模型hosting，这些场景很多都是AI应用发展初期的探索场景，有很高的成长性，可以大幅降低MaaS生态扩展的成本。  
  
**Aegaeon仍适用于大规模商用部署场景**。大规模部署往往是少数几个模型大规模部署，而且对SLO的要求比较严格，从而压缩了模型切换的必要性和收益空间。但是请求波峰波谷比较明显，资源利用率往往不高，通过实时的模型切换和token级调度，仍然可以提高资源利用率。  
  
**Aegaeon不适用于不同并行度、不同大小模型的统一部署**。这种情况需要分别部署Aegaeon实例，增加了系统复杂度和资源碎片。如果需要一个Aegaeon实例支持不同并行度实例的调度，需要支持多个通信域并存，可能会大幅增加VRAM开销。如果重新初始化推理引擎，模型切换成本将导致整个方案不可用。这可能是Aegaeon 2.0要做的事情。
