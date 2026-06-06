---
title: "Agentic AI Deep Dive — Evaluation & Production"
seoTitle: "How to Evaluate AI Agents — Output and Trajectory"
seoDescription: " Learn how to
evaluate agents properly — output quality, trajectory
evaluation, LLM-as-judge, production monitoring,
and the improvement loop."
datePublished: 2026-06-06T17:45:54.376Z
cuid: cmq2n9fic000b1spq477i58r8
slug: agentic-ai-deep-dive-evaluation-production
cover: https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/d75a95d8-244f-4935-a0ab-32552cd4ad3b.jpg
tags: ai, ai-engineering, agentic-ai

---

* * *

## Where We Left Off

Part 8 was about failure modes — the six ways agents break in production without anyone trying to break them. Infinite loops. Hallucinated tool calls. Silent failures. Context poisoning. Cascading multi-agent failures. Confident wrong answers. We built safety nets: cutoffs, explicit error states, retry logic, fallback tools, human escalation paths. We built observability in from day one.

The system is running. The safety nets are in place.

Now the hardest question of the entire series: how do you actually know if it's working?

* * *

## The Hook

A consulting firm doesn't just deliver work and disappear.

The engagement partner presents the final report. The client reviews it. Were the recommendations accurate? Did the market analysis hold up against reality? Six months later — did the strategy actually work?

Quality isn't assumed. It's measured. Client satisfaction scores. Accuracy of predictions against outcomes. Repeat engagement rates. The firm that never measures its own quality is the firm that slowly drifts without knowing it.

Agents are the same. Running is not the same as working. Producing output is not the same as producing correct, reliable, useful output. The gap between those two things is what evaluation closes.

And it's the gap most teams discover — quietly, painfully — weeks after deploying something they assumed was working fine.

* * *

## Why Agent Evaluation Is Hard

Evaluating a traditional ML classification model is relatively straightforward. You have a test set. You run predictions. You compare to ground truth labels. You get a number — accuracy, F1, AUC. The number tells you something real.

Agent evaluation is harder for three distinct reasons.

**Reason 1 — Output is often text, not a number.** An agent that answers questions about company policy doesn't produce a label or a probability. It produces a paragraph. Whether that paragraph is correct, complete, and appropriately nuanced requires judgement — not just comparison to a reference answer.

**Reason 2 — The path matters as much as the answer.** An agent that produces the correct final answer via three unnecessary tool calls, a reasoning detour, and a lucky recovery is not a well-functioning agent. It got lucky. Next time, the same flawed path will produce a wrong answer. Evaluating only the output misses this entirely.

**Reason 3 — Evaluation often requires another LLM.** Because the output is text and judgement is required, many evaluation approaches use a second LLM to assess the first one's output. This works surprisingly well — but introduces its own failure modes. The evaluator LLM can be wrong. It can be biased toward outputs that sound confident. It can miss domain-specific errors.

These three properties mean agent evaluation requires a combination of approaches — no single method is sufficient.

* * *

## What You're Actually Evaluating

Agent evaluation has two dimensions. Both matter. Neither alone is sufficient.

**Output quality** — was the final answer correct? Did it address the user's goal? Was it accurate, complete, and appropriately nuanced? This is what most teams measure first because it's the most visible.

**Trajectory quality** — did the agent get there the right way? Did it use the right tools in the right order? Did it reason correctly at each step? Was the path efficient? Did it handle unexpected observations appropriately?

A correct answer via a flawed trajectory is still a problem. The flawed trajectory will produce a wrong answer on a slightly different input. You want to catch the flawed trajectory before that happens — not after.

Think of it like grading a maths exam. The answer matters. But the working matters too. A student who gets the right answer via incorrect working has a fragile understanding that will fail on the next problem.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/5d58e09a-b6e0-4ccc-afb0-5c9d81096052.png align="center")

* * *

## Output Evaluation

Three approaches to evaluating agent outputs, each suited to different situations.

**Exact match** — compare the agent's output to a reference answer character by character. Simple, fast, objective. Appropriate when the output is structured and deterministic — a JSON object, a specific code snippet, a precise data extraction. Completely useless for natural language outputs where multiple phrasings of the same correct answer are all valid.

