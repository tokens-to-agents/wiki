---
icon: book-open
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

# Overview

This section explains how AI models work: how they are made, how they produce text, images and sound on your own machine, and how an app turns a model into an agent that uses tools. This page is the map: every other page of the section zooms into one part of it. The running example is **Llama 3.1 8B Instruct**, a text model made by Meta.

{% hint style="info" %}
**A model is made once and used many times.** Its creator trains it and publishes its files. Every use, called **inference**, loads one of those files into the GPU's memory and computes an answer from it. An **agent** is a model that a program, its harness, runs in a loop with tools: the model writes text, and the harness acts on it.
{% endhint %}

### The map

The map has three stages. **Model** is how the model is made, once, by its creator. **Inference** is what happens every time you ask something. **Agents** is what happens when a program gives the model tools and runs it in a loop.

<figure><img src="../.gitbook/assets/0.overview_ai-overview-map_v3.png" alt=""><figcaption></figcaption></figure>

**Model**, made once. See [Model](model/) for the detailed map.

* **Training data:** Text collected before training, not prompts: web pages, books, code and math. Meta used about 15 trillion **tokens** (words or pieces of words) for Llama 3.1. Image models are trained on images paired with their descriptions.
* **Design:** Before training, Meta chose the model's **architecture**, its shape: 32 layers, 4,096 neurons wide. Training never changes it.
* **Training:** Meta adjusted every weight, step by step, in two phases: first to predict the next token of the training text, then to follow instructions, which gives the **Instruct** model that chat apps use. It took 1.46 million GPU-hours.
* **Model:** The result: the architecture plus 8.03 billion learned numbers, the **parameters** (also called **weights**).
* **Versions:** The same model saved at different precisions, the number of bits per weight, and published on [Hugging Face](https://huggingface.co): Meta's **BF16** version takes 16.1 GB in four `.safetensors` files, a **quantized** Q4\_K\_M version 4.92 GB in a single `.gguf` file, made by the creator or by community uploaders.
* **LoRA:** A small extra file of corrections added on top of a model's weights to specialise it, made by extra training on your own examples (fine-tuning).

**Inference**, every use. See [Inference](inference/) for the detailed map. The numbers follow the badges:

1. **Download:** You download one version of the model, once. It stays on your disk.
2. **Prompt:** You type in an **app** such as [Ollama](https://ollama.com), [LM Studio](https://lmstudio.ai), [TextGen](https://github.com/oobabooga/textgen) or [ComfyUI](https://github.com/Comfy-Org/ComfyUI). The app is built on an **engine**, the program that runs the model: [llama.cpp](https://github.com/ggml-org/llama.cpp) for most local text models, [PyTorch](https://pytorch.org) for images and video. A few others exist, such as MLX on Macs.
3. **Use the hardware:** The engine drives the GPU in two parts.
   * **3a, load:** The weights are copied from the disk into the GPU's own memory, the **VRAM**, once. For text, VRAM also holds the **KV cache**, the model's memory of the conversation.
   * **3b, run:** The GPU's cores do the model's math, reading the weights from VRAM at every step. A disk would be far too slow for that.
4. **Compute:** The GPU's results feed one of two loops, text or image.
   * **Text:** The model gives a score to each of the 128,256 tokens of its vocabulary, a **sampler** picks one, the token is added to the answer, and the loop repeats until the answer is complete.
   * **Image:** A **diffusion** model starts from random noise and removes part of it at each step, about 20 steps with common settings. A **VAE** (a small decoder model) then turns the result into pixels.

On a Mac, the GPU has no separate VRAM: it uses the same **unified memory** as the processor. A rented GPU works the same way, with the machine in a data centre instead of on your desk.

**Agents**, every turn of a task. See [Agents](agents.md) for the detailed map.

An **agent** is a model plus the program that runs it, its **harness** (also called the agent app): the harness runs the model in a loop, gives it tools and decides what enters its context. The map shows one agent, the usual case; a **multi-agent** system runs several such loops. The numbers continue those of Inference:

5. **Context:** The **context** is everything the model reads at one turn, and the model sees nothing else: the harness's files, keys and code stay outside it unless a tool puts them in. The harness builds it from a system prompt listing the tools the model may call (here `get_current_weather(city)`), the conversation, earlier tool results, and anything remembered or retrieved from documents (**RAG**); it is sent whole at every turn, because the model keeps nothing between turns.
6. **Model:** The model runs as in Inference and writes text. That text is either an answer for you or a **tool call**, a short structured message such as `{"name": "get_current_weather", "arguments": {"city": "Paris"}}`. Llama 3.1 Instruct models were trained to write such calls for tools described in their prompt.
7. **Tool:** The harness reads the tool call and runs the tool with its arguments, after your approval if you required one. A tool can be a function inside the harness, a web service, a file on disk, an **MCP** server (a standard way to plug tools into any harness), or another agent with its own loop (a sub-agent).
8. **Result:** The tool's output is added to the context, and the loop goes back to step 5 until the model writes an answer. This text was written neither by you nor by the harness: a web page or an email can contain instructions that steer the model, the main security risk of agents.

The model only writes text; everything an agent does in the world, the harness does. The same loop runs in a chat app with web search, in coding agents such as Claude Code or Codex, in personal agents such as OpenClaw, and in a script of a few lines around Ollama's API.

### Pages in this section

* **Model**: how a model is made, what its versions contain, and how they are changed.
  * [Inside a model](model/inside-a-model.md): parameters, tables, the files of a version, and how the tables connect. Two sub-pages zoom on the two parts of a layer: **Transformer** (attention) and **Mixture of experts** (a feed-forward part split into experts, a few used per token).
  * [How a model learns](model/how-a-model-learns.md): neurons, matrices and training: pre-training, post-training, reinforcement learning on checkable answers, distillation and scaling laws.
  * [Precision](model/precision.md): how a weight is written in bits (FP32, BF16), and how precision sets the size of a model.
  * [Quantization](model/quantization/): why and how a 16 GB model becomes a 5 GB version, block by block. Its sub-page, [Quantization formats](model/quantization/quantization-formats.md), reads names like Q4\_K\_M or IQ3\_M.
  * [LoRA](model/lora.md): specialising a model with a small file instead of a full copy.
* **Inference**: what runs on your machine when you use a model.
  * [GPU & memory](inference/gpu-and-memory.md): why AI runs on GPUs, why memory matters more than raw power, and how software reaches the chip.
  * [Text generation](inference/text-generation/): tokens, the generation loop and the context window, with [Tokens](inference/text-generation/tokens.md) (pieces of text and the tokenizer), [Sampling](inference/text-generation/sampling.md) (the settings that pick each token), [KV cache](inference/text-generation/kv-cache.md) (the memory of a conversation) and **Reasoning** (thinking tokens before the answer).
  * **Multimodal models**: how a picture or a sound becomes tokens a language model reads.
  * [Image & video](inference/image-and-video.md): diffusion, the latent space, CFG and video.
  * **Audio & speech**: speech-to-text, text-to-speech and real-time voice.
  * **Serving at scale**: how one GPU answers many users at once, and why APIs bill per token.
* **Agents**: the harness and the loop that turn a model into an agent.
  * **Context & prompt**: what the model actually reads at each turn.
  * **Tool use**: how a model calls a tool, with its sub-page **MCP**, the standard plug for tools.
  * **RAG**: answering from your own documents, with **Embeddings**, the lists of numbers that find them.
  * **Memory**: what an agent keeps between turns and between sessions.
  * **Multi-agent**: an agent that hands parts of a task to other agents.
  * **Agent security**: prompt injection, and the permissions and sandboxes that limit it.
* **Limits**: hallucinations, knowledge cutoff and the other limits of today's models, each linked to its cause.
* **Research frontiers**: what research is working on next.
* [Glossary](../glossary.md): every term of the section, with its other names (parameter and weight, VRAM and GPU memory…).

### Resources

* Model card: [Meta Llama 3.1](https://huggingface.co/meta-llama/Llama-3.1-8B) (GPU-hours, BF16 release) | Paper: [The Llama 3 Herd of Models](https://arxiv.org/abs/2407.21783) (training data, tool use)
* Tool calling: [Ollama, Tool support](https://ollama.com/blog/tool-support) (the `get_current_weather` example, Llama 3.1)
* What an agent is: [Simon Willison, "An LLM agent runs tools in a loop to achieve a goal"](https://simonwillison.net/2025/Sep/18/agents/) | The harness: [de Macedo, What makes a harness a harness](https://arxiv.org/pdf/2606.10106) (preprint, June 2026: its four parts, and the horse's harness) | [LangChain, The anatomy of an agent harness](https://langchain.com/blog/the-anatomy-of-an-agent-harness)
