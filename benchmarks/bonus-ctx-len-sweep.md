# Bonus — Context-length sweep (prefill cost)

Model: `qwen2.5-1.5b-instruct-q4_k_m.gguf`  ·  threads: `12`  ·  n_gpu: `99`

| ctx tokens | pp (tok/s) | prefill latency (ms) |
|--:|--:|--:|
| 128 | 197.5 | 648.0 |
| 256 | 190.5 | 1343.6 |
| 512 | 192.1 | 2664.9 |
| 1024 | 171.8 | 5958.7 |
| 2048 | 173.3 | 11817.7 |

Prefill scales **super-linearly** with context length — that's where TTFT comes from in long-context RAG. This is also why the deck's *disaggregated prefill/decode* pattern (Mooncake / llm-d / Dynamo) exists: give prefill its own GPU pool so long-context requests don't block decode.
