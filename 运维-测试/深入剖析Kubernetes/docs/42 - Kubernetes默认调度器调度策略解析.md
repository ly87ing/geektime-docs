你好，我是张磊。今天我和你分享的主题是：Kubernetes默认调度器调度策略解析。

在上一篇文章中，我主要为你讲解了Kubernetes默认调度器的设计原理和架构。在今天这篇文章中，我们就专注在调度过程中Predicates和Priorities这两个调度策略主要发生作用的阶段。

首先，我们一起看看Predicates。

**Predicates在调度过程中的作用，可以理解为Filter**，即：它按照调度策略，从当前集群的所有节点中，“过滤”出一系列符合条件的节点。这些节点，都是可以运行待调度Pod的宿主机。

而在Kubernetes中，默认的调度策略有如下四种。

**第一种类型，叫作GeneralPredicates。**

顾名思义，这一组过滤规则，负责的是最基础的调度策略。比如，PodFitsResources计算的就是宿主机的CPU和内存资源等是否够用。

当然，我在前面已经提到过，PodFitsResources检查的只是 Pod 的 requests 字段。需要注意的是，Kubernetes 的调度器并没有为 GPU 等硬件资源定义具体的资源类型，而是统一用一种名叫 Extended Resource的、Key-Value 格式的扩展字段来描述的。比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo
spec:
  containers:
  - name: extended-resource-demo-ctr
    image: nginx
    resources:
      requests:
        alpha.kubernetes.io/nvidia-gpu: 2
      limits:
        alpha.kubernetes.io/nvidia-gpu: 2
```

可以看到，我们这个 Pod 通过`alpha.kubernetes.io/nvidia-gpu=2`这样的定义方式，声明使用了两个 NVIDIA 类型的 GPU。

而在PodFitsResources里面，调度器其实并不知道这个字段 Key 的含义是 GPU，而是直接使用后面的 Value 进行计算。当然，在 Node 的Capacity字段里，你也得相应地加上这台宿主机上 GPU的总数，比如：`alpha.kubernetes.io/nvidia-gpu=4`。这些流程，我在后面讲解 Device Plugin 的时候会详细介绍。

而PodFitsHost检查的是，宿主机的名字是否跟Pod的spec.nodeName一致。

PodFitsHostPorts检查的是，Pod申请的宿主机端口（spec.nodePort）是不是跟已经被使用的端口有冲突。

PodMatchNodeSelector检查的是，Pod的nodeSelector或者nodeAffinity指定的节点，是否与待考察节点匹配，等等。

可以看到，像上面这样一组GeneralPredicates，正是Kubernetes考察一个Pod能不能运行在一个Node上最基本的过滤条件。所以，GeneralPredicates也会被其他组件（比如kubelet）直接调用。

我在上一篇文章中已经提到过，kubelet在启动Pod前，会执行一个Admit操作来进行二次确认。这里二次确认的规则，就是执行一遍GeneralPredicates。

**第二种类型，是与Volume相关的过滤规则。**

这一组过滤规则，负责的是跟容器持久化Volume相关的调度策略。

其中，NoDiskConflict检查的条件，是多个Pod声明挂载的持久化Volume是否有冲突。比如，AWS EBS类型的Volume，是不允许被两个Pod同时使用的。所以，当一个名叫A的EBS Volume已经被挂载在了某个节点上时，另一个同样声明使用这个A Volume的Pod，就不能被调度到这个节点上了。

而MaxPDVolumeCountPredicate检查的条件，则是一个节点上某种类型的持久化Volume是不是已经超过了一定数目，如果是的话，那么声明使用该类型持久化Volume的Pod就不能再调度到这个节点了。

而VolumeZonePredicate，则是检查持久化Volume的Zone（高可用域）标签，是否与待考察节点的Zone标签相匹配。

此外，这里还有一个叫作VolumeBindingPredicate的规则。它负责检查的，是该Pod对应的PV的nodeAffinity字段，是否跟某个节点的标签相匹配。

在前面的第29篇文章[《PV、PVC体系是不是多此一举？从本地持久化卷谈起》](https://time.geekbang.org/column/article/42819)中，我曾经为你讲解过，Local Persistent Volume（本地持久化卷），必须使用nodeAffinity来跟某个具体的节点绑定。这其实也就意味着，在Predicates阶段，Kubernetes就必须能够根据Pod的Volume属性来进行调度。

此外，如果该Pod的PVC还没有跟具体的PV绑定的话，调度器还要负责检查所有待绑定PV，当有可用的PV存在并且该PV的nodeAffinity与待考察节点一致时，这条规则才会返回“成功”。比如下面这个例子：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - my-node
```

