# KV Cache Hit Rate Simulator CLI

Run the KVCache.AI hit-rate simulator locally on JSONL traces. The Python implementation uses the same model accounting formulas as the web KV Cache Size Calculator and the same prefix-aware hit-rate semantics as the web simulator.

## Quick Start

```bash
cd /path/to/kvcache-blog

python3 -m kvcache_sim sweep \
  --trace trace.jsonl.gz \
  --model glm-5.2 \
  --kv-precision fp8_int8 \
  --indexer-precision fp4_int4 \
  --jobs 8
```

The default output is a readable table. Use `--format json` when another script needs to consume the result.

By default, the CLI runs the replay simulation core in C++. Python still handles CLI parsing, trace parsing, model memory accounting, and output formatting. If a C++ compiler is not available, install `c++` / `clang++` or run with `--backend python`.

## Input Trace Format

Input is JSONL or JSONL.GZ, one request per line. The minimal accepted format is:

```json
{"block_size":64,"hash_ids":[2001,2002],"input_length":128}
```

Required fields:

- `hash_ids`: cache block identities in request-prefix order.
- `input_length`: prefill input token count for this request.
- `block_size`: source-native block size. It can be omitted only when `--block-size` is provided.

Optional fields:

- `timestamp`: ignored by the simulator. Requests are replayed in file order, so sort production traces by timestamp before running the command.
- `output_length`: ignored by the hit-rate denominator. Generated output matters only if it appears later in another request's `hash_ids`.
- `block_tokens`: advanced field for exact per-block token weights. If present, it must be a positive integer list with the same length as `hash_ids`.

`--block-size` is only a fallback for records that omit `block_size`. If any record declares `block_size`, the trace-declared value is used and overrides the CLI fallback for the whole trace.

## Web Preset Traces

The web page ships precomputed curves for its public preset traces, not the full raw request streams. This CLI therefore expects a user-provided JSONL/JSONL.GZ trace. If you want to replay one of the public datasets locally, first convert it to the JSONL format above with the repo's trace normalization scripts, then pass that converted file through `--trace`.

## Options

| Option | Meaning |
| --- | --- |
| `--trace PATH` | JSONL/JSONL.GZ trace path, or `-` for stdin. |
| `--model ID` | Model id from `data/kv_cache_calculator/models.yaml`. Use `python3 -m kvcache_sim list-models` to list ids. |
| `--kv-precision ID` | KV cache precision: usually `bf16_fp16`, `fp8_int8`, or `fp4_int4`. Defaults follow the web calculator. |
| `--indexer-precision ID` | Indexer cache precision for models with an indexer cache, such as DeepSeek V4 / GLM / MiniMax M3. |
| `--include-draft-kv-cache` | Include draft/MTP KV layers when the selected model defines them. Default is off. |
| `--block-size N` | Fallback block size when trace records omit `block_size`; trace-declared `block_size` overrides it. |
| `--estimate-tokens N` | Override the token count used for token-dependent bytes/token formulas. By default the trace average input length is used. |
| `--budgets-gib A,B,C` | Comma-separated KV cache memory budgets in GiB. Default matches the web sweep: `1,2,4,...,16384`. |
| `--policies fifo,lru,optimal` | Eviction policies to simulate. Defaults to all three. |
| `--backend cpp\|python` | Simulation backend. Default is `cpp`; use `python` for debugging or machines without a compiler. |
| `--jobs N` | Number of worker processes for the Python backend. The C++ backend runs one batch process and ignores this option. |
| `--no-progress` | Disable terminal progress output. Progress is written to stderr only when stderr is interactive, so JSON stdout stays valid. |
| `--format table\|json` | Output format. Default is `table`. |
| `--output PATH` | Write output to a file. Default `-` prints to stdout. |
| `--max-records N` | Debug/testing limit: stop after N valid requests. |
| `--max-events N` | Debug/testing limit: stop after N trace blocks. |

## Output Semantics

- Hit rate is measured over the last 50% of requests.
- Budget points that do not fill the cache before that measurement window are omitted, because they are not under memory pressure yet.
- Hit tokens count only the longest continuous cached prefix of each request. If a middle block misses, later blocks in that same request do not count as prefill hits even if their ids are already cached.
- `speedup` is an ideal prefill-only upper bound: `1 / (1 - hit_rate)`. `1.0x` means no-cache prefill throughput where every prefill input token is computed. It does not include decode, KV lookup, network, batching, scheduling, or memory bandwidth overhead.

## Performance Notes

The default C++ backend runs all cache-budget points in one batch after loading the trace and building the prefix trie once. This is usually faster than the Python backend, especially on large traces.

`--jobs` applies only to the Python backend. It parallelizes independent `(policy, cache budget)` simulation tasks. More jobs are not always faster: the default sweep has only a small number of budget points, tasks have uneven runtimes, and large traces can become memory-bandwidth limited. On many machines, `--jobs 8` can be close to the practical limit; `--jobs 32` may add process overhead without much speedup.

## Examples

Table output:

```bash
python3 -m kvcache_sim sweep \
  --trace trace.jsonl \
  --model deepseek-v4-pro \
  --kv-precision fp8_int8 \
  --indexer-precision fp4_int4 \
  --budgets-gib 1,2,4,8,16,32 \
  --jobs 8
```

JSON output:

```bash
python3 -m kvcache_sim sweep \
  --trace trace.jsonl.gz \
  --model kimi-k2.6 \
  --kv-precision bf16_fp16 \
  --format json \
  --output result.json
```

Trace without per-record `block_size`:

```bash
python3 -m kvcache_sim sweep \
  --trace sglang-converted.jsonl \
  --block-size 64 \
  --model qwen3-32b \
  --kv-precision bf16_fp16
```
