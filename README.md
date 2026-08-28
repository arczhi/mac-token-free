# Mac Token Free

<p>
  <a href="#english"><strong>English</strong></a>
  |
  <a href="#中文">中文</a>
</p>

> The English version is shown first by default. Use the language links above as README tabs.

<a id="english"></a>

## English

Mac Token Free helps Mac users build a local coding-agent setup that feels close to "token freedom".

"Token freedom" does not mean inference has no hardware, power, or time cost. It means routine code reading, editing, shell work, and long-context agent conversations can move from cloud token quotas and per-token billing to a local Mac that you control.

This baseline combines open-source community research with my own local practice. Thanks to the MLX, mlx-lm, oMLX, Qwen, mlx-works, Pi Agent, and oMLX community contributors who made this possible. The result is a smooth local model + long-context coding-agent baseline that has been tested on a 32GB Apple Silicon Mac.

### Current Baseline

Core stack:

- Local inference server: oMLX
- Apple Silicon inference foundation: MLX / mlx-lm
- Model: `mlx-works/Qwen3.6-35B-A3B-oQ2-mtp`
- Agent: Pi Agent connected to an oMLX OpenAI-compatible endpoint
- Long-context strategy: 128K configured context, with tool-result truncation and prefix cache to control real context growth
- Decode acceleration: built-in Lightning MTP enabled
- Prefill acceleration: Qwen3.5/3.6 ANE prefill + GDN enabled
- Cache: 8GB SSD cache + 2GB hot cache
- Explicitly disabled: DFlash2 and SpecPrefill

Startup script:

[skills/mac-token-free-coding-agent/scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh](skills/mac-token-free-coding-agent/scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh)

Reusable Codex skill:

[skills/mac-token-free-coding-agent/SKILL.md](skills/mac-token-free-coding-agent/SKILL.md)

### License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE).

### Reproducible Device

Tested hardware:

| Item | Value |
| --- | --- |
| Model Name | MacBook Air |
| Model Identifier | `Mac17,3` |
| Chip | Apple M5 |
| CPU cores | 10 cores, 4 performance + 6 efficiency |
| Unified memory | 32 GB |
| macOS | 26.3 |
| Build | `25D2125` |
| oMLX profile | `qwen36-35b-a3b-oq2-mtp-cache-coding-agent` |
| oMLX endpoint | `http://127.0.0.1:8012/v1` |

Key runtime signals observed in oMLX logs:

| Item | Observed Value |
| --- | --- |
| `model_settings` | Loaded settings for 1 model |
| Speculative backend | Lightning MTP active |
| ANE prefill | warmed 136 ANE procedures |
| ANE programs | 41 MLP + 27 GDN procedures, sequence length 2048 |
| Loaded model memory | actual 14.25GB |
| TurboQuant KV | 9/40 cache layers converted to 4-bit, skipped last KVCache layer |
| Paged SSD cache | max 8.00GB |
| Hot cache | 2.00GB |
| Metal cap warning | Apple default cap about 25GB; `iogpu.wired_limit_mb` unset |

### Agent Setup

The observed agent was Pi Agent connected to local oMLX:

| Item | Value |
| --- | --- |
| Agent | Pi Agent |
| Connection | OpenAI-compatible API |
| Base URL | `http://127.0.0.1:8012/v1` |
| Model alias | `qwen36-35b-a3b-oq2-mtp-cache-coding-agent` |
| Workload | Multi-turn coding-agent interaction, mostly reading existing code |
| Typical prompt range | 9k to 45k tokens |
| Typical stop reason | `tool_calls` and `stop` |

### Parameter Baseline

