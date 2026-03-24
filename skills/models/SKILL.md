---
name: Models
description: Choose AI models for coding, reasoning, and agents with cost-aware, task-matched recommendations.
metadata: {"clawdbot":{"emoji":"🤖","os":["linux","darwin","win32"]}}
---

# AI Model Selection Rules

## Core Principle
- No single model is best for everything — match model to task, not brand loyalty
- A $0.75/M model often performs identically to a $40/M model for simple tasks
- Test cheaper alternatives before committing to expensive defaults

## Cost Reality
- Output tokens cost 3-10x more than input tokens — advertised input prices are misleading
- Calculate real cost with your actual input/output ratio, not theoretical pricing
- Batch/async APIs offer 50% discounts — use them for non-real-time workloads
- Prompt caching reduces repeated context costs significantly

## Task Matching

### Coding
- Architecture and design decisions: Use frontier models (Opus-class) — they catch subtle issues cheaper models miss
- Day-to-day implementation: Mid-tier models (Sonnet-class) offer 90% of capability at 20% of cost
- Parallel subtasks and scaffolding: Fast/cheap models (Haiku-class) — speed matters more than depth
- Code review: Thorough models catch async bugs and edge cases that fast models miss

### Non-Coding
- Complex reasoning and math: Extended thinking modes justify their cost for hard problems
- General assistance: User preference studies favor models different from benchmark leaders
- High-volume simple queries: Cheapest models perform identically — don't overpay
- Long documents: Context window size determines viability — some offer 1M+ tokens

## Claude Code vs Codex CLI
- Claude Code: Fast iteration, UI/frontend, interactive debugging — developer stays in the loop
- Codex CLI: Long-running background tasks, large refactors, set-and-forget — accuracy over speed
- Both tools have value — use Claude Code for implementation, Codex for final review
- File size limits differ — Claude Code struggles with files over 25K tokens

## Orchestration Pattern
- Planning phase: Use expensive/smart models to break down problems correctly
- Execution phase: Use balanced models, parallelize where possible
- Review phase: Use accurate models for final verification — catches bugs others miss
- This pattern beats using one model for everything at similar total cost

## Benchmark Skepticism
- Benchmark scores vary 2-3x based on scaffolding and evaluation method
- User preference rankings differ significantly from benchmark rankings
- SWE-bench scores don't predict real-world coding quality reliably
- Models drift week-to-week — last month's best may underperform today

## Open Source Viability
- DeepSeek and similar models approach frontier performance at 1/50th API cost
- Self-hosting eliminates API rate limits and price variability
- MIT/Apache licensed models allow commercial use without restrictions
- Consider for: data privacy, cost predictability, custom fine-tuning

## Model Selection Mistakes
- Using premium models for chatbot responses that cheap models handle identically
- Ignoring context window limits — chunking long documents costs more than using large-context models
- Expecting consistency — same prompt gives different results over time as models update
- Trusting speed over accuracy for complex tasks — fast models trade thoroughness for latency

## Practical Guidelines
- Default to mid-tier for most tasks, escalate to frontier only when quality suffers
- Track actual costs per workflow, not just per-token rates
- Build verification into pipelines — don't trust any model blindly
- Reassess model choices quarterly — pricing and capabilities shift constantly
