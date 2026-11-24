# Q1. How do you batch LLM requests on one GPU while users are waiting?
[← Back to questions](../questions.md)

## TL;DR
- **Decode is memory-bandwidth-bound.** At batch size 1 the GPU reads every weight just to make one token. Batching shares that weight read across many sequences, so throughput goes up almost linearly until you hit the compute limit or run out of KV cache.
- **Static batching** (wait for N requests, then run them together until all finish) wastes GPU time, because short answers sit idle while the longest one finishes. **Dynamic batching** (send the batch when it fills or after a timeout) helps with time to first token (TTFT), but it still works at the request level.
- **Continuous (in-flight) batching**, also called iteration-level scheduling (Orca, 2022), re-forms the batch after *every decode step*. A finished sequence leaves right away and a waiting request joins at the next step. vLLM, TGI, TensorRT-LLM and SGLang all work this way.
- **PagedAttention** (vLLM) stores the KV cache in fixed-size blocks, much like OS paging. Fragmentation drops to near zero, so more sequences fit in memory, and a bigger batch means more throughput.
- **Chunked prefill** breaks long prompts into token-budgeted chunks and mixes them with decode steps. One 32K-token prompt then can't freeze every other user's token stream.
- The main controls are `max_num_seqs` (batch width) and `max_num_batched_tokens` (token budget per step). Wider batches give more throughput but worse inter-token latency (ITL/TPOT). Tune them against your TTFT and ITL SLOs, not for peak tokens/s.

## Detailed Answer

### 1. Why batching works: the roofline argument
A forward pass for one decode token on a 7–8B model in FP16 must stream about **16 GB of weights** from HBM. An H100 has about 3.35 TB/s of HBM bandwidth:

```
time per decode step (B=1) ≈ 16 GB / 3.35 TB/s ≈ 4.8 ms  → ~200 tok/s for ONE user
```

The math done per weight is tiny: about 2 FLOPs per parameter per token, which is roughly 1 FLOP per byte at FP16. The H100's ridge point is around 300 FLOPs/byte, so at B=1 the GPU is **>99% idle on compute**. If you batch B sequences, every weight you load is used B times:

```
B=1   → ~200 tok/s total
B=32  → ~5,500 tok/s total (each step slightly slower, ~5.5 ms)
B=128 → ~15,000+ tok/s total, but step time grows → per-user ITL rises
```

Throughput keeps rising until (a) attention reads of the KV cache dominate, (b) you become compute-bound, or (c) **you run out of KV cache memory**. In practice (c) is usually the limit, which is why Q2 matters.

### 2. Prefill and decode are different workloads
| Phase | What happens | Bound by | Latency metric |
|---|---|---|---|
| Prefill | Process all prompt tokens in parallel, fill the KV cache | **Compute** (big matmuls) | TTFT |
| Decode | Generate one token per step per sequence | **Memory bandwidth** | TPOT / ITL |

A 4K-token prefill on an 8B model takes about 2 × 8e9 × 4096 ≈ 65 TFLOPs, which is roughly 100–150 ms on an H100. If that prefill runs as its own step, **every decoding user stalls for those 100–150 ms**, and they see a visible hiccup in their stream. Good schedulers deal with this interference.

### 3. The batching strategies

```
Static batching (request-level)
t →   ████████████████████████  req A (long)
      ██████░░░░░░░░░░░░░░░░░░  req B (done early, slot wasted)
      ████████████░░░░░░░░░░░░  req C (slot wasted)
      new req D waits until A finishes ...........▶ starts here

Continuous batching (iteration-level)
t →   ████████████████████████  req A
      ██████ D D D D D D D D D  B finishes → D joins next iteration
      ████████████ E E E E E E  C finishes → E joins
```

| Strategy | Unit of scheduling | Pros | Cons | Where used |
|---|---|---|---|---|
| No batching | request | lowest ITL at low load | terrible throughput | demos |
| Static | fixed batch, run to completion | simple | head-of-line blocking, padding, idle slots | classic TF Serving, offline jobs |
| Dynamic (timeout) | request; batch closes at size N or T ms | bounded wait | still runs to completion; padding | Triton dynamic batcher (non-LLM models) |
| **Continuous / in-flight** | **one decode iteration** | high utilization, new requests join quickly | complex scheduler and memory manager | vLLM, TGI, TRT-LLM, SGLang |
| Continuous + chunked prefill | iteration with a token budget | smooth ITL, good TTFT | tuning the chunk size | vLLM (default in V1), TRT-LLM, SGLang |
| Disaggregated prefill/decode | separate GPU pools | removes interference completely | KV transfer cost, more GPUs | large-scale deployments (DistServe, Mooncake, Dynamo) |

