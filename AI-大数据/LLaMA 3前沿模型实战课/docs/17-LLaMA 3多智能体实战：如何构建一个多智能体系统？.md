你好，我是Tyler！

在上一节课中，我们学习了多智能体系统的必要性，并探讨了如何通过引入多个智能体来提升系统的效率和灵活性。本节课我会带你深入理解开发过程中关键的设计方法和原则。

## **多智能体系统设计：智能体之间的最大区别是什么？**

在上一节课，我们提到LLaMA 3对智能体系统的优化要求我们遵循单一职责原则，而今天我们要深入探讨这个原则。设计多智能体系统时，最核心的问题之一是：智能体之间的最大区别是什么？答案是——人设（Persona）。

人设决定了每个智能体的角色和定位，也就是它在系统中具体要做的事情。你可以把人设想成公司里每个员工的岗位描述，通常包含以下几个重要元素：

1. **使命（Mission）：**智能体的任务或目标，它告诉我们智能体存在的意义。举个例子，有的智能体的使命可能是“确保公司的合规运作”，另一个可能是“确保产品技术领先”。使命不仅明确了要做什么，还决定了任务的优先级和范围。
2. **专业领域（Expertise）：**智能体擅长的领域，类似于人类员工的专长。通过专注于某个领域，智能体能更高效地完成任务。
3. **技能集（Skill Set）：**智能体所具备的能力、工具和方法。例如，某些智能体擅长处理自然语言、访问外部API，或者执行数据库查询等。技能集的不同让每个智能体能处理不同类型的任务，从而让系统更灵活。

### **合理分工：构建高效的智能体系统**

在多智能体系统中，合理的分工非常重要。首先，我们得根据系统的需求分析任务的复杂度，然后把任务分成不同的模块，最后将这些模块分配给合适的智能体。这样，每个智能体都有清晰的职责，大家互相配合，才能让整个系统高效运行。

为了确保每个智能体都能稳定地完成任务，我们会用到种子记忆。

#### **什么是种子记忆？**

种子记忆是指在智能体的提示词中埋入一些关键信息，作为智能体执行任务时的“记忆种子”。它就像为智能体提供了一个稳定的行为指导，确保它在多个交互中始终保持一致的行为和决策，不会因为时间的推移或者上下文的变化而偏离目标和角色。

#### **种子记忆的作用**

1. **稳定性**：种子记忆可以帮助智能体保持其行为的一致性。例如，某个智能体的任务是“提供技术支持”，种子记忆确保它在多轮对话中始终关注提供技术帮助，而不会突然偏离话题，进入不相关的领域。
2. **避免偏差**：智能体在执行任务时，可能会受到外部输入的影响，从而出现不符合预期的行为。种子记忆为智能体提供了一个稳定的参考框架，帮助它抵抗不必要的干扰，减少误差和偏差的发生。
3. **提高效率**：通过种子记忆，智能体能够快速了解其角色和目标，从而在任务执行中更高效，不会因为重新定义目标或角色而浪费时间和资源。

#### **种子记忆的实现方式**

通常，种子记忆是通过编写特定的提示词（prompt）来实现的。假设智能体扮演的是一个客户支持员工的角色，主要负责解答客户的咨询、处理售后问题、提供产品信息等。为了确保智能体在每次与客户的互动中始终符合员工角色的要求，我们为它设定了种子记忆，以确保它的行为和回答符合设定的员工职责。

这样，无论智能体与用户互动多少次，它都会根据这些信息执行任务。

```python
seed_memory = """
你是公司的一名客户支持员工。你的任务是提供高效、礼貌、专业的客户支持。无论客户提出什么问题，你都应该做到以下几点：

1. 始终保持友好和专业的语气。
2. 为客户提供清晰、准确的信息，尽可能解决他们的问题。
3. 如果问题超出了你的知识范围，应主动引导客户联系其他专业团队或提供进一步帮助的途径。
4. 你不能提供任何超出公司政策或产品范围的承诺。
5. 你可以使用公司提供的FAQ和支持文档，但不能随意修改或猜测信息。

员工的行为规范：在与客户互动时，始终表现出耐心和尊重。避免过度推销或引导客户购买不必要的产品。

语言风格：简洁、礼貌、专业，不使用俚语或过于随意的语言。
"""
```

## 智能体**护栏：行为安全与稳定的保障**

在设计多智能体系统时，除了明确智能体的角色和任务，更重要的是确保它们的行为能够安全、稳定地执行。尤其是智能体直接参与任务时，任何异常行为都可能导致系统不稳定。因此，我们需要引入“智能体护栏”（Guardrails）来限制智能体的行为，你可以把智能体护栏想象成是火车的轨道。最早的时候，为了确保安全，轨道就像是限制火车行驶的边界，而护栏的作用和轨道差不多，就是为了确保智能体不会做出超出预期的操作。

