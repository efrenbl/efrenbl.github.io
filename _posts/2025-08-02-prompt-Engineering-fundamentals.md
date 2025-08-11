---
layout: post
title: "Prompt Engineering Fundamentals & In-Context Learning"
date: 2025-08-02 10:00:00 -0000
categories: [blog]
tags: [prompt-engineering, llms, in-context-learning, ai, nlp]
---

Ever typed something into ChatGPT and gotten a response that wasn’t quite what you wanted? Totally normal. Large language models aren’t mind-readers—they’re **pattern-followers**. Prompt engineering is how you give them the right patterns to follow: clear instructions, a role, examples, and a target format.

---

## TL;DR (What actually works)

- **Describe the task crisply** (role → goal → constraints → format).
- **Show an example or two** (few-shot) when formatting or tone matters.
- **Use structured outputs** (JSON schema) to remove ambiguity. [OpenAI Structured Outputs][1]
- **Add retrieval when facts matter** (RAG), and **verify** claims (CoVe). [RAPTOR][2]; [CoVe][12]
- **Adopt proven patterns**: ReAct, ToT, Plan-and-Solve, Self-Consistency. [ReAct][3]; [ToT][9]; [Plan-and-Solve][10]; [Self-Consistency][7]
- **Scale with modern techniques**: prompt compression (LLMLingua), automatic prompt optimization (OPRO/DSPy/APE), and multi-agent prompting (Mixture-of-Agents). [LLMLingua][4]; [OPRO][19]; [DSPy][5][20]; [APE][21][22]; [MoA][15]
- **Harden against prompt injection** (OWASP LLM Top 10). [OWASP GenAI][6]; [Cloudflare guide][27]

---

## 1) In-Context Learning: Teaching without training

LLMs adapt *within a single prompt* via examples and instructions—no fine-tuning required.

- **Zero-shot**: instructions only.  
- **One/few-shot**: give 1–5 examples to lock down structure/tone.  
- **Self-Consistency**: sample multiple answers and take a majority/consensus—often a big win for math/reasoning. [7]

> **Research snapshot:** Self-Consistency improves Chain-of-Thought by sampling diverse reasoning paths and aggregating the final answer, boosting GSM8K and other reasoning benchmarks. [7][8]

**Example (sentiment):**

- *Zero-shot:* “Classify the sentiment of this review: ‘The food was amazing!’”
- *One-shot:*  
  `Review: 'The service was terrible.' → Sentiment: Negative`  
  `Review: 'The food was amazing!' → Sentiment: ?`
- *Few-shot:* Add 2–3 more labeled examples to cement format and edge cases.

---

## 2) Prompt anatomy that holds up in production

**ROLE** (who the model is) → **GOAL** (what to achieve) → **CONSTRAINTS** (rules, scope, sources) → **FORMAT** (JSON/markdown schema) → **EXAMPLES** (good/bad) → **ACCEPTANCE TEST** (how you’ll judge it).

**Skeleton**

