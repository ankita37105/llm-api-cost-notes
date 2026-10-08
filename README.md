# DeepSeek vs Qwen vs GLM API Cost for Coding Agents: A Worked Example With Real Prices (Oct 2026)

"Which is cheapest for a coding agent: DeepSeek, Qwen or GLM?" sounds like a question you can answer from a price table. Usually you can't. A coding agent doesn't send one prompt and get one answer. It sends the whole growing conversation back on every turn, so the bill depends far more on **how many turns you run, how big the context gets, and how much of it the provider caches** than on the headline price per million tokens.

Below I work through one realistic agent task with official list prices, then scale it to a month. I build a flat-rate gateway (disclosure at the end), so I've also included the comparison with flat pricing. In some cases that comparison doesn't come out in my favour.

## The list prices (as of October 2026)

All prices are USD per 1M tokens, taken from each vendor's own pricing page:

| Model | Input (cache miss) | Input (cache hit) | Output | Source |
|---|---|---|---|---|
| DeepSeek V4 Pro, peak | $1.32 | $0.044 | $3.96 | [DeepSeek](https://api-docs.deepseek.com/quick_start/pricing) |
| DeepSeek V4 Pro, off-peak | $0.66 | $0.022 | $1.98 | same |
| DeepSeek Flash (V4.1), peak | $0.30 | $0.006 | $1.20 | same |
| Qwen3.8-Max (International / Singapore) | $2.00 | $0.25 | $6.00 | [Alibaba Model Studio](https://docs.modelstudio.console.alibabacloud.com/en/model-studio/qwen3-8-max.md) |
| GLM-5.3 | $1.40 | $0.26 | $4.40 | [Z.ai](https://docs.z.ai/guides/overview/pricing) |
| GLM-5.3-Flash | $0.15 | $0.03 | $0.50 | same |

Some details that matter for agents:

- **DeepSeek's off-peak rates are half the peak rates.** Peak is 01:00â€“04:00 and 06:00â€“10:00 UTC on weekdays, which is 06:30â€“09:30 and 11:30â€“15:30 here in India. Weekends are off-peak all day.
- **Qwen3.8-Max costs less outside Singapore.** Alibaba lists $1.65 input and $4.951 output in its Global-scope regions. I used the International price above because that's what most non-China accounts default to.
- **Reasoning tokens are billed as output.** Alibaba's table literally says "chain of thought + answer". If you leave thinking mode on, which is DeepSeek's default, your output token count is much bigger than the visible reply.

## The task I'm costing

Take one typical "fix this failing test" run in an agent like Cline or Claude Code:

- **25 model calls** (read files, run tests, edit, re-run).
- **The first call sends 15,000 tokens**: system prompt, tool definitions and the relevant files.
- **Each later call adds about 3,000 tokens** (the last tool output plus the model's previous reply), so the context grows from 15k to 87k.
- **About 1,500 output tokens per call**, including reasoning.

That gives:

```text
input  = sum(15,000 + 3,000 Ã— n) for n = 0..24 = 1,275,000 tokens
output = 25 Ã— 1,500                            =    37,500 tokens
```

Look at the ratio. **Input is 97% of the tokens.** In agent loops, the input price and the cache-hit price decide the bill, not the output price.

### Caching changes everything

Each call starts with the exact text of the previous call, so in theory almost all of it can be a cache hit. In this example about 1.19M of the 1.275M input tokens are a repeated prefix, roughly 93%. Real hit rates are lower. Caches expire, some agents rewrite earlier messages, and context compaction breaks the prefix. So I've costed both ends:

```python
def task_cost(miss, hit, out, cached_share):
    inp, outp = 1_275_000, 37_500
    return (inp * (1 - cached_share) * miss + inp * cached_share * hit + outp * out) / 1e6
```

| Model | Per task, no cache | Per task, ~93% cached |
|---|---|---|
| DeepSeek V4 Pro, peak | $1.83 | $0.32 |
| DeepSeek V4 Pro, off-peak | $0.92 | $0.16 |
| DeepSeek Flash, peak | $0.43 | $0.08 |
| Qwen3.8-Max (Intl) | $2.78 | $0.70 |
| GLM-5.3 | $1.95 | $0.60 |
| GLM-5.3-Flash | $0.21 | $0.07 |

Caching alone changes the per-task cost by 3â€“6x. The difference between "DeepSeek vs Qwen vs GLM" is smaller than the difference between "cached vs not cached" for any one of them.

## Scaling to a month

Now say you run 12 of these tasks per working day, for 22 days a month. That's 264 tasks and 6,600 model calls, or about **300 calls a day**.

| Model | Month, no cache | Month, ~93% cached |
|---|---|---|
| DeepSeek V4 Pro, peak | $484 | $83 |
| DeepSeek V4 Pro, off-peak | $242 | $42 |
| DeepSeek Flash, peak | $113 | $21 |
| Qwen3.8-Max (Intl) | $733 | $184 |
| GLM-5.3 | $515 | $157 |
| GLM-5.3-Flash | $55 | $18 |

Two takeaways:

1. **The small models are really cheap per token.** With good caching, DeepSeek Flash and GLM-5.3-Flash cost about $20 a month for this workload. It's hard for any subscription to beat that.
2. **The flagship models have a wide range.** The same month on Qwen3.8-Max can cost $184 or $733 depending on cache behaviour you don't fully control. That uncertainty is the real cost of per-token billing for agents.

## How flat pricing compares

Flat plans meter something other than tokens. For a single vendor, Z.ai sells a [GLM Coding Plan](https://docs.z.ai/devpack/overview) from $18/month, with 5-hour and weekly credit limits that are themselves computed from tokens. For one model family that's often the cheapest predictable option.

For several vendors behind one key, my own gateway counts **requests per day** instead: $19/month for 500 a day, $39/month for 1,000. Heavier models count as more than one request. The public model list currently gives `deepseek-v4-pro` a weight of 1, and `qwen3.8-max`, `glm-5.3` and `glm-5.3-flash` a weight of 2. For the 300 calls a day above:

| Model | Weighted requests/day | Flat plan needed | Per-token range (month) |
|---|---|---|---|
| DeepSeek V4 Pro | 300 | $19 (500/day) | $42â€“$484 |
| Qwen3.8-Max | 600 | $39 (1,000/day) | $184â€“$733 |
| GLM-5.3 | 600 | $39 (1,000/day) | $157â€“$515 |
| GLM-5.3-Flash | 600 | $39 (1,000/day) | $18â€“$55 |

Read honestly, that table says:

- **GLM-5.3-Flash or DeepSeek Flash with decent caching:** pay per token. Flat is more expensive.
- **DeepSeek V4 Pro:** at this volume flat comes out ahead ($19 against at least $42). Halve the workload, run it off-peak with good caching, and per-token drops to about $21, which is roughly even.
- **Qwen3.8-Max or GLM-5.3 as a daily driver:** flat is cheaper even in the best per-token case for this workload.
- **Different models for different jobs:** the flat price stays the same whichever one you pick, which is the main reason I built it.

## Run your own numbers

Don't trust my 25 calls and 3,000 tokens per turn. Your agent, repo and habits will be different. Most agents and gateways log token usage per request. Take one real day, add up the cache-miss, cache-hit and output tokens per model, multiply by the table at the top, and count your calls. If your monthly per-token estimate is under about $20, stay per-token. If it's well above your flat-plan price, or it swings a lot from week to week, the flat meter is probably the better fit.

*Disclosure: I build [APIClaw](https://apiclaw.biz), the request-based gateway mentioned above. Vendor prices are from their official pricing pages as of October 2026, and vendors change prices, so recheck them before relying on these numbers.*
