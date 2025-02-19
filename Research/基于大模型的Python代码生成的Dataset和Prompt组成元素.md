# 备注
* ReCode这篇细看一下，似乎可以作为本文的相项目的相关工作（关于大模型鲁棒性的探讨，组合生成各种prompt）
* StudentEval这篇看一下，这篇包含多个非专家编写的提示词，描述了同一个问题，能够帮助探索提示成功的关键因素。
# 参考A Survey on Large Language Models for Code Generation ---- 5.10.3,Table 6中General和Repository的内容
[A Survey on Large Language Models for Code Generation](Research/A%20Survey%20on%20Large%20Language%20Models%20for%20Code%20Generation)

# 1 Genaral
## **1.1 HumanEval** [48]
* 简介：HumanEval 包含 164 个手动编写的 Python 编程问题，每个问题都包含**一个函数签名、文档字符串、主体和多个单元测试（a function signature, docstring, body, and multiple unit tests）**。  
### links
* 论文：Mark Chen, Jerry Tworek, Heewoo Jun, Qiming Yuan, Henrique Ponde de Oliveira Pinto, Jared Kaplan, Harri Edwards, Yuri Burda, Nicholas Joseph, Greg Brockman, et al. 2021. Evaluating large language models trained on code. arXiv preprint arXiv:2107.03374 (2021).
* 论文精读：
	* [Evaluating Large Language Models Trained on Code - 知乎](https://zhuanlan.zhihu.com/p/619403997)
	* [HumanEval是如何进行代码评估的：从数据构成、评估逻辑到pass@k指标计算-CSDN博客](https://blog.csdn.net/qq_27590277/article/details/135163862)
	* [代码生成大模型评估指标以及数据集 - 知乎](https://zhuanlan.zhihu.com/p/691397120)
* dataset：
	* https://huggingface.co/datasets/openai_humaneval
	* [openai/human-eval: Code for the paper "Evaluating Large Language Models Trained on Code"](https://github.com/openai/human-eval)
### 规模
* size：164，人工编写
* 问题领域：语言理解力、算法和简单的数学
* PL：Python
### 单条数据组成
* task_id：任务的ID
	* "task_id":"HumanEval/0"
* prompt：表示题目（通常直接请求大模型获取答案）
	* **"prompt":库+函数签名+文档字符串（功能描述，输入输出样例）+函数主体**
	``` Python
	from typing import List
	def has_close_elements(numbers: List[float], threshold: float) -> bool:
	    """
	    Check if in given list of numbers, are any two numbers closer to each other than given threshold.
	
    >>> 	has_close_elements([1.0, 2.0, 3.0], 0.5)
	    False
	
    >>> 	has_close_elements([1.0, 2.8, 3.0, 4.0, 5.0, 2.0], 0.3)
	    True
	    """
	```
* entry_point：唯一标记
	* "entry_point":"has_close_elements"
* canonica_solution：参考答案
``` Python
	for idx, elem in enumerate(numbers):
	    for idx2, elem2 in enumerate(numbers):
	        if idx != idx2:
	            distance = abs(elem - elem2)
	            if distance < threshold:
	                return True
	return False
	```
* test：测试单元
``` Python
METADATA = {
    'author': 'jt',
    'dataset': 'test'
}
def check(candidate):
    assert candidate([1.0, 2.0, 3.9, 4.0, 5.0, 2.2], 0.3) == True
    assert candidate([1.0, 2.0, 3.9, 4.0, 5.0, 2.2], 0.05) == False
    assert candidate([1.0, 2.0, 5.9, 4.0, 5.0], 0.95) == True
    assert candidate([1.0, 2.0, 5.9, 4.0, 5.0], 0.8) == False
    assert candidate([1.0, 2.0, 3.0, 4.0, 5.0, 2.0], 0.1) == True
    assert candidate([1.1, 2.2, 3.1, 4.1, 5.1], 1.0) == True
    assert candidate([1.1, 2.2, 3.1, 4.1, 5.1], 0.5) == False

```

### 评估逻辑
每一个测试问题重复实验n次，然后通过单元测试，计算平均通过率。
**1、读取每个样本，请求模型获得结果**
* generate_one_completion为请求大模型生成结果的函数，输入每道题的prompt，然后得到结果。
* num_samples_per_task用来控制每道题生成多少个结果(代码中设置为200次)，从而计算通过率。
* completion作为补全结果的存储字段。

``` Python
from human_eval.data import write_jsonl, read_problems
 
problems = read_problems()
 
num_samples_per_task = 200
samples = [
    dict(task_id=task_id, completion=generate_one_completion(problems[task_id]["prompt"]))
    for task_id in problems
    for _ in range(num_samples_per_task)
]
write_jsonl("samples.jsonl", samples)
```

**2、获得模型的结果，进行单元测试**
* 针对得到的补全结果，通过构造一个完整的测试用例，送入单元测试中进行测试。
* 测试样例的构造：将题目的prompt、模型预测的内容completion、题目的test的按照换行符进行拼接。
* 进行单元测试：给定超时timeout时间，如果测试通过，则标记为passed，如果不是，则不通过。

### 评估指标
**1. pass@k指标计算**
* Kulal等人（2019年）使用pass@k指标评估功能正确性，**每个问题生成k个代码样本，如果任何样本通过单元测试，则认为问题已解决**，并报告总分数。
* **一次实验随机性太大，需要多次实验求平均值。pass@k需要对每一个测试问题重复实验t次，并且每次都生成k个代码，最后计算平均通过率。假如重复实验100次来估计pass@100，就需要生成 100 x 100=10000个代码，这样的计算量是难以接受的。而t越小，估计的pass@k就越不准（方差越大）。**
**2. pass@k公式理解**
* ![pass_k.png](assets/Research/pass_k.png)
* 解释：`Pass@k` 的计算是基于从 `n` 个样本中选择 `k` 个样本的组合概率，减去从错误样本中选择 `k` 个的概率。最终结果越接近 1，表示模型生成的正确样本越多。
* 无偏估计：为了应对高采样方差，使用了无偏估计方法来计算 `Pass@k`，这有助于确保评估结果的稳定性和可靠性。
- 参数解释
	- **𝑛**：实际为每个问题生成的代码样本总数，即生成的代码候选样本的数量。
	- **𝑐**：为每个问题生成的正确代码样本的数量，表示通过测试用例的正确生成的代码样本数量。
	- **𝑘**：**在此公式中是一个固定的常数**。
	- **pass@k：评估在 n 个生成样本中，至少有 k 个样本是正确的概率。**
	- **论文工作为每个任务生成n≥k个样本（本文中使用n=200，k≤100），计算通过单元测试的正确样本c≤n的数量，并计算无偏估计值。**
	- **𝑛 - 𝑐**：指剩余的错误样本数量，即没有通过测试的代码样本。
	- **(n  k)**：组合数，表示从 `n` 个样本中选择 `k` 个样本的所有可能方式。这个数值用于表示从所有样本中选择 `k` 个样本的组合数量。
	- **(n−c  k)**：表示从错误样本中选择 `k` 个样本的组合数。这用来计算从错误样本中选择的 `k` 个样本的可能性。
* [代码生成模型评价指标 pass@k 的计算 - 知乎](https://zhuanlan.zhihu.com/p/653063532)


## 1.2 HumanEval+ [162]
* 简介：HumanEval+ 扩展了原始的 HumanEval [48] 基准，**通过将测试用例的规模增加 80 倍来扩展测试范围。** 随着测试用例的增加，HumanEval+ 能够捕捉到大量以前未检测到的 LLM 合成的错误代码。  
### links
* 论文：
	* Jiawei Liu, Chunqiu Steven Xia, Yuyao Wang, and Lingming Zhang. 2024. Is your code generated by chatgpt really correct? rigorous evaluation of large language models for code generation. Advances in Neural Information Processing Systems 36 (2024).
* 论文解读：
	* [LLM评测一：HumanEval+ - 知乎](https://zhuanlan.zhihu.com/p/656306032)
* dataset
	* [evalplus/humanevalplus · Datasets at Hugging Face](https://huggingface.co/datasets/evalplus/humanevalplus)
	* [evalplus/evalplus: Rigourous evaluation of LLM-synthesized code - NeurIPS 2023 & COLM 2024](https://github.com/evalplus/evalplus)

### 规模
* size：164，**在HumanEval的基础上添加自动化测试用例生成**
* 问题领域：语言理解力、算法和简单的数学
* PL：Python
### 单条数据组成
同HumanEval
### 评估逻辑
同HumanEval
### 评估指标
* **同HumanEval，实验结果表明HumanEval+能够捕捉到大量之前未检测到的错误代码，显著降低了LLM生成代码的通过率（pass@k）。** 例如，WizardCoder-CodeLlama和Phind-CodeLlama在HumanEval+上的表现优于ChatGPT，而在原始HumanEval上则相反。
* 通过自动化测试生成和多样化测试输入，EvalPlus能够更准确地评估LLM生成代码的功能正确性。
## 1.3 HumanEvalPack [187]

* 简介：HumanEvalPack 扩展了 HumanEval [48]，**将其扩展到包括六种编程语言的三项编程任务，分别是代码合成、代码修复和代码解释**。  
### links
* 论文
	* Niklas Muennighoff, Qian Liu, Armel Zebaze, Qinkai Zheng, Binyuan Hui, Terry Yue Zhuo, Swayam Singh, Xiangru Tang, Leandro Von Werra, and Shayne Longpre. 2023. Octopack: Instruction tuning code large language models. arXiv preprint arXiv:2308.07124 (2023).
	* [[2308.07124] OctoPack: Instruction Tuning Code Large Language Models](https://arxiv.org/abs/2308.07124)
* dataset
	* [bigcode/humanevalpack · Datasets at Hugging Face](https://huggingface.co/datasets/bigcode/humanevalpack)
### 规模
* size：164，人工编写。Python 拆分与 OpenAI 的 Python HumanEval 完全相同。**其他拆分由人工翻译。**
* 问题领域：语言理解力、算法和简单的数学
* **三项编程任务：是代码合成、代码修复和代码解释**
* **PL：cpp, go, java, js,  python, rust**
### 单条数据组成
在HumanEval的基础上补充了几个键，其他的相同。
- **task_id**：表示问题的语言（Python/JavaScript/Java/Go/C++/Rust）和任务 ID（从 0 到 163）
	- "Python/0"
- **prompt**：用于依赖代码续写的模型的提示
- **declaration**：函数声明（与 prompt 相同，但没有 docstring）
- **canonical_solution**：通过所有单元测试的正确解答
- **buggy_solution**：与 canonical_solution 相同，但包含一个微妙的人为编写的错误，导致单元测试失败
- **bug_type**：buggy_solution 中的错误类型（可能的值包括 [missing logic, excess logic, value misuse, operator misuse, variable misuse, function misuse]）
- **failure_symptoms**：错误导致的问题（可能的值包括 [incorrect output, stackoverflow, infinite loop]）
- **entry_point**：函数名称
- **import**：解决方案所需的导入（仅对于 Go 语言存在）
- **test_setup**：测试执行所需的导入（仅对于 Go 语言存在）
- **test**：问题的单元测试
- **example_test**：与 test 不同的附加单元测试，可能是提供给模型的额外测试（这些在论文中未使用）
- **signature**：函数签名
- **docstring**：描述问题的 docstring
- **instruction**：**HumanEvalSynthesize 的指令，格式为：  
    `Write a {language_name} function {signature} to solve the following problem:\n{docstring}`**

### 三项编程任务

**1. HUMANEVALFIX (NL+C→C)**  
* **给定一个包含细微 bug 的错误代码函数（即buggy_solution）以及相关的单元测试，模型的任务是修复该函数。**
* bug添加
	* 手动为所有 6 种语言的 164 个 HumanEval 解决方案添加了 bug（总共 984 个 bug）。对于每个样本，bug 在 6 种语言中尽可能相似，以便进行有意义的跨语言分数比较。
	* 编写方式是代码仍然能运行，但会产生错误的结果，导致至少一个单元测试失败。
* **prompt组成：buggy_solution + test + fix instruction**


**2. HUMANEVALEXPLAIN (NL+C→NL)**  
* 第一步，给定一个正确的代码函数（即canonical_solution），模型的任务是生成该代码的解释。
* 第二步，仅根据自己的解释重新生成代码。（**第二步使我们能够通过代码执行来对该任务进行评分，并使用 pass@k进行评估，而不是使用诸如 BLEU（Papineni et al., 2002）或 ROUGE（Lin, 2004）等启发式指标来评估解释本身**） 
* **prompt组成：第一步的是canonical_solution + explain instruction；第二步的是第一步的explain + generate instruction**

**3. HUMANEVALSYNTHESIZE (NL→C)**
* **给定一个自然语言的 docstring 或注释（即docstring）**，描述所需的代码，模型的任务是合成正确的代码。
* **在输入的最前面添加一个明确的指令（即instruction），解释模型应该做什么。**
* **prompt组成：instruction + docstring**

![fig3_HUMANEVALPACK_overview.png](assets/Research/fig3_HUMANEVALPACK_overview.png)
### 评估逻辑
同HumanEval
### 评估指标
同HumanEval

## 1.4 MBPP(Mostly Basic Programming Problems) [17]

* 简介：MBPP 是一个包含大约 974 个 Python 编程问题的数据集，经过众包设计，适用于初级程序员。**每个问题都提供了一个英文任务描述、一个代码解决方案和三个自动化测试用例。**  
### links
* 论文
	* Jacob Austin, Augustus Odena, Maxwell Nye, Maarten Bosma, Henryk Michalewski, David Dohan, Ellen Jiang, Carrie Cai, Michael Terry, Quoc Le, et al. 2021. Program synthesis with large language models. arXiv preprint arXiv:2108.07732 (2021).
	* [[2108.07732] Program Synthesis with Large Language Models](https://arxiv.org/abs/2108.07732)
* dataset
	* [google-research-datasets/mbpp · Datasets at Hugging Face](https://huggingface.co/datasets/google-research-datasets/mbpp)
### 规模
* size：974，人工编写
	* 该数据集有两个版本（full and sanitized）
	* full: 一些问题使用了不常见的函数签名（例如，将列表和其长度作为两个独立的参数传递给函数）、缺乏细节、存在一定模糊性（例如，“编写一个Python函数计算矩形中的正方形数量”）或在与提供的测试配对时执行了意外的操作（例如，在返回结果之前将浮点数转换为整数，而测试则进行整数比较）。
	* sanitized: 手动检查、编辑并修剪了以上部分问题，最终得到了427个经过人工验证的问题。
``` Python
dataset_full = load_dataset("mbpp")
DatasetDict({
    test: Dataset({
        features: ['task_id', 'text', 'code', 'test_list', 'test_setup_code', 'challenge_test_list'],
        num_rows: 974
    })
})

dataset_sanitized = load_dataset("mbpp", "sanitized")
DatasetDict({
    test: Dataset({
        features: ['source_file', 'task_id', 'prompt', 'code', 'test_imports', 'test_list'],
        num_rows: 427
    })
})
```
* 问题领域：编程基础、标准库函数（programming fundamentals, standard library functionality）（注：58%的问题涉及数学（例如，计算球体的体积），43%涉及列表处理，19%需要字符串处理，9%涉及整数序列，2%围绕其他数据结构的使用展开）
* PL：Python
### 单条数据组成
* source_file: 未知  
* **text/prompt: 编程任务的描述，不包含函数签名信息**
* code: 编程任务的解决方案  
* test_setup_code/test_imports: 执行测试所需的代码导入  
* test_list(3个自动化测试用例): 用于验证解决方案的测试列表  
* challenge_test_list: 用于进一步探测解决方案的更具挑战性的测试列表

mbpp-full:
``` json
{
    'task_id': 1,
    'text': 'Write a function to find the minimum cost path to reach (m, n) from (0, 0) for the given cost matrix cost[][] and a position (m, n) in cost[][].',
    'code': 'R = 3\r\nC = 3\r\ndef min_cost(cost, m, n): \r\n\ttc = [[0 for x in range(C)] for x in range(R)] \r\n\ttc[0][0] = cost[0][0] \r\n\tfor i in range(1, m+1): \r\n\t\ttc[i][0] = tc[i-1][0] + cost[i][0] \r\n\tfor j in range(1, n+1): \r\n\t\ttc[0][j] = tc[0][j-1] + cost[0][j] \r\n\tfor i in range(1, m+1): \r\n\t\tfor j in range(1, n+1): \r\n\t\t\ttc[i][j] = min(tc[i-1][j-1], tc[i-1][j], tc[i][j-1]) + cost[i][j] \r\n\treturn tc[m][n]',
    'test_list': [
        'assert min_cost([[1, 2, 3], [4, 8, 2], [1, 5, 3]], 2, 2) == 8',
        'assert min_cost([[2, 3, 4], [5, 9, 3], [2, 6, 4]], 2, 2) == 12',
        'assert min_cost([[3, 4, 5], [6, 10, 4], [3, 7, 5]], 2, 2) == 16'],
    'test_setup_code': '',
    'challenge_test_list': []
}
```

mbpp-sanitized:
```json
{
    'source_file': 'Benchmark Questions Verification V2.ipynb',
    'task_id': 2,
    'prompt': 'Write a function to find the shared elements from the given two lists.',
    'code': 'def similar_elements(test_tup1, test_tup2):\n  res = tuple(set(test_tup1) & set(test_tup2))\n  return (res) ',
    'test_imports': [],
    'test_list': [
        'assert set(similar_elements((3, 4, 5, 6),(5, 7, 4, 10))) == set((4, 5))',
        'assert set(similar_elements((1, 2, 3, 4),(5, 4, 3, 7))) == set((3, 4))',
        'assert set(similar_elements((11, 12, 14, 13),(17, 15, 14, 13))) == set((13, 14))'
        ]
}
```

### 评估逻辑及评估指标

* MBPP的合成实验（synthesis experiments）
	* **关注的是采样代码的功能正确性（functional correctness）：检查生成的代码在执行时是否能够通过一组测试用例**
	* **不关注代码质量的代理指标**，如令牌准确度或BLEU分数（第4.7节实验验证了BLUE分数与功能正确性的little correlation）。
	* 对于测试数据集中的每个问题，使用**温度采样**（温度设置为0.5）生成80个代码样本，然后执行这些样本来测试它们的语义正确性。
* MBPP的执行实验（execution experiments）
	* 检查模型是否能够产生与实际执行代码完全相同的结果。
	* 使用**贪心解码**（温度设置为0.0）生成一个最可能的单一输出，**并将其与执行代码生成的结果字符串进行比较。**
* 原文没有提语义正确性评估的详细标准。根据下图推测，应该是生成的n个代码样本中有通过的，即认为这条problem解决了。
![table2_MBPP.png](assets/Research/table2_MBPP.png)

## 1.5 MBPP+ [162]
* 简介：MBPP+ 在 MBPP [17] 的基础上，**通过去除不规范的问题和修正实现错误的问题进行了增强。MBPP+ 的测试规模也增加了 35 倍，** 用于测试集的扩展。  
### links
* 论文
	* Jiawei Liu, Chunqiu Steven Xia, Yuyao Wang, and Lingming Zhang. 2024. Is your code generated by chatgpt really correct? rigorous evaluation of large language models for code generation. Advances in Neural Information Processing Systems 36 (2024).
* dataset
	* [evalplus/mbppplus · Datasets at Hugging Face](https://huggingface.co/datasets/evalplus/mbppplus)
### 规模
* size：**378，在MBPP的基础上去除不规范问题，但自动化生成测试用例扩大了测试用例的规模**
* 问题领域：编程基础、标准库函数（programming fundamentals, standard library functionality）
* PL：Python
### 单条数据组成
* task_id
* code: 编程任务的解决方案 
* **prompt: 编程任务的描述**  
* source_file: 未知  
* test_imports: 执行测试所需的代码导入  
* test_list(3个自动化测试用例, 这是原MBPP中的3个测试用例):  用于验证解决方案的测试列表  
* **test: 扩大测试用例规模后的测试代码**： 通过使用ChatGPT来检查地面真实数据（即白盒方法），为初始化有趣的种子提供支持，然后基于这些种子，采用类型感知的变异方法（即黑盒方法）将测试输入扩展到大量样本。
### 评估逻辑及评估指标
* 使用无偏版本的 pass@k  来准确评估 LLM 合成代码的功能正确性。
* 对 26 个流行且最先进的 LLM 进行评估。
	* (i) 随机采样：在四种温度设置（{0.2, 0.4, 0.6, 0.8}）下生成 200 个程序样本；展示了每个 k ∈{1, 10, 100} 的最佳 pass@k 结果，并标明其对应的温度 T∗k。
	* (ii) 贪婪搜索解码。仅为每个任务合成一个确定性样本，并评估其 pass@1⋆ 的通过率。

## 1.6 CoNaLa [297]
* 简介：CoNaLa 包含大约 597K 个数据样本，用于评估 Python 代码生成。
	* **CoNaLa 的策划部分是从 Stack Overflow 爬取的，经过自动筛选后由注释员整理( 2,379 training and 500 test examples)。**
	* **挖掘部分是自动挖掘的，包含近 60 万个示例。**
### links
* 论文
	* Pengcheng Yin, Bowen Deng, Edgar Chen, Bogdan Vasilescu, and Graham Neubig. 2018. Learning to mine aligned code and natural language pairs from stack overflow. In Proceedings of the 15th international conference on mining software repositories. 476–486.
	* [Learning to mine aligned code and natural language pairs from stack overflow | Proceedings of the 15th International Conference on Mining Software Repositories](https://dlnext.acm.org/doi/10.1145/3196398.3196408)
	* [Learning to Mine Aligned Code and Natural Language Pairs from Stack Overflow](https://arxiv.org/pdf/1805.08949)
	* PPT：[code mining](https://pdfs.semanticscholar.org/1a53/e7446274016f737236bdd48e3ff05d966384.pdf)， [CoNaLa: The Code/Natural Language Challenge](https://conala-corpus.github.io/mining.html)
* dataset
	* [neulab/conala · Datasets at Hugging Face](https://huggingface.co/datasets/neulab/conala)
### 规模
* size：596.88K 
	* 策划部分（dataset_curated）
	``` Python
	dataset_curated = load_dataset("neulab/conala")
	DatasetDict({
	    train: Dataset({
	        features: ['question_id', 'intent', 'rewritten_intent', 'snippet'],
	        num_rows: 2379
	    })
	    test: Dataset({
	        features: ['question_id', 'intent', 'rewritten_intent', 'snippet'],
	        num_rows: 500
	    })
	})
	```
	
	* 挖掘部分（dataset_mined）
	```Python
	dataset_mined = load_dataset("neulab/conala", "mined")
	DatasetDict({
	    train: Dataset({
	        features: ['question_id', 'parent_answer_post_id', 'prob', 'snippet', 'intent', 'id'],
	        num_rows: 593891
	    })
	})
	```
* 问题领域：无细分，Stack Overflow (SO)
* PL：Python
### 单条数据组成
* 策划部分（dataset_curated）
	* question_id, int64, Stack Overflow 问题的 ID  
	* intent, string, 自然语言意图（即 Stack Overflow 问题的标题）
	* **rewritten_intent, string, 众包修订的意图，旨在更好地反映代码的完整含义，不包含函数签名信息**  
	* snippet, string, 实现该意图的代码片段
* 挖掘部分（dataset_mined）
	* question_id, int64, Stack Overflow 问题的 ID  
	* parent_answer_post_id, int64, 提取候选代码片段的回答帖子的 ID  
	* **intent, string, 自然语言意图（即 Stack Overflow 问题的标题），不包含函数签名信息**
	* snippet, string, 实现该意图的代码片段
	* id, string, 该意图/代码片段对的唯一 ID  
	* prob, float64, 挖掘模型给出的概率
### 评估逻辑与评估指标
* **关于该数据集，论文中并没有给出评估nl-to-code generation方法的有效性评估指标。**
* **这篇工作给nl-to-code的数据驱动模型创造nl和code对（要求两者之间具有细粒度对齐的平行数据）。** 通过使用两组特征从SO中挖掘高质量的对齐数据：一组是手工制作的特征，考虑了提取代码片段的结构，另一组是通过训练一个概率模型，利用神经网络捕捉NL和代码之间的关联性得到的对应特征。这些特征被输入到分类器中，来判断挖掘的NL-代码对的质量。**论文评估部分的评估对象是自动挖掘nl和code对的方法有效性。**


## 1.7 Spider [300] (pass)
* 简介：Spider 是一个大规模复杂的文本到 SQL 数据集，**覆盖了 138 个不同的领域。它包含超过 1 万个问题和 5.6K 个复杂的 SQL 查询，涉及 200 个数据库。** 该数据集旨在测试模型在 SQL 查询、数据库模式和新领域中的泛化能力。  
### links
* dataset
	* [xlangai/spider · Datasets at Hugging Face](https://huggingface.co/datasets/xlangai/spider)
	* [taoyds/spider: scripts and baselines for Spider: Yale complex and cross-domain semantic parsing and text-to-SQL challenge](https://github.com/taoyds/spider)
* 数据集网站： [Spider: Yale Semantic Parsing and Text-to-SQL Challenge](https://yale-lily.github.io//spider)
* spider的简单概括：[Text-to-SQL学习整理（八）Spider数据集介绍导语 前面的一系列博客中，我们已经了解到Text2SQL任务的 - 掘金](https://juejin.cn/post/7085557671528660999)
* spider的数据集大小说明：[Spider数据集论文研读 - 阿帆fann - 博客园](https://www.cnblogs.com/tyfann/p/15727093.html)
* 论文
	* Spider ：[[1809.08887] Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task](https://arxiv.org/abs/1809.08887)
	* [Can LLM already serve as a database interface? a big bench for large-scale database grounded text-to-SQLs | Proceedings of the 37th International Conference on Neural Information Processing Systems](https://dl.acm.org/doi/10.5555/3666122.3667957)

### 规模
* size：8034 ，人工标注
* 问题领域：138 个不同领域
* PL：SQL
### 单条数据组成
* db_id: 数据库名称  
* question: 需要转化为SQL的自然语言  
* query: 目标SQL查询  
* query_toks: 查询的令牌列表  
* query_toks_no_value: 不包含值的查询令牌列表  
* question_toks: 问题的令牌列表

### 评估逻辑及评估指标
2. Exact Match(EM): 模型预测的SQL语句必须与ground truth完全一样。
3. Execution Accuracy(EX)：模型预测的语句执行后所得的结果与ground truth一样。

## 1.8 CONCODE [113]
* 简介：CONCODE 是**一个包含来自公共 GitHub 仓库的 10 万多个 Java 类的数据集**。它提供了近乎zero-shot 的条件，可以测试模型**在未见过的自然语言标记和未见过的环境中的泛化能力**。  
### links
* 论文
	* Srinivasan Iyer, Ioannis Konstas, Alvin Cheung, and Luke Zettlemoyer. 2018. Mapping Language to Code in Programmatic Context. In Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing. 1643–1652.
	* [Mapping Language to Code in Programmatic Context - ACL Anthology](https://aclanthology.org/D18-1192/)
* dataset
	* [semeru/Text-Code-concode-Java · Datasets at Hugging Face](https://huggingface.co/datasets/semeru/Text-Code-concode-Java)
### 规模
* size：104K，自动化
* 问题领域：多领域，文中未提
* PL：Java
### 单条数据组成

* **nl(即prompt) : 结合了自然语言描述和类环境。
	* 类环境是由类中其他成员变量和成员函数提供的编程上下文。
	* 类环境中的元素通过特殊符号（如 con_elem_sep 和 con_func_sep）进行分隔。
``` java
check if details are parsed
concode_field_sep: Container parent
concode_elem_sep: boolean isParsed
concode_elem_sep: long offset
concode_elem_sep: long contentStartPosition
concode_elem_sep: ByteBuffer deadBytes
concode_elem_sep: boolean isRead
concode_elem_sep: long memMapSize
concode_elem_sep: Logger LOG
concode_elem_sep: byte[] userType
concode_elem_sep: String type
concode_elem_sep: ByteBuffer content
concode_elem_sep: FileChannel fileChannel

concode_field_sep: Container getParent
concode_elem_sep: byte[] getUserType
concode_elem_sep: void readContent
concode_elem_sep: long getOffset
concode_elem_sep: long getContentSize
concode_elem_sep: void getContent
concode_elem_sep: void setDeadBytes
concode_elem_sep: void parse
concode_elem_sep: void getHeader
concode_elem_sep: long getSize
concode_elem_sep: void parseDetails
concode_elem_sep: String getType
concode_elem_sep: void _parseDetails
concode_elem_sep: String getPath
concode_elem_sep: boolean verify
concode_elem_sep: void setParent
concode_elem_sep: void getBox
concode_elem_sep: boolean isSmallBox
```
* code：答案
``` java
boolean function () { return isParsed ; }
```


### 评估逻辑及评估指标

* 本文任务（下图所示）：**根据自然语言文档、类成员变量（名称和数据类型）以及其他类成员方法（方法名称和返回类型）生成方法的源代码推导（如上面的数据实例所示），这些构成了代码环境。**
* 论文评估了基于上下文的**代码生成任务**在测试集和开发集上的结果。
	* 参考代码和生成代码之间的精确匹配 （Exact match accuracy）
	* BLEU score
![fig2_CONCODE.png](assets/Research/fig2_CONCODE.png)


## 1.9 ODEX [273]
* 简介：ODEX 是一个开放域数据集，专注于从自然语言生成 Python 代码的执行基准。**它包含 945 对自然语言查询及其对应的 Python 代码，所有数据都来自 Stack Overflow 论坛。**
### links
* 论文
	* Zhiruo Wang, Shuyan Zhou, Daniel Fried, and Graham Neubig. 2022. Execution-based evaluation for open-domain code generation. arXiv preprint arXiv:2212.10481 (2022).
	* [[2212.10481] Execution-Based Evaluation for Open-Domain Code Generation](https://arxiv.org/abs/2212.10481)
* dataset
	* [neulab/odex · Datasets at Hugging Face](https://huggingface.co/datasets/neulab/odex)
* github
	* [zorazrw/odex: [EMNLP'23] Execution-Based Evaluation for Open Domain Code Generation](https://github.com/zorazrw/odex)
### 规模
* size：945，人工编写（其中有 1,707 个人工编写的测试用例）
* 问题领域：来自于Stack Overflow；四种不同的自然语言意图：439 个英文样本（en），90 个西班牙语样本（es），164 个日语样本（ja）和 252 个俄语样本（ru ）。
* PL：Python
### 单条数据组成
- task_id ：原始 StackOverflow 帖子的帖子 ID，样本是从该帖子构建的；
- intent ：人类注释员根据合格的重新编写的自然语言描述；
- prompt ：函数前缀（定义、输入参数等），用于正确执行代码片段；
- canonical_solution：编码问题的参考解决方案（经过人工注释员验证）；
- suffix：函数后缀（返回值，如果有的话），用于正确执行代码；
- test_start：测试函数的定义，包括程序需要的库导入；
- test：人工注释员创建的测试用例列表；
- entry_point：在评估过程中需要调用的“检查”函数名。

```
{
    'task_id': 3844801,
    'intent': "check if all elements in list `myList` are identical", 
    'prompt': "def f_3844801(myList):\n\treturn ",
    'canonical_solution': "all(x == myList[0] for x in myList)",
    'suffix': "",
    'test_start': "\ndef check(candidate):",
    'test': [
        "\n    assert candidate([1,2,3]) == False\n", 
        "\n    assert candidate([1,1,1,1,1,1]) == True\n",
        "\n    assert candidate([1]) == True\n",
        "\n    assert candidate(['k','k','k','k','k']) == True\n",
        "\n    assert candidate([None,'%$#ga',3]) == False\n"
    ],
    'entry_point': "f_3844801",
}
```


**prompt的组成如下图所示（原文提到，这一prompt构建方式是借鉴前面的1.1 HumanEval的）：将函数上下文和文档字符串拼接在一起来构造提示。文档字符串包含了 prompt（其实是上述的函数签名），suffix，NL intent和可选的单元测试。**
![ODEX_prompt.png](assets/Research/ODEX_prompt.png)

### 评估逻辑及评估指标
文章指出，同1.1 HumanEval的评估方式，即pass@k



## 1.10 CoderEval [299]
* 简介：CoderEval 是一个实用的代码生成基准，**包含 230 个 Python 和 230 个 Java 代码生成问题。它可用于评估模型在生成实际代码（不仅仅是生成独立函数）方面的表现。**  
### links
* 论文
	* Hao Yu, Bo Shen, Dezhi Ran, Jiaxin Zhang, Qi Zhang, Yuchi Ma, Guangtai Liang, Ying Li, Qianxiang Wang, and Tao Xie. 2024. Codereval: A benchmark of pragmatic code generation with generative pre-trained models. In Proceedings of the 46th IEEE/ACM International Conference on Software Engineering. 1–12.
* dataset
	* [CoderEval/CoderEval: A collection of practical code generation tasks and tests in open source projects. Complementary to HumanEval by OpenAI.](https://github.com/CoderEval/CoderEval)
* 论文阅读
	* [【论文阅读ICSE2024】CoderEval: A Benchmark of Pragmatic Code Generation with Generative Pre-trained Models-CSDN博客](https://blog.csdn.net/HeeKaai/article/details/143466194)
	* [【论文精读-大模型评估】CoderEval: A Benchmark of Pragmatic Code Generation with Generative Pre-trained Models-CSDN博客](https://blog.csdn.net/qq_43510916/article/details/136030213)
	* [从HumanEval到CoderEval: 你的代码生成模型真的work吗？-云社区-华为云](https://bbs.huaweicloud.com/blogs/416964?utm_source=luntan&utm_medium=bbs-ex&utm_campaign=other&utm_content=content)
### 规模
* size：460 , 自动化爬取工具+人工筛选+huma label+半自动化生成半人工添加测试用例
* 问题领域：github项目，按照函数可运行范围分类，self-contained, slib-runnable, plib-runnable, class-runnable, file-runnable, and project-runnable
* PL：Python, Java
* **CoderEval的创新之处**
	* 先前的datasets领域单一，仅覆盖了语言本身基础的编程知识，如数据结构操作、简单算法等；CoderEval直接来源于真实的开源项目，覆盖多个领域，从而可以全面评估代码生成。
	* **先前的datasets任务本身过于简单，参考代码均为自包含的单一函数，并未考虑复杂类型、自定义类型、三方库、跨过程调用等情况；CoderEval考虑了复杂数据类型或项目代码中开发者自定义的类型，支持面向对象特性和跨过程调用。**
### 单条数据组成
CoderEval主要由三部分组成：
* 生成任务：以函数/方法为基本单位的代码生成任务，包括任务描述（即docstring和human_label）、函数签名、参考代码（即code）、所在文件所有上下文代码（即file_content）、所在项目其他文件内容等；
* 测试代码：针对某一编程任务的单元测试，一个编程任务可能对应一到多个测试文件、一到多个测试方法（即test_name），以及附加的测试数据；
* 测试环境：由于CoderEval中的函数/方法允许使用自定义类型、调用语言标准库或三方库、调用项目中其他方法等，因此需要在配置好所在项目的环境中执行。


* ***_id**: 唯一的标识符
* **all_context**: 包含当前代码片段的上下文信息，包括导入的模块以及当前类和文件的名称。
* **code**: 代码片段的具体内容。
* **dependency**: 用于列出该函数或模块的依赖项。
* **docstring**: 函数的文档字符串，简要描述函数的功能及其参数和返回值。
* **end_lineno**: 表示代码片段在文件中的结束行号。
* **file_content**: 该字段包含了与代码片段相关的文件内容。它提供了完整的文件源代码。
* **file_path**: 代码文件的路径，相对于项目的根目录。
* **human_label**: 给定代码的标签或简要描述，提供一个更易懂的功能说明。
* **level**: 指明该代码的级别，通常表示它的复杂度或应用范围。
* **lineno**: 代码片段在文件中的起始行号。
* **name**: 函数或方法的名称。
* **oracle_context**: 描述了代码片段的 "oracle" 上下文。包括该函数使用的 API、类以及变量等信息。
* **package**: 表示该代码片段所属的包或模块。
* **project**: 表示该代码片段所属的项目。
* **test_lineno**: 用于表示与测试代码相关的行号。
* **test_name**: 用于表示与测试代码相关的函数或方法名。

### 评估逻辑
* Docker环境配置（Docker Environments）：通过基于Linux的Docker镜像构建评估环境。
* Python的评估（Evaluation for Python）：Python项目的评估步骤包括版本控制和依赖管理。通过pyenv和venv管理Python版本和虚拟环境，pip用于安装依赖，**并在运行时以生成的程序片段替换目标函数**。**测试通过直接调用生成函数的测试用例输入来完成，并将结果与期望输出进行对比。**
* Java的评估（Evaluation for Java）：Java需要在运行前进行编译。**首先将生成的函数替换到目标项目中，再进行增量编译和测试。 使用javac编译更改后的文件，并通过java命令执行生成的字节码。** 通过“-cp”参数来管理依赖关系。

### 评估指标
* Pass@K：在生成的前K个样本中，至少有一个样本能够通过所有测试用例的比例，评估模型的整体生成成功率。使用无偏估计器Pass@K降低高采样方差，保证结果的准确性。
	* 无偏估计器是一种统计方法，能够在有限采样的情况下，得到对真实比例更准确的估计，不会系统性地偏向某一方向。
* Acc@K：该指标用于评估代码生成模型在生成代码所需的**上下文依赖信息时的准确性的指标**，旨在衡量模型在生成代码时对外部依赖（如变量、类型和API调用等）信息的正确引用程度。
	* 对于每个目标函数，检查它的每个oracle_context标记，看看是否在模型生成的前K个样本中的至少一个中正确地出现，则该目标函数计为“成功”。
	* 统计所有目标函数中，符合上述条件的函数的比例，即可得到Acc@K。


## 1.11 ReCode [263]
* 简介：ReCode 作为一个全面的鲁棒性评估基准，**通过对文档字符串、函数和变量名、代码语法以及代码格式进行扰动，提供了对模型鲁棒性表现的多维评估。**  
### links
* 论文
	* Shiqi Wang, Li Zheng, Haifeng Qian, Chenghao Yang, Zijian Wang, Varun Kumar, Mingyue Shang, Samson Tan, Baishakhi Ray, Parminder Bhatia, Ramesh Nallapati, Murali Krishna Ramanathan, Dan Roth, and Bing Xiang. 2022. ReCode: Robustness Evaluation of Code Generation Models. (2022). https://doi.org/10.48550/arXiv.2212.10264
* dataset
	* [amazon-science/recode: Releasing code for "ReCode: Robustness Evaluation of Code Generation Models"](https://github.com/amazon-science/recode)
### 规模
* size：1138 ，人工编写
* 问题领域：语言理解力、算法和简单的数学
* PL：Python
### 单条数据组成

prompt的组成

### 评估逻辑

### 评估指标


代码生成模型已经取得了令人印象深刻的成绩。然而，它们往往比较脆弱，稍微修改提示词就可能导致生成结果的巨大变化；这些鲁棒性特性对于在实际应用中部署时用户体验至关重要，但目前还没有被充分理解。

现有的大部分关于文本或代码任务鲁棒性的研究都集中在分类任务上，而生成任务中的鲁棒性仍然是一个未被探索的领域，目前还没有针对代码生成的鲁棒性进行全面评估的基准。

在本文中，我们提出了 **ReCode**，这是一个综合的代码生成模型鲁棒性评估基准。
**我们为代码定制了超过30种转换，专门针对文档字符串、函数和变量名、代码语法和代码格式。** 这些转换经过精心设计，既符合现实编程实践的自然性，又能保留原始语义，从而提供多维度的评估，以衡量模型的鲁棒性表现。所有扰动都通过自动生成很好地实现，提供了简单的使用和定制。通过我们的基准测试中提供的这些扰动，用户可以了解模型鲁棒性性能的全面分析。

通过人工标注者的验证，我们确认超过90%的扰动提示词不会改变原始提示词的语义。

此外，我们还定义了代码生成模型的鲁棒性度量，考虑了每种扰动类型下的最坏情况行为，利用执行生成的代码作为客观评估依据。

我们在SOTA（最先进的）模型上演示了ReCode，在这个基准上发布了HumanEval和MBPP的扰动数据集的标准版本（函数补全任务）。有趣的观察结果包括：**CodeGen**相比**InCoder**和**GPT-J**具有更好的鲁棒性；模型对于语法扰动最为敏感；在**MBPP**上的鲁棒性评估比**HumanEval**更具挑战性。



```json
{"task_id": "HumanEval/0", 
"prompt": "from typing import List\n\n\ndef has_close_elements(numbers: List[float], threshold: float) -> bool:\n    \"\"\" Check if in given list of numbers, are any two numbers closer to each other than\n    given threshold.\n    >>> has_close_elements([1.0, 2.0, 3.0], 0.5)\n    False\n    >>> has_close_elements([1.0, 2.8, 3.0, 4.0, 5.0, 2.0], 0.3)\n    True\n    \"\"\"\n\n    for idx, elem in enumerate(numbers):\n        for idx2, elem2 in enumerate(numbers):\n            if idx != idx2:\n\n                distance = abs(elem - elem2)\n\n", 

"entry_point": "has_close_elements", 
"canonical_solution": "                if distance < threshold:\n                    return True\n\n    return False\n", 
"test": "\n\nMETADATA = {\n    'author': 'jt',\n    'dataset': 'test'\n}\n\n\ndef check(candidate):\n    assert candidate([1.0, 2.0, 3.9, 4.0, 5.0, 2.2], 0.3) == True\n    assert candidate([1.0, 2.0, 3.9, 4.0, 5.0, 2.2], 0.05) == False\n    assert candidate([1.0, 2.0, 5.9, 4.0, 5.0], 0.95) == True\n    assert candidate([1.0, 2.0, 5.9, 4.0, 5.0], 0.8) == False\n    assert candidate([1.0, 2.0, 3.0, 4.0, 5.0, 2.0], 0.1) == True\n    assert candidate([1.1, 2.2, 3.1, 4.1, 5.1], 1.0) == True\n    assert candidate([1.1, 2.2, 3.1, 4.1, 5.1], 0.5) == False\n\n", 

???"partial": "from typing import List\n\n\ndef has_close_elements(numbers: List[float], threshold: float) -> bool:\n    \"\"\" Check if in given list of numbers, are any two numbers closer to each other than\n    given threshold.\n    >>> has_close_elements([1.0, 2.0, 3.0], 0.5)\n    False\n    >>> has_close_elements([1.0, 2.8, 3.0, 4.0, 5.0, 2.0], 0.3)\n    True\n    \"\"\"\n    for idx, elem in enumerate(numbers):\n        for idx2, elem2 in enumerate(numbers):\n            if idx != idx2:\n                distance = abs(elem - elem2)\n                # print('@@this is the line to split##')\n                if distance < threshold:\n                    return True\n\n    return False\n"}

```



* task_id：任务的ID
* prompt：表示题目（通常直接请求大模型获取答案）
	* **"prompt":库+函数签名+文档字符串（功能描述，输入输出样例）+函数主体**
	``` Python
	from typing import List
	def has_close_elements(numbers: List[float], threshold: float) -> bool:
	    """
	    Check if in given list of numbers, are any two numbers closer to each other than given threshold.
	
    >>> 	has_close_elements([1.0, 2.0, 3.0], 0.5)
	    False
	
    >>> 	has_close_elements([1.0, 2.8, 3.0, 4.0, 5.0, 2.0], 0.3)
	    True
	    """
	```
* entry_point：唯一标记
* canonica_solution：参考答案
* test：测试单元



## 1.12 StudentEval [19]
* 简介：StudentEval 是一个包含 1,749 个提示的数据库，涵盖 48 个问题，由 80 名仅完成过一个学期 Python 编程课程的学生编写。**与其他许多基准不同，每个问题有多个提示，并且同一参与者进行了多次尝试，每个问题还附带了一组由教师编写的测试用例。**
### links
* 论文
	* Hannah McLean Babe, Sydney Nguyen, Yangtian Zi, Arjun Guha, Molly Q Feldman, and Carolyn Jane Anderson. 2023. StudentEval: A Benchmark of Student-Written Prompts for Large Language Models of Code. arXiv:2306.04556 [cs.LG]
	* [StudentEval: A Benchmark of Student-Written Prompts for Large Language Models of Code - ACL Anthology](https://aclanthology.org/2024.findings-acl.501/)
* dataset
	* [wellesley-easel/StudentEval · Datasets at Hugging Face](https://huggingface.co/datasets/wellesley-easel/StudentEval)
### 规模
* size：1749 ，人工编写prompt（强调说是非专家编写，由只学习python一个月的学生编写）+ 专家编写测试用例
* 问题领域
	* 这些提示是通过 48 个适合初学者的问题收集的，每个问题都有多个不同的提示（平均有36个）。
	* 数据来自cs教材：列表、循环、字符串、条件语句、数学、嵌套数据、排序和字典
* PL：Python
* **这是第一个针对每个问题有多个提示词且同一参与者有多次尝试的数据集。** **论文旨在探索如何编写“好的”提示，帮助 Code LLM 更好地与非专家程序员对接。**  为每个问题-参与者对识别了四个关键的、不重叠的子集：
	- **First Success**：第一次尝试生成了正确的代码。
	- **First Failure**：第一次尝试未能成功，且参与者转向下一个问题。
	- **Last Success**：最后一次尝试生成了正确的代码。
	- **Last Failure**：最后一次尝试未能成功，且参与者转向下一个问题。
### 单条数据组成

* problem：问题名称，和entrypoint基本上一致
* entrypoint：函数名
* assertions：3个左右的专家编写的测试用例
* prints：输出函数结果
* username：编写prompt的学生
* **submitted_text：学生编写的注释**
* tests_passed：本学生编写的prompt，得到的code的测试用例通过数目
* total_tests：测试用例总数
* **prompt：函数签名+submitted_text**
* completion： 正确code答案
* first_attempt：
* last_attempt
* is_success
* is_first_success
* is_last_success
* is_first_failure
* is_last_failure
* __ index_level_0__: 下标
### 评估逻辑

* 使用 **STUDENTEVAl** 来评估 12 个 Code LLM，并发现 **STUDENTEVAl** 是一个比现有基准更好的模型性能区分器。
	* 使用了没有文档字符串的函数签名生成了 Codex (Chen et al., 2021) 的结果，并测量了平均 pass@1 率。
	* 验证每个问题的测试用例。
		* 使用测试覆盖率和突变测试来衡量 **STUDENTEVAl** 的测试用例质量。力求在全面性和可理解性之间找到平衡：每个问题都有 3-4 个测试用例，能够实现 100% 的代码覆盖率。
### 评估指标
* 通过反复采样这些提示词的完成结果来计算每个子集的 **pass@k** 评分，从而将这些提示词作为基准进行评估。


## **1.13 BigCodeBench** [333]
* 简介：**BigCodeBench 包含 1,140 个复杂的 Python 编程任务，涵盖了来自 139 个流行库的 723 个函数调用，跨越 7 个领域。 该基准专门设计用于评估 LLM 在跨领域库中调用多个函数并遵循复杂指令解决编程任务的能力，帮助弥合孤立编码练习与实际编程场景之间的评估差距。**  
### links
* 论文
	* Terry Yue Zhuo, Minh Chien Vu, Jenny Chim, Han Hu, Wenhao Yu, Ratnadira Widyasari, Imam Nur Bani Yusuf, Haolan Zhan, Junda He, Indraneil Paul, et al. 2024. Bigcodebench: Benchmarking code generation with diverse function calls and complex instructions. arXiv preprint arXiv:2406.15877 (2024).
	* [[2406.15877] BigCodeBench: Benchmarking Code Generation with Diverse Function Calls and Complex Instructions](https://arxiv.org/abs/2406.15877)
* dataset
	* [bigcode/bigcodebench · Datasets at Hugging Face](https://huggingface.co/datasets/bigcode/bigcodebench)
* 论文解读
	* [BigCodeBench: 继 HumanEval 之后的新一代代码生成基准测试](https://huggingface.co/blog/zh/leaderboard-bigcodebench)
### 规模
* size：1140，以[ODEX](https://github.com/zorazrw/odex)作为“种子数据集”+人工指导 GPT-4 将这些一行代码扩展为全面的函数级任务。
* 问题领域：Computation (63%)，General (44%)，Visualization (31%)，System (30%)，Time (10%)，Network (8%)，Cryptography (5%)
* PL：Python
* 关于数据的详细数量指标（长度，测试用例数目等等），选取的函数的信息等等，见[bigcode/bigcodebench · Datasets at Hugging Face](https://huggingface.co/datasets/bigcode/bigcodebench)
* **评估LLMs解决实际和具有挑战性的编程任务的能力。**
### 单条数据组成
该数据集有 2 个变体：
- **BigCodeBench-Complete**：基于结构化文档字符串的代码补全。
- **BigCodeBench-Instruct**：基于面向自然语言指令的代码生成。

下面是相关的fields：
- **task_id (string)**：任务的唯一标识符。
- **complete_prompt (string)**：**基于PEP257结构的文档字符串提示。即prompt=intent+args+returns+requirements+examples**
``` Python
import itertools
from random import shuffle

def task_func(numbers=list(range(1, 3))):
    """
    Calculates the average of the sums of absolute differences between each pair of consecutive numbers 
    for all permutations of a given list. Each permutation is shuffled before calculating the differences.

    Args:
        - numbers (list): A list of numbers. Default is numbers from 1 to 10.

    Returns:
        float: The average of the sums of absolute differences for each shuffled permutation of the list.

    Requirements:
        - itertools
        - random.shuffle

    Example:
        >>> result = task_func([1, 2, 3])
        >>> isinstance(result, float)
        True
    """
```
- **instruct_prompt (string)**：面向自然语言的指令提示。
- **canonical_solution (string)**：标准解决方案（无注释）。
- **code_prompt (string)**：仅包含代码的提示。
- **test (string)**：用于测试的代码片段，包装在 `unittest.TestCase` 类中。Bench基准测试包含5.6个测试用例，并且有99%的平均分支覆盖率。
- **entry_point (string)**：代码片段的入口点，即 `task_func`。
- **doc_struct (string[dictionary])**：结构化的文档字符串。
	- **description (string)**：任务的主要描述，使用自然语言编写。
	- **note (string)**：任务的附加备注，使用自然语言编写。
	- **reqs (string, optional)**：任务解决方案中可以使用的模块。
	- **params (string, optional)**：任务解决方案中使用的参数。
	- **returns (string, optional)**：任务解决方案中返回的值。
	- **raises (string, optional)**：任务解决方案中应抛出的异常。
	- **examples (string, optional)**：用于提示任务解决方案的交互式 Python 示例。
- **libs (string)**：任务解决方案中可以使用的库。

![BigCodeBench_prompt.png](assets/Research/BigCodeBench_prompt.png)

### 评估逻辑
* Pass@1 是评估整体表现的好指标。
* Elo 评分：使用 Elo 评分（[Chatbot Arena](https://lmsys.org/blog/2023-05-03-arena/)）来对`BigCodeBench-Complete`上的模型进行排名。将每个任务视为一场比赛，每个模型视为一个玩家。Elo 评分更新基于比赛结果和预期，使用任务级校准 Pass@1（0%或100%），排除平局。从初始 Elo 评分1000开始，使用最大似然估计和500次自举来获得最终分数。的·
### 评估指标
* 使用贪婪解码的 Pass@1，测量通过精心设计的测试用例生成的第一个代码片段正确解决任务的百分比。

## 1.14 ClassEval [72]
  * 简介：**ClassEval 是一个手工制作的基准，包含 100 个类和 412 个方法，用于评估 LLM 在类级别代码生成场景中的表现。** 特别地，ClassEval 的任务样本具有较高的复杂性，涉及长代码生成和复杂的文档字符串信息，因此有助于评估 LLM 在生成复杂代码方面的能力。
### links
* 论文
	* Xueying Du, Mingwei Liu, Kaixin Wang, Hanlin Wang, Junwei Liu, Yixuan Chen, Jiayi Feng, Chaofeng Sha, Xin Peng, and Yiling Lou. 2024. Evaluating large language models in class-level code generation. In Proceedings of the IEEE/ACM 46th International Conference on Software Engineering. 1–13.
	* [[2408.16498] A Survey on Evaluating Large Language Models in Code Generation Tasks](https://arxiv.org/abs/2408.16498)
* dataset
	* [FudanSELab/ClassEval · Datasets at Hugging Face](https://huggingface.co/datasets/FudanSELab/ClassEval)
### 规模
* size：包含 100 个类级 Python 编程任务，共有 100 个类和 **412 个方法**，每个类平均 33.1 个测试用例；人工编写
* 问题领域：管理系统、数据格式化、数学运算、游戏开发、文件处理、数据库操作和自然语言处理等
* PL：Python
* **评估 LLM 在类级别代码生成场景中的表现**
### 单条数据组成
每个任务的具体数据字段如下所示：
- **task_id**：每个任务的唯一标识符。
- **skeleton**：
	- **类的骨架，包括我们在类级别编码任务中的所有输入描述。**
	- **是prompt的一部分，不过相比其他datatset，是以类为粒度的代码生成任务。**
	- **skeleton=dependency+class constructor+method functionality+Method Parameter+Method Reture Value+example**
![ClassEval_fig2.png](assets/Research/ClassEval_fig2.png)
- **test**：整个类的所有测试用例。
- **solution_code**：每个任务的真实类级别代码。

从类骨架中获取的更细粒度的类级信息，包括：
- **import_statement**：每个任务的导入语句，和前面的skeleton的加载库一致。
- **class_name**：类的名称。
- **class_description**：类的简要描述，说明类的目的和功能。
- **class_constructor**：类的完整构造函数。
- **fields**：在类构造函数中定义的字段。

每个方法的详细信息位于“methods_info”字段中，包括：
- **method_name**：方法签名。
- **method_input**：方法契约设计，包括方法中的所有输入描述。
- **test_code**：方法的测试用例。
- **solution_code**：方法的真实代码。
- **dependencies**：方法的依赖信息。

给定一个类级别的代码生成任务，每个模型设计了三种不同生成策略。
- **整体生成（Holistic Generation）**：模型被要求一次性生成整个类，输入为类的骨架。
- **增量生成（Incremental Generation）**：模型被要求以逐方法的方式生成类。每次迭代基于之前迭代中生成的方法体。该迭代过程会重复，直到类中的所有方法都生成完毕。
- **组合生成（Compositional Generation）**：模型被要求以逐方法的方式生成类。每次迭代是独立的，不考虑其他已生成的方法。最后，将所有生成的方法组合起来形成完整的类。

针对上面的三种生成策略，其prompt template如下(上述三种分别为Instruction-H, Instruction-I, Instruction-C)。
```txt
System Prompt: Provided below is an instruction detailing a task. Compose a response that aptly fulfills the request. 

Instruction-H: Please complete the class ${Class Name} in the subsequent code. ${Class Skeleton} 

Instruction-I: Please complete the method ${Method Name} within the following class ${Class Name}. ${Class-level Info}${Generated Methods with Contract Designs}${TargetMethod Contract Design}  

Instruction-C:Please complete the method${MethodName} within the following class ${Class Name}. ${Class-level Info} ${Other Method Signatures} ${Target Method Contract Design}
```
### 评估逻辑及评估指标
* Pass@k 指标
	* 目的：计算代码正确性
	* 类级 Pass@k： 关注类粒度的代码样本。如果一个类级代码样本通过了所有的方法级和类级测试用例，则认为它是正确的。
	* 方法级 Pass@k： 关注方法粒度的代码样本。如果一个方法级样本通过了所有的方法级测试用例，则认为它是正确的。
	* 设置：n=5，𝑘 = {1,3,5}，也是采用的pass@k的无偏估计方法。
* **DEP**
	* 目的：衡量模型生成与上下文相关代码的能力（即调用类中声明的其他方法，或类中的字段）。
	* 定义：该指标计算每个方法生成的依赖项的百分比，与标准解决方案方法中实际的依赖项数量进行比较。
	* ![ClassEval_DEP.png](assets/Research/ClassEval_DEP.png)
	* 方法依赖项 **DEP(M)** 
	* 字段依赖项 **DEP(F)**
	* 𝐺𝑖(𝑀/𝐹) 是第 𝑖 个方法中生成的 方法/字段 依赖项的数量
	* 𝑆𝑖(𝑀/𝐹) 是标准解决方案中第 𝑖 个方法的实际 方法/字段 依赖项的数量。
## 1.15 NaturalCodeBench [314]
* 简介：NaturalCodeBench 是一个综合性的代码基准，包含 402 个高质量的 Python 和 Java 编程问题。这些问题来源于在线编码服务中的自然用户查询，涵盖了 6 个不同的领域，塑造了一个与实际应用相一致的评估环境。
### links
* 论文
	* Shudan Zhang, Hanlin Zhao, Xiao Liu, Qinkai Zheng, Zehan Qi, Xiaotao Gu, Xiaohan Zhang, Yuxiao Dong, and Jie Tang. 2024. NaturalCodeBench: Examining Coding Performance Mismatch on HumanEval and Natural User Prompts. arXiv preprint arXiv:2405.04520 (2024).
	* [[2405.04520] NaturalCodeBench: Examining Coding Performance Mismatch on HumanEval and Natural User Prompts](https://arxiv.org/abs/2405.04520)
* dataset
	* [THUDM/NaturalCodeBench: NaturalCodeBench (Findings of ACL 2024)](https://github.com/THUDM/NaturalCodeBench)
### 规模
* size：402，自动化获取及过滤高质量nl问题+semi-automated solution and test cases generation（LLM+human-annotated）
	* ![NaturalCodeBench_overview.png](assets/Research/NaturalCodeBench_overview.png)
* 问题领域：Front-End, Algorithm, Data Science, Artificial Intelligence, Software Engineering, System Administration（前端、算法与数据结构、数据科学、人工智能、软件工程和系统管理）
* 问题领域的详细占比，参考[THUDM/NaturalCodeBench: NaturalCodeBench (Findings of ACL 2024)](https://github.com/THUDM/NaturalCodeBench)
* PL：Python，Java(中英文双语)
### 单条数据组成
- **_id**（整数）：每个问题的唯一标识符。
- **prompt**（字符串）：**包含问题描述和指令的提示。如下图所示。**
- **problem**（字符串）：问题描述。
- **testcases**（字符串）：测试用例的代码。
- **setup_code**（字符串）：测试环境设置的代码。
- **reference_solution**（字符串）：解决问题的参考答案。
- **classification**（字符串）：问题所属的领域。

![NaturalCodeBench_fig4.png](assets/Research/NaturalCodeBench_fig4.png)
### 评估逻辑及评估指标
* pass@k：生成代码的功能正确性
	* 贪心搜索解码，温度为0：k = 1
	* 随机采样：采样温度设置为 0.2，top-p 设置为 0.9。展示模型在每个 k ∈ {10,50} 时的最佳 pass@k 结果。
* 代码覆盖率：评估测试用例有效性和质量。


# 2 Repository

## 2.1 RepoEval [309]
* 简介：**RepoEval 用于评估代码库级别的代码补全。** 通过使用单元测试，它能够提供不同粒度的评估，并提高评估的准确性。
### links
* 论文
	* Fengji Zhang, Bei Chen, Yue Zhang, Jacky Keung, Jin Liu, Daoguang Zan, Yi Mao, Jian-Guang Lou, and Weizhu Chen. 2023. RepoCoder: Repository-Level Code Completion Through Iterative Retrieval and Generation. In Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing. 2471–2484.
	* [[2306.03091] RepoBench: Benchmarking Repository-Level Code Auto-Completion Systems](https://arxiv.org/abs/2306.03091)
* 论文笔记
	* [RepoCoder：Repository-Level Code Completion Through Iterative Retrieval and Generation笔记_repocoder: repository-level code completion throug-CSDN博客](https://blog.csdn.net/chinaren69fy/article/details/136259588)
* dataset
	* https://paperswithcode.com/dataset/repoeval
* 工具github
	* [RepoCoder/README.md at main · Banana-Boat/RepoCoder](https://github.com/Banana-Boat/RepoCoder/blob/main/README.md
### 规模
* size：3573 ，自动获取（包括存储库中的单元测试）
* 问题领域：来自Github的最新高质量存储库，涵盖了三个级别的代码完成粒度：行、API调用和函数体。
* PL：Python, Java
### 单条数据组成
prompt的组成：没找到相关数据集
![RepoEval_fig3.png](assets/Research/RepoEval_fig3.png)
* RepoCoder 框架从代码库中检索出最相关的代码示例，记作 Cret，并将其与未完成的代码 X 连接起来。
* 检索到的代码片段根据它们与查询的相似度得分按升序排列。每个代码片段都附有其原始文件路径，且提示中包含的最大代码片段数量，记作 K，取决于可用的提示长度。

### 评估逻辑及评估指标
* Exact Match(EM)
* Edit Similarity(ES)
## 2.2 Stack-Repo (太少了，pass)[239]
* 简介：Stack-Repo 是一个包含 200 个来自 GitHub 的 Java 仓库数据集，这些仓库的文件经过去重处理。这些文件通过三种类型的仓库上下文进行增强：提示建议上下文、基于 BM25 相似度得分的 BM25 上下文，以及通过嵌入模型表示空间中的最近邻获取的 RandomNN 上下文。  
### links
* 论文
	* Disha Shrivastava, Denis Kocetkov, Harm de Vries, Dzmitry Bahdanau, and Torsten Scholak. 2023. RepoFusion: Training Code Models to Understand Your Repository. arXiv preprint arXiv:2306.10998 (2023).
* 论文解析
	* [蚂蚁 CodeFuse 代码大模型技术解析：基于全仓库上下文的代码补全-阿里云开发者社区](https://developer.aliyun.com/article/1580069)
* dataset
	* [RepoFusion/Stack-Repo · Datasets at Hugging Face](https://huggingface.co/datasets/RepoFusion/Stack-Repo)
### 规模
* size：200
* 问题领域：
* PL：Java
### 单条数据组成

prompt的组成

### 评估逻辑

### 评估指标





## 2.3 Repobench [167]
* 简介：Repobench 是一个专门用于评估代码库级别代码自动补全系统的基准，支持 Python 和 Java。**它包括三个相互关联的评估任务：RepoBench-R（检索）、RepoBench-C（代码补全）和 RepoBench-P（管道）。**
### links
* 论文
	* Tianyang Liu, Canwen Xu, and Julian McAuley. 2023. Repobench: Benchmarking repository-level code autocompletion systems. arXiv preprint arXiv:2306.03091 (2023).
* dataset
	* [Leolty/repobench: ✨ RepoBench: Benchmarking Repository-Level Code Auto-Completion Systems - ICLR 2024](https://github.com/Leolty/repobench)
	* [tianyang/repobench_python_v1.1 · Datasets at Hugging Face](https://huggingface.co/datasets/tianyang/repobench_python_v1.1)
### 规模
* size：27k
* 问题领域：
* PL：Python, Java
### 单条数据组成
数据集中的数据：
- **repo_name** (字符串)：仓库的名称
- **file_path** (字符串)：当前文件的路径
- **context** (列表)：可能对写下一行代码有帮助的跨文件代码片段：
    - **identifier** (字符串)：代码片段的标识符
    - **path** (字符串)：代码片段的路径
    - **snippet** (字符串)：代码片段
    - **import_statement** (字符串)：当前文件的导入语句
    - **cropped_code** (字符串)：当前文件的裁剪代码（最多前120行）
    - **all_code** (字符串)：当前文件的全部代码（未裁剪）
    - **next_line** (字符串)：下一行代码（作为目标）
    - **gold_snippet_index** (整数)：黄金片段在上下文中的索引（将用于下一行，仅供参考，不应在下一行预测中使用）
    - **created_at** (字符串)：仓库的创建时间
    - **level** (字符串)：下一行代码完成的级别，通过整个提示（包括所有上下文、导入语句、裁剪代码以及一些必要的分隔符标记）的 token 数量来衡量


**根据论文构建提示的官方，可知其prompt结构如下：**

根据给定的 `construct_prompt` 函数，最终构建的 `prompt` 是用于下一行代码预测的字符串。该函数的结构可以分为以下几个主要部分：

1. **函数参数解释**：
    - `data`: 包含来自数据集的输入数据的字典，包含信息如仓库名称、文件路径、上下文等。
    - `language`: 编程语言，默认为 Python，影响注释符号。
    - `tokenizer`: 用于评估模型的分词器，帮助计算令牌数量。
    - `max_token_nums`: 构建提示时的最大令牌数，用于确保生成的提示不超过模型的最大令牌数限制。
2. **注释符号选择**：
    - 根据编程语言选择注释符号。若语言为 `python`，则使用 `#`，否则使用 `//`。
3. **构建跨文件（cross-file）提示**：
    - 提示首先包括仓库名称，并对每个代码片段（`context`）进行处理。每个代码片段显示其路径和代码内容。
4. **构建当前文件（in-file）提示**：
    - 提示包括文件路径、导入语句和裁剪的代码。裁剪的代码是指在当前文件中，最多保留120行的代码。
5. **截断提示以满足令牌数限制**：
    - 如果指定了 `tokenizer` 和 `max_token_nums`，则会根据给定的最大令牌数限制来截断提示。首先计算 `cross_file_prompt` 和 `in_file_prompt` 的令牌数量，确定是否需要截断。
    - 截断策略是：从 `cross_file_prompt` 中删除行，直到总令牌数小于或等于 `max_token_nums`。
6. **合并跨文件提示和当前文件提示**：
    - 最终的提示由 `cross_file_prompt` 和 `in_file_prompt` 合并而成。跨文件提示与当前文件提示之间通过空行分隔。
7. **规范化空行**：
    - 使用正则表达式 `re.sub(r'\n{4,}', '\n\n', prompt)` 来将多个连续的空行规范化为两个空行，确保提示结构整洁。

最终构建的 `prompt` 包括以下部分：
- **跨文件提示（cross-file prompt）**：包含仓库名称、每个代码片段的路径和代码内容。
- **当前文件提示（in-file prompt）**：包含当前文件的路径、导入语句和裁剪的代码。
- **令牌数控制**：根据模型的最大令牌数限制，可能对跨文件提示进行截断。
- **空行规范化**：确保没有多余的空行。
![Repobench_fig1.png](assets/Research/Repobench_fig1.png)
### 评估逻辑

### 评估指标







## 2.4 EvoCodeBench [144]
* 简介：EvoCodeBench 是一个进化式代码生成基准，通过严格的流程构建，并与实际的代码库对齐。该基准还提供了全面的注释和强大的评估指标。
### links

### 规模
* size：164，人工编写
* 问题领域：语言理解力、算法和简单的数学
* PL：
### 单条数据组成

prompt的组成

### 评估逻辑

### 评估指标
## 2.5 SWE-bench [123]
* 简介：SWE-bench 是一个数据集，用于测试模型自动解决 GitHub 问题的能力。该数据集包含来自 12 个流行 Python 仓库的 2,294 对问题-拉取请求。
### links

### 规模
* size：164，人工编写
* 问题领域：语言理解力、算法和简单的数学
* PL：
### 单条数据组成

prompt的组成

### 评估逻辑

### 评估指标
## **2.6 CrossCodeEval** [68]
* 简介：CrossCodeEval 是一个多样化的多语言范围补全数据集，涵盖了四种编程语言：Python、Java、TypeScript 和 C#。**该基准测试了模型理解跨文件信息的深度能力**，并准确完成代码的能力。
### links
* 论文
	* Yangruibo Ding, Zijian Wang, Wasi Ahmad, Hantian Ding, Ming Tan, Nihal Jain, Murali Krishna Ramanathan, Ramesh Nallapati, Parminder Bhatia, Dan Roth, et al. 2024. Crosscodeeval: A diverse and multilingual benchmark for cross-file code completion. Advances in Neural Information Processing Systems 36 (2024). 
	* [CrossCodeEval](https://crosscodeeval.github.io/)
* 论文解析
	* [CrossCodeEval:仓库级别代码补全评估 - 知乎](https://zhuanlan.zhihu.com/p/673151089)
* dataset
	* [amazon-science/cceval: CrossCodeEval: A Diverse and Multilingual Benchmark for Cross-File Code Completion (NeurIPS 2023)](https://github.com/amazon-science/cceval)
### 规模
* size：10K，自动化收集工具
* 问题领域：
* PL：Python, Java, TypeScript, C#
* CROSSCODEEVAL严格要求**跨文件上下文才能正确补全缺失的代码**，而不是只需使用当前文件的上下文即可预测正确答案。
### 单条数据组成
每个数据划分包含一个独立的仓库文件夹，每个仓库文件夹包含该仓库中所有的 .java 源代码文件，并保留原始的目录结构，**同时还包括三个 .json 文件，分别对应于 PP、BM25 和 RandomNN 仓库上下文。** 在 HuggingFace Datasets 的术语中，有四个子数据集或配置：
- PP_contexts：提示提案（Prompt Proposal）仓库上下文。
- bm25_contexts：BM25 仓库上下文。
- randomNN_contexts：RandomNN 仓库上下文。
- sources：实际的 Java (.java) 源代码文件。


单条这段JSON数据包含了几个键（key）以及它们对应的内容。
8. **prompt**: 这是一个字符串，包含了用于生成Python代码的提示文本。它描述了一个函数和其行为的详细要求，并提供了如何处理代码片段的指示。`prompt`字段内包含了用于代码生成的完整代码框架，和一些辅助性描述、设置等。
![CrossCodeEval_fig3.png](assets/Research/CrossCodeEval_fig3.png)
9. **groundtruth**: 参考答案的代码部分。这部分代码代表着预期的正确行为。
10. **right_context**: 这是字符串类型，表示在生成代码时，提供的正确的参考上下文信息。在这里，`right_context`包含了代码生成过程中紧接着`groundtruth`后续的部分，帮助模型理解后续的代码逻辑。
11. **metadata**: 这是一个对象，包含了以下子字段：
    - **task_id**: 任务的唯一标识符，用于标识这个特定的代码生成任务。
    - **repository**: 与此任务相关的代码仓库名称。
    - **file**: 执行代码生成任务的具体文件名。
    - **context_start_lineno**: 在原始代码中，给定上下文的起始行号，标识该部分代码的起始位置。
    - **groundtruth_start_lineno**: 参考答案（groundtruth）部分代码的起始行号，标识该部分代码的起始位置。
    - **right_context_start_lineno**: 对应右侧上下文的起始行号，表示该部分代码（right context）的位置。

### 评估逻辑及评估指标
- **代码匹配**：直接比较补全的代码与 reference代码,评估代码完成过程的整体准确性，考虑到标识符、关键字、操作符、分隔符和文字等元素。  
	- 精确匹配（EM）
	- 编辑相似度（ES）
- **标识符匹配**：该指标评估模型预测正确应用编程接口（API）的能力。标识符是不包含注释、关键字和字符串等其他字符。
	- 解析代码并从模型预测和 reference中提取标识符，得到两个有序的标识符列表。
	- 将预测的标识符与 reference进行比较，并以EM和F1分数的形式报告结果。



## 2.7 SketchEval [308]
* 简介：SketchEval 是一个面向仓库的基准，涵盖了来自 19 个不同仓库的数据，这些仓库在复杂性上各不相同。除了数据集，SketchEval 还引入了一种度量标准，称为 SketchBLEU，用于衡量两个仓库在结构和语义上的相似性。
### links
* 论文
	* Daoguang Zan, Ailun Yu, Wei Liu, Dong Chen, Bo Shen, Wei Li, Yafen Yao, Yongshun Gong, Xiaolin Chen, Bei Guan, et al. 2024. CodeS: Natural Language to Code Repository via Multi-Layer Sketch. arXiv preprint arXiv:2403.16443 (2024).
* dataset
	* 
### 规模
* size：164，人工编写
* 问题领域：语言理解力、算法和简单的数学
* PL：
### 单条数据组成

prompt的组成

### 评估逻辑

### 评估指标