```text
You are a {role}.
Goal: {what outcome looks like}
Constraints: {grounding, style, limits, sources}
Output format: {JSON schema or markdown template}
Quality bar: {acceptance criteria}
Examples:
- Input ... → Output ...
- Input ... → Output ...
Task: {the new input}
````

### 2.1 System vs. user prompts

* **System prompt:** personality, durable rules (“never invent citations”), domain.
* **User prompt:** the actual request (“Plan a 3-day trip to Buenos Aires…”).

This split keeps “big-picture” guidance separate from one-offs.

---

## 3) Classic prompting patterns (still S-tier)

* **ReAct (Reason + Act):** interleave reasoning with tool calls (search, DB, code). Reduces hallucinations in QA/verification tasks. [3]
* **Tree-of-Thought (ToT):** explore multiple solution branches; backtrack and choose the best path for hard problems. [9]
* **Plan-and-Solve:** draft a plan, then execute step-by-step; great when decomposition helps. [10][11]
* **Chain-of-Verification (CoVe):** draft → generate verification questions → answer separately → finalize. [12][13]

> **Caution:** Recent analyses show limits to ReAct depending on environment fidelity and action design—measure before you standardize. [14]

---

## 4) What’s *new* (2024–2025) and worth adopting

### 4.1 Multi-agent & ensemble prompting

**Mixture-of-Agents (MoA):** multiple LLMs (or personas) draft and refine iteratively; strong gains on instruction following, summarization, and coding. [15]

### 4.2 Retrieval that actually helps

**RAPTOR** (hierarchical summaries) for long docs; **HyDE** (hypothetical docs) for zero-shot retrieval; **CRAG** (corrective logic) when confidence is low; **CoV-RAG** blends verification with retrieval. [2][16][17][18]

### 4.3 Make prompts cheaper & faster

**Prompt compression** (LLMLingua / LongLLMLingua / LLMLingua-2) cuts tokens with minimal quality loss—big latency/cost wins. [4][26]

### 4.4 Stop hand-tuning—*optimize* prompts automatically

**OPRO** (LLMs as optimizers), **DSPy** (declarative pipelines + compiled prompts/weights), **APE** (instruction search). [19][5][20][21][22]

### 4.5 Safety-by-design, not by vibes

Use **constitutional-style critiques** to elicit safe revisions, pair with **OWASP LLM Top 10** guidance for injection, insecure output handling, and excessive agency. [23][6]

---

## 5) SDK recipes (OpenAI Python)

> See the official docs for the latest signatures and model names. [24]

### 5.1 Structured outputs (always return valid JSON)

Enforce schema to eliminate format drift. [1]

```python
from openai import OpenAI
client = OpenAI()

schema = {
  "type": "object",
  "properties": {
    "title": {"type": "string"},
    "summary": {"type": "string"},
    "entities": {"type": "array", "items": {"type": "string"}}
  },
  "required": ["title", "summary", "entities"],
  "additionalProperties": False
}

resp = client.responses.create(
  model="gpt-4.1-mini",
  input="Extract a title, 1-paragraph summary, and key entities from the article below...",
  response_format={"type": "json_schema", "json_schema": {"name": "ArticleCard", "schema": schema}}
)

print(resp.output_text)  # JSON string conforming to the schema
```

### 5.2 Tool calling (a.k.a. function calling) [25]

```python
tools = [{
  "type": "function",
  "function": {
    "name": "get_order_status",
    "description": "Lookup order status by ID",
    "parameters": {
      "type": "object",
      "properties": {"order_id": {"type": "string"}},
      "required": ["order_id"]
    }
  }
}]

resp = client.responses.create(
  model="gpt-4.1-mini",
  input=[{"role": "user", "content": "Where is order 19A7?"}],
  tools=tools
)

for item in resp.output:
  if item.type == "tool_call":
    result = get_order_status(item.tool_call.function.arguments)
    followup = client.responses.create(
      model="gpt-4.1-mini",
      input=[{"role": "tool", "content": result, "name": "get_order_status"}],
    )
    print(followup.output_text)
```

### 5.3 Self-Consistency (majority vote) [7]

```python
from collections import Counter
answers = []
for _ in range(5):
  r = client.responses.create(
    model="gpt-4.1-mini",
    input="Solve and return only the integer: 27 * 31"
  )
  answers.append(r.output_text.strip())

final = Counter(answers).most_common(1)[0][0]
print(final)
```

### 5.4 Lightweight CoVe verifier (draft → verify → finalize) [12]

```python
draft = client.responses.create(
  model="gpt-4.1-mini",
  input="Draft a factual answer about {topic} with citations."
).output_text

plan = client.responses.create(
  model="gpt-4.1-mini",
  input=f"Given this draft, list 3 verification questions to fact-check it:\n{draft}"
).output_text

answers = client.responses.create(
  model="gpt-4.1-mini",
  input=f"Answer these verification questions independently:\n{plan}"
).output_text

