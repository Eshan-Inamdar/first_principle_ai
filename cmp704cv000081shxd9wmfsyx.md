---
title: "Agentic AI Deep Dive — How Agents Think"
seoTitle: "How AI Agents Think — ReAct and Planning Loops"
seoDescription: "Learn how agents actually reason — ReAct loops, chain-of-thought,
Plan-and-Execute, and dynamic replanning when
things go wrong."
datePublished: 2026-05-15T14:17:15.043Z
cuid: cmp704cv000081shxd9wmfsyx
slug: agentic-ai-deep-dive-how-agents-think
cover: https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/c22755c3-c7fe-43e9-8794-edb1ec53db55.jpg
tags: ai, artificial-intelligence, data-science, react, machine-learning, ai-agents, reasoning

---

## Where We Left Off

In [Part 2](https://eshaninamdar.hashnode.dev/agentic-ai-deep-dive-anatomy-of-an-agent) we opened the hood. Every agent — regardless of framework — has five core components: LLM, planner, executor, tools, and memory. The harness manages the loop between them. CLAUDE.md configures the agent before the loop begins. We know what's inside the system.

What we haven't answered is how the system *thinks*.

Knowing the components is like knowing the parts of an engine. It doesn't tell you how combustion works. This part is about combustion.

* * *

## The Hook

A chess player doesn't plan twenty moves ahead and execute blindly.

They plan a few moves, make one, watch what the opponent does, re-evaluate the position, and plan again. The plan isn't fixed. It evolves with every new piece of information. The player who wins isn't the one with the most rigid strategy — it's the one who adapts fastest to what's actually happening on the board.

This is exactly how a well-designed agent reasons. Not a static script. Not a waterfall of pre-planned steps. A continuous loop of plan, act, observe, adapt — where each observation changes what happens next.

Understanding this loop is what separates engineers who build agents that work from those who build agents that look like they work until something unexpected happens.

* * *

## Why Static Planning Fails

The naive approach to building an agent is to generate a full plan upfront and execute it step by step.

Give the LLM a goal. Ask it to produce a complete plan. Run each step in sequence. Done.

This works in demos. It breaks in production — and here's why.

The real world doesn't cooperate with upfront plans. APIs time out. Search results don't contain what you expected. A document you needed turns out to be behind a paywall. A database query returns an empty result. The fourth step assumes something the third step was supposed to produce — and didn't.

A static plan has no mechanism to handle any of this. It either executes blindly through failures and produces garbage, or it stops dead and throws an error.

What you need is a reasoning loop — something that generates the *next* action based on what just happened, not what was planned three steps ago.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/e9cdc4b6-2fe0-4dd3-a62a-ea70c56160ea.png align="center")

* * *

## ReAct — The Dominant Reasoning Pattern

In 2022, researchers introduced a pattern called **ReAct** — short for *Reason + Act*. It's now the foundation of how most production agents think.

The idea is simple but powerful. Before every action, the agent writes out its reasoning — explicitly, in the context window. Then it acts. Then it observes the result. Then it reasons again. The cycle continues until the goal is complete.

Each iteration looks like this:

```plaintext
Thought: I need to find the current price of Product X.
         I'll search for their pricing page first.

Action: search("Product X pricing 2025")

Observation: Found pricing page. Basic plan is $29/month,
             Pro is $79/month, Enterprise is custom.

Thought: I have the pricing. Now I need to find
         their main competitor and do the same.

Action: search("Product X competitor pricing")

...
```

This isn't just logging. The **thought** step is doing real work — the model is reasoning about what it knows, what it needs, and what the best next action is. Writing it out explicitly forces the model to reason before acting, which dramatically reduces the rate of incorrect tool calls.

**Why ReAct works:** The thought step gives the model a scratchpad. It externalises reasoning that would otherwise happen implicitly. When reasoning is explicit, the model catches its own mistakes before they become bad actions. And when an agent fails, the thought trace gives you a precise record of where reasoning broke down — invaluable for debugging.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/359c1792-af60-4e24-ba18-25a7c706a521.png align="center")

* * *

## Chain-of-Thought — The Scratchpad Inside the Loop

ReAct works because it forces the model to think before it acts. The mechanism behind this is **chain-of-thought** reasoning.

**Chain-of-thought (CoT)** is a prompting technique where the model is encouraged — or required — to work through a problem step by step before producing a final answer. Instead of jumping straight to a conclusion, the model writes out its intermediate reasoning.

In an agentic context, chain-of-thought is always on. Every thought step in the ReAct loop is chain-of-thought reasoning applied to the question: *"given what I know right now, what should I do next?"*

The **scratchpad** is the space in the context window where this reasoning happens. It's not shown to the end user. It's not the final output. It's the working memory the model uses to reason through complexity before committing to an action.

Think of it as the difference between a student who reads a math problem and immediately writes an answer versus one who works through it on rough paper first. The rough paper student makes fewer errors — not because they're smarter, but because the process forces them to catch mistakes before they're committed.

* * *

## Plan-and-Execute — When ReAct Isn't Enough

ReAct is excellent for tasks where the path forward isn't fully known in advance — search, research, exploration. It handles uncertainty well because it re-evaluates at every step.

But ReAct has a weakness. For long, complex tasks — tasks with twenty or thirty steps, tasks that span multiple sessions, tasks where some steps must happen in a precise order — re-evaluating from scratch at every step is expensive and fragile. The model can lose track of the bigger picture when it's focused on the immediate next action.

**Plan-and-Execute** is a different pattern designed for exactly this scenario.

It works in two phases:

**Phase 1 — Plan:** A dedicated planning step generates the full task breakdown upfront. This is done by a planner component (often a separate LLM call) that has the full goal in view and produces a structured plan.

**Phase 2 — Execute:** Each step of the plan is executed in sequence. But unlike static planning, the executor can feed observations back to the planner — allowing the plan to be updated if something unexpected happens.

The key difference from static planning: the plan is a *living document*, not a fixed script.

|  | ReAct | Plan-and-Execute |
| --- | --- | --- |
| **Best for** | Exploration, research, uncertain paths | Long structured tasks, known workflows |
| **Planning** | Inline, one step at a time | Upfront, full task breakdown |
| **Adaptability** | High — re-evaluates every step | Medium — plan updates on major failures |
| **Context usage** | Lower per step | Higher upfront |
| **Failure recovery** | Naturally handles step failures | Requires explicit replanning logic |

Neither pattern is universally better. The choice depends on the task structure. Many production agents use a hybrid — Plan-and-Execute for the overall task, with ReAct-style reasoning within each step.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/68468a6a-d6d2-45b2-b826-c9b021fa2615.png align="center")

* * *

## Dynamic Replanning — When the Plan Breaks Mid-Task

Even with Plan-and-Execute, things break. A step fails. A dependency turns out not to exist. New information changes what needs to happen.

**Dynamic replanning** is the mechanism that handles this. When a step fails or produces an unexpected result, instead of stopping or blindly continuing, the agent triggers a replanning step — feeding the current state, the original goal, and the failure into the planner and asking it to produce a revised plan.

This is what separates production-grade agents from brittle demos.

The flow looks like this:

```plaintext
Original plan: Step 1 → Step 2 → Step 3 → Step 4

Step 2 fails — API is unavailable

Replan: Given original goal and current state
        (Step 1 complete, Step 2 failed),
        what's the best path forward?

Revised plan: Step 1 ✓ → Step 2b (alternative) → Step 3 → Step 4
```

Dynamic replanning adds complexity. It requires the planner to receive structured state — what's been completed, what failed, what was produced so far. It requires clear failure signals from the executor. And it requires careful design to avoid infinite replan loops.

But for any agent running tasks longer than a few steps in production, it's not optional.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/6cb0435f-c082-4262-930e-240a5f3e7941.png align="center")

* * *

## How Agents Know When They're Done

This sounds obvious. It isn't.

A static function terminates when it reaches a return statement. An agent has no equivalent. It terminates when the LLM decides the goal has been met — which means the termination condition is itself a reasoning decision.

There are three common patterns:

**Goal-state checking:** The LLM explicitly evaluates whether the current state satisfies the original goal. *"Was I asked to find three competitors and their pricing? I have three competitors and their pricing. Goal complete."*

**Tool-based termination:** The agent calls a special `finish` tool with the final response as its argument. The harness intercepts this call and exits the loop. This is explicit and easy to detect.

**Max-step cutoff:** A hard limit on the number of loop iterations. The harness forces termination after N steps regardless of the LLM's assessment. Crude but necessary as a safety net.

Production agents use all three — goal-state checking as the primary mechanism, tool-based termination as the clean exit path, and max-step cutoff as the backstop.

Without a reliable termination mechanism, agents loop. And loops are the most common failure mode in production — which is why we dedicate Part 8 entirely to failure modes.

* * *

## What This Looks Like in a Real Framework

These aren't abstract patterns. Here's how they map to real tools:

**LangChain** implements ReAct natively through its `AgentExecutor`. You define tools, pass them to the agent, and the ReAct loop runs automatically. The thought/action/observation trace is logged and accessible.

**LangGraph** gives you explicit control over the reasoning loop as a state graph. Each node in the graph is a step — you define the transitions, the conditions, and the replanning logic yourself. More verbose, but more controllable for complex tasks.

**LangSmith** — LangChain's observability layer — lets you inspect the full thought trace for every agent run. When an agent produces a wrong answer, you read the trace and see exactly where reasoning diverged. This is the practical benefit of the ReAct pattern's explicit thought steps.

The pattern you choose shapes which framework serves you best. ReAct-heavy workflows fit naturally into LangChain. Tasks needing precise loop control belong in LangGraph.

* * *

## Key Takeaways

*   Static planning fails in production because the real world doesn't cooperate with upfront plans — you need a reasoning loop, not a fixed script
    
*   **ReAct** (Reason + Act) is the dominant pattern — explicit thought before every action, dramatically reduces incorrect tool calls and makes failures debuggable
    
*   **Chain-of-thought** is the mechanism behind ReAct — externalizing reasoning into a scratchpad before committing to an action
    
*   **Plan-and-Execute** is better for long structured tasks — upfront planning with the ability to replan when steps fail
    
*   **Dynamic replanning** is what separates production agents from demos — the ability to revise the plan mid-task when something breaks
    
*   Termination is a reasoning problem, not a technical one — production agents need goal-state checking, tool-based exits, and a max-step backstop
    
*   The thought trace in ReAct is your primary debugging tool — it tells you exactly where reasoning broke down
    

* * *

## What's Next

We now understand how agents think — the reasoning patterns, the planning approaches, the mechanisms for handling failure and knowing when to stop.

But all of this reasoning is in service of one thing: taking action in the world. And actions require tools.

A thought without a tool is just text. The moment an agent calls a tool, it's reaching outside itself — into APIs, databases, browsers, file systems. That interface between the agent and the world is where most of the interesting engineering happens.

And it's where MCP — Model Context Protocol — is quietly changing everything.

**Part 4: Tools — How Agents Act on the World →**

* * *

*Agentic AI Deep Dive — Part 3 of 9*