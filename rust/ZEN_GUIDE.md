# Claw Code + OpenCode Zen Integration Guide

Use OpenCode Zen free models with Claw Code — zero cost, full feature parity.

## Quick Start

```bash
# Set your Zen API key
export OPENCODE_API_KEY="sk-your-zen-key"

# Use default model (qwen3.6-plus-free)
./target/release/claw "Hello world"

# Use a specific model
./target/release/claw --model qwen "Write a Python script"
./target/release/claw --model minimax "Analyze this code"
./target/release/claw --model nemotron "Explain this architecture"
```

## Model Aliases

| Alias | Full Model | Tokens | Best For |
|-------|-----------|--------|----------|
| `qwen` | `qwen3.6-plus-free` | 32K | General coding, reasoning |
| `minimax` | `minimax-m2.5-free` | 64K | Long context, code generation |
| `nemotron` | `nemotron-3-super-free` | 64K | Complex analysis, math |
| `mimo` | `mimo-v2-omni-free` | 32K | Multimodal (image + text) |

## How It Works

### Architecture

```
┌─────────────────────────────────────────────────┐
│                   claw CLI                      │
│  Default: qwen3.6-plus-free                     │
├─────────────────────────────────────────────────┤
│              ProviderClient                     │
│  OpenCode → OpenAiCompatClient                  │
│    ├─ base_url: https://opencode.ai/zen/v1     │
│    ├─ auth: Bearer token (OPENCODE_API_KEY)    │
│    ├─ headers: x-opencode-client: cli ← CRITICAL│
│    ├─ headers: x-opencode-session: <uuid>       │
│    └─ endpoint: /chat/completions              │
├─────────────────────────────────────────────────┤
│           OpenCode Zen Free Models              │
│  qwen3.6-plus-free  (32K tokens)               │
│  minimax-m2.5-free  (64K tokens)               │
│  nemotron-3-super-free (64K tokens)            │
│  mimo-v2-omni-free  (32K tokens)               │
└─────────────────────────────────────────────────┘
```

### Critical Headers

Three headers are **mandatory** for Zen to work:

1. **`x-opencode-client: cli`** — Without this, Zen applies a punitive rate limit (~1-5 requests/day instead of ~50-200+)
2. **`x-opencode-session: <uuid>`** — Without this, ALL accounts return `FreeUsageLimitError` even with valid API keys
3. **`Authorization: Bearer <key>`** — Standard bearer token authentication

### Why `opencode.ai/zen/v1` and not `api.opencode.ai/v1`?

The `api.opencode.ai` endpoint returns `404 Not Found` for `/v1/chat/completions`. Zen uses a separate routing path at `opencode.ai/zen/v1`. This is handled internally by OpenCode's infrastructure — the `api.opencode.ai` path is for OpenCode's own TUI, not external clients.

## Implementation Details

### Files Modified

1. **`crates/api/src/providers/openai_compat.rs`**
   - Added `custom_headers: Vec<(String, String)>` field to `OpenAiCompatClient`
   - Added `with_custom_headers()` builder method
   - Modified `send_raw_request()` to apply custom headers before `content-type`
   - Added `zen()` config to `OpenAiCompatConfig`
   - Made `ChatCompletionChunk.id` optional: `#[serde(default)]`

2. **`crates/api/src/providers/mod.rs`**
   - Added `OpenCode` variant to `ProviderKind` enum
   - Added 4 Zen model entries to `MODEL_REGISTRY`
   - Added model alias resolution for Zen models
   - Added OpenCode detection in `metadata_for_model()` and `detect_provider_kind()`

3. **`crates/api/src/client.rs`**
   - Added `OpenCode(OpenAiCompatClient)` variant to `ProviderClient`
   - Wired up with `x-opencode-client: cli` + `x-opencode-session` headers
   - Updated all match arms for the new variant
   - Added `read_zen_base_url()` helper

4. **`crates/rusty-claude-cli/src/main.rs`**
   - Changed `AnthropicRuntimeClient.client` from `AnthropicClient` to `ProviderClient`
   - Updated `new()` to use provider detection with automatic Zen routing
   - Changed `DEFAULT_MODEL` to `qwen3.6-plus-free`
   - Added Zen model aliases and `max_tokens_for_model()` handling

### Zen Streaming Quirks

1. **Comment line**: Streaming starts with `: OPENROUTER PROCESSING` — our SSE parser skips `:` comment lines
2. **Post-DONE frame**: After `[DONE]`, Zen sends `{"choices":[],"cost":"0"}` with no `id` field — handled by `#[serde(default)]`

## Troubleshooting

### FreeUsageLimitError

```
error: api failed after 3 attempts: api returned 429 Too Many Requests (FreeUsageLimitError)
```

**Cause**: Your Zen account has hit its daily rate limit.

**Solutions**:
1. Wait for the daily reset (typically midnight UTC)
2. Use a different Zen API key
3. Switch to a different model (some models have separate limits)

### Not Found (404)

```
error: http error: 404 Not Found
```

**Cause**: Wrong base URL. Make sure it's `https://opencode.ai/zen/v1`, NOT `https://api.opencode.ai/v1`.

**Fix**: Set `OPENCODE_BASE_URL=https://opencode.ai/zen/v1` or use the default (which is correct).

### missing field `id`

```
error: json error: missing field `id` at line 1 column 25
```

**Cause**: This was a bug in early versions. Fixed by making `ChatCompletionChunk.id` optional with `#[serde(default)]`.

**Fix**: Update to the latest version of this repo.

### missing Anthropic credentials

```
error: missing Anthropic credentials; export ANTHROPIC_AUTH_TOKEN or ANTHROPIC_API_KEY
```

**Cause**: The `OPENCODE_API_KEY` env var is not set, so provider detection falls back to Anthropic.

**Fix**: `export OPENCODE_API_KEY="sk-zen-key"`

## Testing

```bash
# Simple prompt
export OPENCODE_API_KEY="sk-zen-key"
./target/release/claw --model qwen "What is 2+2? Answer in one word."

# Complex code generation
./target/release/claw --model minimax "Write a Python function to calculate fibonacci numbers."

# Interactive REPL
./target/release/claw --model qwen

# Check version
./target/release/claw --version
```

## Build

```bash
cd rust/
cargo build --release
# Binary: target/release/claw (~13MB)
```

## Tests

```bash
cargo test -p api
# 66 tests pass, 0 fail
```
