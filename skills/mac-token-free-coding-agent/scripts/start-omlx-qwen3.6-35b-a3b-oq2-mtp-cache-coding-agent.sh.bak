#!/usr/bin/env bash
set -euo pipefail

PORT="${OMLX_PORT:-8012}"
HOST="${OMLX_HOST:-127.0.0.1}"
BASE_PATH="${OMLX_BASE_PATH:-$HOME/.omlx-qwen36-35b-a3b-oq2-mtp-cache-coding-agent}"
MODEL_DIR="${OMLX_MODEL_DIR:-$HOME/.lmstudio/models}"
TARGET_MODEL="${OMLX_TARGET_MODEL:-Qwen3.6-35B-A3B-oQ2-mtp}"
TARGET_MODEL_PATH="${OMLX_TARGET_MODEL_PATH:-$HOME/.lmstudio/models/mlx-works/Qwen3.6-35B-A3B-oQ2-mtp}"
MODEL_ALIAS="${OMLX_MODEL_ALIAS:-qwen36-35b-a3b-oq2-mtp-cache-coding-agent}"
OMLX_CLI="${OMLX_CLI:-}"

CACHE_DIR="${OMLX_CACHE_DIR:-$HOME/.omlx-cache/${MODEL_ALIAS}}"
SSD_CACHE_SIZE="${OMLX_SSD_CACHE_SIZE:-8GB}"
HOT_CACHE_SIZE="${OMLX_HOT_CACHE_SIZE:-2GB}"
MAX_CONCURRENT_REQUESTS="${OMLX_MAX_CONCURRENT_REQUESTS:-1}"
MEMORY_GUARD_GB="${OMLX_MEMORY_GUARD_GB:-27}"
MAX_CONTEXT_WINDOW="${OMLX_MAX_CONTEXT_WINDOW:-131072}"
MAX_TOOL_RESULT_TOKENS="${OMLX_MAX_TOOL_RESULT_TOKENS:-800}"
MAX_TOKENS="${OMLX_MAX_TOKENS:-2048}"
TEMPERATURE="${OMLX_TEMPERATURE:-0.2}"
TOP_P="${OMLX_TOP_P:-0.95}"
TOP_K="${OMLX_TOP_K:-20}"

TURBOQUANT_KV_ENABLED="${OMLX_TURBOQUANT_KV_ENABLED:-true}"
TURBOQUANT_KV_BITS="${OMLX_TURBOQUANT_KV_BITS:-4}"
TURBOQUANT_SKIP_LAST="${OMLX_TURBOQUANT_SKIP_LAST:-true}"

QWEN35_ANE_PREFILL_ENABLED="${OMLX_QWEN35_ANE_PREFILL_ENABLED:-true}"
QWEN35_ANE_PREFILL_SEQUENCE_LENGTH="${OMLX_QWEN35_ANE_PREFILL_SEQUENCE_LENGTH:-2048}"
QWEN35_ANE_PREFILL_FRACTION="${OMLX_QWEN35_ANE_PREFILL_FRACTION:-0.53}"
QWEN35_ANE_PREFILL_MAX_LAYERS="${OMLX_QWEN35_ANE_PREFILL_MAX_LAYERS:-64}"
QWEN35_ANE_PREFILL_DUAL_ANE="${OMLX_QWEN35_ANE_PREFILL_DUAL_ANE:-true}"
QWEN35_ANE_PREFILL_GDN="${OMLX_QWEN35_ANE_PREFILL_GDN:-true}"
QWEN35_ANE_PREFILL_GDN_FRACTION="${OMLX_QWEN35_ANE_PREFILL_GDN_FRACTION:-0.5}"
QWEN35_ANE_PREFILL_GDN_MAX_LAYERS="${OMLX_QWEN35_ANE_PREFILL_GDN_MAX_LAYERS:-48}"

if [[ -z "$OMLX_CLI" ]]; then
  for candidate in \
    "/Volumes/oMLX/oMLX.app/Contents/MacOS/omlx-cli" \
    "/Applications/oMLX.app/Contents/MacOS/omlx-cli" \
    "$HOME/.omlx/bin/omlx" \
    "/opt/homebrew/bin/omlx"
  do
    if [[ -x "$candidate" ]]; then
      OMLX_CLI="$candidate"
      break
    fi
  done
fi

if [[ ! -x "$OMLX_CLI" ]]; then
  cat >&2 <<EOF
oMLX CLI not found.