**Semantic similarity** — embed both the agent's output and the reference answer as vectors and measure their cosine similarity. Captures paraphrases and synonyms that exact match misses. Appropriate for factual question answering where the meaning matters more than the phrasing. Fails on nuance — two answers can be semantically similar but one can be subtly wrong in ways that matter.

**LLM-as-judge** — use a second LLM to evaluate the first one's output against a rubric. Provide the judge with the original question, the reference answer (if available), the agent's output, and an evaluation rubric. The judge scores the output and optionally explains its reasoning.

LLM-as-judge is the most powerful approach for complex outputs. It handles nuance, catches subtle errors, and can evaluate dimensions like tone, completeness, and appropriateness that rule-based approaches miss entirely.

But it has failure modes:

*   The judge LLM is biased toward outputs that *sound* confident and well-structured — a fluent wrong answer often scores higher than a correct but hesitant one
    
*   The judge LLM can have the same domain blind spots as the agent being evaluated
    
*   Without a reference answer, the judge has no ground truth — it's assessing plausibility, not correctness
    
*   Using the same model family for both the agent and the judge introduces correlated errors
    

Use LLM-as-judge with these limitations in mind. Always combine it with human review samples to calibrate whether the judge's scores match human judgement on your specific domain.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/0da84c3d-b8e1-44fc-8c46-7a0d7df6f315.png align="center")

* * *

## Trajectory Evaluation

Trajectory evaluation assesses the reasoning path — not just where the agent ended up, but how it got there.

**Step correctness** — at each step in the reasoning loop, was the agent's decision reasonable given the information available? Did the thought step correctly interpret the last observation? Did the tool call match what the thought step decided to do?

**Tool call appropriateness** — did the agent use the right tools? Did it avoid unnecessary tool calls? Did it call tools in a logical order? Unnecessary tool calls indicate reasoning inefficiency. Wrong tool calls indicate a tool design or description problem.

**Efficiency** — how many steps did the agent take to complete the task? A task that should take five steps but takes twelve has a reasoning problem, even if the final answer is correct. Track step counts over time — increases often signal prompt drift or model degradation.

**Recovery behaviour** — when the agent encountered an unexpected observation or a tool failure, did it handle it correctly? Did it replan appropriately? Or did it continue down a path that should have been abandoned?

**How to evaluate trajectories in practice:** log the full thought trace for every agent run. For your eval set, manually review the traces on a sample of runs — especially runs where the output was incorrect. The trace tells you whether the failure was a reasoning failure, a tool failure, or a termination failure. That distinction determines the fix.

* * *

## Building Your Eval Set

An eval set is your most important quality assurance asset. Build it deliberately, not accidentally.

**What makes a good eval case:**

*   A specific input that exercises a real capability the agent is supposed to have
    
*   A known correct output or rubric for evaluating the output
    
*   Coverage of the happy path, edge cases, and known failure-prone inputs
    

**Coverage your eval set needs:**

*Happy path cases* — straightforward inputs where the agent should work correctly. These establish your baseline and catch broad regressions.

*Edge cases* — inputs that are technically within scope but unusual. Ambiguous queries. Inputs with missing information. Tasks that require the agent to express uncertainty or ask a clarifying question.

*Adversarial cases* — inputs designed to trigger failure modes. Inputs that might cause hallucinated tool calls. Inputs where the correct answer requires resisting a plausible-sounding but wrong path.

*Regression cases* — every production failure that gets discovered and fixed should become an eval case. The failure should never recur without being caught.

**How many cases do you need?** Enough to detect meaningful regressions — typically 50-200 cases for a focused agent, more for broad-purpose agents. Quality matters more than quantity. Twenty well-designed eval cases that cover real failure modes are worth more than two hundred cases that all test the same happy path.

* * *

## Production Monitoring

Eval sets catch regressions in testing. Production monitoring catches failures in the real world — including failure modes your eval set didn't anticipate.

**What to monitor in production:**

**Latency** — how long does each task take end-to-end? Sudden increases often signal a reasoning loop running longer than expected, a slow tool, or an external dependency degrading.

**Token consumption per run** — track average and p95 token consumption. Increases over time can signal prompt drift, context management problems, or the agent taking more steps than necessary.

**Tool call success rates** — what percentage of calls to each tool succeed? A drop in success rate for a specific tool signals a problem with that tool or its upstream dependency.

**Step count distribution** — how many steps does the agent take to complete tasks? Track this over time. Increases indicate reasoning degradation.