final = client.responses.create(
  model="gpt-4.1-mini",
  input=f"Using the draft and verified answers, produce a corrected final answer with citations.\nDraft:\n{draft}\nVerified:\n{answers}"
).output_text
```

### 5.5 Cost control with prompt compression (LLMLingua family) [4][26]

Compress long prompts *before* calling your model, especially in RAG/chat-history scenarios.

---

## 6) RAG prompting that doesn’t hallucinate

**Do:**

* Be explicit: “Answer *only* from the provided context. If missing, say `INSUFFICIENT_CONTEXT`.”
* Use **document IDs + line spans** in the prompt and require **quotes** for claims.
* Add **verification** (CoVe) and **corrective** logic (CRAG) for low-quality retrieval. [12][17]
* Explore **RAPTOR** (long docs) and **HyDE** (zero-shot retrieval). [2][16]

**Don’t:**

* Stuff the entire wiki; compress and curate (LLMLingua / LongLLMLingua). [4][26]

---

## 7) Security & reliability (non-negotiable)

* **Prompt Injection (LLM01:2025):** never blindly obey user-supplied text; gate actions and sanitize outputs.
* **Insecure output handling:** treat model output as untrusted input.
* **Excessive agency:** impose rate limits, scopes, and human-in-the-loop for risky actions.

Start with **OWASP Top 10 for LLM Apps** and thread it through prompts and tool policy. [6][27]

---

## 8) Evaluation & iteration

* **Unit tests for prompts:** small fixtures of input → expected behaviors.
* **Regression tests:** ensure changes don’t break previous wins.
* **Frameworks:** OpenAI Evals; grader/judge patterns with rubrics. [28][29]
* **Production signals:** disagreement rate, abstentions, citation coverage, escalation rate, latency, cost.

---

## 9) Copy-paste prompt templates

### 9.1 JSON-first extraction

```text
System: You are a precise information extractor. If data is missing, use null.
User: Extract to the JSON schema below from the text.
Schema:
{ "type":"object","properties":{"company":{"type":"string"},"revenue_usd":{"type":"number"},"source":{"type":"string"}},"required":["company","source"] }
Rules:
- Use numbers only for revenue_usd (no currency symbols).
- If multiple values, prefer the most recent.
Text:
{{DOCUMENT}}
Return only JSON.
```

### 9.2 RAG-answer with verification

```text
System: You answer strictly from context; otherwise reply "INSUFFICIENT_CONTEXT".
User:
Question: {{Q}}
Context (cite with [doc#line-start–line-end]):
{{CHUNKS}}
Steps:
1) Draft a 3-sentence answer with inline citations.
2) List 2 verification questions.
3) Answer them independently.
4) Revise the final answer if needed.
Output: final answer + citation list.
```

### 9.3 Plan-and-Solve for tasks with dependencies

```text
System: Senior engineer.
User: Build a stepwise plan before answering.
Plan:
- Break the task into atomic steps.
- Identify tools/data needed per step.
- Execute steps, marking any assumption.
- Provide final answer + checklist.
Task: {{TASK}}
```

### 9.4 Multi-agent MoA (lightweight, single-model personas)

```text
System: You are a committee of 3 reviewers: (A) Skeptic, (B) Stylist, (C) Fact-checker.
User: Given the draft below, each reviewer writes a short critique. Then produce a final revised output that addresses all critiques explicitly.
Draft:
{{DRAFT}}
Output:
- Critiques A/B/C
- Final revision
```

---

## 10) Common anti-patterns (and fixes)

* **Underspecified asks** → Add constraints, examples, and an acceptance test.
* **“Write JSON” drift** → Use structured outputs and validation. [1]
* **Hallucinated facts** → RAG + CoVe + citations + allow `INSUFFICIENT_CONTEXT`. [12]
* **Cost spikes** → Compress prompts; trim history; retrieve selectively. [4]
* **Prompt injection** → Isolate untrusted content; never let it override system/tool rules. [6]

---

## Further reading (hand-picked)

* **OpenAI Docs**: Prompting, Structured Outputs, Tool Calling / Responses API. [30][1][25]
* **Reasoning & Acting**: ReAct; ToT; Plan-and-Solve; Chain-of-Verification. [3][9][10][12]
* **RAG Upgrades**: RAPTOR; HyDE; CRAG; CoV-RAG. [2][16][17][18]
* **Optimization**: OPRO; DSPy; APE. [19][5][20][21][22]
* **Compression**: LLMLingua / LongLLMLingua / LLMLingua-2. [4][26]
* **Security**: OWASP Top 10 for LLM Apps. [6][27]

---

## Final takeaway

You don’t need magic prompts. You need **clear specs**, **a couple of examples**, **structured outputs**, and **tight feedback loops** (retrieval + verification + tests). Add modern accelerants—**automatic prompt optimization**, **compression**, and **multi-agent critiques**—when they move a metric you care about.

**Ship prompts like code:** specify, test, measure, iterate.

[1]: https://platform.openai.com/docs/guides/structured-outputs "Structured model outputs - OpenAI API"
[2]: https://arxiv.org/abs/2401.18059 "RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval"
[3]: https://arxiv.org/abs/2210.03629 "ReAct: Synergizing Reasoning and Acting in Language Models"
[4]: https://arxiv.org/abs/2310.05736 "LLMLingua: Compressing Prompts for Accelerated Inference of LLMs"
[5]: https://dspy.ai/ "DSPy"
[6]: https://genai.owasp.org/llm-top-10/ "OWASP GenAI: LLM Top 10"
[7]: https://arxiv.org/abs/2203.11171 "Self-Consistency Improves Chain-of-Thought Reasoning in Language Models"
[8]: https://openreview.net/forum?id=1PL1NIMMrw "Self-Consistency Improves Chain-of-Thought Reasoning (OpenReview)"
[9]: https://arxiv.org/abs/2305.10601 "Tree of Thoughts: Deliberate Problem Solving with LLMs"
[10]: https://arxiv.org/abs/2305.04091 "Plan-and-Solve Prompting"
[11]: https://aclanthology.org/2023.acl-long.147/ "Plan-and-Solve Prompting (ACL 2023)"
[12]: https://arxiv.org/abs/2309.11495 "Chain-of-Verification Reduces Hallucination in LLMs"
[13]: https://openreview.net/forum?id=VP20ZB6DHL "Chain-of-Verification (OpenReview)"
[14]: https://arxiv.org/abs/2405.13966 "On the Brittle Foundations of ReAct Prompting for Agentic LMs"
[15]: https://arxiv.org/abs/2406.04692 "Mixture-of-Agents Enhances LLM Capabilities"
[16]: https://arxiv.org/abs/2212.10496 "HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels"
[17]: https://arxiv.org/abs/2401.15884 "CRAG: Corrective Retrieval Augmented Generation"
[18]: https://arxiv.org/abs/2410.05801 "The Chain-of-Verification Can Improve Retrieval-Augmented Generation"
[19]: https://arxiv.org/abs/2309.03409 "Large Language Models as Optimizers (OPRO)"
[20]: https://hai.stanford.edu/research/dspy-compiling-declarative-language-model-calls-into-state-of-the-art-pipelines "DSPy @ Stanford HAI"
[21]: https://arxiv.org/abs/2211.01910 "Large Language Models Are Human-Level Prompt Engineers (APE)"
[22]: https://sites.google.com/view/automatic-prompt-engineer "Automatic Prompt Engineer (project page)"
[23]: https://arxiv.org/abs/2212.08073 "Constitutional AI: Harmlessness from AI Feedback"
[24]: https://platform.openai.com/docs/api-reference/introduction "OpenAI API Reference"
[25]: https://platform.openai.com/docs/guides/function-calling "Function (Tool) Calling - OpenAI"
[26]: https://arxiv.org/abs/2310.06839 "LongLLMLingua: Accelerating and Enhancing LLMs in Long Contexts"
[27]: https://www.cloudflare.com/learning/ai/owasp-top-10-risks-for-llms/ "Cloudflare: OWASP Top 10 risks for LLMs"
[28]: https://github.com/openai/evals "openai/evals"
[29]: https://platform.openai.com/docs/guides/graders "Grader/Judge Patterns - OpenAI"
[30]: https://platform.openai.com/docs/guides/prompt-engineering "Prompt engineering - OpenAI API"


