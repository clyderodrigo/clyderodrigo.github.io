# Agent Economy: Tokenization, Inference Parameters, and the Cost of Deployed Agents

## Key Takeaways

- Tokenization sets the unit of agent economy. JSON, code, and structured data tokenize at 1.5–3x prose density — every tool schema and structured result is paid at a premium.
- Context accumulation is the structural cost driver. Sliding window + summarization cuts costs 60–80% on long-running agents.
- `max_tokens` and `temperature` are paired controls. Low temperature without a token cap is still vulnerable to over-generation on open-ended tasks.
- `top_p` and `top_k` control vocabulary distribution. Don't adjust both simultaneously with temperature. For extraction tasks, tight `top_k` + low temperature produces short, predictable outputs.
- `presence_penalty` and `frequency_penalty` reduce verbosity. Keep at 0 for structured output tasks — they can corrupt format compliance.
- Stop sequences are free cost reduction. Combine with `max_tokens` for a hard cap plus an early exit.
- Model routing is the exchange rate multiplier. Route 70–80% of agent steps to cheap models. The quality difference is minimal for structured subtasks; the cost difference is 10–18x.
- Instrument at the step level. Session totals hide the expensive steps.

---

Every agent deployment has an economy. Tokens are the currency. Inference parameters are the controls that determine how much of that currency gets spent on each call. Understanding both — together — is what separates teams that run agents profitably at scale from teams that get surprised by the bill.

This article covers tokenization as the unit of agent cost, the full set of inference parameters and their economic effects, how costs compound across agent turns, and the model-level decisions that multiply or divide everything else.

---

## Tokenization: Unit of Agent Economy

A token is not a word. It's a subword unit determined by the model's vocabulary. Understanding what tokenizes expensively is the first economic skill.

```python
import tiktoken  # for OpenAI models

enc = tiktoken.encoding_for_model("gpt-4o")

def count_tokens(text: str) -> int:
    return len(enc.encode(text))

# Density comparison — same semantic content, different token cost
count_tokens("revenue declined in Q3")                         # 5 tokens
count_tokens('{"metric": "revenue", "period": "Q3", "trend": "decline"}')  # 18 tokens
count_tokens("```python\ndef calculate_margin(r, c):\n    return (r-c)/r\n```")  # 22 tokens
count_tokens("SELECT (revenue-cogs)/revenue AS margin FROM sales WHERE quarter=3")  # 17 tokens
```

**The expensive formats:** JSON, code, SQL, and XML tokenize at 1.5–3x the density of plain prose. Every tool call result returned to the agent, every schema definition passed in the system prompt, every structured output the model emits — these are all paid at a premium.

For Anthropic models, use their token counter directly:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-sonnet-4-5",
    system="You are an enterprise data analyst.",
    messages=[{"role": "user", "content": "What was Q3 net margin by region?"}]
)
print(response.input_tokens)  # exact pre-call token count
```

### Tool Schemas Are Paid on Every Call

Tool definitions are injected into the context on every single inference call. A verbose schema that could be written in 40 tokens written at 180 tokens costs 4.5x more — and that overhead compounds across every agent step that uses that tool.

```python
# 180 tokens — paying for prose no model needs
{
    "name": "search_knowledge_base",
    "description": "This function searches through the entire internal knowledge base using semantic similarity to find the most relevant documents that match the user's query. It returns the top K results ranked by relevance score.",
    "parameters": {
        "query": {
            "type": "string",
            "description": "The natural language search query provided by the user that will be used to find semantically similar documents in the knowledge base"
        }
    }
}

# 40 tokens — same behavior
{
    "name": "search_kb",
    "description": "Semantic search over internal documents. Returns top-K results by relevance.",
    "parameters": {
        "query": {"type": "string", "description": "Search query"}
    }
}
```

Audit every tool definition against its token count. Descriptions should tell the model what to use the tool for — nothing else.

---

## How Costs Compound Across Agent Turns

Single calls are affordable. Agentic loops are where costs accumulate into a structural problem.

```
Turn 1:  system_prompt (2,000t) + user_input (200t) + response (500t)    =  2,700 tokens billed
Turn 2:  system_prompt (2,000t) + history (2,700t) + input (150t) + tool_result (800t) + response (600t) = 6,250 tokens billed
Turn 3:  system_prompt (2,000t) + history (6,250t) + input (100t) + response (800t) = 9,150 tokens billed
```

By turn 5, you're billing 15,000+ tokens for a task that started as a 200-token request. The system prompt and history are re-billed in full every single call.

### Cap History With a Sliding Window

```python
from collections import deque

MAX_HISTORY_TURNS = 6  # tune per use case

class ManagedContext:
    def __init__(self, system_prompt: str):
        self.system = system_prompt
        self.history = deque(maxlen=MAX_HISTORY_TURNS * 2)  # user+assistant pairs

    def add_turn(self, user_msg: str, assistant_msg: str):
        self.history.append({"role": "user", "content": user_msg})
        self.history.append({"role": "assistant", "content": assistant_msg})

    def build_messages(self, current_input: str) -> list:
        return (
            [{"role": "system", "content": self.system}]
            + list(self.history)
            + [{"role": "user", "content": current_input}]
        )