| Parameter | Baseline | Notes |
| --- | ---: | --- |
| `MAX_CONTEXT_WINDOW` | `131072` | 128K upper bound for long-context coding, not an invitation to keep unlimited history |
| `MAX_TOOL_RESULT_TOKENS` | `800` | Critical for keeping agent context growth under control |
| `MAX_TOKENS` | `2048` | Caps single-turn output to keep interaction responsive |
| `TEMPERATURE` | `0.2` | Stable behavior for coding |
| `TOP_P` | `0.95` | Standard Qwen sampling setting |
| `TOP_K` | `20` | Standard Qwen sampling setting |
| `MTP` | on | Uses the model's retained MTP heads for decode acceleration |
| `DFlash2` | off | MTP and DFlash are separate speculative decode paths; this baseline prioritizes MTP |
| `SpecPrefill` | off | Reduces complexity around Qwen/ANE/cache paths |
| `TurboQuant KV` | on, 4-bit | Prioritizes memory headroom for long contexts |
| `ANE prefill` | on | Lets Apple Neural Engine share part of Qwen prefill work |
| `ANE fraction` | `0.53` | Community-benchmark-style balanced value |
| `GDN` | on | Enables Qwen3.5/3.6 prefill optimizations |
| `SSD cache` | `8GB` | Cold cache for more reusable context blocks |
| `hot cache` | `2GB` | Hot cache for recently used blocks |
| `max_concurrent_requests` | `1` | Keeps local coding-agent inference predictable |
| `memory_guard_gb` | `27` | Conservative guard for a 32GB Mac |

### Measured Results

Log source:

`/Users/alex/.omlx-qwen36-35b-a3b-oq2-mtp-cache-coding-agent/logs/server.log`

Measurement window: 2026-08-28 16:00:38 to 16:29:53.

#### Overview

| Metric | Value |
| --- | ---: |
| Completed requests | 76 |
| Prompt range | 9,404 -> 44,866 tokens |
| Total output | 11,052 tokens |
| Total end-to-end time | 1,249.04s |
| Average end-to-end time per request | 16.43s |
| Median end-to-end time | 13.18s |
| P90 end-to-end time | 30.85s |
| Weighted end-to-end throughput | 8.85 tok/s |
| Median oMLX-reported decode speed | 23.45 tok/s |

#### Prefill / Decode Split

The table below uses 67 requests that could be reliably paired through `MTP path activated -> MTP finish -> Chat completion`.

`Approx prefill/dispatch = Chat completion total time - MTP decode wall time`. This includes prefill, queueing, cache snapshots, and prefill throttling: the time a user actually feels before output arrives.

| Prompt Range | Requests | End-to-End | Approx Prefill/Dispatch | Decode Wall | Decode Throughput | Avg MTP Acceptance |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `<20k` | 23 | 182.79s | 144.83s | 37.96s | 34.5 tok/s | 60.4% |
| `20k-35k` | 26 | 323.38s | 266.88s | 56.50s | 45.1 tok/s | 59.9% |
| `>=35k` | 18 | 404.50s | 359.88s | 44.62s | 61.0 tok/s | 47.9% |
| All paired | 67 | 910.67s | 771.59s | 139.08s | 47.3 tok/s | 56.9% |

#### Recent Long-Context Turns

| Time | Prompt | Output | End-to-End | Approx Prefill | Decode | Stop |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| 16:18:50 | 36,718 | 249 | 27.65s | 25.41s | 2.24s | tool_calls |
| 16:20:05 | 38,168 | 241 | 26.33s | 23.12s | 3.21s | tool_calls |
| 16:21:27 | 38,884 | 230 | 21.55s | 19.95s | 1.60s | tool_calls |
| 16:22:40 | 40,060 | 173 | 19.60s | 17.04s | 2.56s | tool_calls |
| 16:23:16 | 40,277 | 230 | 34.51s | 32.17s | 2.34s | tool_calls |
| 16:26:23 | 41,748 | 67 | 30.85s | 29.23s | 1.62s | tool_calls |
| 16:26:57 | 42,228 | 227 | 17.07s | 15.29s | 1.78s | stop |
| 16:27:33 | 42,470 | 245 | 23.80s | 21.17s | 2.63s | tool_calls |
| 16:28:14 | 43,064 | 161 | 36.34s | 32.50s | 3.84s | tool_calls |
| 16:29:02 | 43,988 | 64 | 16.37s | 13.01s | 3.36s | tool_calls |

### Observations

Decode is healthy and responsive. MTP was active, and most measured decode windows were 1 to 4 seconds.

