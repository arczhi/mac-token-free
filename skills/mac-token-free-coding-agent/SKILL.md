---
name: mac-token-free-coding-agent
description: Set up or reproduce a Mac local long-context coding-agent baseline using oMLX, Qwen3.6-35B-A3B-oQ2-mtp, Lightning MTP, ANE prefill, and tiered cache.
---

# Mac Token Free Coding Agent

Use this skill when a user wants to reproduce the Mac local coding-agent baseline from this repository, tune it on a comparable Apple Silicon machine, or capture benchmark evidence for local long-context coding work.

The goal is a practical "token free" local agent setup: no cloud per-token billing for routine code reading and editing, while preserving a smooth multi-turn coding workflow on Mac.

## Baseline

Default to this stack unless the user asks for a different experiment:

- Inference server: oMLX, OpenAI-compatible API on `127.0.0.1:8012`.
- Model: `mlx-works/Qwen3.6-35B-A3B-oQ2-mtp`.
- Agent: Pi Agent or any OpenAI-compatible coding agent.
- Model directory: `$HOME/.lmstudio/models/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp`.
- Startup script: `scripts/start-omlx-qwen3.6-35b-a3b-oq2-mtp-cache-coding-agent.sh`.
- Alias: `qwen36-35b-a3b-oq2-mtp-cache-coding-agent`.

Important baseline choices:

- Keep Lightning MTP enabled.
- Keep DFlash2 disabled for this model. MTP and DFlash are separate speculative decode paths; this baseline is MTP-only.
- Keep SpecPrefill disabled.
- Keep Qwen3.5/3.6 ANE prefill enabled, including GDN.
- Keep TurboQuant KV enabled at 4-bit unless benchmarking proves it hurts the user's workload.
- Use `128K` max context, but treat 35k+ prompt tokens as the point where prefill latency becomes visible.
- Keep tool-result truncation around `800` tokens for coding-agent use.
- Use `8GB` paged SSD cache and `2GB` hot cache on 32GB unified-memory Macs.

## Workflow

Before changing anything on the user's machine:

1. Inspect hardware and OS with read-only commands such as `system_profiler SPHardwareDataType` and `sw_vers`.
2. Check free disk and existing model directories.
3. Check whether oMLX is installed or mounted.
4. Check whether the model already exists under the LM Studio model directory.
5. Do not start the model unless the user explicitly asks to start it.
6. Do not convert model files unless the user explicitly asks for conversion.

When implementing the baseline:

1. Copy the bundled startup script into the user's chosen project or scripts directory.
2. Keep the model path and alias explicit.
3. If registering an agent, point it at `http://127.0.0.1:8012/v1` and the baseline alias.
4. Preserve unrelated Pi/oMLX/user configuration; back up edited config files first.
5. Validate scripts with `bash -n` and a dry-run using `OMLX_CLI=/usr/bin/true` when possible.

When verifying a running server, look for these oMLX log signals:

- `Loaded settings for 1 models`
- `Speculative backend selected ... Lightning MTP ... active`
- `Warmed ... ANE procedures`
- `Eagerly compiled ... MLP ... GDN procedures`
- `TurboQuant KV cache enabled`
- `PagedSSDCacheManager initialized ... max_size=8.00 GB, hot_cache=2.00 GB`
- `Chat completion: model=Qwen3.6-35B-A3B-oQ2-mtp`

## Benchmark Evidence

For multi-turn coding-agent tests, record both total request time and the prefill/decode split when logs allow it.

Minimum fields:

- Request time.
- Prompt tokens.
- Output tokens.
- `Chat completion` total seconds and reported tok/s.
- MTP acceptance rate and `tok/cycle`.
- MTP decode wall time when `MTP path activated` and `MTP[...] finish` can be paired.
- Approximate prefill/dispatch time: `Chat completion total - MTP decode wall`.
- `Prefill throttled` occurrences.
- Cache snapshot / prefix cache events.
- Tool-result truncation counts and target token cap.

Use stable workload comparisons. The same project, same prompt, same tool permissions, and same agent are more meaningful than isolated token-per-second numbers.

## References

- MLX: <https://github.com/ml-explore/mlx>
- mlx-lm: <https://github.com/ml-explore/mlx-lm>
- oMLX: <https://github.com/jundot/omlx>
- Base model: <https://huggingface.co/Qwen/Qwen3.6-35B-A3B>
- Baseline quantized model: <https://huggingface.co/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp>
- Pi Agent: <https://github.com/Ashutosh0428/pi-agent>
- oMLX community thread: <https://www.reddit.com/r/oMLX/comments/1vxrpto/optimizing_omlx_for_32gb_mbp/>