护栏的重要性体现在几个方面：

1. **防止错误决策**：有时候，智能体可能会因为数据误判或算法偏差，做出错误的决策，导致生产系统出问题。没有护栏的情况下，这些错误的决策可能会带来严重后果。护栏的作用就是防止这种情况发生，确保智能体的决策符合预期。
2. **保障系统稳定性**：智能体通常需要在在线环境中长时间、高频率地工作。如果它们做出了不合适的行为，可能会导致整个系统的崩溃或不稳定。通过设置护栏，可以减少这种异常操作的发生，从而确保系统的稳定性和可靠性。

### **护栏是怎么设计的？**

设计智能体护栏时，需要综合考虑智能体的任务需求以及它们的行为规范。这里的**行为规范**是指每个智能体都应该有明确的行为边界，确保它们只做自己被允许做的事。比如，订单处理系统中的智能体，只能修改订单的特定字段，而不能删除订单。我们可以通过规则引擎或者策略表来定义这些限制，确保智能体的行为在预定范围内。

为了确保智能体的行为符合预期，系统会**实时监控**它们的操作。通过记录操作日志并分析智能体的行为，可以及时发现问题。如果发现异常，系统可以立即发出警报，或者执行回滚操作，防止异常行为影响系统的正常运行。

在一些行业内，智能体护栏显得尤为重要，比如金融行业。智能体可能会涉及客户咨询或投资建议等任务，但并非所有智能体都具备金融专业知识。如果没有护栏，智能体可能会给客户提供错误的投资建议，带来风险。通过护栏程序，我们可以限制智能体的讨论内容，确保它们不会“胡说八道”。

## 智能体系统的完整设计流程

在了解了人设和护栏的重要性之后，我把多智能体系统的完整流程整理成一张图片，你可以参考。

1. **明确需求**
   
   - 分析系统的整体目标和需要解决的核心问题。
   - 将目标任务分解为多个模块，为智能体的分工设计提供基础。
2. **设计人设**
   
   - 针对每个任务模块，定义智能体的使命、专业领域和技能集。
   - 确保人设之间的职责明确，无交叉或冲突。
3. **配置护栏**
   
   - 针对每个智能体设计行为边界，防止权限滥用和不必要的操作。
   - 定义智能体的权限范围和具体操作限制。
4. **构建交互机制**
   
   - 设计智能体之间的通信协议。例如，使用消息队列、共享内存或事件驱动机制。
   - 确保智能体能够高效协作。
5. **测试与优化**
   
   - 在模拟环境中运行多智能体系统，观察其行为。
   - 根据测试结果调整人设和护栏，优化整体性能。
6. **上线运行**
   
   - 将经过测试的智能体部署到生产环境，并配置监控工具。
   - 持续收集运行数据，以便改进系统的稳定性和可靠性。

#### 总结

学到这里，我们做个总结吧。在多智能体系统的设计中，**人设和种子记忆的应用是确保智能体行为一致性和任务执行稳定性的关键。**通过合理分工、明确角色设定和行为规范，可以让每个智能体在复杂的系统中高效运作。

设计一个多智能体系统，不仅要考虑智能体的能力和任务，还要确保它们在实际应用中的稳定性和安全性。使智能体可以在复杂的系统环境中协同工作，完成各自的任务。在实际操作中，通过智能体行为的规范和种子记忆的持续维护，可以最大程度地避免系统中的偏差和风险，提升整体效率。

课程即将接近尾声，后续的任务也会变得越来越复杂。在这里，我想提前预告一下，即使在课程结束后，大家依然可以通过访问我的 [GitHub仓库](https://github.com/tylerelyt/llama)，继续获得不断更新的示例。对于一些有趣的示例，我们也会不定期进行更新。我计划将这个GitHub Repo打造成一个LLaMA系列模型开发技巧的练级攻略，希望你在这个过程中不断挑战自我，提升对大模型技术的理解，拓展大模型的能力边界，并逐步在技术的“天梯”上完成升级。

#### 思考题

根据今天学习的种子记忆和护栏技术，试着构建一个多智能体系统。我的实现会在 GitHub 上发布，你可以参考并对比。**提示：**你可以使用 LangGraph 或 Dify 这样的智能体系统来进行低代码建模。

欢迎你把你构建的智能体系统发布在留言区，我们一起交流探讨，如果你觉得这节课的内容对你有帮助的话，也欢迎你分享给其他朋友，我们下节课再见！
<div><strong>精选留言（1）</strong></div><ul>
<li><span>Fenix</span> 👍（1） 💬（1）<p>sedd memory和system Prompt的区别是啥呀？</p>2024-11-22</li><br/>
</ul>