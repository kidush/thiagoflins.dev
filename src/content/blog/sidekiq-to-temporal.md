---
title: "Why I moved from Sidekiq to Temporal for long-running workflows"
date: 2026-03-01
description: "Sidekiq is great for fire-and-forget jobs. But when you need durable, observable, multi-step workflows, Temporal changes the game."
tags: ["Ruby", "Temporal", "Sidekiq", "Architecture"]
draft: false
---

At TrustLayer, we process third-party compliance workflows that span multiple external API calls, retries, and state transitions. For a long time, Sidekiq served us well — it's battle-tested, simple, and integrates cleanly with Rails.

But as our workflows grew more complex, we kept hitting the same wall: **visibility and durability**.

## The problem with Sidekiq for complex workflows

Sidekiq jobs are stateless by design. When a job fails mid-way through a multi-step process, you lose context. You either rebuild state from scratch or bolt on a home-grown state machine — which quickly becomes a maintenance burden.

Some specific pain points we ran into:

- **No native workflow state** — you have to reconstruct what step you were on from external DB records
- **Limited observability** — the Web UI shows you a queue, not a business process
- **Cascading job chains** — one job enqueues the next, making tracing a nightmare

## Enter Temporal

Temporal treats workflows as first-class, durable, replayable units. A workflow is just code — regular Ruby (or Go, TypeScript, etc.) — that Temporal makes fault-tolerant for you.

```ruby
class InsuranceVerificationWorkflow < Temporal::Workflow
  def execute(vendor_id:)
    policy = workflow.execute_activity(
      FetchPolicyActivity,
      vendor_id: vendor_id,
      schedule_to_close_timeout: 30
    )

    workflow.execute_activity(
      ValidateCoverageActivity,
      policy: policy,
      schedule_to_close_timeout: 60
    )

    workflow.execute_activity(
      NotifyComplianceActivity,
      vendor_id: vendor_id
    )
  end
end
```

If `ValidateCoverageActivity` fails, Temporal retries it automatically — and when it resumes, the workflow continues exactly where it left off. No extra DB columns, no custom state machines.

## What we gained

- **Full workflow history** — every step, retry, and result is persisted and queryable
- **Deterministic replay** — if the worker crashes mid-execution, Temporal replays the workflow from the beginning, skipping already-completed activities
- **Clean separation** — workflows describe *what* happens, activities describe *how*

## When to keep Sidekiq

Temporal isn't free — it adds operational complexity and a new mental model. For simple background jobs (sending emails, processing images, one-shot tasks), Sidekiq is still the right tool.

We now run both in parallel: Sidekiq for simple async work, Temporal for anything that requires durability and observability across multiple steps.

---

If you're hitting the same walls with long-running Sidekiq chains, Temporal is worth the investment.