Checked:
  /Volumes/oMLX/oMLX.app/Contents/MacOS/omlx-cli
  /Applications/oMLX.app/Contents/MacOS/omlx-cli
  $HOME/.omlx/bin/omlx
  /opt/homebrew/bin/omlx

Install or mount oMLX, or run with:
  OMLX_CLI=/path/to/omlx ./$(basename "$0")
EOF
  exit 1
fi

if [[ ! -d "$TARGET_MODEL_PATH" ]]; then
  echo "Target model not found: $TARGET_MODEL_PATH" >&2
  echo "Download with:" >&2
  echo "  hf download mlx-works/Qwen3.6-35B-A3B-oQ2-mtp --local-dir '$TARGET_MODEL_PATH'" >&2
  exit 1
fi

mkdir -p "$BASE_PATH/logs" "$CACHE_DIR"

cat > "$BASE_PATH/model_settings.json" <<EOF
{
  "version": 1,
  "models": {
    "$TARGET_MODEL": {
      "model_alias": "$MODEL_ALIAS",
      "model_type_override": "llm",
      "max_context_window": $MAX_CONTEXT_WINDOW,
      "max_tool_result_tokens": $MAX_TOOL_RESULT_TOKENS,
      "enable_thinking": false,
      "preserve_thinking": false,
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": false
      },
      "forced_ct_kwargs": [
        "enable_thinking",
        "preserve_thinking"
      ],
      "turboquant_kv_enabled": $TURBOQUANT_KV_ENABLED,
      "turboquant_kv_bits": $TURBOQUANT_KV_BITS,
      "turboquant_skip_last": $TURBOQUANT_SKIP_LAST,
      "specprefill_enabled": false,
      "mtp_enabled": true,
      "vlm_mtp_enabled": false,
      "dflash_enabled": false,
      "qwen35_ane_prefill_enabled": $QWEN35_ANE_PREFILL_ENABLED,
      "qwen35_ane_prefill_sequence_length": $QWEN35_ANE_PREFILL_SEQUENCE_LENGTH,
      "qwen35_ane_prefill_fraction": $QWEN35_ANE_PREFILL_FRACTION,
      "qwen35_ane_prefill_max_layers": $QWEN35_ANE_PREFILL_MAX_LAYERS,
      "qwen35_ane_prefill_dual_ane": $QWEN35_ANE_PREFILL_DUAL_ANE,
      "qwen35_ane_prefill_gdn": $QWEN35_ANE_PREFILL_GDN,
      "qwen35_ane_prefill_gdn_fraction": $QWEN35_ANE_PREFILL_GDN_FRACTION,
      "qwen35_ane_prefill_gdn_max_layers": $QWEN35_ANE_PREFILL_GDN_MAX_LAYERS,
      "max_tokens": $MAX_TOKENS,
      "temperature": $TEMPERATURE,
      "top_p": $TOP_P,
      "top_k": $TOP_K
    }
  }
}
EOF

echo "Starting oMLX Qwen3.6 35B A3B oQ2 MTP + cache coding-agent server"
echo "  endpoint: http://$HOST:$PORT/v1"
echo "  base:     $BASE_PATH"
echo "  cache:    $CACHE_DIR (ssd=$SSD_CACHE_SIZE, hot=$HOT_CACHE_SIZE)"
echo "  model:    $TARGET_MODEL"
echo "  alias:    $MODEL_ALIAS"
echo "  target:   $TARGET_MODEL_PATH"
echo "  options:  thinking=false, MTP=on, DFlash2=off, SpecPrefill=off"
echo "            TurboQuant KV=$TURBOQUANT_KV_ENABLED/${TURBOQUANT_KV_BITS}bit, ANE prefill=$QWEN35_ANE_PREFILL_ENABLED"
echo "            max_concurrent_requests=$MAX_CONCURRENT_REQUESTS, max_context=$MAX_CONTEXT_WINDOW, tool_result_tokens=$MAX_TOOL_RESULT_TOKENS"
echo
echo "Stop with Ctrl-C."

exec "$OMLX_CLI" serve \
  --model-dir "$MODEL_DIR" \
  --base-path "$BASE_PATH" \
  --host "$HOST" \
  --port "$PORT" \
  --max-concurrent-requests "$MAX_CONCURRENT_REQUESTS" \
  --log-level info \
  --paged-ssd-cache-dir "$CACHE_DIR" \
  --paged-ssd-cache-max-size "$SSD_CACHE_SIZE" \
  --hot-cache-max-size "$HOT_CACHE_SIZE" \
  --memory-guard-gb "$MEMORY_GUARD_GB"
