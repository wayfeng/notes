---
theme: gaia
style: |
    code { font-family: mononoki, consolas, monospace;  }
    img[alt~="center"] { display: block; margin: 0 auto; }
    img[alt~="right"] { display: float; margin: 0 0 0 auto;}
_class: lead
paginate: true
#footer: "Intel Internal Only"
#backgroundColor: #fff
marp: true
title: Intoduction to llama.cpp

---

# Introduction to **llama.cpp**

Wayne Feng
June 2024

---

## Agenda

- How to use `llama.cpp`
- Necessary background knowledges
- Implementation

---

<!-- _class: lead -->
## Usage

---

## Run with ollama

Install ``ollama``

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Run ``llama3``

```bash
# ollama pull llama3:8b
ollama run llama3:8b
```

Combine with ``vscode`` and plugin [``continue``](continue.dev).

---

## Compile source code

```bash
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
make -j$(nproc)
# make LLAMA_CUDA=1 -j$(nproc)
```

For Intel devices with SYCL enabled.
```bash
source /opt/intel/oneapi/setvars.sh
# Option 1: Use FP32 (recommended for better performance in most cases)
cmake -B build -DLLAMA_SYCL=ON -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx
# Option 2: Use FP16
cmake -B build -DLLAMA_SYCL=ON -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx -DLLAMA_SYCL_F16=ON
cmake --build build --config Release -j -v
```

---

## Converting and quantizing GGUF manually

Converting models to GGUF
```bash
python3 -m venv .env
pip install -r requirements/requirements-convert-hf-to-gguf.txt
python convert-hf-to-gguf.py  ~/checkpoints/Llama-2-7b-chat-hf
```
Quantizing GGUF model
```bash
./build/bin/quantize ~/checkpoints/Llama-2-7b-chat-hf/ggml-model-f16.gguf Q4_K
```
<!-- _k quantization is optimal to _0 -->

---

## Run llama.cpp manually
Run the model with CPU.
```bash
./build/bin/main --color \
 -m ~/checkpoints/Llama-2-7b-chat-hf/ggml-model-Q4_K.gguf \
 -p "What is openvino"
```
With GPU.
```bash
./build/bin/main --color \
 -m ~/checkpoints/Llama-2-7b-chat-hf/ggml-model-Q4_K.gguf \
 -p "What is openvino" -ngl 9999
```

---

## Bonus: llamafile

- Based on `llama.cpp`
- Compile once, run everywhere with [Actual Portable Executable](https://justine.lol/ape.html).

---

## Some background information

How do tokenizers work?
How are transformers calculated?
Why is KV cache needed?
Decoder-only models V.S. encoder-decoder models?
Why the fuzz about model formats?

---

## Model Formats

- pickle (pytorch)
- h5 (tensorflow)
- protobuf (onnx)
- safetensors
- GGUF
- ...

---

<!-- _class: lead -->
## GGUF details
![width:800px](./img/gguf-spec.png)

---

<!-- _class: lead -->
## Implementation

---

<!-- _class: lead -->
## Repo structure

---

## Memory management

- Custom allocator
- Lazy allocation
- Free inactive tensors

---

## KV cache

- **Stores pre-computed token representations**: The cache stores key-value pairs where the key is a token and the value is its vector representation. This avoids redundant calculations for already encountered tokens.
- **Targeted Storage**: The cache prioritizes storing frequently used tokens or those with complex representations.
- **Eviction Strategies**: The cache evicts less important entries (LRU, LFU, or hybrid approaches) to make space for new ones.
- **Fragmentation Mitigation**: Techniques like compaction or pre-allocation help maintain efficient cache space utilization.

---

## Tokenizers

- SPM
- BPE
- WPM

```C++
struct llama_vocab {};
struct llm_tokenizer_bpe {};
struct llm_tokenizer_spm {};
struct llm_tokenizer_wpm {};
```
<!--
---

## Sampling?
-->
---

## Inferencing process
```C++
main() {
    ...
    llama_backend_init();
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    inp = ::llama_tokenize(ctx, params.prompt, true, true);
    for (...) { llama_decode(ctx, llama_batch_get_one(...)); }
    output_ss << llama_token_to_piece(ctx, token);
    llama_free(ctx);
    llama_free_model(model);
    llama_backend_free();
    ...
}
```

---

## Decode process

```C++
llama_decode() {
    ...
    if (!llama_kv_cache_find_slot(kv_self, u_batch)) {
        return 1;
    }
    ggml_backend_sched_reset(lctx.sched);
    ggml_backend_sched_set_eval_callback(...);
    ggml_cgraph * gf = llama_build_graph(lctx, u_batch, false);
    ggml_backend_sched_alloc_graph(lctx.sched, gf);
    llama_set_inputs(lctx, u_batch);
    llama_graph_compute(lctx, gf, n_threads);
    ggml_backend_tensor_get_async(...);
    ...
}
```

---

## Graph Compute

Compute graphs are composed with `ggml_tensor`
A graph with only `ggml_add` looks like:
![center](./img/cgraph_add.png)
<!--
---

## Tricks

- Lookup tables of fp16->fp32, exp() and activation functions: GELU, QuickGELU, and SiLU.

---

## Inferencing customized modes

Executing llava requires CLIP. Refer to `examples/llava/clip.cpp`.
-->