可以看到，这个 PV 对应的持久化目录，只会出现在名叫 my-node 的宿主机上。所以，任何一个通过 PVC 使用这个 PV 的 Pod，都必须被调度到 my-node 上才可以正常工作。VolumeBindingPredicate，正是调度器里完成这个决策的位置。

**第三种类型，是宿主机相关的过滤规则。**

这一组规则，主要考察待调度 Pod 是否满足 Node 本身的某些条件。

比如，PodToleratesNodeTaints，负责检查的就是我们前面经常用到的Node 的“污点”机制。只有当 Pod 的 Toleration 字段与 Node 的 Taint 字段能够匹配的时候，这个 Pod 才能被调度到该节点上。

> 备注：这里，你也可以再回顾下第21篇文章[《容器化守护进程的意义：DaemonSet》](https://time.geekbang.org/column/article/41366)中的相关内容。

而NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足，如果是的话，那么待调度 Pod 就不能被调度到该节点上。

**第四种类型，是 Pod 相关的过滤规则。**

这一组规则，跟 GeneralPredicates大多数是重合的。而比较特殊的，是PodAffinityPredicate。这个规则的作用，是检查待调度 Pod 与 Node 上的已有Pod 之间的亲密（affinity）和反亲密（anti-affinity）关系。比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-antiaffinity
spec:
  affinity:
    podAntiAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution: 
      - weight: 100  
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security 
              operator: In 
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: docker.io/ocpqe/hello-pod
```

这个例子里的podAntiAffinity规则，就指定了这个 Pod 不希望跟任何携带了 security=S2 标签的 Pod 存在于同一个 Node 上。需要注意的是，PodAffinityPredicate是有作用域的，比如上面这条规则，就仅对携带了Key 是`kubernetes.io/hostname`标签的 Node 有效。这正是topologyKey这个关键词的作用。

而与podAntiAffinity相反的，就是podAffinity，比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution: 
      - labelSelector:
          matchExpressions:
          - key: security 
            operator: In 
            values:
            - S1 
        topologyKey: failure-domain.beta.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: docker.io/ocpqe/hello-pod
```

这个例子里的 Pod，就只会被调度到已经有携带了 security=S1标签的 Pod 运行的 Node 上。而这条规则的作用域，则是所有携带 Key 是`failure-domain.beta.kubernetes.io/zone`标签的 Node。

此外，上面这两个例子里的requiredDuringSchedulingIgnoredDuringExecution字段的含义是：这条规则必须在Pod 调度时进行检查（requiredDuringScheduling）；但是如果是已经在运行的Pod 发生变化，比如 Label 被修改，造成了该 Pod 不再适合运行在这个 Node 上的时候，Kubernetes 不会进行主动修正（IgnoredDuringExecution）。

上面这四种类型的Predicates，就构成了调度器确定一个 Node 可以运行待调度 Pod 的基本策略。

**在具体执行的时候， 当开始调度一个 Pod 时，Kubernetes 调度器会同时启动16个Goroutine，来并发地为集群里的所有Node 计算 Predicates，最后返回可以运行这个 Pod 的宿主机列表。**

需要注意的是，在为每个 Node 执行 Predicates 时，调度器会按照固定的顺序来进行检查。这个顺序，是按照 Predicates 本身的含义来确定的。比如，宿主机相关的Predicates 会被放在相对靠前的位置进行检查。要不然的话，在一台资源已经严重不足的宿主机上，上来就开始计算 PodAffinityPredicate，是没有实际意义的。

接下来，我们再来看一下 Priorities。

在 Predicates 阶段完成了节点的“过滤”之后，Priorities 阶段的工作就是为这些节点打分。这里打分的范围是0-10分，得分最高的节点就是最后被 Pod 绑定的最佳节点。

Priorities 里最常用到的一个打分规则，是LeastRequestedPriority。它的计算方法，可以简单地总结为如下所示的公式：

```
score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2
```

可以看到，这个算法实际上就是在选择空闲资源（CPU 和 Memory）最多的宿主机。

而与LeastRequestedPriority一起发挥作用的，还有BalancedResourceAllocation。它的计算公式如下所示：

```
score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10
```

其中，每种资源的 Fraction 的定义是 ：Pod 请求的资源/节点上的可用资源。而 variance 算法的作用，则是计算每两种资源 Fraction 之间的“距离”。而最后选择的，则是资源 Fraction 差距最小的节点。

所以说，BalancedResourceAllocation选择的，其实是调度完成后，所有节点里各种资源分配最均衡的那个节点，从而避免一个节点上 CPU 被大量分配、而 Memory 大量剩余的情况。

此外，还有NodeAffinityPriority、TaintTolerationPriority和InterPodAffinityPriority这三种 Priority。顾名思义，它们与前面的PodMatchNodeSelector、PodToleratesNodeTaints和 PodAffinityPredicate这三个 Predicate 的含义和计算方法是类似的。但是作为 Priority，一个 Node 满足上述规则的字段数目越多，它的得分就会越高。

在默认 Priorities 里，还有一个叫作ImageLocalityPriority的策略。它是在 Kubernetes v1.12里新开启的调度规则，即：如果待调度 Pod 需要使用的镜像很大，并且已经存在于某些 Node 上，那么这些Node 的得分就会比较高。

当然，为了避免这个算法引发调度堆叠，调度器在计算得分的时候还会根据镜像的分布进行优化，即：如果大镜像分布的节点数目很少，那么这些节点的权重就会被调低，从而“对冲”掉引起调度堆叠的风险。

以上，就是 Kubernetes 调度器的 Predicates 和 Priorities 里默认调度规则的主要工作原理了。

**在实际的执行过程中，调度器里关于集群和 Pod 的信息都已经缓存化，所以这些算法的执行过程还是比较快的。**

此外，对于比较复杂的调度算法来说，比如PodAffinityPredicate，它们在计算的时候不只关注待调度 Pod 和待考察 Node，还需要关注整个集群的信息，比如，遍历所有节点，读取它们的 Labels。这时候，Kubernetes 调度器会在为每个待调度 Pod 执行该调度算法之前，先将算法需要的集群信息初步计算一遍，然后缓存起来。这样，在真正执行该算法的时候，调度器只需要读取缓存信息进行计算即可，从而避免了为每个 Node 计算 Predicates 的时候反复获取和计算整个集群的信息。

## 总结

在本篇文章中，我为你讲述了 Kubernetes 默认调度器里的主要调度算法。

需要注意的是，除了本篇讲述的这些规则，Kubernetes 调度器里其实还有一些默认不会开启的策略。你可以通过为kube-scheduler 指定一个配置文件或者创建一个 ConfigMap ，来配置哪些规则需要开启、哪些规则需要关闭。并且，你可以通过为 Priorities 设置权重，来控制调度器的调度行为。

## 思考题

请问，如何能够让 Kubernetes 的调度器尽可能地将 Pod 分布在不同机器上，避免“堆叠”呢？请简单描述下你的算法。

感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。
<div><strong>精选留言（15）</strong></div><ul>
<li><span>鱼自由</span> 👍（7） 💬（1）<p>老师，最后你提到为 Priorities 设置权重，请问，这个操作在哪里进行？</p>2019-01-03</li><br/><li><span>Alex</span> 👍（4） 💬（1）<p>podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: name
                operator: In
                values:
                - nginx-demo
            topologyKey: &quot;kubernetes.io&#47;hostname&quot;

我用了反亲和的特性让pod分到了不同的机器上，不知道是否回答了你的问题</p>2018-11-28</li><br/><li><span>芒果少侠</span> 👍（90） 💬（7）<p>思考题答案，个人认为有三个解决思路
1. 为pod.yaml设置PreferredDuringSchedulingIgnoredDuringExecution（注意不是required），可以指定【不想和同一个label的pod】放在一起。调度器随后会根据node上不满足podAntiAffinity的pod数量打分，如果不想一起的pod数量越多分数越少。就能够尽量打散同一个service下多个副本pod的分布。
关于这一思路，k8s官网也给出了相同应用的例子。【preferredDuringSchedulingIgnoredDuringExecution 反亲和性的一个例子是 “在整个域内平均分布这个服务的所有 pod”（这里如果用一个硬性的要求是不可行的，因为您可能要创建比域更多的 pod）。】 -- https:&#47;&#47;k8smeetup.github.io&#47;docs&#47;concepts&#47;configuration&#47;assign-pod-node&#47;

2. SelectorSpreadPriority，kubernetes内置的一个priority策略。具体：与services上其他pod尽量不在同一个节点上，节点上同一个Service里pod数量越少得分越高。
3. 自定义策略，实现自己的负载均衡算法（一次性哈希等）。

参考资料：
0. PreferredDuringSchedulingIgnoredDuringExecution ：在调度期间尽量满足亲和性或者反亲和性规则，如果不能满足规则，POD也有可能被调度到对应的主机上。在之后的运行过程中，系统不会再检查这些规则是否满足。
1. https:&#47;&#47;wilhelmguo.cn&#47;blog&#47;post&#47;william&#47;Kubernetes%E8%B0%83%E5%BA%A6%E8%AF%A6%E8%A7%A3
2. https:&#47;&#47;blog.fleeto.us&#47;post&#47;adv-scheduler-in-k8s&#47;
3. https:&#47;&#47;zhuanlan.zhihu.com&#47;p&#47;56088355
4. https:&#47;&#47;kuboard.cn&#47;learning&#47;k8s-advanced&#47;schedule&#47;#filtering</p>2020-03-10</li><br/><li><span>Dale</span> 👍（14） 💬（1）<p>在工作中遇到这个问题，需要将pod尽量分散在不通的node上，通过kubernetes提供的反亲和性来解决的。从回答中老师希望从Priorities阶段中实现，我的想法如下：
1、首先在Predicates阶段，已经选择出来一组可以使用的node节点
2、在Priorities阶段，根据资源可用情况将node从大到小排序，加上node节点个数为m
3、根据pod中配置replicate个数n，在node列表中进行查找出资源可用的top n的node节点
4、如果node节点个数m不满足pod中的replicate个数，每次选择top m之后，重新计算node的资源可用情况，在选择top（n-m）的node，会存在node上有多个情况，最大程度上保证pod分散在不同的node上</p>2018-12-13</li><br/><li><span>陈斯佳</span> 👍（11） 💬（0）<p>第四十二课:Kubernetes默认调度器调度策略解析
Predicates在调度过程中的作用，可以理解为Filter，也就是按照调度策略从当前的集群所有节点中“过滤”出一些符合条件的节点来运行调度的 Pod。

默认的调度策略有三种，一种是GerenalPredicates，负责最基础的调度策略，比如计算宿主机CPU和内存资源等是否够用的PodFitsResource；还有检查宿主机名字是否和Pod的spec.nodeName一致的PodFitsHost。第二类是和Volume相关的过滤规则，比如NoDiskConflict是检查多个Pod申明挂载的持久化Volume是否有冲突。第三类是宿主机相关的过滤条件，主要考察待调度的Pod是否满足Node本身条件，比PodToleratesNodeTaints，负责检查Node 的“污点”taint机制，而 NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足，如果是的话，那么待调度 Pod 就不能被调度到该节点上。第四种类型是和Pod相关的过滤规则，这一组规则，跟 GeneralPredicates 大多数是重合的。而比较特殊的，是 PodAffinityPredicate。在具体执行的时候， 当开始调度一个 Pod 时，Kubernetes 调度器会同时启动 16 个 Goroutine，来并发地为集群里的所有 Node 计算 Predicates，最后返回可以运行这个 Pod 的宿主机列表。

在 Predicates 阶段完成了节点的“过滤”之后，Priorities 阶段的工作就是为这些节点打分。这里打分的范围是 0-10 分，得分最高的节点就是最后被 Pod 绑定的最佳节点。Priorities 里最常用到的一个打分规则，是LeastRequestedPriority。这个算法实际上就是在选择空闲资源（CPU 和 Memory）最多的宿主机。此外，还有 NodeAffinityPriority、TaintTolerationPriority 和 InterPodAffinityPriority 这三种 Priority。在默认 Priorities 里，还有一个叫作 ImageLocalityPriority 的策略。它是在 Kubernetes v1.12 里新开启的调度规则，即：如果待调度 Pod 需要使用的镜像很大，并且已经存在于某些 Node 上，那么这些 Node 的得分就会比较高。</p>2021-11-03</li><br/><li><span>freeman</span> 👍（9） 💬（1）<p>首先筛选出满足资源的机器。如果可用节点大于等于需求副本集则一个node一份，顺序取node调度即可，如果node节点少于副本数量，则进行一次调度后，剩下的副本重复上面的事情。直到为每个副本找到对应node或者调度失败。</p>2018-11-29</li><br/><li><span>陈小白( ´･ᴗ･` )</span> 👍（5） 💬（2）<p>这个我们线上就遇到了，一开始编排好差不多100个系统，发现其中几台主机一堆pod ，内存cpu 都很吃紧，而其他的主机却十分空闲。重启也没用。另外还想请教老师一个问题，我们遇到主机内存不足的时候，经常出现docker  hang 住了，就是docker 的命令完全卡死，没反应，不知道老师有遇到过呢？</p>2019-12-30</li><br/><li><span>tyamm</span> 👍（4） 💬（0）<p>老师，我有个疑问。这个课程后面会讲到如何搭建master节点高可用么？？</p>2018-11-28</li><br/><li><span>王景迁</span> 👍（3） 💬（1）<p>根据老师上面说的，默认的调度策略不是应该有四种类型吗？为什么文章开头说是三种类型？</p>2019-10-23</li><br/><li><span>小朱</span> 👍（1） 💬（0）<p>请教一个问题，自己启动了另一个scheduler，某一个node上同时存在default-scheduler和second-scheduler调度的资源。但是scheduler只统计schedulerName是自己的pod，这样就和node上面kubelet统计的资源就出现了不一致，这种设计是为什么呢？</p>2018-12-05</li><br/><li><span>周娄子</span> 👍（1） 💬（0）<p>写一个类似nginx一致性hash的算法</p>2018-11-28</li><br/><li><span>Geek_4df222</span> 👍（0） 💬（0）<p>思考题：将pod尽可能分布在不同节点上，这是一个优先的调度策略，应该在Priorities 阶段定义，具体来说，根据根据节点上的pod数量，pod数量越少，node的分数越高。</p>2023-08-27</li><br/><li><span>卢俊义</span> 👍（0） 💬（0）<p>一个dev开启后pod运行在node1的话，重启后是否不管node1的资源占用情况如何，新的pod是否还会运行在node1呢。线上项目遇到很多pod都会堆积在一个node节点，其他节点有空闲资源</p>2023-06-06</li><br/><li><span>浅陌</span> 👍（0） 💬（1）<p>请问默认情况下，这些predicate的算法和priority的算法都会被执行吗</p>2022-10-22</li><br/><li><span>zhoufeng</span> 👍（0） 💬（1）<p>Predicates 单词字面意思是“谓词”，谁知道为啥调度算法要取这个名字，有什么说法吗</p>2022-09-30</li><br/>
</ul>