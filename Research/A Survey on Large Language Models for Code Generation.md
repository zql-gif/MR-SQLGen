# links


# 5 LARGE LANGAUGE MODELS FOR CODE GENERATION
## 5.1 Data Curation & Processing
### 5.1.3 Benchmark

我们将这些基准大致分为六大类，依据其应用背景，包括通用型、竞赛编程、数据科学、多语言、逻辑推理和代码库级别（general-purpose, competitive programming, data science, multilingual, logical reasoning, and repository-level）。
#### General
• **HumanEval [48]**：HumanEval 包含 164 个手动编写的 Python 编程问题，每个问题都包含一个函数签名、文档字符串、主体和多个单元测试。  
• **HumanEval+ [162]**：HumanEval+ 扩展了原始的 HumanEval [48] 基准，通过将测试用例的规模增加 80 倍来扩展测试范围。随着测试用例的增加，HumanEval+ 能够捕捉到大量以前未检测到的 LLM 合成的错误代码。  
• **HumanEvalPack [187]**：HumanEvalPack 扩展了 HumanEval [48]，将其扩展到包括六种编程语言的三项编程任务，分别是代码合成、代码修复和代码解释。  
• **MBPP [17]**：MBPP 是一个包含大约 974 个 Python 编程问题的数据集，经过众包设计，适用于初级程序员。每个问题都提供了一个英文任务描述、一个代码解决方案和三个自动化测试用例。  
• **MBPP+ [162]**：MBPP+ 在 MBPP [17] 的基础上，通过去除不规范的问题和修正实现错误的问题进行了增强。MBPP+ 的测试规模也增加了 35 倍，用于测试集的扩展。  
• **CoNaLa [297]**：CoNaLa 包含大约 597K 个数据样本，用于评估 Python 代码生成。CoNaLa 的策划部分是从 Stack Overflow 爬取的，经过自动筛选后由注释员整理。挖掘部分是自动挖掘的，包含近 60 万个示例。  
• **Spider [300]**：Spider 是一个大规模复杂的文本到 SQL 数据集，覆盖了 138 个不同的领域。它包含超过 1 万个问题和 5.6K 个复杂的 SQL 查询，涉及 200 个数据库。该数据集旨在测试模型在 SQL 查询、数据库模式和新领域中的泛化能力。  
• **CONCODE [113]**：CONCODE 是一个包含来自公共 GitHub 仓库的 10 万多个 Java 类的数据集。它提供了近乎零-shot 的条件，可以测试模型在未见过的自然语言标记和未见过的环境中的泛化能力。  
• **ODEX [273]**：ODEX 是一个开放域数据集，专注于从自然语言生成 Python 代码的执行基准。它包含 945 对自然语言查询及其对应的 Python 代码，所有数据都来自 Stack Overflow 论坛。  
• **CoderEval [299]**：CoderEval 是一个实用的代码生成基准，包含 230 个 Python 和 230 个 Java 代码生成问题。它可用于评估模型在生成实际代码（不仅仅是生成独立函数）方面的表现。  
• **ReCode [263]**：ReCode 作为一个全面的鲁棒性评估基准，通过对文档字符串、函数和变量名、代码语法以及代码格式进行扰动，提供了对模型鲁棒性表现的多维评估。  
• **StudentEval [19]**：StudentEval 是一个包含 1,749 个提示的数据库，涵盖 48 个问题，由 80 名仅完成过一个学期 Python 编程课程的学生编写。与其他许多基准不同，每个问题有多个提示，并且同一参与者进行了多次尝试，每个问题还附带了一组由教师编写的测试用例。
• **BigCodeBench [333]**：BigCodeBench 包含 1,140 个复杂的 Python 编程任务，涵盖了来自 139 个流行库的 723 个函数调用，跨越 7 个领域。该基准专门设计用于评估 LLM 在跨领域库中调用多个函数并遵循复杂指令解决编程任务的能力，帮助弥合孤立编码练习与实际编程场景之间的评估差距。  
• **ClassEval [72]**：ClassEval 是一个手工制作的基准，包含 100 个类和 412 个方法，用于评估 LLM 在类级别代码生成场景中的表现。特别地，ClassEval 的任务样本具有较高的复杂性，涉及长代码生成和复杂的文档字符串信息，因此有助于评估 LLM 在生成复杂代码方面的能力。  
• **NaturalCodeBench [314]**：NaturalCodeBench 是一个综合性的代码基准，包含 402 个高质量的 Python 和 Java 编程问题。这些问题来源于在线编码服务中的自然用户查询，涵盖了 6 个不同的领域，塑造了一个与实际应用相一致的评估环境。
#### Repository
• **RepoEval [309]**：RepoEval 用于评估代码库级别的代码补全。通过使用单元测试，它能够提供不同粒度的评估，并提高评估的准确性。  
• **Stack-Repo [239]**：Stack-Repo 是一个包含 200 个来自 GitHub 的 Java 仓库数据集，这些仓库的文件经过去重处理。这些文件通过三种类型的仓库上下文进行增强：提示建议上下文、基于 BM25 相似度得分的 BM25 上下文，以及通过嵌入模型表示空间中的最近邻获取的 RandomNN 上下文。  
• **Repobench [167]**：Repobench 是一个专门用于评估代码库级别代码自动补全系统的基准，支持 Python 和 Java。它包括三个相互关联的评估任务：RepoBench-R（检索）、RepoBench-C（代码补全）和 RepoBench-P（管道）。  
• **EvoCodeBench [144]**：EvoCodeBench 是一个进化式代码生成基准，通过严格的流程构建，并与实际的代码库对齐。该基准还提供了全面的注释和强大的评估指标。  
• **SWE-bench [123]**：SWE-bench 是一个数据集，用于测试模型自动解决 GitHub 问题的能力。该数据集包含来自 12 个流行 Python 仓库的 2,294 对问题-拉取请求。

