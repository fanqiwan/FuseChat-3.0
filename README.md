# FuseChat-3.0: Preference Optimization for Implicit Model Fusion

In this blog post, we introduce FuseChat-3.0, a series of models designed to elevate model performance by fusing the strengths of multiple source models into smaller target models. Our approach involves using three mainstream smaller models—Llama-3.1-8B-Instruct, Gemma2-9B-it, and Qwen2.5-7B-Instruct—and two even smaller models—Llama-3.2-3B-Instruct and Llama-3.2-1B-Instruct—as target models. To achieve fusion, we utilized four powerful open-source source models: Gemma-2-27B-it, Mistral-Large-Instruct-2407, Qwen2.5-72B-Instruct, and Llama-3.1-70B-Instruct. The process employed a two-stage training pipeline comprising Supervised Fine-Tuning (SFT) and Direct Preference Optimization (DPO). The resulting FuseChat-3.0 models demonstrated substantial improvements in general conversation, instruction following, mathematics, and coding tasks. Notably, when Llama-3.1-8B-Instruct was the target model, our fusion approach delivered a 6.8-point average improvement across 14 benchmarks. We release the [FuseChat-3.0](https://huggingface.co/FuseAI) models on Huggingface, stay tuned for the forthcoming dataset and code.

## Overview

Combining the strengths of multiple large language models (LLMs) offers a promising pathway for enhancing individual model capabilities. Previous iterations of the FuseChat series adopted explicit model fusion (EMF), leveraging probabilistic distribution matrices generated by source models to transfer knowledge to target models. While effective, this method posed challenges, including vocabulary alignment and the merging of diverse distribution matrices, which could introduce noise and errors. Inspired by preference alignment techniques such as Direct Preference Optimization (DPO) and Simple Preference Optimization (SimPO), we developed implicit model fusion (IMF) in FuseChat-3.0. This approach introduces preference optimization to implicitly transfer the capabilities of source models into target models. Specifically, our method consists of a three-stage recipe, including dataset construction, supervised fine-tuning (SFT), and direct preference optimization (DPO). We will detail the overall pipeline in the following paragraph.

## Dataset Construction

### Prompt Selection

Our datasets were designed to enhance model's instruction following, general conversation, mathematics, coding, and Chinese-language capabilities. We selected data from open-source community datasets, applying targeted filtering and preprocessing. Key datasets and filtering criteria included:

- **Instruction Following & General Conversation**: Sourced from [**UltraFeedback**](https://huggingface.co/datasets/openbmb/UltraFeedback), [**Magpie-Pro-DPO-100K-v0.1**](https://huggingface.co/datasets/Magpie-Align/Magpie-Pro-DPO-100K-v0.1), and [**HelpSteer2**](https://huggingface.co/datasets/nvidia/HelpSteer2), excluding code and math data.
- **Mathematics**: Selected from [**OpenMathInstruct-2**](https://huggingface.co/datasets/nvidia/OpenMathInstruct-2), with nearly 60,000 unique samples.
- **Coding**: Curated from [**leetcode**](https://huggingface.co/datasets/greengerong/leetcode) and [**self-oss-instruct-sc2-exec-filter-50k**](https://huggingface.co/datasets/bigcode/self-oss-instruct-sc2-exec-filter-50k), retaining prompts with test cases.
- **Chinese Language**: Integrated [**alpaca_gpt4_zh**](https://huggingface.co/datasets/llamafactory/alpaca_gpt4_zh) and [**Magpie-Qwen2-Pro-200K-Chinese**](https://huggingface.co/datasets/Magpie-Align/Magpie-Qwen2-Pro-200K-Chinese), filtering out code and math prompts to retain approximately 10,000 high-quality samples.


### Sampling

For each dataset's prompts, we synthesized responses mainly from four different series of source models, specifically [gemma-2-27b-it](https://huggingface.co/google/gemma-2-27b-it), [Mistral-Large-Instruct-2407](https://huggingface.co/mistralai/Mistral-Large-Instruct-2407), [Qwen2.5-72B-Instruct](https://huggingface.co/Qwen/Qwen2-72B-Instruct), and [Llama-3.1-70B-Instruct](https://huggingface.co/meta-llama/Llama-3.1-70B-Instruct). For instruction following and general conversation data, we sampled each prompt five times from all the source models. For code data, we sampled each prompt eight times for all source models. For Chinese data, we included single response sampled exclusively from Qwen2.5-72B-Instruct.  For math data (OpenMathInstruct-2), we retained the responses generated by Llama-3.1-405B-Instruct from the original dataset and additionally sampled responses using [Qwen2.5-Math-72B-Instruct](https://huggingface.co/Qwen/Qwen2.5-Math-72B-Instruct). The sampling parameters for different models are detailed in Table 2.

|       **Source LLMs**       | **Sampling Params** |
|:---------------------------:| --- |
|       gemma-2-27b-it        | Temp 0.8 Top-p 0.95 |
| Mistral-Large-Instruct-2407 | Temp 0.8 Top-p 0.95 |
| Qwen2.5-(Math)-72B-Instruct | Temp 0.7 Top-p 0.8 Repetition penalty 1.05 |
|   Llama-3.1-70B-Instruct    | Temp 0.8 Top-p 0.95 |


### Filtering
**Instruction Following**: For the five responses generated by each source model, we employed [ArmoRM](https://huggingface.co/RLHFlow/ArmoRM-Llama3-8B-v0.1) for annotation to obtain RM scores. We divided the data into SFT and DPO datasets using a 4:6 ratio. For the SFT phase, we selected responses with the highest RM scores. For the DPO phase, we paired responses from the same source model, using those with the highest RM scores as positive samples and lowest RM scores as negative samples. We maintained an RM score difference between 0.01 and 0.1 for all pairs.

**Math**: For mathematical data, we leveraged the openmathinstruct2 dataset, evaluating responses from all source models and Llama3-405B-Instruct using ArmoRM. We strategically divided the data between SFT and DPO phases based on RM scores and gold answer. The DPO phase used paired samples from the same source model: correct answers with highest RM scores as positive samples and incorrect answers with lowest RM scores as negative samples. We maintained an RM score differential of 0.01 to 0.1 between positive and negative pairs. The left data samples are reserved for the SFT phase, incorporating correct answers with the highest RM scores.

**Code**: In the code domain, we sourced data from Leetcode and self-oss-instruct-sc2-exec-filter-50k from Huggingface, which included test cases. Our evaluation implemented a dual-scoring system: ArmoRM annotation of code segments and additional points for passing both static and dynamic checks. The SFT phase included responses that passed all tests with the highest RM scores, while the DPO phase contrasted high-scoring, test-passing responses (positive samples) with low-scoring, test-failing responses (negative samples). We excluded cases where all model responses failed the testing criteria.

**Chinese**: For Chinese language data, we utilizde only Qwen2.5-72B-Instruct sampled responses for the SFT phase, deliberately excluding Chinese data from the DPO phase. 

Our final dataset comprised 158,784 total entries, with 94,539 entries for the SFT phase and 64,245 preference pairs for the DPO phase. The overall composition of the datasets is shown below.


| **Dataset** | **Selected Count** | **SFT Count** | **DPO Count** | **Category** |
| --- | :---: | :---: | :---: | :---: |
| [**UltraFeedback**](https://huggingface.co/datasets/openbmb/UltraFeedback) | 51098 | 20439 | 30659 | General & Instruction following |
| [**Magpie-Pro-DPO-100K-v0.1**](https://huggingface.co/datasets/Magpie-Align/Magpie-Pro-DPO-100K-v0.1) | 20374 | 8149 | 12225 | General & Instruction following |
| [**HelpSteer2**](https://huggingface.co/datasets/nvidia/HelpSteer2) | 9435 | 3774 | 5661 | General & Instruction following |
| [**OpenMathInstruct-2**](https://huggingface.co/datasets/nvidia/OpenMathInstruct-2) | 58546 | 40188 | 11615 | Math |
| [**leetcode**](https://huggingface.co/datasets/greengerong/leetcode) | 3113 | 1877 | 1236 | Coding |
| [**self-oss-instruct-sc2-exec-filter-50k**](https://huggingface.co/datasets/bigcode/self-oss-instruct-sc2-exec-filter-50k) | 13696 | 10160 | 2849 | Coding |
| [**alpaca_gpt4_zh**](https://huggingface.co/datasets/llamafactory/alpaca_gpt4_zh) | 2471 | 2471 | 0 | Chinese |
| [**Magpie-Qwen2-Pro-200K-Chinese**](https://huggingface.co/datasets/Magpie-Align/Magpie-Qwen2-Pro-200K-Chinese) | 7481 | 7481 | 0 | Chinese |
| **Total** | 158784 | 94539 | 64245 | |


## Training Pipeline

The entire training process includes supervised fine-tuning (SFT) and direct preference optimization (DPO).

### SFT

We used [Llama-Factory](https://github.com/hiyouga/LLaMA-Factory) as our fine-tuning library. For all target models, we fine-tuned for 3 epochs, with a batch size of 128 and a maximum sequence length of 2048 tokens.
A cosine learning rate schedule with a warmup ratio of 0.1 is employed. Different models' learning rates are shown in the table below:

| Taregt Models | Learning rate |
| :---: | :---: |
| Llama-3.1-8B-Instruct | 5e-6 |
| Qwen2.5-7B-Instruct | 2e-6 |
| Gemma-2-9B-it | 2e-6 |
| Llama-3.2-(1/3)B-Instruct | 5e-6 |


### DPO

We used [alignment-handbook](https://github.com/huggingface/alignment-handbook) as our DPO training library. For all Target SFT models, we trained for 1 epoch, set maximum sequence length to 2048, used cosine learning rate with a warmup ratio of 0.1. We saved checkpoints every 100 steps and selected the best from the last two checkpoints. For Llama-3.1 and Llama-3.2 series models, we introduced length normalized in DPO training, as shown in the formula below.

![](https://latex.codecogs.com/svg.image?\mathcal{L}_{\text{LN-DPO}}=-\log\sigma\left(\frac{\beta}{|y_w|}\log\frac{\pi_\theta(y_w|x)}{\pi_{\text{ref}}(y_w|x)}-\frac{\beta}{|y_l|}\log\frac{\pi_\theta(y_l|x)}{\pi_{\text{ref}}(y_l|x)}\right))

Different models' hyperparameters are shown in the table below. 

| Target SFT Models             | Learning rate | β | Length nomalize |
|-------------------------------| --- |---------| --- |
| FuseChat-Llama-3.1-8B-SFT     | 8e-7 | 10      | Yes |
| FuseChat-Qwen2.5-7B-SFT       | 3e-7 | 0.01    | No |
| FuseChat-gemma2-9b-SFT        | 5e-7 | 0.01    | No |
| FuseChat-Llama-3.2-(1/3)B-SFT | 1e-6 | 10      | Yes |

## Evaluation

The evaluation of instruction-tuned models mainly focuses on the model performance of instruction following, natural language understanding, general question answering, reasoning,  mathematics, coding, etc. For the evaluation of FuseChat-3.0, we include 14 benchmarks and organize them into four categories:
- **Instruction Following** Tasks: AlpacaEval-2, Arena-Hard, MTbench, AlignBench v1.1 (Chinese).
- **General** Tasks: LiveBench-0831, MMLU-Pro, MMLU-redux, GPQA-Diamond.
- **Math** Tasks: GSM8K, MATH, AMC 23.
- **Python Coding** Tasks: HumanEval, MBPP, LiveCodeBench 2408-2411.
We include more details and release our evaluation code at [FuseEval]().

The evaluation results of five series fused models are as follows, showing that our FuseChat-3.0 models achieved varying degrees of improvement across different target models. When selecting Llama3.1-8B-Instruct as the target model, our fusion model FuseChat-Llama-3.1-8B-Instruct achieved an average performance improvement of 6.8 points across 14 benchmarks. Notably, it showed significant improvements of 37.1 and 30.1 points on instruction-following test sets AlpacaEval-2 and Arena-Hard respectively. Additionally, FuseChat-Llama-3.1-8B-Instruct outperformed AllenAI's recently released Llama-3.1-Tulu-3-8B model on all benchmarks except GSM8K and GPQA-Diamond. All these results demonstrate the effectiveness and success of FuseChat-3.0.


## FuseChat-Llama3.1-8B-Instruct Performance

|             **Benchmarks**             | **Llama3.1-8B-Instruct** | **Llama-3.1-Tulu-3-8B** | **FuseChat-Llama-3.1-8B-SFT** | **FuseChat-Llama-3.1-8B-Instruct** |
|:--------------------------------------:| :---: | :---: | :---: | :---: |
|        **AlpacaEval-2 (LC %)**        | 28.3 |          33.4           |             41.3              | **65.4** |
|         **Arena-Hard (WR %)**         | 28.1 |          45.6           |             38.7              | **58.2** |
|              **MT-Bench**              | 8.38 |          8.34           |             8.54              | **9** |
|          **AlignBench v1.1**           | 4.61 |           6.2           |             6.25              | **6.69** |
|           **LiveBench 0831**           | 27.6 |          30.1           |             30.2              | **32** |
|               **GSM8K**                | 85.9 |        **88.6**         |              87               | 88 |
|                **MATH**                | 50.7 |          47.5           |             54.7              | **55.2** |
|               **AMC 23**               | 25 |           25            |              30               | **37.5** |
|              **MMLU-Pro**              | **50** |          42.9           |             47.8              | 49.2 |
|             **MMLU-redux**             | 67.2 |          66.3           |             68.4              | **69.2** |
|            **GPQA-Diamond**            | 33.8 |          35.9           |           **37.9**            | 34.9 |
|             **HumanEval**              | 69.5 |          66.5           |             69.5              | **71.3** |
|                **MBPP**                | **75.4** |          56.3           |             71.4              | 72 |
| **LiveCodeBench 2408-2411** | 12.3 |        10.6        |             12.6              | **13.1** |
|              **Average**               | 40.5 |          40.2           |             43.2              | **47.3** |


## FuseChat-Qwen2.5-7B-Instruct Performance

|              **Datasets**              | **Qwen2.5-7B-Instruct** | **FuseChat-Qwen2.5-7B-SFT** | **FuseChat-Qwen2.5-7B-Instruct** |
|:--------------------------------------:| :---: | :---: | :---: |
|        **AlpacaEval-2 (LC %)**        | 33.2 | 34.2 | **63.6** |
|         **Arena-Hard (WR %)**         | 50.7 | 45.2 | **61.4** |
|              **MT-Bench**              | 8.42 | 8.46 | **8.98** |
|          **AlignBench v1.1**           | 7.49 | 7.35 | **7.61** |
|           **LiveBench-0831**           | **35.4** | 33.7 | 33.2 |
|               **GSM8K**                | 91.7 | **92.3** | 91.7 |
|                **MATH**                | **75** | 72.7 | 73.6 |
|               **AMC 23**               | 52.5 | 45 | **57.5** |
|              **MMLU-Pro**              | **54.1** | 51.7 | 53 |
|             **MMLU-redux**             | **75.1** | 72.7 | 74.4 |
|            **GPQA-Diamond**            | 34.9 | **38.4** | 33.8 |
|             **HumanEval**              | **85.4** | 81.7 | 79.9 |
|                **MBPP**                | 80.2 | **84.1** | 83.1 |
| **LiveCodeBench 2408-2411** | 15.8 | 17.3 | **18.9** |
|              **Average**               | 50 | 48.9 | **52.9** |


## FuseChat-gemma2-9b-Instruct Performance

|              **Datasets**              | **gemma-2-9b-it** | **FuseChat-gemma2-9b-SFT** | **FuseChat-gemma2-9b-Instruct** |
|:--------------------------------------:| :---: | :---: | :---: |
|        **AlpacaEval-2 (LC %)**        |       51.1        |            49.8            | **70.2** |
|         **Arena-Hard (WR %)**         |       40.8        |            58.3            | **63.4** |
|              **MT-Bench**              |       8.49        |          **8.68**          | 8.64 |
|          **AlignBench v1.1**           |       7.04        |            7.05            | **7.41** |
|           **LiveBench-0831**           |       31.6        |          **33.3**          | 33.2 |
|               **GSM8K**                |       88.5        |            90.5            | **91** |
|                **MATH**                |       49.6        |           **58**           | 57.8 |
|               **AMC 23**               |        20         |            27.5            | **35** |
|              **MMLU-Pro**              |       50.5        |            52.5            | **52.9** |
|             **MMLU-redux**             |       72.8        |            72.8            | **73.7** |
|            **GPQA-Diamond**            |     **39.4**      |            33.3            | 35.4 |
|             **HumanEval**              |     **67.1**      |            65.9            | 64 |
|                **MBPP**                |     **75.1**      |            70.6            | 71.7 |
| **LiveCodeBench 2408-2411** |     **11.9**      |             11             | 10.1 |
|              **Average**               |       43.9        |            45.7            | **48.2** |

## FuseChat-Llama-3.2-3B-Instruct Performance

|             **Benchmarks**             | **Llama3.2-3B-Instruct** | **FuseChat-Llama-3.2-3B-SFT** | **FuseChat-Llama-3.2-3B-Instruct** |
|:--------------------------------------:| :---: | :---: | :---: |
|        **AlpacaEval-2 (LC %)**        |           21.4           | 31.1 |               **54**               |
|         **Arena-Hard (WR %)**         |           16.6           | 21.3 |              **30.2**              |
|              **MT-Bench**              |           6.87           | 7.33 |              **7.66**              |
|          **AlignBench v1.1**           |           3.83           | 5.5 |              **5.91**              |
|           **LiveBench 0831**           |           23.4           | 24.5 |              **24.9**              |
|               **GSM8K**                |            82            | **82.8** |                 82                 |
|                **MATH**                |           51.4           | 52.9 |              **53.1**              |
|               **AMC23**                |           22.5           | 20 |               **35**               |
|              **MMLU-Pro**              |           39.3           | **40.3** |              **40.3**              |
|             **MMLU-redux**             |           58.5           | 58.2 |               **59**               |
|            **GPQA-Diamond**            |           29.8           | 33.3 |              **33.8**              |
|             **HumanEval**              |            61            | **62.8** |                60.4                |
|                **MBPP**                |         **68.5**         | 67.5 |                67.5                |
| **LiveCodeBench 2408-2411** |         8.3        | 7.1 |               **9**                |
|              **Average**               |           35.2           | 36.8 |              **40.2**              |

## FuseChat-Llama-3.2-1B-Instruct Performance

|      **Benchmarks**      | **Llama3.2-1B-Instruct** | **FuseChat-Llama-3.2-1B-SFT** | **FuseChat-Llama-3.2-1B-Instruct** |
|:------------------------:| :---: | :---: | :---: |
| **AlpacaEval-2 (LC %)** | 9.7 | 14 | **25.3** |
|  **Arena-Hard (WR %)**  | 5.1 | 6 | **8.6** |
|       **MT-Bench**       | 4.71 | 5.17 | **5.74** |
|   **AlignBench v1.1**    | 2.88 | 3.86 | **4.25** |
|    **LiveBench 0831**    | 14.0 | 13.9 | **15.8** |
|        **GSM8K**         | 46.3 | **55.6** | 54.5 |
|         **MATH**         | 32.7 | **34.7** | 33.6 |
|        **AMC23**         | 17.5 | 15 | **20** |
|       **MMLU-Pro**       | **22.3** | 21.5 | 21.3 |
|         **MMLU**         | **45.8** | 45 | 44.8 |
|     **GPQA-Diamond**     | 21.2 | **25.3** | 24.2 |
|      **HumanEval**       | 39.6 | 36.6 | **40.2** |
|         **MBPP**         | **49.5** | 42.1 | 46.6 |
|       **Average**        | 24 | 24.5 | **26.5** |

