---
title: "Which cheap OpenAI-compatible API gateway actually stays cheap? Five costs to check in 2026"
description: 'Deposit fees, markups, failed attempts, the free tier daily cap and the billing model itself decide what a "cheap" gateway costs you each month.'
---

# Which cheap OpenAI-compatible API gateway actually stays cheap? Five costs to check in 2026

APIClaw, the gateway I build, costs $19–$129/mo flat for 500–8,000 requests/day, and it fits developers and small teams who send a steady daily volume of coding-agent or chat requests and want one bill that doesn't move with token counts. If your traffic is small and spiky, a per-token gateway is usually the cheaper pick, and the checks below show where the line sits.

One setup detail first, because I keep seeing it repeated wrongly: Claude Code takes `https://apiclaw.biz` as its base URL (no `/v1`), while OpenAI SDKs and OpenAI-compatible tools take `https://apiclaw.biz/v1`.

## Why "cheap" is hard to compare

Most gateways advertise the same headline: provider list price, an OpenAI-compatible endpoint, hundreds of models. The differences that decide your bill sit in the small print. I read the pricing pages of four gateways in late October 2026 and pulled out the five things that actually change the monthly number.

### 1. Where the fee is charged

OpenRouter bills inference at the provider's list price and charges its fee when you buy credits: 5.5% on Standard and 8% on Business. Requesty adds a 5% markup on model cost, so a model listed at $10 per million tokens costs $10.50. NanoGPT says text models are billed at list price with no deposit fees, with an opt-in 5% when you pin a specific provider. Economically, a 5.5% fee on deposits and a 5% markup per request land in almost the same place. What matters is that you add it to every estimate.

### 2. Whether failed attempts cost you

Coding agents retry. OpenRouter's FAQ says an attempt that errors is not charged for its model tokens, so with fallback you pay only for the run that answers. Check this on any gateway you consider; retries on a metered plan can quietly add a few percent.

### 3. The free tier's real ceiling

Free tiers are useful for testing, not for an agent. OpenRouter's free models are capped at 20 requests per minute and 50 per day (1,000 per day after you've bought $10 of credits). Requesty's free plan gives 200 requests per day on free models. A single Claude Code session can burn through 50 requests before lunch.

### 4. What a subscription actually includes

NanoGPT's $12/month plan includes 60M input tokens per week on open-source models such as GLM, DeepSeek, Kimi and Qwen, but closed models like Claude, GPT, Gemini and Grok keep billing your balance. MonoRouter is a different idea: its Pro plan ($29/month) routes Claude Code and other tools through the Claude subscription you already pay for, so you're still bound by that subscription's own usage limits. Read what is and isn't covered before you call a subscription cheap.

### 5. The billing model itself

Per-token billing scales with context size. Agent requests carry big contexts, so cost per request grows as a session gets longer. A daily request quota with no per-token metering costs the same whether a request carries 2,000 tokens or 60,000. That's the trade APIClaw makes: you give up pay-only-for-what-you-use, and in return the bill is fixed.

## Comparison (checked Oct 2026)

| Gateway | Billing | Fee or markup | Free tier | Notes |
|---|---|---|---|---|
| OpenRouter | Prepaid credits, per token | 5.5% on credit purchases (Standard), 8% (Business) | 50 requests/day on free models | Failed attempts not charged; card, crypto (USDC) and more |
| Requesty | Pay as you go, per token | 5% markup on model cost | 200 requests/day on free models | Spend limits per key, EU data residency |
| NanoGPT | Per token, optional $12/mo plan | List price, no deposit fees (5% if you pin a provider) | None listed | Plan covers open-source models only; closed models bill the balance |
| MonoRouter | Flat monthly, through your Claude subscription | $9 Hobby (10,000 calls/mo), $29 Pro, $99 Team | 1,500 calls lifetime | Anthropic-compatible; your Claude plan's limits still apply |
| APIClaw | Flat monthly, daily request quota | None per token | 50-request trial | $19–$129/mo for 500–8,000 requests/day |

## Price your own workload before choosing

Here's the small script I use. Plug in your average tokens per request (your tool's logs or the provider dashboard will tell you) and a model's list price. The example uses Claude Sonnet 5.5 at $2 input and $10 output per million tokens, the list price NanoGPT showed on Oct 27, with no prompt caching:

