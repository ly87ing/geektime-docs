你好，我是徐昊，今天我们来继续学习AI时代的软件工程。

上节课，我们介绍了如何使用LLM辅助进行模型展开，从而应用业务知识。具体做法是通过Mermaid将业务模型转化为大语言模型（Large Language Model，LLM）易于理解的方式，再用半结构化自然语言补充上下文，让思维链的构造更加简单。

但是如果我们没有模型要怎么处理呢？利用LLM辅助建模是否也能比传统建模方法更有效率呢？这是我们今天要讨论的问题。

## 构造反馈循环

建模的过程，是学习业务知识并提炼知识的过程。我们在[第六节课](https://time.geekbang.org/column/article/760312)提到使用LLM时，可以通过缩短反馈周期提高知识学习的效率。那么我们可以通过构造一个反馈循环，使用LLM帮助我们完成建模的过程。

在反馈循环的过程中，我们处在复杂的认知模式下，也就是遵循**探测（Probe）- 感知（Sense）- 响应（Respond）**的认知行为模式。

我们之前提到过，由于**探测**环节费力费时，往往会成为整个反馈周期的瓶颈。但是有了LLM的辅助，我们可以在**探测**阶段快速产生初始结果，或是根据反馈重新执行任务。那么**感知**这一环节就可能会成为新的瓶颈。

在我们前面课程的例子里，我们希望LLM帮助我们把一个基于关系型数据的解决方案，改造为使用MongoDB的解决方案。**感知**就比较直接了：将LLM生成的代码直接执行就可以了。但对于模型就没有这么简单了。

**模型是对于现实世界的抽象，**是没有对错之分的，只有不同的角度和抽象的方式。我们评价一个模型，首先看能否适用于业务场景，然后是能否应对业务场景的变化。**那么我们可以通过模型展开验证模型的适用度，以及应对变化的能力**。

正如我们在[上节课](https://time.geekbang.com/column/article/761647)的演示，这个过程也可以通过LLM辅助。因而，我们使用LLM辅助建模的过程，就是利用LLM快速构建一个模型，并在不同的业务场景中展开这个模型，收集反馈再让LLM帮助我们调整模型。

![](https://static001.geekbang.org/resource/image/2f/72/2fa32e8f37b9d0dc1ae08b15bd151a72.jpg?wh=1920x1080)

如上图所示，在这里我们需要利用两个LLM的会话（Session），一个用于生成模型，一个用于模型展开。同样我们也需要两个任务模板，一个负责建模，一个负责模型的展开。

但是这里还有一点需要注意，上一节课我们介绍模型展开的时候，不涉及模型的修改。只需要在特定的上下文中，使用模型中的概念给出解释就行了。

但是在建模的过程中，不可能一次就建模正确。我们很可能碰到**概念缺失**的情况，也就是在用户故事或其他业务上下文中提及的概念，在模型中不存在。或是**关系错置**的情况，也就是模型中对象间的关联关系不正确的情况。因而在展开之前，我们可以先行反馈这些问题。于是，我们最终的反馈循环看起来是这个样子的：

![](https://static001.geekbang.org/resource/image/7e/cd/7e7b5fb2937700a96d375815e96508cd.jpg?wh=1920x1080)

我们使用的三个模板分别如下。

建模任务模板：

```plain
业务描述
=======
{context}

任务
====
根据业务描述，为系统建立模型。可以添加你认为必要的实体和关系。并将模型表示为mermaid的class diagram
```

模型检查任务模板：

````
```plain
领域模型
======
```mermaid
{model}
```

用户故事
======
{user_story}

验收场景
======
{ac}

任务
===
针对这个用户故事和验收场景，领域模型中缺少哪些概念？或者存在哪些不正确的关联关系。
请用文字表示缺失的概念是什么？以及存在哪些不正确的关联。
```
````

模型展开任务模板：

````
```plain
领域模型
======
```mermaid
{model}
```

用户故事
======
{user_story}

验收场景
======
{ac}

任务
===
数据都以yaml格式给出。
首先，请根据领域模型理解用户故事中的场景，并针对验收场景中Given的部分，给出样例数据。
然后，参看验收场景中When的部分，给出样例数据会产生怎样的改变。
```
````

在模型展开的模板中，我们需要用户故事和验收场景。同样，我们可以使用上一节课中的用户故事和验收条件。

> **作为**学校的教职员工（**As** a faculty），  
> **我希望**学生可以根据录取通知将学籍注册到教学计划上（**I want** the student to be able to enroll in an academic program with given offer），  
> **从而**我可以跟踪他们的获取学位的进度（**So that** I can track their progress）

> **如果**获取了录取通知的学生没有注册学籍时（**Given** student with offer hasn’t enrolled any program），  
> **当**这个学生注册时（**When** the student enroll），  
> **那么**这个学生将能成功注册学籍到录取通知指定的教学计划中（**Then** the student will successfully enroll the program specified in the offer）x

下面让我们使用这两个模板，通过反馈循环，对这个系统进行建模。

## 使用反馈循环完成建模

首先，让我们先描述一下我们要建模的业务系统，因为我们存在一个让ChatGPT检查的环节，出于展示的目的，我们不需要放入太多的内容。但在正常的使用过程中，更多的上下文总是会产生更好的结果。

```plain
这是一个教学学籍管理系统
```

我们套入模板，先让ChatGPT帮我们建模。我这里使用了ChatGPT-4的插件，可以直接显示模型图。如果没有插件的话，可以根据ChatGPT生成的Mermaid在使用在线编辑器看图。

![](https://static001.geekbang.org/resource/image/9d/20/9dfb1091a4f5908a7f8a17769ce59e20.jpg?wh=1325x2611)

可以看到，结果跟我们建模的结果相去甚远。因为我们并没有给出具体的业务描述，ChatGPT只能根据对于学籍管理的泛泛理解，完成建模的过程。那么我们可以把这次生成的结果，代入到模型检查的任务模板中去。

![](https://static001.geekbang.org/resource/image/e2/a4/e2110f341bac99f85c32455dc9fc16a4.jpg?wh=1374x1238)

ChatGPT给出了对于概念缺失的提示。我们可以根据这个反馈，修改对于业务的描述。一个简单的办法是，我们可以列出当前业务中的核心概念，让LLM重新生成模型：

```plain
这是一个教学学籍管理系统。系统中应该包含以下的核心概念：
- 教学计划: 一系列相关课程和活动，这些课程和活动旨在培养特定领域的知识和技能。比如，计算机科学与技术学士学位教学计划，或是计算机科学与技术硕士学位教学计划
- 录取通知: 学生需要根据录取通知注册学籍。录取通知应该包含学生被录取的信息，如录取的教学计划
```

![](https://static001.geekbang.org/resource/image/e5/d4/e533f66467727f25da019b9f0b4663d4.jpg?wh=1356x3259)

我们可以看到，这次已经非常接近我们之前给出的结果了。让我们再继续验证一下这个模型。

![](https://static001.geekbang.org/resource/image/1f/f3/1f6cc2f54ed3b1a41a8yy61f602947f3.jpg?wh=1352x1661)  
可以看到ChatGPT给出的建议非常类似我们前面课程中给出的模型。接下来，我们继续补充需要的核心概念：

```plain
这是一个教学学籍管理系统。系统中应该包含以下的核心概念：
- 教学计划: 一系列相关课程和活动，这些课程和活动旨在培养特定领域的知识和技能。比如，计算机科学与技术学士学位教学计划，或是计算机科学与技术硕士学位教学计划
- 录取通知: 学生需要根据录取通知注册学籍。录取通知应该包含学生被录取的信息，如录取的教学计划
- 学籍：当学生注册之后，学籍记录学生在校将按照哪个教学计划学习
- 学生
```

![](https://static001.geekbang.org/resource/image/d9/18/d925330ba9b7541878cf3f651f04ba18.jpg?wh=1270x3053)

已经非常接近了。让我们再检查一次：

![](https://static001.geekbang.org/resource/image/aa/62/aa2233b8b37728edbb30f22fb048f062.jpg?wh=1402x1441)

好了，现在对于实体已经没有遗漏了，只有对于关联关系的建议，这些目前我们并不需要非常关注，我们建模第一步先把实体找清楚，然后再来看建模关系。下面让我们进行模型展开：

![](https://static001.geekbang.org/resource/image/e8/1d/e8897a43b1340c6ecd5d5084de8c931d.jpg?wh=1344x3569)

至此为止，我们就通过LLM辅助，借助反馈循环得到了与之前类似的模型，并且这个模型也可以解释当前的业务。

## 小结

今天我们介绍了如何通过构造反馈循环，并使用LLM加速反馈循环中不同的环节，辅助我们完成建模的过程。

![图10](https://static001.geekbang.org/resource/image/7e/cd/7e7b5fb2937700a96d375815e96508cd.jpg?wh=1920x1080)

我们首先描述了系统的大致情况，然后通过LLM在不同的业务上下文中，验证模型的完备性，提出缺失的概念。之后将缺失的概念添加回业务上下文中，再做模型改进。利用这种方法，我们哪怕预先不知道太多业务上下文，也可以仅仅通过LLM的反馈来辅助建模。

我们展示的方式，也是知识工程的一个基本模式，**构造复杂认知模式的反馈循环并通过LLM加速**。在后面的课程中，我们会反复看到这个模式的应用。

## 思考题

有没有其他的方式构造反馈的循环？

欢迎你在留言区分享自己的思考或疑惑，我们会把精彩内容置顶供大家学习讨论。
<div><strong>精选留言（11）</strong></div><ul>
<li><span>赫伯伯</span> 👍（5） 💬（2）<p>能不能直接给大模型用户故事？让他根据用户故事建模</p>2024-04-02</li><br/><li><span>术子米德</span> 👍（1） 💬（1）<p>🤔☕️🤔☕️🤔
【R】建模时处于复杂认知模式，构造复杂认知模式的反馈循环，通过LLM加速建模过程、以及认知负担下降。
模板 = 建模任务 + 模型检查 + 模型展开。
关键 = 用户故事 + 反馈循环。
【.I.】建模 = 实体 + 关系 = 概要设计 = 总体设计 = 用户故事从自然语言、转变成结构化描述 = 对代码实现而言的半结构化描述。
【Q】用户故事，如果Clear，那么建模是否为Complicated，如果用户故事本身也Complicated，那么建模才是Complex？
— by 术子米德@2024年3月27日</p>2024-03-27</li><br/><li><span>范飞扬</span> 👍（5） 💬（0）<p>这节课需要代码来反复生成 prompt，不过老师没贴代码，所以我让 GPT 写了一下，放到公众号了，希望对大家有帮助~

链接：
https:&#47;&#47;mp.weixin.qq.com&#47;s?__biz=Mzg3MDg5MzYyMA==&amp;mid=2247483819&amp;idx=1&amp;sn=64bb8ce513260a785e589ae45eccd19f&amp;chksm=ce8790f0f9f019e63aa26e10cba54eb8eab1def0af5f2a192c7566bb773541ffa1d9f2310098&amp;token=597248625&amp;lang=zh_CN#rd</p>2024-03-27</li><br/><li><span>aoe</span> 👍（1） 💬（0）<p>第二次学习收获

当处于一个陌生的领域时，可以

1. 简述需求，让 AI 生成 mermaid 的 class diagram
2. 将 mermaid 带入 CoT；补充用户故事、验收场景到 CoT；开始新一轮 RAG 任务获取更多缺失概念与不正确的概念
3. 验收与 AI 互动后的学习成果

以上三步分别对应文中的三个模板：建模任务模板 -&gt; 模型检查任务模板 -&gt; 模型展开任务模板</p>2024-03-30</li><br/><li><span>6点无痛早起学习的和尚</span> 👍（0） 💬（0）<p>我用 claude 做建模任务模板，还给了一个答案：缺少模型 Degree(学位):用户故事提到了&quot;获取学位的进度&quot;,但是当前模型中没有学位的概念。学位应该是一个独立的实体,包含学位名称、级别等属性。</p>2024-05-03</li><br/><li><span>6点无痛早起学习的和尚</span> 👍（0） 💬（0）<p>2024年05月03日11:51:10
蛮有意思的 3 个会话模板，有了模板关于就在于“用户故事”、“验收场景”这 2 个了</p>2024-05-03</li><br/><li><span>Morty</span> 👍（0） 💬（0）<p>一定要用三个 Session 吗？不能在一个对话里完成吗？</p>2024-05-01</li><br/><li><span>姑射仙人</span> 👍（0） 💬（0）<p>用国内的智谱清言走了一遍课程的流程，发现结果非常发散，不停的在增加实体，无法收敛。不知道这种情况算好事，还是坏事，如何让它停止发散，快速收敛。</p>2024-04-28</li><br/><li><span>范飞扬</span> 👍（0） 💬（0）<p>我注意到模板里面有个“```plain”，这个是笔误还是为了和mermaid区分出来加的呢？</p>2024-04-18</li><br/><li><span>王海</span> 👍（0） 💬（0）<p>老师，可以把完整的 prompt 或者完整的 LLM 对话内容分享一下吗？文章中只有prompt模板。</p>2024-03-30</li><br/><li><span>术子米德</span> 👍（0） 💬（0）<p>【Q】检查模型的模板、展开模型的模板，输入一样、输出一样，它们的差别是啥？</p>2024-03-30</li><br/>
</ul>