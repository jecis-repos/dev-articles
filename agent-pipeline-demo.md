---
title: "Self-Correcting Agent Pipelines: How I Stopped My AI From Lying to Users"
published: false
description: "4-layer directed pipelines, failure feedback loops, swarm decomposition, and the 7-state conversation machine that makes autonomous AI actually reliable in production."
tags: ai, agents, laravel, architecture
cover_image:
canonical_url:
---

Most agent tutorials show you a single LLM call with a system prompt. That works for demos. It does not work when your bot handles real conversations 24/7 and a hallucinated response means a confused human staring at their phone.

This article covers the production patterns I built into an autonomous AI assistant backed by 2,700+ tests across 17 modules. The core insight: agents should not be monolithic. They should be **pipelines of specialists that correct each other's failures**.

I will walk through the architecture, show real code, and explain how each layer prevents the kind of silent failures that make agent systems unreliable.

---

## The Problem with Single-Agent Systems

A single agent with a fat system prompt will eventually:

1. Forget context halfway through a complex task
2. Produce output that passes its own vibe check but fails in production
3. Retry the exact same broken approach when something goes wrong
4. Get stuck in a loop with no exit condition

The fix is not "better prompts." The fix is **structure** — decomposing agent work into a directed pipeline where each stage has one job, clear inputs, clear outputs, and a feedback mechanism when things go wrong.

---

## Pattern 1: The 4-Layer Directed Pipeline

Every complex task in the system flows through four specialist layers:

```
Coordinator → E2E Mapping → Coding → Verification
     ↑                                    |
     └────── failure feedback ─────────────┘
```

**Coordinator** receives the raw task (e.g., "add a /stats command that shows message counts per conversation"). It decomposes the task into structured sub-goals: what needs to change, what the acceptance criteria are, what modules are affected.

**E2E Mapping** takes the coordinator's output and generates concrete test scenarios. Not unit tests — end-to-end flows. "User sends /stats in a group chat with 3 participants. Bot responds with a formatted table within 2 seconds. Table includes columns for participant name and message count."

**Coding** receives both the task decomposition and the test scenarios. It writes the implementation. Because it has the E2E scenarios upfront, it does not have to guess what "done" looks like.

**Verification** runs the generated code against the test scenarios. It does not just check "does it compile" — it validates behavioral correctness against the E2E mapping output.

Each layer is a separate agent invocation with a focused system prompt. The coordinator prompt never mentions code. The coding prompt never mentions task decomposition. Separation of concerns, applied to AI.

---

## Pattern 2: The Failure Feedback Loop (Max 3 Iterations)

This is the pattern that made the biggest difference in production reliability.

When verification fails, the system does not blindly retry. It **appends the failure context to the coding agent's prompt** and re-dispatches. The coding agent now knows exactly what went wrong on the previous attempt.

```php
// When verification fails, re-dispatch with failure context
$codingTask->update([
    'prompt' => $originalPrompt . "\n\nPrevious Attempt Failed: " . $failureContext,
    'iteration_count' => $iteration + 1,
]);
```

This is not retry logic. This is **structured self-correction**. The difference matters:

- **Retry**: run the same thing again, hope for a different result
- **Self-correction**: run a modified version that knows what failed last time

The max iteration count of 3 is critical. Without it, you get infinite loops where the agent keeps producing variations of the same broken approach. Three attempts is enough for the agent to try genuinely different strategies. If it cannot solve the problem in three attempts, it escalates to a human — and the failure log gives the human full context on what was tried.

Here is a simplified but runnable class showing the core loop:

```php
<?php

declare(strict_types=1);

namespace App\Agent;

use App\Agent\Contracts\AgentInterface;
use App\Agent\Contracts\VerifierInterface;
use App\Agent\DTO\TaskResult;
use App\Agent\DTO\VerificationResult;

final class SelfCorrectingPipeline
{
    private const MAX_ITERATIONS = 3;

    public function __construct(
        private AgentInterface $codingAgent,
        private VerifierInterface $verifier,
    ) {}

    /**
     * Execute a task with self-correcting feedback loop.
     *
     * @param string $prompt       The original task prompt
     * @param array  $testScenarios E2E scenarios from the mapping layer
     */
    public function execute(string $prompt, array $testScenarios): TaskResult
    {
        $currentPrompt = $prompt;
        $attempts = [];

        for ($iteration = 1; $iteration <= self::MAX_ITERATIONS; $iteration++) {
            // Coding agent produces implementation
            $implementation = $this->codingAgent->generate($currentPrompt);

            // Verification agent validates against test scenarios
            $verification = $this->verifier->validate(
                implementation: $implementation,
                scenarios: $testScenarios,
            );

            $attempts[] = [
                'iteration'      => $iteration,
                'prompt_hash'    => hash('xxh3', $currentPrompt),
                'passed'         => $verification->passed,
                'failure_reason' => $verification->failureReason,
            ];

            if ($verification->passed) {
                return new TaskResult(
                    success: true,
                    implementation: $implementation,
                    attempts: $attempts,
                );
            }

            // Append failure context for the next iteration.
            // The coding agent sees exactly what went wrong.
            $currentPrompt = $this->buildCorrectionPrompt(
                original: $prompt,
                iteration: $iteration,
                verification: $verification,
            );
        }

        // Exhausted all iterations — escalate to human
        return new TaskResult(
            success: false,
            implementation: null,
            attempts: $attempts,
            escalationReason: sprintf(
                'Failed after %d iterations. Last failure: %s',
                self::MAX_ITERATIONS,
                $verification->failureReason,
            ),
        );
    }

    private function buildCorrectionPrompt(
        string $original,
        int $iteration,
        VerificationResult $verification,
    ): string {
        return implode("\n\n", [
            $original,
            "--- CORRECTION CONTEXT (attempt {$iteration} of " . self::MAX_ITERATIONS . ") ---",
            "Previous implementation FAILED verification.",
            "Failure reason: {$verification->failureReason}",
            "Failed scenarios: " . implode(', ', $verification->failedScenarios),
            "Do NOT repeat the same approach. Address the specific failure above.",
        ]);
    }
}
```

A few things worth noting about this code:

- The `prompt_hash` in the attempts array lets you verify that each iteration actually received different input. If two hashes match, something is wrong with your feedback injection.
- The correction prompt explicitly says "do NOT repeat the same approach." Without this, LLMs have a strong tendency to regenerate near-identical output.
- The `TaskResult` carries the full attempt history. This is your audit trail.

---

## Pattern 3: Swarm Decomposition

Some tasks are not sequential — they are parallelizable. When the coordinator identifies independent sub-tasks, it decomposes the master task into N children using a structured AI call:

```php
$subtasks = $this->aiClient->generate(
    prompt: "Decompose this task into independent sub-tasks: {$masterTask}",
    temperature: 0.3,       // Low creativity — we want structured output
    responseFormat: 'json',  // Force JSON array output
);
```

Temperature 0.3 is deliberate. Decomposition is an analytical task, not a creative one. Higher temperatures produce inconsistent decomposition — sometimes 3 sub-tasks, sometimes 12 for the same input.

Each sub-task is dispatched as a **Laravel queue job**. They execute in parallel across workers. Reconciliation happens only when **all** children reach a terminal state (success or failure):

```php
// In the reconciliation listener
$children = $masterTask->children;
$allTerminal = $children->every(
    fn ($child) => in_array($child->status, ['success', 'failure'])
);

if (!$allTerminal) {
    return; // Not ready yet — wait for remaining children
}

$failures = $children->where('status', 'failure');

if ($failures->isEmpty()) {
    $masterTask->update(['status' => 'success']);
} else {
    // Partial failure: some children succeeded, some didn't.
    // Log which ones failed and why, mark master as partial.
    $masterTask->update([
        'status' => 'partial_failure',
        'metadata->failed_children' => $failures->pluck('id', 'failure_reason'),
    ]);
}
```

Handling partial failure is important. If 8 out of 10 sub-tasks succeed, you do not throw away the 8 successes. You surface the 2 failures for review while keeping the successful work.

---

## Pattern 4: The 7-State Conversation Machine

Every WhatsApp conversation in the system is governed by a finite state machine with 7 states:

| State | Purpose | Valid transitions |
|---|---|---|
| `idle` | Waiting for user input | greeting |
| `greeting` | Initial acknowledgment | collecting |
| `collecting` | Gathering requirements | processing, idle |
| `processing` | AI is working | confirming, collecting |
| `confirming` | User approves the plan | executing, collecting |
| `executing` | Running the pipeline | completed, collecting |
| `completed` | Done, results delivered | idle |

Every state transition is **explicit and logged**. There is no implicit state. The conversation object always knows exactly where it is, and invalid transitions are rejected:

```php
if (!$this->canTransition($currentState, $targetState)) {
    Log::warning('Invalid state transition attempted', [
        'conversation_id' => $this->id,
        'from' => $currentState,
        'to' => $targetState,
    ]);
    return false;
}
```