The main wait is prefill/dispatch/cache work. Once prompts pass roughly 35k tokens, prefill becomes the dominant latency source even though decode remains fast.

The logs showed 18 `Prefill throttled` events. This is not a script bug; it is the practical constraint of running this workload on a 32GB Mac with Apple's default Metal working-set cap around 25GB. Optimize context growth first. Raising `iogpu.wired_limit_mb` may help, but should be treated as a local system tuning experiment.

### Reproduction

1. Install oMLX.

   Use the official oMLX app or Homebrew. The script auto-detects:

   - `/Volumes/oMLX/oMLX.app/Contents/MacOS/omlx-cli`
   - `/Applications/oMLX.app/Contents/MacOS/omlx-cli`
   - `$HOME/.omlx/bin/omlx`
   - `/opt/homebrew/bin/omlx`

2. Download the model into the LM Studio model directory.

   ```bash
   mkdir -p "$HOME/.lmstudio/models/mlx-works"
   huggingface-cli download mlx-works/Qwen3.6-35B-A3B-oQ2-mtp \
     --local-dir "$HOME/.lmstudio/models/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp"
   ```

3. Start oMLX.

   ```bash
   ./skills/mac-token-free-coding-agent/scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh
   ```

4. Point Pi Agent or another OpenAI-compatible coding agent to:

   ```text
   base_url = http://127.0.0.1:8012/v1
   model = qwen36-35b-a3b-oq2-mtp-cache-coding-agent
   ```

5. Verify these log signals:

   - `Loaded settings for 1 models`
   - `Speculative backend selected ... Lightning MTP ... active`
   - `Warmed 136 ANE procedures`
   - `PagedSSDCacheManager initialized ... max_size=8.00 GB, hot_cache=2.00 GB`
   - `Chat completion: model=Qwen3.6-35B-A3B-oQ2-mtp`

### Key Open-Source Technologies

| Technology / Project | Role in This Baseline | Link |
| --- | --- | --- |
| MLX | Apple Silicon array and machine-learning foundation with unified-memory support | <https://github.com/ml-explore/mlx> |
| mlx-lm | LLM inference and model tooling in the Apple MLX ecosystem | <https://github.com/ml-explore/mlx-lm> |
| oMLX | Local OpenAI/Anthropic-compatible serving, model management, KV cache, and Mac optimization patches | <https://github.com/jundot/omlx> |
| Qwen3.6-35B-A3B | 35B total / 3B active MoE base model with long-context and MTP support | <https://huggingface.co/Qwen/Qwen3.6-35B-A3B> |
| Qwen3.6-35B-A3B-oQ2-mtp | mlx-works oQ2 MLX quantized model with retained MTP heads; default model for this baseline | <https://huggingface.co/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp> |
| Pi Agent | Coding-agent tool-use loop connected through an OpenAI-compatible endpoint | <https://github.com/Ashutosh0428/pi-agent> |
| oMLX community practice | Community experience around Mac 32GB long-context local agents, DFlash/MTP/cache tradeoffs | <https://www.reddit.com/r/oMLX/comments/1vxrpto/optimizing_omlx_for_32gb_mbp/> |

### Practical Advice

This baseline is best for coding agents that mostly read existing code: file reads, grep, local edits, repository explanation, and small patches.

Do not blindly stuff an entire repository, huge logs, or unbounded tool outputs into context. Long context is not free. Above roughly 35k tokens, decode can still be fast, but prefill latency becomes visible.

For serious comparisons, keep the same project, same prompt, same tool permissions, and same agent. Record:

- prompt tokens
- output tokens
- `Chat completion` total time
- MTP acceptance
- decode wall time
- `Prefill throttled` count
- cache snapshot / prefix hit events

These measurements are more meaningful for real agent experience than a single isolated tok/s number.

---

<a id="中文"></a>

## 中文

Mac Token Free 的目标很直接：帮助 Mac 用户在本地实现接近“token 自由”的编码 agent 体验。

