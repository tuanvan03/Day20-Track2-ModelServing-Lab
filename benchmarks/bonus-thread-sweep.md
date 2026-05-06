# Bonus — Thread sweep

Model: `qwen2.5-1.5b-instruct-q4_k_m.gguf`  ·  GPU layers: `99`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 0.0 |
| 2 | 0.0 |
| 6 | 0.0 |
| 12 | 0.0 |
| 24 | 0.0 |

**Best**: `-t 1` at 0.0 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
