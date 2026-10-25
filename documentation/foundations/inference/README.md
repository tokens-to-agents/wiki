---
icon: '1'
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

# Inference

**Inference** is every use of a finished model: loading it and computing an answer. This page follows one question to **Llama 3.1 8B Instruct**, in its Q4\_K\_M version (4.92 GB), run by [Ollama](https://ollama.com) on an RTX 5090 graphics card with 32 GB of VRAM. The steps are numbered as on the [overview map](../overview.md).

{% hint style="info" %}
**The weights are read at every step.** The model's weights sit in the GPU's memory, and the GPU reads all of them for every token it writes. Memory size decides which models fit; memory speed decides how fast they answer.
{% endhint %}

### The map

<figure><img src="../../.gitbook/assets/1.inference_ai-inference-hub-map_v1.png" alt=""><figcaption></figcaption></figure>

1. **Download:** The version comes from the model's page on Hugging Face (see [Model](../model/)) and is stored on the disk once.
2. **Prompt:** You type in the **app**, here Ollama. The app is built on an **engine**, here [llama.cpp](https://github.com/ggml-org/llama.cpp). The engine does all the work: it loads the file, cuts the prompt into tokens, runs the model's math and runs the sampler. Other engines exist (MLX on Macs, ExLlamaV3 on NVIDIA cards, vLLM on servers); image models run on [PyTorch](https://pytorch.org), inside apps such as [ComfyUI](https://github.com/Comfy-Org/ComfyUI). Model and engine are independent: a version runs in any engine that reads its format (`.gguf` in llama.cpp, `.safetensors` in PyTorch), and updating the engine does not change the model.
3. **Use the hardware**, in two parts:
   * **3a, load:** The engine copies the weights from the disk into the **VRAM**, the GPU's own memory, once. It also reserves room for the [KV cache](text-generation/kv-cache.md), the keys and values of every token of the conversation: 128 KB per token, so 1.1 GB for an 8,192-token window. Layers that do not fit in VRAM can wait in the computer's **system RAM** (offloading), at a large cost in speed.
   * **3b, run:** For each operation of the model, such as "multiply these two tables", the engine calls a **kernel**, a small GPU program provided by the GPU's toolkit (CUDA on NVIDIA, Metal on Macs), and the driver hands it to the chip. The GPU's 21,760 cores each take a share of the work: about 8 billion multiply-adds per token, one per weight. See [GPU & memory](gpu-and-memory.md).
4. **Compute:** The results feed one of two loops.

**Text**, one token at a time. See [Text generation](text-generation/).

* **4a, tokens:** The prompt is cut into tokens from the model's vocabulary, plus a few special tokens from the chat template that mark who is speaking.
* **4b, prefill:** The model reads all the prompt's tokens at once, through its 32 layers, because they are all known in advance. What each token leaves behind, its keys and values, is stored in the KV cache, so it is never computed again. A long pasted document makes this step take a few seconds: it is the wait before the first word, the **time to first token**.
* **4c, the loop:** The answer is written one token per turn, because each token depends on the one before. Each turn, the newest token goes through the 32 layers, the model scores all 128,256 tokens of the vocabulary, the **sampler** picks one (see [Sampling](text-generation/sampling.md)), and the chosen token joins the answer and the KV cache. The map shows the sixth turn: the answer so far is "The capital of France is", and " Paris" comes out at 91 % (an illustrative figure). The loop stops at an end-of-text token or at the length limit.

**Image**, in a fixed number of steps. See [Image & video](image-and-video.md).

* **Text encoder:** A language model turns the prompt into numbers that guide the image.
* **Noise:** A grid of random numbers, made from the **seed**, in the **latent space**: 128 × 128 × 16 numbers for a 1024 × 1024 image with Flux.1, 12 times fewer than the pixels.
* **Diffusion model:** Removes part of the noise at each step, about 20 steps, guided by the prompt with **CFG**.
* **VAE:** Turns the final latent into pixels, once. Video works the same way on a block of frames.

On a Mac, the GPU and the processor share one **unified memory**, so there is no separate VRAM: the model sits in the same memory as everything else.

### How fast it can go

Every token reads all the weights from VRAM, so the speed ceiling is the memory bandwidth divided by the size of the weights: 1,792 GB/s ÷ 4.92 GB = about 364 tokens per second for this example. Real speed is lower, since the KV cache is read too and the engine has other work. A version twice as small can go up to twice as fast, which is one reason to [quantize](../model/quantization/).

### Resources

* GPU specs: [NVIDIA GeForce RTX 5090](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/rtx-5090/)
