## 调研
### 0 A Survey on Large Language Models for Code Generation
[A Survey on Large Language Models for Code Generation](Research/A%20Survey%20on%20Large%20Language%20Models%20for%20Code%20Generation)
### 1 LLM-as-a-Judge的LLM Code Generation方法
[LLM-as-a-Judge的LLM Code Generation方法](Research/LLM-as-a-Judge的LLM%20Code%20Generation方法)

### 2 基于大模型的Python代码生成的Dataset和Prompt组成元素
[基于大模型的Python代码生成的Dataset和Prompt组成元素](Research/基于大模型的Python代码生成的Dataset和Prompt组成元素)



## 任务
Empirical study Prompt中的各种因素对于代码生成的影响并提出改进方法。

Motivation： 研究prompt的组成（例如role-play、format、NL）对于LLM generated code的影响，提出一种prompt优化的方法。

现有工作：Evaluation Benchmark、 LLM for Code Generation（Pre-train、fine-tune、prompt engineering、Multi-Agents....） 针对LLM，没有针对prompt、Autoprompting、 扰动测试...

创新点：换个角度，低代价，性能研究

Research Method：

● Phase 1：探索Docstring（Prompt）对LLM生成代码质量的影响因素是什么？

○ Collect dataset（Python HumanEval，类型注释、函数描述、上下文...） ---》 生成一些等价（蜕变）prompt来研究影响 （LLM、人工... 多样性即可） ---》 评估（可读性、正确性...），可不可以训练一个奖励模型？

● Phase 2：针对第一步优质的prompt, 训练一个专门用于改进Prompt的模型。 Instruct fine-tuning。 （升华.. usefulness）