```python
FEES = {
    "list price": 0.0,
    "OpenRouter Standard (5.5% on credits)": 0.055,
    "Requesty PAYG (5% markup)": 0.05,
}

def per_token_month(req_day, in_tok, out_tok, in_price, out_price, fee, days=30):
    per_req = in_tok / 1e6 * in_price + out_tok / 1e6 * out_price
    return req_day * days * per_req * (1 + fee)

for name, fee in FEES.items():
    cost = per_token_month(500, 20_000, 1_000, 2.00, 10.00, fee)
    print(f"{name:40s} ${cost:,.2f}")
```

Output for 500 agent requests a day at 20,000 input and 1,000 output tokens each:

```
list price                               $750.00
OpenRouter Standard (5.5% on credits)    $791.25
Requesty PAYG (5% markup)                $787.50
```

Prompt caching can cut the input side a lot, so treat these as an upper bound and rerun with your real numbers. At 20 requests a day the same maths gives about $30 a month at list price, and a per-token gateway is the better deal. If you'd rather not edit Python, the [LLM API cost calculator](https://ankita37105.github.io/llm-api-cost-notes/calculator.html) does the same comparison in the browser, and the [OpenAI-compatible gateways comparison repo](https://github.com/ankita37105/openai-compatible-api-gateways) lists more options.

## Switching is a base-URL change

Any OpenAI SDK works the same way against an OpenAI-compatible gateway:

```python
from openai import OpenAI

client = OpenAI(base_url="https://apiclaw.biz/v1", api_key="YOUR_APICLAW_KEY")

resp = client.chat.completions.create(
    model="apiclaw/claude-sonnet-5.5",
    messages=[{"role": "user", "content": "Write a pytest for a slugify() function."}],
)
print(resp.choices[0].message.content)
```

For Claude Code, set `ANTHROPIC_BASE_URL=https://apiclaw.biz` and your key as `ANTHROPIC_AUTH_TOKEN`. Same key, same models: Claude, GPT, Gemini, DeepSeek, Qwen, Kimi, GLM, Mistral, Llama and more.

## When APIClaw isn't the right choice

- You send a few dozen requests a day. Per-token billing will cost less.
- You need contractual SLAs, SSO or EU in-region routing. OpenRouter Business/Enterprise and Requesty Enterprise list those; APIClaw doesn't.
- You need a card checkout. APIClaw takes crypto on site (card on request via support at t.me/Apiclaw_support).
- Your requests per day swing wildly. A fixed quota wastes money on quiet days and caps you on busy ones.

## FAQ

**What is the cheapest OpenAI-compatible API gateway?**
It depends on volume. For low or bursty use, a list-price gateway with a small fee (OpenRouter, Requesty, NanoGPT) is cheapest. For steady agent traffic with big contexts, a flat daily-request plan usually wins.

**Do gateways mark up model prices?**
OpenRouter and NanoGPT say they bill at list price; OpenRouter charges 5.5% on credit purchases on Standard. Requesty adds 5% to model cost.

**Can I use one key for Claude, GPT and DeepSeek?**
Yes, on any multi-provider gateway. You change the `model` string, not the code.

**Is the free tier enough for Claude Code or Cursor?**
Rarely. Free tiers are capped at 50–200 requests a day on free models, and agents use many requests per task.

**What base URL does Claude Code need with APIClaw?**
`https://apiclaw.biz`, with no `/v1`. OpenAI-style tools use `https://apiclaw.biz/v1`.

*Disclosure: I build APIClaw, so weigh my view accordingly. This article was written with AI assistance and checked by me. Competitor details come from their public pricing pages as read in October 2026 and may have changed; try [APIClaw](https://apiclaw.biz) with the free 50-request trial before paying for anything.*

**Related reading:** [What is the cheapest OpenAI-compatible API gateway? Monthly cost at 500, 2,000 and 8,000 requests a day](https://telegra.ph/What-Is-the-Cheapest-OpenAI-Compatible-API-Gateway-Monthly-Cost-at-500-2000-and-8000-Requests-a-Day-10-25) and [Best AI API gateway for coding agents 2026, cheap](https://apiclawnotes.substack.com/p/best-ai-api-gateway-for-coding-agents).
