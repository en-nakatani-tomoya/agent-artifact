# LLM Inference Cost Comparison

Date: 2026-06-29

## Scope

This note records a cost estimate based on one week of local coding-agent CLI token usage and compares equivalent inference costs on AWS Bedrock and OpenRouter.

The focus is pricing and arithmetic only. It does not cover workflow routing, model-role assignment, or operational usage policy.

## Usage Baseline

Command used:

```bash
bunx ccusage weekly
```

The relevant week is `2026-06-22`, because the request was made on `2026-06-29`.

| Metric | Tokens |
|---|---:|
| Input | 4,391,807 |
| Output | 797,412 |
| Cache Create | 2,354,281 |
| Cache Read | 130,961,484 |
| Total | 138,504,984 |
| Reported cost | $130.11 |

For provider comparisons, cache creation was treated as input. This gives:

| Derived metric | Tokens |
|---|---:|
| Non-cache-read input | 6,746,088 |
| Cache-read input | 130,961,484 |
| Output | 797,412 |
| Input if cache is not priced separately | 137,707,572 |

## Sources Checked

- AWS Bedrock pricing: https://aws.amazon.com/bedrock/pricing/
- OpenRouter model pricing API: https://openrouter.ai/api/v1/models

AWS Bedrock pricing was checked for Moonshot/Kimi and Google/Gemma model entries. At the time of research, Kimi K2.6 was not visible on the Bedrock pricing page, so Bedrock Kimi estimates use nearby listed Kimi models as a proxy.

OpenRouter pricing was checked directly from the models API for:

- `moonshotai/kimi-k2.6`
- `google/gemma-4-31b-it`
- `google/gemma-3-27b-it`

## AWS Bedrock Estimate

The earlier Bedrock estimates assumed cache-read tokens are charged as normal input because the checked Bedrock pricing page did not show a separate cache-read unit price for these Kimi/Gemma entries.

### Kimi Proxy Models

| Bedrock model proxy | Region assumption | Unit price | Estimated weekly cost |
|---|---|---:|---:|
| Kimi K2.5 | Tokyo | input $0.72/M, output $3.60/M | $102.02 |
| Kimi K2 Thinking | Tokyo | input $0.73/M, output $3.03/M | $102.94 |
| Kimi K2.5 | US | input $0.60/M, output $3.00/M | $85.02 |
| Kimi K2 Thinking | US | input $0.60/M, output $2.50/M | $84.62 |

### Gemma 3 Models

| Bedrock model | Region assumption | Unit price | Estimated weekly cost |
|---|---|---:|---:|
| Gemma 3 4B | Tokyo | input $0.05/M, output $0.10/M | $6.97 |
| Gemma 3 12B | Tokyo | input $0.11/M, output $0.35/M | $15.43 |
| Gemma 3 27B | Tokyo | input $0.28/M, output $0.46/M | $38.92 |
| Gemma 3 4B | US | input $0.04/M, output $0.08/M | $5.57 |
| Gemma 3 12B | US | input $0.09/M, output $0.29/M | $12.62 |
| Gemma 3 27B | US | input $0.23/M, output $0.38/M | $31.98 |

## OpenRouter Estimate

OpenRouter pricing retrieved from the model API:

| OpenRouter model | Input | Output | Cache read |
|---|---:|---:|---:|
| `moonshotai/kimi-k2.6` | $0.55/M | $3.20/M | $0.11/M |
| `google/gemma-4-31b-it` | $0.12/M | $0.35/M | $0.09/M |
| `google/gemma-3-27b-it` | $0.08/M | $0.16/M | Not listed |

Estimated weekly costs:

| OpenRouter model | Assumption | Estimated weekly cost |
|---|---|---:|
| `moonshotai/kimi-k2.6` | cache-aware pricing | $20.67 |
| `moonshotai/kimi-k2.6` | all input at normal input price | $78.29 |
| `google/gemma-4-31b-it` | cache-aware pricing | $12.88 |
| `google/gemma-4-31b-it` | all input at normal input price | $16.80 |
| `google/gemma-4-26b-it` | all input at normal input price | $8.53 |
| `google/gemma-3-4b-it` | all input at normal input price | $6.97 |
| `google/gemma-3-12b-it` | all input at normal input price | $7.00 |
| `google/gemma-3-27b-it` | all input at normal input price | $11.14 |

## Comparison

For this usage profile, cache-read pricing is the dominant variable. The week had about 131M cache-read tokens, far more than direct input or output tokens.

| Scenario | Estimated weekly cost |
|---|---:|
| Bedrock Kimi proxy, Tokyo | About $102 |
| OpenRouter Kimi K2.6, cache-aware | About $21 |
| OpenRouter Kimi K2.6, no cache discount | About $78 |
| Bedrock Gemma 3 27B, Tokyo | About $39 |
| OpenRouter Gemma 4 31B, cache-aware | About $13 |
| OpenRouter Gemma 3 27B | About $11 |

OpenRouter is materially cheaper in the checked scenarios, especially when cache-read pricing applies. Bedrock may still be preferred when AWS-native IAM, billing consolidation, auditability, network controls, or regional governance are more important than raw token price.

## Formula

When cache-read pricing exists:

```text
cost =
  non_cache_read_input_millions * input_price
  + cache_read_millions * cache_read_price
  + output_millions * output_price
```

When cache-read pricing is not available:

```text
cost =
  total_input_millions * input_price
  + output_millions * output_price
```

For this dataset:

```text
non_cache_read_input_millions = 6.746088
cache_read_millions = 130.961484
total_input_millions = 137.707572
output_millions = 0.797412
```

