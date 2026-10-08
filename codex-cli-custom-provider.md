# How to Run OpenAI Codex CLI Against a Third-Party OpenAI-Compatible Endpoint (config.toml That Works in 2026)

Codex CLI is happy to talk to providers other than OpenAI, but the setup has a few traps that waste an afternoon: the key goes in the wrong field, the provider block sits in the wrong file, or an old `wire_api = "chat"` line breaks every command. These are the notes I keep for wiring Codex to a custom endpoint.

My examples use [APIClaw](https://apiclaw.biz), a flat-rate OpenAI-compatible gateway. I build it, so weigh that accordingly. The same steps apply to any gateway or proxy that serves the Responses API.

## What you need before you touch config.toml

1. **A base URL that serves `/v1/responses`.** Current Codex only speaks the Responses API. OpenAI's config reference lists `responses` as the only supported `wire_api` value, and it's the default when you omit the line. A gateway that only serves `/v1/chat/completions` won't work with current Codex, no matter what you put in the config.
2. **An API key for that endpoint**, not an OpenAI key.
3. **The exact model ID** the endpoint uses. Copy it from the provider's model list. Don't guess from the vendor's marketing name.

A quick check that the endpoint exists before you blame Codex:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://apiclaw.biz/v1/responses \
  -H "Content-Type: application/json" -d '{}'
```

A `401` without a key is a good sign: the route exists and wants auth. A `404` means the path is wrong or the provider doesn't serve the Responses API.

## Step 1: put the key in an environment variable

Codex reads provider keys from an environment variable at runtime. The `env_key` field in the config is the **name** of that variable, never the key itself.

```bash
# ~/.zshrc or ~/.bashrc
export APICLAW_API_KEY="sk-your-key"
```

Open a new terminal and run `echo $APICLAW_API_KEY` before you start Codex. On Windows, Codex runs under WSL2, so set the variable inside your WSL distribution's `~/.bashrc`, not in PowerShell.

## Step 2: add a provider block to the user-level config

Edit `~/.codex/config.toml`:

```toml
model_provider = "apiclaw"
model = "claude-sonnet-5-5"
model_reasoning_effort = "high"

[model_providers.apiclaw]
name = "APIClaw"
base_url = "https://apiclaw.biz/v1"
env_key = "APICLAW_API_KEY"
wire_api = "responses"
```

Three details matter here:

- **It has to be the user-level file.** Codex ignores `model_provider` and `model_providers` in a project-local `.codex/config.toml`, per OpenAI's config docs. If you put the block in your repo's `.codex/` folder, Codex silently keeps using its default provider.
- **Don't reuse a reserved provider id.** `openai`, `ollama` and `lmstudio` are taken. If you only want to move the built-in OpenAI provider to a different URL, OpenAI's docs say to use the top-level `openai_base_url` key instead of creating `[model_providers.openai]`.
- **`base_url` ends at `/v1`.** Codex appends `/responses` itself.

## Step 3: verify

```bash
cd /path/to/a/project
codex
```

Ask it something small, like "list the files in this folder". If it answers, the provider is wired up. If the gateway has a request log, check that the call landed there and not at OpenAI.

## The errors I've hit, and what each one means

**`Missing environment variable: sk-...`**
The key was pasted into `env_key`. Codex went looking for a variable literally named `sk-...`. Set `env_key = "APICLAW_API_KEY"` (or whatever you named it) and export the key separately.

**`401 Unauthorized`**
The variable isn't set in the terminal you launched Codex from, the name doesn't match `env_key`, or the key is inactive. `env | grep APICLAW` in the same shell settles the first two.

**`404 Not Found`**
Either `base_url` is wrong (missing `/v1`, or it already includes `/responses`), or the model ID doesn't exist on that endpoint. List the provider's models and copy the ID exactly.

**`wire_api = chat is no longer supported`**
You have an old provider block from when Codex could talk Chat Completions. Codex validates every provider block at startup, so even a block you no longer use breaks every command. Change it to `wire_api = "responses"`, drop the line, or delete the unused block.

**`stream disconnected before completion` or a `Reconnecting 1/5` counter**
The stream ended without a completion event and Codex is retrying. OpenAI's config reference documents `stream_max_retries` (default 5) and `stream_idle_timeout_ms` (default 300000, so five minutes) per provider. With slow reasoning models behind a gateway, raising the idle timeout often helps:

```toml
[model_providers.apiclaw]
name = "APIClaw"
base_url = "https://apiclaw.biz/v1"
env_key = "APICLAW_API_KEY"
wire_api = "responses"
stream_idle_timeout_ms = 600000
stream_max_retries = 10
request_max_retries = 4
```

If it keeps happening at the same point every time, it's usually the gateway or a proxy in between closing long-lived connections, not Codex.

## Extra headers and query parameters

Some company proxies want a tenant header or an API version in the query string. The provider block supports both, per OpenAI's advanced config page:

```toml
[model_providers.companyproxy]
name = "Company proxy"
base_url = "https://proxy.example.com/v1"
env_key = "COMPANY_PROXY_KEY"
http_headers = { "X-Team" = "platform" }
env_http_headers = { "X-Project" = "PROXY_PROJECT_ID" }
query_params = { api-version = "2025-04-01-preview" }
```

`env_http_headers` maps a header name to the name of an environment variable, the same pattern as `env_key`, so secrets stay out of the file.

## A short checklist

- Endpoint serves `/v1/responses` (curl returns 401, not 404, without a key).
- Key lives in an env var, and `env_key` holds the variable's name.
- Provider block is in `~/.codex/config.toml`, not the project's `.codex/`.
- Provider id isn't `openai`, `ollama` or `lmstudio`.
- `base_url` ends in `/v1`.
- Model ID copied from the provider's model list.
- No leftover `wire_api = "chat"` blocks anywhere in the file.

That covers every Codex-to-gateway problem I've run into so far. If you hit one that isn't here, the Codex config reference on developers.openai.com is the source of truth for which keys exist.