这里的“token 自由”不是说推理没有硬件、电费和时间成本，而是把日常读代码、改代码、跑命令、长上下文对话这类工作，尽量从云端按 token 计费和限额里解放出来，转到自己的 Mac 上稳定运行。

这套基线方案结合了开源社区的研究成果和我的本地实践。感谢 MLX、mlx-lm、oMLX、Qwen、mlx-works、Pi Agent 以及 oMLX 社区里持续做 Mac 本地推理优化的人。这个仓库沉淀的是一个已经在 32GB Apple Silicon Mac 上跑得比较顺的本地模型 + 长上下文 coding agent 配置。

### 当前基线

核心组合：

- 本地推理框架：oMLX
- 底层 Apple Silicon 推理生态：MLX / mlx-lm
- 模型：`mlx-works/Qwen3.6-35B-A3B-oQ2-mtp`
- Agent：Pi Agent，连接 oMLX 的 OpenAI-compatible endpoint
- 长上下文策略：128K 配置上限，coding 过程通过工具结果截断和 prefix cache 控制实际上下文膨胀
- Decode 加速：保留模型内置 Lightning MTP
- Prefill 加速：启用 Qwen3.5/3.6 ANE prefill + GDN 路径
- Cache：8GB SSD cache + 2GB hot cache
- 明确不启用：DFlash2、SpecPrefill

启动脚本在：

[skills/mac-token-free-coding-agent/scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh](skills/mac-token-free-coding-agent/scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh)

可复用的 Codex skill 在：

[skills/mac-token-free-coding-agent/SKILL.md](skills/mac-token-free-coding-agent/SKILL.md)

### License

本项目使用 Apache License 2.0。详见 [LICENSE](LICENSE)。

### 可复现设备

本次实测机器：

| 项 | 信息 |
| --- | --- |
| Model Name | MacBook Air |
| Model Identifier | `Mac17,3` |
| Chip | Apple M5 |
| CPU cores | 10 cores, 4 performance + 6 efficiency |
| Unified memory | 32 GB |
| macOS | 26.3 |
| Build | `25D2125` |
| oMLX profile | `qwen36-35b-a3b-oq2-mtp-cache-coding-agent` |
| oMLX endpoint | `http://127.0.0.1:8012/v1` |

oMLX 日志里的关键运行状态：

| 项 | 观测值 |
| --- | --- |
| `model_settings` | Loaded settings for 1 model |
| Speculative backend | Lightning MTP active |
| ANE prefill | warmed 136 ANE procedures |
| ANE programs | 41 MLP + 27 GDN procedures, sequence length 2048 |
| Loaded model memory | actual 14.25GB |
| TurboQuant KV | 9/40 cache layers converted to 4-bit, skipped last KVCache layer |
| Paged SSD cache | max 8.00GB |
| Hot cache | 2.00GB |
| Metal cap warning | Apple default cap about 25GB; `iogpu.wired_limit_mb` unset |

### Agent 信息

本次实测的 agent 是 Pi Agent，连接本机 oMLX：

| 项 | 信息 |
| --- | --- |
| Agent | Pi Agent |
| 连接方式 | OpenAI-compatible API |
| Base URL | `http://127.0.0.1:8012/v1` |
| Model alias | `qwen36-35b-a3b-oq2-mtp-cache-coding-agent` |
| 工作负载 | 多轮 coding agent 交互，读已有代码为主 |
| 典型 prompt 范围 | 9k 到 45k tokens |
| 典型 stop reason | `tool_calls` 和 `stop` |

### 参数基线