```

### Compress Old History With Summarization

When the window overflows, summarize what gets evicted — use a cheap model for it:

```python
def summarize_history(messages: list, model: str = "gpt-4o-mini") -> str:
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": "Summarize this conversation in 3–5 bullet points. Be factual and concise."},
            {"role": "user", "content": str(messages)}
        ],
        max_tokens=300
    )
    return response.choices[0].message.content
```

Capping at 6 turns with summarization cuts context-window costs by 60–80% on long-running agents without meaningful quality loss for most task types.

---

## Inference Parameters as Economic Controls

Every inference parameter has an economic effect, not just a quality effect. The token budget is the output of your parameter choices — set them deliberately.

### `max_tokens` — The Hard Budget

The most direct control. If the model doesn't need 2,000 tokens to extract a date from a sentence, don't provide room for 2,000 tokens. A runaway verbose response on an extraction step doesn't just cost more — it inflates the history for every subsequent turn.

Set budgets by task type, not globally:

| Task | Recommended `max_tokens` |
|---|---|
| Classification / routing | 10–50 |
| Entity / data extraction | 100–300 |
| Summarization | 200–500 |
| Structured JSON output | 150–600 |
| Code generation | 500–2000 |
| Analysis / reasoning | 300–1000 |

```python
# Task-aware budget enforcement
TASK_BUDGETS = {
    "classify": 50,
    "extract": 300,
    "summarize": 500,
    "generate_code": 1500,
    "analyze": 800,
}

def call_with_budget(task_type: str, model: str, messages: list):
    return client.chat.completions.create(
        model=model,
        messages=messages,
        max_tokens=TASK_BUDGETS[task_type]
    )
```

### `temperature` — Randomness and Output Length

Temperature controls the sharpness of the probability distribution over the next token. At `temperature=0`, the model always picks the highest-probability token — deterministic and typically concise. At higher values, it samples more broadly — more creative, more verbose, more variable in length.

For agents doing structured work (extraction, classification, SQL generation), `temperature=0` or `0.1` is the right setting. It reduces output variance and average token length simultaneously.

```python
# Classification — zero temperature, hard cap
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
    max_tokens=20,
    temperature=0
)

# Creative synthesis — higher temperature, larger budget
response = client.chat.completions.create(
    model="claude-sonnet-4-5",
    messages=messages,
    max_tokens=800,
    temperature=0.7
)
```

**Economic rule:** Temperature and `max_tokens` are paired controls. Low temperature without a tight `max_tokens` is still vulnerable to verbose outputs when the task is open-ended. Set both.

### `top_p` — Nucleus Sampling

`top_p` (nucleus sampling) restricts token selection to the smallest vocabulary subset that accounts for `p` probability mass. At `top_p=0.1`, only the top ~10% probability tokens are considered. At `top_p=1.0`, the full distribution is available.

`top_p` and `temperature` both control output entropy — **don't adjust both simultaneously**. Pick one. In practice:
- Use `temperature` when you want a continuous control over creativity
- Use `top_p` when you want to hard-constrain vocabulary diversity regardless of temperature

For cost control, `top_p=0.9` is a reasonable default that slightly reduces tail token selections without materially affecting quality.

### `top_k` (Anthropic) — Hard Vocabulary Cap

Claude exposes `top_k` as an integer cap on the candidate token set. `top_k=1` is equivalent to greedy decoding (same effect as `temperature=0`). `top_k=40` is a common middle ground.

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=300,
    top_k=20,         # only consider top 20 tokens at each step
    top_p=0.9,
    messages=[{"role": "user", "content": "Extract the invoice total from this text: ..."}]
)
```

For extraction tasks, low `top_k` (10–20) combined with low temperature produces tighter, shorter, more predictable outputs.

### `presence_penalty` and `frequency_penalty` — Repetition and Verbosity Controls

These are OpenAI parameters (`presence_penalty` and `frequency_penalty`, both ranging from -2.0 to 2.0):

- **`presence_penalty`**: Penalizes tokens that have appeared *at all* in the output so far. Encourages introducing new concepts rather than restating existing ones. Positive values reduce repetitive elaboration.
- **`frequency_penalty`**: Penalizes tokens proportional to how *many times* they've already appeared. Specifically suppresses repeated words and phrases.

Both reduce verbosity, but target different patterns:

```python
# Suppress repetitive elaboration in analysis tasks
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    max_tokens=600,
    temperature=0.3,
    presence_penalty=0.4,   # discourages re-covering the same ground
    frequency_penalty=0.3   # suppresses literal word repetition
)
```

**Caution:** Values above 1.0 start producing unnatural outputs. For agent tasks with structured output requirements (JSON, SQL), keep penalties at 0 — they can corrupt format compliance.

