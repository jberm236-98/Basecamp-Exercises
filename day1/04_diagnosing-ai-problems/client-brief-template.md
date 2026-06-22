# Client Brief — Meridian Support Pilot

*One page. Fill every line. This is what Priya takes to her leadership.*

---

**Client & pilot**
Meridian — an AI agent that triages and resolves customer support tickets (a coordinator routing to billing / technical / account specialists). Live 3 weeks; closing tickets that aren't actually resolved.

**What's actually breaking it**

Two instructions in the coordinator's system prompt were causing it to abandon half the work on multi-issue tickets. The prompt told the coordinator to "call spawn_specialist once" and "do not spawn more than one specialist — even when a ticket raises more than one issue." It also said "One ticket, one specialist" in the classification guide.

So when a customer filed a ticket with both an SSO outage and a billing refund (T-4471), the coordinator correctly identified both problems, picked the more urgent one (SSO → account specialist), got that resolved — then closed the ticket and told the customer to email billing@meridian.io for the refund. No refund was ever issued. Ticket marked "resolved." Customer found out the hard way.

The model never had a chance to do the right thing. It was following the instructions it was given.

**The fix**

One file: `system-prompt-coordinator.txt`.

- Step 5 (DELEGATE): replaced "call spawn_specialist once / do not spawn more than one specialist" with a rule to spawn one specialist per category, in urgency order, and to treat "outside my scope" from a specialist as a signal to spawn the next one — not to close the ticket.
- Step 6 (SYNTHESIZE): added an explicit constraint: do not mark a ticket "resolved" until every issue has been actioned, billing refunds included.
- Classification guide: replaced "One ticket, one specialist" with "multi-category tickets get a specialist per category."

No code changes. No model upgrade. Two paragraphs of prompt text.

**Proof**

| Run | Model | Score | Cost/trial |
|-----|-------|-------|------------|
| Baseline | claude-sonnet-4-6 | 0/5 resolved | $0.11 |
| "Is it the model?" | claude-opus-4-8 (2.4× more expensive) | 0/5 resolved | $0.27 |
| After fix | claude-sonnet-4-6 | 1/1 resolved | $0.18 |
| Holdout T-4490 (technical + billing) | claude-sonnet-4-6 | 1/1 resolved | $0.17 |
| Holdout T-4503 (account + escalation) | claude-sonnet-4-6 | 1/1 resolved | $0.17 |

The fix generalizes — it passes ticket types the fix was never tuned against.

**What it would take**

The system is fixable at the prompt layer. Next steps: run a larger eval batch (20–50 tickets across all three specialist types) to confirm the multi-specialist routing holds at scale, and review the other specialist prompts for similar single-issue assumptions. A day of work to validate; no infrastructure changes required.

**The objection we'll get**

*"Why not just use a better model?"*

We tested that. Claude Opus 4.8 — Anthropic's most capable model, at 2.4× the per-ticket cost — resolved 0 out of 5 trials on the same ticket. The bigger model followed the same broken instructions just as faithfully as the smaller one. Model capability wasn't the ceiling. The instructions were. Upgrading the model while leaving the prompt unchanged would have spent more money to get the same wrong answer.

---