### 4. Iteration-level scheduling (Orca)
Orca (OSDI '22) made two contributions:
1. **Iteration-level scheduling.** The scheduler runs once per forward pass, not once per request.
2. **Selective batching.** Non-attention ops (QKV projections, MLP) are batched by flattening all tokens from all sequences into one `[total_tokens, hidden]` matrix, so sequences of different lengths need no padding. Attention runs per sequence, because each sequence has its own KV length.

Pseudo-loop of a continuous-batching engine:

```python
while True:
    # 1. admit new requests if KV blocks and slots allow
    while waiting and len(running) < max_num_seqs and kv.can_allocate(waiting[0]):
        running.append(waiting.popleft())

    # 2. build a step under the token budget
    budget = max_num_batched_tokens
    step = []
    for seq in running:                      # decodes first (1 token each) -> protects ITL
        if seq.is_decoding and budget >= 1:
            step.append((seq, 1)); budget -= 1
    for seq in running:                      # then prefill chunks
        if seq.is_prefilling and budget > 0:
            n = min(seq.remaining_prompt, budget)
            step.append((seq, n)); budget -= n

    # 3. if KV cache is full -> preempt lowest-priority/newest seq (recompute or swap)
    # 4. run ONE forward pass over the flattened tokens
    outputs = model.forward(step)
    # 5. stream new tokens, retire finished seqs, free their KV blocks
```

### 5. PagedAttention: why it raises the batch size
Older servers pre-allocated a contiguous KV region for `max_seq_len` per request. Most requests never reach that length, so **60–80% of KV memory was wasted** through internal and external fragmentation (vLLM paper, SOSP '23). PagedAttention:
- splits the KV cache into **blocks** (vLLM's default is 16 tokens per block) and gives each sequence a block table, like a page table;
- allocates blocks only as tokens are produced, so waste is limited to under one block per sequence;
- lets sequences share blocks copy-on-write, which serves parallel sampling, beam search and **prefix caching**.

Result: about 2–4× more concurrent sequences at the same memory, which translates directly into throughput.

### 6. Chunked prefill
Without chunking, a step is either "all prefill" or "all decode". With chunking, each step has a token budget (`max_num_batched_tokens`, e.g. 2048 or 8192). Decode tokens get priority and the leftover budget goes to prefill chunks:

```
step k:   [ 120 decode tokens ][ 1928-token chunk of new 10K prompt ]
step k+1: [ 120 decode tokens ][ next 1928-token chunk ]
...
```
- **Smaller budget** → smoother ITL for decoding users, slightly worse TTFT for long prompts, and lower prefill efficiency.
- **Larger budget** → better TTFT and throughput, and more ITL jitter.
- Rule of thumb: start at 2048 tokens when latency matters most, and 8192–16384 for throughput or offline work.

### 7. Configuring vLLM

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="meta-llama/Meta-Llama-3-8B-Instruct",
    gpu_memory_utilization=0.90,     # fraction of VRAM vLLM may use (weights + KV + activations)
    max_model_len=8192,              # caps per-request context -> bounds worst-case KV
    max_num_seqs=128,                # max concurrent sequences in a step (batch width)
    max_num_batched_tokens=4096,     # token budget per iteration (chunked prefill)
    enable_chunked_prefill=True,
    enable_prefix_caching=True,      # reuse KV for shared system prompts
)

params = SamplingParams(temperature=0.7, max_tokens=512)
outs = llm.generate(["Explain continuous batching."] * 64, params)  # engine batches internally
```

As an OpenAI-compatible server, with streaming:

```bash
vllm serve meta-llama/Meta-Llama-3-8B-Instruct \
  --max-num-seqs 128 \
  --max-num-batched-tokens 4096 \
  --enable-chunked-prefill \
  --enable-prefix-caching \
  --gpu-memory-utilization 0.90 \
  --scheduling-policy priority      # allows per-request "priority" field
```

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="x")

stream = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3-8B-Instruct",
    messages=[{"role": "user", "content": "Summarize PagedAttention"}],
    stream=True,
    max_tokens=300,
    extra_body={"priority": 0},       # lower value = served earlier under --scheduling-policy priority
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

TGI's equivalents are `--max-batch-total-tokens`, `--max-concurrent-requests`, `--max-batch-prefill-tokens` and `--waiting-served-ratio`.

### 8. "While users are waiting": the latency side
Users experience three numbers:
- **TTFT**: queue time + prefill. Streaming hides total generation time, but TTFT is what makes an app feel slow.
- **ITL/TPOT**: the gap between streamed tokens. People read at about 5–10 tokens/s, so ITL under 50–100 ms feels instant. Pushing ITL lower adds little perceived value, which is exactly why you can **spend that headroom on a bigger batch**.
- **E2E**: TTFT + ITL × output_tokens. This matters for agents and tool chains that wait for the full response.

The trade-off curve (typical shape):

```
throughput (tok/s)                    ITL p95 (ms)
   ^                                     ^
   |            ________                 |                 /
   |         __/                         |              __/
   |      __/                            |          ___/
   |   _/                                |   ______/
   |__/__________________> batch         |__/________________> batch
         knee ≈ best operating point
```
Choose the largest batch (`max_num_seqs`) where p95 ITL and p95 TTFT still meet the SLO under realistic load. Find it with a load test (e.g. `vllm bench serve` or `benchmarks/benchmark_serving.py`, GuideLLM, or LLMPerf) using **your real prompt and output length distribution**.

### 9. Fairness and priority
- **FCFS** (the default) can be gamed. One user sending 50 long requests can fill the KV cache.
- **Priority scheduling**: vLLM `--scheduling-policy priority` plus a per-request `priority`. Use it for interactive vs batch traffic, or paid vs free tiers. Under memory pressure, low-priority sequences get preempted first.
- **Per-tenant fairness** usually lives in the gateway, not the engine: token-bucket rate limits on *tokens* rather than requests, per-tenant concurrency caps, and weighted fair queueing. Research systems such as VTC (Virtual Token Counter) do fair scheduling by token count.
- **Admission control**: cap the queue length and return 429 or `Retry-After` instead of letting TTFT climb without bound. A request that waits 30 s and then times out wastes the prefill compute you spent on it.
- **Length-aware scheduling**: very long prompts can go to a separate pool or replica, so short chat requests aren't stuck behind them (head-of-line blocking).

### 10. Preemption under memory pressure
When the KV cache is full and running sequences need new blocks, the engine must evict something:
- **Recompute** (the vLLM default): drop the sequence's KV and re-prefill it later. This is cheap for short sequences.
- **Swap**: copy the blocks to CPU memory over PCIe and bring them back later. This is better for long sequences, but it is bounded by PCIe bandwidth.
If you see frequent preemptions (`vllm:num_preemptions_total`), `max_num_seqs` is too high for your memory, or you need more KV capacity (FP8 KV, more GPUs, shorter `max_model_len`).

## Common Pitfalls
- **Tuning for peak tokens/s in a benchmark** with fixed 128-in/128-out prompts. Production has long-tail prompts that wreck TTFT.
- **Building your own request batcher in front of vLLM/TGI** (collecting N requests in FastAPI and calling generate). This puts static batching back on top of an engine that already does continuous batching. Just send requests concurrently.
- **Setting `max_model_len` to the model maximum (128K)** when the app never goes above 8K. Admission becomes pessimistic, and one huge request can fill the cache.
- **No admission control**, so queues grow, everything times out, and goodput collapses (a retry storm).
- **Ignoring prefill/decode interference**, with chunked prefill disabled on mixed long/short workloads, which causes ITL spikes.
- **Measuring averages.** Batching problems show up at p95/p99 TTFT, not in the mean.
- Leaving `gpu_memory_utilization` at a value that OOMs when another process (another model, CUDA graphs) shares the GPU.

## Likely Follow-up Questions
1. **Why does continuous batching need special attention kernels?**
   Each sequence in the batch has a different KV length and non-contiguous blocks. Kernels such as PagedAttention and FlashAttention/FlashInfer with variable-length and block-table support process these ragged batches without padding.
2. **How would you serve both a chatbot and a nightly batch summarization job on the same GPU?**
   Use priority scheduling (interactive = high), cap batch-job concurrency at the gateway, or keep the batch job off-peak. Better still, send offline work through an offline `LLM.generate` run or a batch API on separate capacity.
3. **What is disaggregated serving and when is it worth it?**
   Prefill and decode run on separate GPU pools, and KV is transferred over NVLink or RDMA. It is worth it at scale with long prompts and strict ITL SLOs, because each pool can be tuned and scaled on its own. It is overkill for a single GPU.
4. **What happens to TTFT when you double `max_num_seqs`?**
   At low load nothing changes. Near saturation, more requests get admitted instead of queued, so queue time drops, but each step gets slower. TTFT for new arrivals can improve while ITL degrades, until KV pressure causes preemptions and both get worse.
5. **How does streaming interact with batching?**
   Streaming is orthogonal. Each iteration emits one token per running sequence, and the server pushes each token over SSE. Batching decides how fast those iterations happen, and streaming decides when the user sees them.
6. **Static batching is never right?**
   It is fine for offline jobs with uniform lengths, or for encoder models and embeddings, where there is no autoregressive decode loop.

[← Back to questions](../questions.md)