| 参数 | 当前基线 | 说明 |
| --- | ---: | --- |
| `MAX_CONTEXT_WINDOW` | `131072` | 128K 上限，适合长上下文 coding，但不鼓励无限堆历史 |
| `MAX_TOOL_RESULT_TOKENS` | `800` | 限制工具结果膨胀，这是 agent 体感流畅的关键之一 |
| `MAX_TOKENS` | `2048` | 单轮输出上限，避免长篇输出拖慢交互 |
| `TEMPERATURE` | `0.2` | coding agent 偏稳定 |
| `TOP_P` | `0.95` | 保持常规采样空间 |
| `TOP_K` | `20` | Qwen 常见设置 |
| `MTP` | on | 使用模型保留的 MTP head 做 decode 加速 |
| `DFlash2` | off | 与 MTP 属于两条 speculative decode 路径；coding 读代码场景优先 MTP |
| `SpecPrefill` | off | 控制变量，避免和 Qwen/ANE/cache 路径叠复杂度 |
| `TurboQuant KV` | on, 4-bit | 长上下文下优先保护内存 |
| `ANE prefill` | on | 让 Apple Neural Engine 分担 Qwen prefill 工作 |
| `ANE fraction` | `0.53` | 社区 benchmark 常用的平衡值 |
| `GDN` | on | 启用 Qwen3.5/3.6 相关 prefill 优化 |
| `SSD cache` | `8GB` | 冷 cache，存更多可复用上下文块 |
| `hot cache` | `2GB` | 热 cache，放最近最常用的块，换取更快命中 |
| `max_concurrent_requests` | `1` | 本地 coding agent 优先单请求低干扰 |
| `memory_guard_gb` | `27` | 32GB 机器保守保护系统可用性 |

### 实测数据

日志来源：

`/Users/alex/.omlx-qwen36-35b-a3b-oq2-mtp-cache-coding-agent/logs/server.log`

统计时间：2026-08-28 16:00:38 到 16:29:53。

#### 总览

| 指标 | 数据 |
| --- | ---: |
| 完成请求数 | 76 |
| prompt 范围 | 9,404 -> 44,866 tokens |
| 输出总量 | 11,052 tokens |
| 端到端总耗时 | 1,249.04s |
| 平均每轮端到端 | 16.43s |
| 中位数端到端 | 13.18s |
| P90 端到端 | 30.85s |
| 端到端加权吞吐 | 8.85 tok/s |
| oMLX 报告 decode 中位数 | 23.45 tok/s |

#### Prefill / Decode 拆分

下面是 67 个能稳定配对 `MTP path activated -> MTP finish -> Chat completion` 的请求。`prefill/调度近似 = Chat completion 总耗时 - MTP decode wall time`，包含 prefill、排队、cache snapshot、prefill throttle 等用户体感等待。

| prompt 区间 | 轮数 | 端到端 | prefill/调度近似 | decode wall | decode 吞吐 | 平均 MTP 接受率 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `<20k` | 23 | 182.79s | 144.83s | 37.96s | 34.5 tok/s | 60.4% |
| `20k-35k` | 26 | 323.38s | 266.88s | 56.50s | 45.1 tok/s | 59.9% |
| `>=35k` | 18 | 404.50s | 359.88s | 44.62s | 61.0 tok/s | 47.9% |
| 全部可配对 | 67 | 910.67s | 771.59s | 139.08s | 47.3 tok/s | 56.9% |

#### 最近 20 轮参考

| 时间 | prompt | 输出 | 端到端 | prefill近似 | decode | stop |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| 16:18:50 | 36,718 | 249 | 27.65s | 25.41s | 2.24s | tool_calls |
| 16:20:05 | 38,168 | 241 | 26.33s | 23.12s | 3.21s | tool_calls |
| 16:21:27 | 38,884 | 230 | 21.55s | 19.95s | 1.60s | tool_calls |
| 16:22:40 | 40,060 | 173 | 19.60s | 17.04s | 2.56s | tool_calls |
| 16:23:16 | 40,277 | 230 | 34.51s | 32.17s | 2.34s | tool_calls |
| 16:26:23 | 41,748 | 67 | 30.85s | 29.23s | 1.62s | tool_calls |
| 16:26:57 | 42,228 | 227 | 17.07s | 15.29s | 1.78s | stop |
| 16:27:33 | 42,470 | 245 | 23.80s | 21.17s | 2.63s | tool_calls |
| 16:28:14 | 43,064 | 161 | 36.34s | 32.50s | 3.84s | tool_calls |
| 16:29:02 | 43,988 | 64 | 16.37s | 13.01s | 3.36s | tool_calls |

### 观测结论