This prevents the most common agent failure mode: getting stuck in a loop. If the agent tries to go from `executing` back to `executing`, the state machine rejects it. The agent must go through `collecting` first, which forces it to re-evaluate.

---

## Pattern 5: 10 Specialist Agent Roles

the system defines 10 distinct roles, each with its own optimized system prompt:

1. **Coordinator** — decomposes tasks, assigns to other roles
2. **Researcher** — gathers context from codebase and documentation
3. **E2E Mapper** — generates test scenarios from requirements
4. **Coder** — writes implementation code
5. **Reviewer** — code review against standards
6. **Deployer** — manages deployment steps
7. **Tester** — executes test scenarios
8. **Analyst** — performance and usage analysis
9. **Troubleshooter** — diagnoses failures from logs
10. **Reporter** — generates human-readable summaries

Not every pipeline uses all 10. A typical code-fix pipeline uses 3-5: Coordinator, Coder, Tester, and maybe Troubleshooter if the first iteration fails. A reporting pipeline might only use Researcher, Analyst, and Reporter.

The key principle: **each role's system prompt is under 200 words and mentions only its specific responsibility**. A Coder prompt never says "also review the code for quality." That is the Reviewer's job. Keeping prompts focused produces dramatically better output than a single mega-prompt that tries to do everything.

---

## Pattern 6: Production Safety

Four mechanisms keep the system safe in production:

**Immutable audit log with hash-chain verification.** Every action — every agent invocation, every state transition, every API call — is logged to an append-only table. Each log entry includes a hash of the previous entry, forming a chain. If any entry is tampered with, the chain breaks and alerting fires.

**Phone number masking.** WhatsApp means phone numbers everywhere. Every log line, every agent prompt, every stored conversation runs through a masking layer that replaces phone numbers with tokens (`[PHONE_a]`, `[PHONE_b]`). The mapping is stored separately with restricted access. Agent prompts never see real phone numbers.

**Rate limiting per conversation.** Each conversation is limited to N agent invocations per time window. This prevents a single runaway conversation from consuming all your AI budget. The limits are per-conversation, not global, so one bad actor does not affect other users.

**Circuit breaker per AI provider.** If OpenAI starts returning 500s, the circuit breaker trips after 5 consecutive failures and the system falls back to a secondary provider. The breaker resets after a cooldown period. Users see slightly different response quality during fallback, but they do not see errors.

---

## What This Looks Like in Production

Here is the actual flow when a user reports a bug via WhatsApp:

1. User sends "hey Joe, the /export command is showing wrong dates for last week's data" to the WhatsApp number.

2. The conversation state machine transitions from `idle` to `greeting`, then to `collecting`. The bot asks a clarifying question: "Which export format — CSV or PDF? And are you seeing this in group chats, DMs, or both?"

3. User replies "CSV, and it's in group chats."

4. State transitions to `processing`. The **Coordinator** agent receives the collected context and decomposes it: "Date calculation bug in the CSV export path, specifically for group chat message aggregation."

5. The **E2E Mapper** generates three test scenarios: export with dates in the current week, export with dates crossing a month boundary, and export with dates in a timezone-offset group.

6. The **Coder** agent receives the task decomposition and test scenarios. It generates a fix — adjusting the date range query to use `startOfWeek()` instead of `subDays(7)`.

7. The **Verifier** runs the implementation against the three scenarios. Scenario 1 passes. Scenario 3 fails — the timezone handling is still wrong.

8. **Iteration 2**: The Coder receives its original prompt plus "Previous Attempt Failed: Scenario 3 (timezone-offset group) — dates are calculated in UTC but should use the group's configured timezone." The Coder produces a revised fix that reads the group timezone setting.

9. Verification passes all three scenarios. State transitions to `confirming`. The bot sends the user a summary: "Found a date calculation bug in CSV exports for group chats. Fix adjusts timezone handling. Want me to apply it?"

10. User replies "yes." State transitions to `executing`, then `completed`. The fix is applied, tests are green, and the user gets a confirmation.

Total time: under 90 seconds. Two iterations. Full audit trail. No human developer involved.

That is what a self-correcting agent pipeline buys you. Not perfect first attempts — but a system that **converges on correct results** through structured feedback, and knows when to stop trying and ask for help.

---

*If you are building something similar or want to dig into any of these patterns, find me at [github.com/jecis-repos](https://github.com/jecis-repos) or [@ananiel_ on Threads](https://threads.net/@ananiel_).*