## 5.10 Evaluation
针对 LLM 在代码生成中的评估策略与通用 LLM 的评估策略类似，可分为三大类：基于指标的评估、人本中心的评估和基于 LLM 的评估方法（three principal categories: metrics-based, human-centered, and LLM-based approaches）。**这些评估策略的详细基准测试将在 5.1.3 节中呈现，并在表 6 中总结。** 

### 5.10.3 LLM-as-a-Judge

LLM 强大的指令跟随能力(instruction-following capabilities)激发了研究人员创新性地探讨基于 LLM 的评估潜力。

* LLM-as-a-Judge
	* 定义：将先进的专有 LLM（例如 GPT-4、Gemini 和 Claud 3）作为人类评估者的代理。这涉及设计具有特定要求的提示，以引导 LLM 进行评估。
	* 方法：AlpacaEval [148] 和 MT-bench [320] 。
	* 优点：
		* 减少了对人类参与的依赖，从而促进了更高效和可扩展的评估。
		* LLM 还可以提供对评分结果的深入解释，从而增强评估的可解释性。
	* 局限性：有效性受到所选 LLM 固有局限性的制约
		* 位置偏差((position biases):位置偏差指的是 LLM 倾向于不成比例地偏爱呈现于某些位置的回答，这可能会根据答案的呈现顺序扭曲对答案质量的认知。（当多个答案被呈现给 LLM 时，如果 LLM 更倾向于给排在前面的回答更高的评分，即便这些回答的质量没有明显优于其他位置的回答，这就可能导致评估结果的失真）
		* 冗长偏差(verbosity biases):冗长偏差描述了 LLM 倾向于偏好较长的回答，即便这些回答未必比简洁的回答质量更高。
		* 自我增强偏差(self-enhancement biases):自我增强偏差则表现为 LLM 一贯地高估其生成文本的质量。
		* 推理能力受限(restricted reasoning ability):由于 LLM 在处理复杂推理任务时固有的局限性，它们可能并不完全可靠，尤其是在需要深入推理的任务中，如涉及数学问题解决的任务。


![fig16_PipelineOfLLM-as-a-judge.png](assets/Research/fig16_PipelineOfLLM-as-a-judge.png)

* ICE-Score 评估指标[332]
	* 作用：该指标指导 LLM 进行代码评估。
	* 优点：
		* 在功能正确性和人类偏好方面与实际结果具有更高的相关性，从而消除了对测试 oracle 或参考答案的需求。





# conferences
[148] Xuechen Li, Tianyi Zhang, Yann Dubois, Rohan Taori, Ishaan Gulrajani, Carlos Guestrin, Percy Liang, and Tatsunori B. Hashimoto. 2023. AlpacaEval: An Automatic Evaluator of Instruction-following Models. https://github.com/tatsu-lab/alpaca_eval.
[320] Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, Siyuan Zhuang, Zhanghao Wu, Yonghao Zhuang, Zi Lin, Zhuohan Li, Dacheng Li, Eric Xing, et al. 2024. Judging llm-as-a-judge with mt-bench and chatbot arena. Advances in Neural Information Processing Systems 36 (2024).
[332] Terry Yue Zhuo. 2024. ICE-Score: Instructing Large Language Models to Evaluate Code. In Findings of the Association for Computational Linguistics: EACL 2024. 2232–2242.



[Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://papers.nips.cc/paper_files/paper/2023/hash/91f18a1287b398d378ef22505bf41832-Abstract-Datasets_and_Benchmarks.html)