**Human escalation rate** — what percentage of tasks escalate to human review? A sudden increase signals the agent is encountering situations outside its design envelope.

**User feedback signals** — thumbs up/down, explicit corrections, repeat queries on the same topic. User behaviour is a noisy but real signal of output quality.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/5380fcd8-e067-4842-9cc7-5728ad1550d2.png align="center")

* * *

## The Improvement Loop

Evaluation is not a one-time exercise. It's a continuous cycle.

```plaintext
Build eval set
      ↓
Run agent against eval set
      ↓
Identify failures — categorise by type
(reasoning? tool? memory? termination?)
      ↓
Fix the root cause
      ↓
Re-run eval set — confirm fix, check for regressions
      ↓
Deploy
      ↓
Monitor production
      ↓
New failures discovered → add to eval set
      ↓
Repeat
```

The teams that build reliable agents are the ones that treat this loop as a core engineering process — not an afterthought. Every production failure is information. Every regression catch before deployment is a win. Every eval case added is institutional memory that prevents the same mistake from happening twice.

The agent gets better. The eval set grows. The monitoring gets tighter. Over time, the system is not just running — it's reliably working.

![]( align="center")

* * *

## Real Tools

**LangSmith** — LangChain's observability and evaluation platform. Captures traces, tool calls, thought steps, and token consumption. Supports eval set management and LLM-as-judge scoring. Best for LangChain-based agents.

**Braintrust** — standalone eval platform that works across frameworks. Supports multiple scoring methods including LLM-as-judge. Good for teams not locked into LangChain.

**AgentEvals** — focused specifically on trajectory evaluation. Assesses step correctness and tool call appropriateness, not just final output. Fills the gap that output-only evaluation leaves.

**Human review workflows** — no tooling replaces periodic human review. Sample agent outputs regularly — especially on high-stakes tasks. Human review calibrates whether your automated eval scores actually reflect real-world quality.

* * *

## Key Takeaways

*   Running is not the same as working — evaluation is how you close the gap between the two
    
*   Agent evaluation is harder than traditional ML evaluation: outputs are text not numbers, the path matters as much as the answer, and evaluation often requires another LLM to judge
    
*   Evaluate two dimensions: output quality (was the answer correct?) and trajectory quality (did it get there the right way?)
    
*   LLM-as-judge is powerful but has failure modes — always calibrate against human review samples
    
*   Build your eval set deliberately: happy path, edge cases, adversarial inputs, and regression cases for every production failure fixed
    
*   Production monitoring catches what eval sets miss — track latency, token consumption, tool call success rates, step counts, and escalation rates
    
*   The improvement loop is the real product: eval → identify → fix → re-eval → deploy → monitor → repeat
    
*   Every production failure is information. Every eval case added is institutional memory.
    

* * *

## The Series Retrospective

Nine parts ago, we started with a question: why do agents have to exist?

We began with a person who wakes up every morning with no memory, no tools, no way to act on the world. Brilliant but broken for real work. That was the LLM. And from that starting point we derived everything else — not by memorising definitions, but by following the thread of what was missing and asking what the minimum solution would be.

We added RAG and gave the LLM access to knowledge it wasn't trained on. We added the agentic loop and gave it the ability to act across multiple steps. We opened the hood and found five components — LLM, planner, executor, tools, memory — held together by a harness. We explored how agents think through ReAct and Plan-and-Execute. We designed the tools that give agents hands. We built the four memory layers that turn a capable agent into a reliable one.

Then we scaled up — multi-agent systems that work like teams, with orchestrators and specialists and parallel execution. We thought about security — prompt injection, trust boundaries, least privilege, the attack surface that comes with capability. We catalogued what breaks — the six failure modes, the safety nets, the observability layer that lets you see what's happening. And now we've closed the loop with evaluation — the continuous process that separates agents that work from agents that just run.

If you've followed all nine parts, you now have something more than knowledge of the vocabulary. You have a mental model. You understand why every part of an agentic system had to exist. You can look at any agent architecture and reason about its components, its failure modes, its trust model, its evaluation strategy.

That was the goal from Part 1. Build the mental model from the ground up.

You've done it.

* * *

*Agentic AI: A Deep Dive from First Principles* *Series complete — Parts 1–9*

* * *

*Thank you for reading. If this series helped you understand agentic AI more deeply, share it with someone who's trying to figure out the same things.*