# AWS team outreach drafts

## Slack draft — main message

Hi team — I've completed a native Qwen3 and Qwen3MoE implementation on vLLM Neuron. Both `Qwen3ForCausalLM` and `Qwen3MoeForCausalLM` are working on Trn2 with BF16, segmented prefill, prefix caching, and TP/EP support.

PR: [link TBD]

Would love to get a review when you have bandwidth — especially interested in what to prioritize next and whether there are any issues with my current approach.

### Thread 1: Implementation status

*Done:*
• Dense Qwen3 (tested with Qwen3-0.6B, TP=2, segmented prefill, APC)
• Qwen3MoE with CTE prefill + TKG decode (tested with Tongyi-DeepResearch-30B-A3B, TP=4 EP=4)
• On-device sampling with logprobs support

*Not yet implemented (could use guidance on priority):*
• FP8 KV cache
• Speculative decoding / EAGLE3
• Sliding window attention / attention sinks
• Context-parallel segmented attention
• Disaggregated inference
• Mixed prefill/decode batching

Would appreciate any feedback on correctness, design, or what you'd want me to tackle next. Happy to share checkpoints, command lines, and compiled graphs for repro.

### Thread 2: Bedrock API usage & quota request

I'm using AWS Bedrock (Claude via API) heavily for this development — AI-assisted model porting, code generation, debugging, and validation scripting. I would like to ask that if this bill can be covered under my existing credit arrangement? If so, I would also like to request increased quotas and model access for the next phase of work.

*Current usage (past week, 6 days of active development):*
```
Model              Tokens     Cost
Opus 4.6           72.6M      ~$226
Sonnet 4.5         136.3M     ~$94
Haiku 4.5          1.3M       ~$1
─────────────────────────────────────
Total              210M       ~$320/week
```

*The main pain point is throttling.* Opus 4.6 is getting 53.7% of requests rejected — 1,215 got `429 too many tokens per day 429 Too many requests, please wait before trying again` vs only 1,046 succeeded. More than half my API calls are being blocked, which seriously slows down development.

*Current model access*
I could only use Opus 4.6 and Sonnet 4.5. The three newest models (Opus 4.8, Fable 5, Sonnet 5) are all blocked with `AccessDeniedException: anthropic.claude-opus-4-8 is not available for this account`.

*Current quotas (too low):*
• Opus 4.6: 5 RPM, 3M TPM, ~2.6M tokens/day
• Sonnet 4.5: 10 RPM, 5M TPM, 10.8M tokens/day
• Opus 4.8 / Fable 5 / Sonnet 5: *no access* — returns `AccessDeniedException: anthropic.claude-opus-4-8 is not available for this account`

*Projected next-phase needs* (FP8, larger model validation with Qwen3-32B and Qwen3-235B-A22B, speculative decoding, performance tuning): 200M–400M tokens/week, ~$400–$1,200/week.

*Request:*
1. Enable access to Opus 4.8, Fable 5, Sonnet 5 (all currently quota = 0)
2. Increase rate limits:
```
Model             RPM     TPM      Tokens/day
Opus 4.6           100     60M      100M
Opus 4.8 (new)     100     60M      100M
Fable 5 (new)      100     60M      100M
Sonnet 5 (new)    100     60M      100M
Sonnet 4.5        100     60M      100M
```

Can this be covered under my existing credit arrangement? If there's a separate process for quota increases or new model access, happy to file whatever is needed.

## Email draft

Subject: Review request: native Qwen3/Qwen3MoE on vLLM Neuron + Bedrock quota request

Hi AWS Neuron team,

I have completed a first native BF16 implementation for Qwen3 and Qwen3MoE on
the vLLM Neuron 0.21.0 codebase and would like to coordinate an upstream review.

The change includes Qwen3/Qwen3MoE architecture registration, Q/K-normalized
GQA, dense and sparse MLP paths, TP/EP checkpoint sharding, full and segmented
prefill, paged KV cache, fused decode, and automatic prefix caching. The
attention path uses the existing segmented-attention NKI implementation.

On Trn2.3xlarge, Qwen3-0.6B at TP=2 matched Hugging Face CPU BF16 for a
1509-token prompt across three 512-token prefill segments (16/16 generated
tokens), including a repeated APC request. Tongyi-DeepResearch-30B-A3B ran at
TP=4 EP=4 with full 48-layer checkpoint, compiled and produced coherent output.
A tiny 2-layer Qwen3MoE model with 32 experts/top-8 also compiled and matched
8/8 teacher-forced tokens for an 807-token segmented prompt.

Not yet implemented: FP8 KV cache, speculative decoding/EAGLE3, sliding window,
context-parallel segmented attention, disaggregated inference. I would appreciate
guidance on priority for these.

I would particularly value feedback on:

1. The canonical paged-cache write followed by segmented-attention read.
2. TP/EP semantics for Qwen3MoE and the preferred large-model qualification.
3. The expected accuracy threshold for a documented 0.25-logit BF16 top-1
   boundary difference at one 989-token prompt.
4. Whether synchronizing gathered logits only for async requests that ask for
   `logprobs` matches the preferred backend design.

I can share the branch, patch, full command lines, compiled graph hashes, and
the tiny reproducible checkpoints.

---

**Separately — Bedrock API quota request:**

I am using AWS Bedrock (Claude models) extensively for this development work
(AI-assisted porting, code generation, debugging, validation scripting). If so, I would also like to request increased quotas and model access for the next phase of work.My usage over the past week (6 days of active development):

- Total: ~210M tokens, ~$320/week
- Opus 4.6: 72.6M tokens ($226), with a 53.7% throttle rate (1,215 requests
  received `ThrottlingException: Too many requests, please wait before trying
  again` vs 1,046 succeeded — more than half my calls are blocked)
- Sonnet 4.5: 136.3M tokens ($94)
- Opus 4.8 / Fable 5 / Sonnet 5: `AccessDeniedException: anthropic.claude-opus-4-8
  is not available for this account`

The throttling on Opus is severely impacting my development velocity.
Additionally, I currently have zero quota for the three newest models (Opus 4.8,
Fable 5, Sonnet 5), which I need for the next phase of work.

For the upcoming FP8 implementation, larger model validation (Qwen3-32B,
Qwen3-235B-A22B), speculative decoding, and performance tuning, I estimate
200M–400M tokens/week (~$400–$1,200/week).

Could you help with:
1. Enabling access to Opus 4.8, Fable 5, and Sonnet 5
2. Increasing rate limits (RPM, TPM, daily cap) — see table below
3. Confirming this can be covered under my existing credit arrangement

Requested quotas:

| Model | RPM | TPM | Tokens/day |
|-------|-----|-----|------------|
| Opus 4.6 | 100 | 60M | 100M |
| Opus 4.8 (new) | 100 | 60M | 100M |
| Fable 5 (new) | 100 | 60M | 100M |
| Sonnet 5 (new) | 100 | 60M | 100M |
| Sonnet 4.5 | 100 | 60M | 100M |

Happy to file whatever formal request is needed. Thank you for any guidance on
both the code review and the quota process.

Best,
Zigeng Xu
