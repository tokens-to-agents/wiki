---
icon: '0'
layout:
  width: default
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
  anchors:
    visible: true
---

# Model

A model is made in two stages: its creator trains it, then publishes it in one or more versions. This page follows **Llama 3.1 8B Instruct** through both, using the figures from Meta's paper, Meta's model card and the file lists on [Hugging Face](https://huggingface.co).

{% hint style="info" %}
**A model is an architecture plus learned numbers.** The architecture is chosen before training; training sets the 8.03 billion **parameters** (also called **weights**). Every version you can download holds the same model, with the weights stored at a different precision.
{% endhint %}

### The map

**Training**, at Meta, once. The whole training took 1.46 million GPU-hours on H100 GPUs.

<figure><img src="../../.gitbook/assets/0.model_ai-model-hub-map_v2.png" alt=""><figcaption></figcaption></figure>

1.  **Design:** Meta chose the **architecture** before training: 32 layers, 4,096 neurons wide, a vocabulary of 128,256 tokens, which gives 8.03 billion parameters, all random at the start. In each layer, a token's list of numbers goes through 9 tables, in the order of the map:

    * **Norm (1 table each):** Rescales the numbers, once before each part.
    * **A, attention:** 4 tables (q, k, v, o) that mix in information from the earlier tokens (see **Transformer**).
    * **B, feed-forward:** Works on each token alone, in one of two designs. **B1, dense:** 3 tables (gate, up, down), all used for every token, as in Llama 3.1 8B. **B2, mixture of experts** (MoE): many sets of 3 smaller tables, the **experts**, and a small learned table, the **router**, that sends each token to a few of them. Qwen3.5-122B-A10B stores 122 billion parameters but reads about 10 billion per token: memory must hold them all, while speed depends on the 10 billion read (see **Mixture of experts**).

    **Token table and output table:** Two more tables sit at the two ends of the stack, each with one row per token of the vocabulary. The token table copies the row of each token read; the output table turns the final list into one score per possible next token (see [how the tables connect](inside-a-model.md#how-the-tables-connect)). Both are weights, learned like the others.
2. **Data:** About 15 trillion tokens of text, mostly from the web: 50 % general knowledge, 25 % math and reasoning, 17 % code, 8 % other languages, nothing after December 2023. A **token** is a word or a piece of one, the unit the model reads and writes (see [Tokens](../inference/text-generation/tokens.md)). How much data suits a model of a given size is predicted from small trial runs, the [scaling laws](how-a-model-learns.md#scaling-laws).
3. **Pre-training:** The model reads a piece of text with the next token hidden, scores every possible next token, and every weight is nudged a little toward the right answer. Repeated over the whole dataset, this turns random weights into a **base model** that continues any text. See [How a model learns](how-a-model-learns.md).
4. **Post-training:** The base model is trained again, with methods that labs combine and repeat in rounds: **A, examples**, an instruction with a good answer (supervised fine-tuning, SFT); **B, preferences**, two answers with the better one marked by a person (reinforcement learning from human feedback, RLHF); and, for reasoning models, **C, checkable answers**, rewarded when a program finds them right (see [reinforcement learning on checkable answers](how-a-model-learns.md#reinforcement-learning-on-checkable-answers)). Meta used A and B, in six rounds, to make the **Instruct** model; small models are often trained on a bigger model's outputs (see [distillation](how-a-model-learns.md#distillation)). Post-training changes the weights once, at the lab, unlike RAG, which adds documents to the prompt at each question.

**One family, many models.** Llama 3.1 comes in three sizes, 8B, 70B and 405B (billions of parameters), each published as a **base** model, which only continues text, and an **Instruct** model, which answers; chat apps and agents use the Instruct one. Others build new models from them by changing their weights: **fine-tunes** (trained further) and **merges** (several models' weights combined). A quantized file such as Q4\_K\_M is not a new model but a version of the same one (step 6); the **Families & names** page explains how to read the names.

**Versions**, on Hugging Face:

5. **BF16 version:** Meta's repository holds the weights at 16 bits each: 16.1 GB in four `.safetensors` files, the format of PyTorch and Hugging Face, which hold only tables. The architecture (`config.json`) and the vocabulary (`tokenizer.json`) are small separate files. See [Inside a model](inside-a-model.md).
6. **Quantized versions:** The weights converted to fewer bits, once per format. A `.gguf` file, the format of [llama.cpp](https://github.com/ggml-org/llama.cpp), holds a whole version (tables, architecture and vocabulary): 4.92 GB for Q4\_K\_M. Meta did not publish GGUF versions of Llama 3.1, so the community made them, 935 on its model page; Qwen and Google publish their own. See [Quantization](quantization/).

Every version holds the same 291 tables (tensors), with the same names and shapes; only the number of bits per weight changes. Weight A1 of the first attention table is 0.11182 in the BF16 version and 0.1172 in the Q4\_K\_M version.

**Fine-tuning**, optional, by anyone:

7. **LoRA:** Training on your own examples, with the model's tables frozen. Only two thin extra tables per targeted table are trained, and they are saved as a small file (about 84 MB for Llama 3.1 8B) that the engine adds on top of the weights. LoRA works for text models as well as image models. See [LoRA](lora.md).

### Resources

* Model card: [Meta Llama 3.1 8B Instruct](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct) (GPU-hours, fine-tuning data, quantized versions) | Paper: [The Llama 3 Herd of Models](https://arxiv.org/abs/2407.21783) (data mix, post-training)
* Mixture of experts: [vLLM recipes, Qwen3.5-122B-A10B](https://recipes.vllm.ai/Qwen/Qwen3.5-122B-A10B) (256 experts, 10 B active) | [Qwen 3.5 medium series deep dive](https://logdew.pages.dev/en/posts/qwen35-medium-series-deep-dive/) (8 routed + 1 shared expert)
* File lists: [bartowski, Meta-Llama-3.1-8B-Instruct-GGUF](https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF) | [Qwen3-0.6B](https://huggingface.co/Qwen/Qwen3-0.6B/tree/main)