### Stop Sequences — Surgical Output Termination

Stop sequences tell the model to halt generation when a specific string appears. For structured output, this is the most reliable way to prevent over-generation:

```python
# Stop immediately after the closing brace — don't let the model add explanation
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "Extract data as JSON. Output only the JSON object."},
        {"role": "user", "content": user_input}
    ],
    max_tokens=400,
    stop=["}\n\n", "```"]  # halt on JSON close or code fence close
)
```

```python
# Anthropic stop sequences
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=400,
    stop_sequences=["</result>", "\n\nHuman:"],
    messages=[{"role": "user", "content": "..."}]
)
```

Stop sequences are zero-cost: they reduce billed tokens by terminating generation early. For structured output tasks, combining `max_tokens` with a stop sequence gives you both a hard cap and an early exit when the model finishes cleanly.

### Structured Output — Forcing Format, Constraining Length

When the output has a known schema, enforce it. Constrained decoding (structured output mode) forces the model to produce schema-valid JSON, which eliminates retries and typically produces shorter outputs than prose-wrapped JSON.

```python
from pydantic import BaseModel

class InvoiceExtraction(BaseModel):
    vendor: str
    total: float
    currency: str
    date: str

response = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "Extract invoice fields."},
        {"role": "user", "content": invoice_text}
    ],
    response_format=InvoiceExtraction,
    max_tokens=150  # schema-constrained output is predictably short
)
result = response.choices[0].message.parsed
```

Structured output eliminates the token overhead of prose framing ("Here are the extracted fields:"), JSON wrapping explanation, and downstream parsing retries.

---

## Model Pricing and the Exchange Rate

All the above parameters operate on tokens. The model choice determines the price per token — and that price spreads across every step.

### Pricing Reference (April 2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|---|---|---|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude Opus 4.5 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |

Output tokens cost 3–5x input tokens. Verbose responses are not just slow — they're expensive. A 1,000-token output on Opus 4.5 costs $0.075. The same output on Haiku 3.5 costs $0.004. That's an 18x spread, not counting the input.

### Route by Task, Not by Default

Most agent steps do not require the most capable model. Routing deliberately is the highest-leverage cost decision you can make:

```python
def select_model(task_type: str) -> str:
    routing = {
        "classify":     "gpt-4o-mini",      # $0.15/$0.60 — trivial task
        "extract":      "gpt-4o-mini",      # structured, deterministic
        "summarize":    "claude-haiku-3-5", # $0.80/$4.00 — cheap, fast
        "generate_sql": "claude-sonnet-4",  # $3/$15 — needs schema reasoning
        "analyze":      "claude-sonnet-4",  # balanced
        "plan":         "claude-opus-4-5",  # only for ambiguous, high-stakes steps
    }
    return routing.get(task_type, "claude-sonnet-4")
```

A 12-step agent chain where 9 steps route to Haiku/GPT-4o-mini and 3 route to Sonnet runs at roughly 15–20% of the all-Sonnet cost — usually with no measurable quality difference on the cheap steps.

---

## Cost Observability: Instrument Before You Optimize

Token cost is invisible without instrumentation. Track at the step level — session-level billing tells you a total, not where to cut.

```python
import time
from dataclasses import dataclass

@dataclass
class AgentStepCost:
    step_name: str
    model: str
    input_tokens: int
    output_tokens: int
    cached_tokens: int
    duration_ms: float

    @property
    def estimated_cost_usd(self) -> float:
        pricing = {
            "gpt-4o":           (0.0000025,  0.00001),
            "gpt-4o-mini":      (0.00000015, 0.0000006),
            "claude-sonnet-4":  (0.000003,   0.000015),
            "claude-haiku-3-5": (0.0000008,  0.000004),
            "claude-opus-4-5":  (0.000015,   0.000075),
        }
        input_rate, output_rate = pricing.get(self.model, (0, 0))
        billable_input = self.input_tokens - self.cached_tokens
        return (billable_input * input_rate) + (self.output_tokens * output_rate)

def tracked_call(step_name: str, model: str, **kwargs):
    start = time.time()
    response = client.chat.completions.create(model=model, **kwargs)
    elapsed = (time.time() - start) * 1000

    usage = response.usage
    cost = AgentStepCost(
        step_name=step_name,
        model=model,
        input_tokens=usage.prompt_tokens,
        output_tokens=usage.completion_tokens,
        cached_tokens=getattr(usage.prompt_tokens_details, "cached_tokens", 0) if usage.prompt_tokens_details else 0,
        duration_ms=elapsed
    )
    log_cost(cost)  # send to your metrics pipeline
    return response
```

What to measure:
- Token count and cost per step — to know which steps are expensive
- Output-to-input ratio per step — high ratios signal verbose steps to constrain
- Cache hit rate — low rates on static prompts mean unstructured prompt layout
- Cost variance per step — high variance means parameter settings are underdetermined