Decode 侧是流畅的：MTP 明确启用，多数轮 decode wall time 在 1 到 4 秒。

真正的等待主要在 prefill/调度/cache 侧。prompt 超过大约 35k tokens 后，prefill 会成为主要延迟来源。

日志里出现 18 次 `Prefill throttled`。这不是脚本错误，而是 32GB Mac 上 Metal working set cap 约 25GB 时的现实约束。继续优化时优先控制上下文增长，其次再考虑是否手动提高 `iogpu.wired_limit_mb`。

### 复现步骤

1. 安装 oMLX。

   推荐使用 oMLX 官方 DMG 或 Homebrew。脚本会按这些路径自动查找：

   - `/Volumes/oMLX/oMLX.app/Contents/MacOS/omlx-cli`
   - `/Applications/oMLX.app/Contents/MacOS/omlx-cli`
   - `$HOME/.omlx/bin/omlx`
   - `/opt/homebrew/bin/omlx`

2. 下载模型到 LM Studio 模型目录。

   ```bash
   mkdir -p "$HOME/.lmstudio/models/mlx-works"
   huggingface-cli download mlx-works/Qwen3.6-35B-A3B-oQ2-mtp \
     --local-dir "$HOME/.lmstudio/models/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp"
   ```

3. 启动 oMLX。

   ```bash
   ./skills/mac-token-free-coding-agent/scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh
   ```

4. 将 Pi Agent 或其他 OpenAI-compatible coding agent 指向：

   ```text
   base_url = http://127.0.0.1:8012/v1
   model = qwen36-35b-a3b-oq2-mtp-cache-coding-agent
   ```

5. 验证日志里出现这些信号：

   - `Loaded settings for 1 models`
   - `Speculative backend selected ... Lightning MTP ... active`
   - `Warmed 136 ANE procedures`
   - `PagedSSDCacheManager initialized ... max_size=8.00 GB, hot_cache=2.00 GB`
   - `Chat completion: model=Qwen3.6-35B-A3B-oQ2-mtp`

### 关键开源技术

| 技术/项目 | 本方案里的作用 | 地址 |
| --- | --- | --- |
| MLX | Apple Silicon 上的数组和机器学习基础框架，利用统一内存 | <https://github.com/ml-explore/mlx> |
| mlx-lm | Apple MLX 生态里的 LLM 推理/模型工具基础 | <https://github.com/ml-explore/mlx-lm> |
| oMLX | OpenAI/Anthropic-compatible 本地服务、模型管理、KV cache、Mac 优化 patch | <https://github.com/jundot/omlx> |
| Qwen3.6-35B-A3B | 35B total / 3B active 的 MoE 基座，支持长上下文和 MTP | <https://huggingface.co/Qwen/Qwen3.6-35B-A3B> |
| Qwen3.6-35B-A3B-oQ2-mtp | mlx-works 的 oQ2 MLX 量化版本，保留 MTP，是本方案默认模型 | <https://huggingface.co/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp> |
| Pi Agent | 本地 coding agent tool-use loop，用 OpenAI-compatible endpoint 连接 oMLX | <https://github.com/Ashutosh0428/pi-agent> |
| oMLX 社区实践 | Mac 32GB 长上下文、本地 agent、DFlash/MTP/cache 配置经验来源之一 | <https://www.reddit.com/r/oMLX/comments/1vxrpto/optimizing_omlx_for_32gb_mbp/> |

### 实践建议

这套配置适合读代码为主的 coding agent。读文件、grep、局部修改、解释仓库、生成小补丁，体感会比较顺。

不建议一上来把完整仓库、超长日志、巨量工具结果全塞进上下文。长上下文本身不是免费的：超过大约 35k tokens 后，decode 仍然快，但 prefill 等待会明显上来。

如果你要做严肃对比，固定同一个项目、同一个 agent prompt、同一组工具权限，记录：

- prompt tokens
- output tokens
- `Chat completion` total time
- MTP acceptance
- decode wall time
- `Prefill throttled` 次数
- cache snapshot / prefix hit 情况

这比只看单轮 tok/s 更接近真实 agent 体验。
