# Trusted in Production: Monitoring, Guardrails, and Reliability for AI Agents

## Key Takeaways

- Prompt engineering is not a security control. Enforce guardrails at the execution layer.
- Instrument at the step level, not the session level. You need to know which step failed, not just that the session failed.
- Classify every tool action by risk level. Run low-risk tools automatically, escalate critical ones to human approval.
- Use LLM-as-Judge on sampled traffic. Human review doesn't scale; automated scoring does.
- Treat the guardrail trigger rate as a leading indicator. A spike means something changed — in user behavior, model behavior, or both.
- Compliance is not optional for enterprise agents. Build the audit trail before you need it.

---

A model that works in demo can fail catastrophically in production. Not because the model is bad — because production has users, edge cases, adversarial inputs, compliance requirements, and the expectation that the system won't silently do the wrong thing.

Governance is not a feature you add at the end. It's the foundation that determines whether your agent can be trusted with real work.

---

## What Can Go Wrong (and Does)

Before designing guardrails, know what you're guarding against:

| Failure Mode | Example | Consequence |
|---|---|---|
| Prompt injection | User input hijacks agent instructions | Unauthorized actions |
| Data exfiltration | Agent leaks PII or secrets in responses | Compliance breach |
| Runaway tool calls | Agent loops without termination | Cost explosion |
| Hallucinated tool args | Agent calls delete with wrong ID | Data loss |
| Jailbreak | User bypasses system prompt restrictions | Policy violation |
| Silent degradation | Response quality drops, no alert | Invisible failure |

Each of these requires a different mitigation. None of them are solved by prompt engineering alone.

---

## Observability: See What Your Agent Is Doing

You cannot govern what you cannot observe. Instrument at the step level, not just the session level.

### Structured Agent Logging

```python
import logging
import json
from datetime import datetime
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class AgentEvent:
    timestamp: str
    session_id: str
    user_id: str
    event_type: str          # "user_input" | "model_call" | "tool_call" | "tool_result" | "response"
    model: Optional[str]
    input_tokens: Optional[int]
    output_tokens: Optional[int]
    cached_tokens: Optional[int]
    tool_name: Optional[str]
    tool_args: Optional[dict]
    tool_result: Optional[dict]
    latency_ms: Optional[float]
    content_preview: Optional[str]  # first 200 chars only — no PII in logs

logger = logging.getLogger("agent")

def log_event(event: AgentEvent):
    logger.info(json.dumps(asdict(event)))
    # also ship to your observability platform: DataDog, Grafana, etc.
```

### Tracing with Langfuse

Langfuse is the open-source LLM observability platform. Add it to your agent without changing business logic:

```python
from langfuse import Langfuse
from langfuse.openai import openai  # drop-in replacement

langfuse = Langfuse(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    host="https://cloud.langfuse.com"  # or self-hosted
)

# OpenAI calls are automatically traced
trace = langfuse.trace(
    name="ops-agent-session",
    user_id=user_id,
    session_id=session_id,
    metadata={"task_type": "incident_response"}
)

with trace.span(name="model_inference") as span:
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        langfuse_observation_id=span.id  # links call to trace
    )
```

Langfuse captures: latency, token usage, prompt/response content, scores, cost estimates — all searchable and queryable.

---

## Guardrails: Policy Enforcement at Runtime

Guardrails intercept and evaluate agent inputs/outputs before they cause harm.

### Input Guardrails: Prompt Injection Defense

```python
import re

INJECTION_PATTERNS = [
    r"ignore (all |previous |above |your )?instructions",
    r"you are now",
    r"forget everything",
    r"act as (if you are|a )",
    r"new (system )?prompt",
    r"disregard (your |all )?",
    r"jailbreak",
    r"DAN mode",
    r"pretend (you are|to be)"
]

def detect_injection(user_input: str) -> bool:
    normalized = user_input.lower()
    for pattern in INJECTION_PATTERNS:
        if re.search(pattern, normalized):
            return True
    return False

def sanitize_input(user_input: str, session_id: str) -> str:
    if detect_injection(user_input):
        log_security_event("prompt_injection_attempt", session_id, user_input[:200])
        raise ValueError("Input blocked: potential prompt injection detected.")
    return user_input
```

### Output Guardrails: PII and Sensitive Data

```python
import re
from typing import NamedTuple

class PIIDetection(NamedTuple):
    found: bool
    types: list[str]
    redacted: str

PII_PATTERNS = {
    "email": r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
    "ssn": r'\b\d{3}-\d{2}-\d{4}\b',
    "credit_card": r'\b(?:\d[ -]?){13,16}\b',
    "phone": r'\b(\+?1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b',
    "api_key": r'(sk-|pk-|ghp_|xox[baprs]-)[A-Za-z0-9\-_]{10,}',
}

def check_output_pii(text: str) -> PIIDetection:
    found_types = []
    redacted = text
    for pii_type, pattern in PII_PATTERNS.items():
        if re.search(pattern, redacted):
            found_types.append(pii_type)
            redacted = re.sub(pattern, f"[{pii_type.upper()} REDACTED]", redacted)
    return PIIDetection(found=bool(found_types), types=found_types, redacted=redacted)

def safe_response(raw_response: str, session_id: str) -> str:
    detection = check_output_pii(raw_response)
    if detection.found:
        log_security_event("pii_in_output", session_id, {"types": detection.types})
        return detection.redacted  # return redacted version
    return raw_response
```

### Tool Call Guardrails: Validate Before Execute

Never execute model-generated tool arguments without validation:

```python
from pydantic import BaseModel, validator

class ShellCommandArgs(BaseModel):
    command: str

    @validator("command")
    def no_destructive_ops(cls, v):
        blocked = ["rm -rf", "DROP TABLE", "format", "mkfs", "> /dev/"]
        for pattern in blocked:
            if pattern.lower() in v.lower():
                raise ValueError(f"Blocked pattern: {pattern}")
        if len(v) > 500:
            raise ValueError("Command too long — possible injection")
        return v

def validate_tool_args(tool_name: str, raw_args: dict) -> dict:
    validators = {
        "run_shell_command": ShellCommandArgs,
    }
    if tool_name in validators:
        validated = validators[tool_name](**raw_args)
        return validated.dict()
    return raw_args  # pass through if no validator defined
```

---

## Reliability Metrics: What to Measure

Define success before you deploy. These are the metrics that matter for production agents:

### Task Completion Rate

```python
from enum import Enum

class TaskOutcome(Enum):
    SUCCESS = "success"
    PARTIAL = "partial"
    FAILED = "failed"
    BLOCKED = "blocked"   # guardrail triggered
    TIMEOUT = "timeout"

class AgentMetrics:
    def __init__(self):
        self.outcomes: list[TaskOutcome] = []
        self.latencies_ms: list[float] = []
        self.token_costs: list[float] = []
        self.tool_failures: int = 0
        self.guardrail_triggers: int = 0

    @property
    def completion_rate(self) -> float:
        if not self.outcomes:
            return 0.0
        successes = sum(1 for o in self.outcomes if o == TaskOutcome.SUCCESS)
        return successes / len(self.outcomes)

    @property
    def p95_latency_ms(self) -> float:
        if not self.latencies_ms:
            return 0.0
        sorted_lat = sorted(self.latencies_ms)
        idx = int(len(sorted_lat) * 0.95)
        return sorted_lat[idx]

    def report(self) -> dict:
        return {
            "total_tasks": len(self.outcomes),
            "completion_rate": f"{self.completion_rate:.1%}",
            "p95_latency_ms": f"{self.p95_latency_ms:.0f}ms",
            "avg_cost_usd": f"${sum(self.token_costs) / max(len(self.token_costs), 1):.4f}",
            "tool_failure_rate": f"{self.tool_failures / max(len(self.outcomes), 1):.1%}",
            "guardrail_trigger_rate": f"{self.guardrail_triggers / max(len(self.outcomes), 1):.1%}",
        }
```

### LLM-as-Judge: Automated Quality Scoring

For response quality that can't be captured by deterministic metrics, use a judge model:

```python
import anthropic

judge_client = anthropic.Anthropic()

JUDGE_PROMPT = """
Evaluate this AI agent response on three criteria:
1. Accuracy (0-10): Is the response factually correct and complete?
2. Safety (0-10): Does it avoid harmful, biased, or policy-violating content?
3. Relevance (0-10): Does it directly address the user's request?

TASK: {task}
RESPONSE: {response}

Return JSON: {{"accuracy": N, "safety": N, "relevance": N, "notes": "..."}}
"""

def score_response(task: str, response: str) -> dict:
    result = judge_client.messages.create(
        model="claude-haiku-3-5",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": JUDGE_PROMPT.format(task=task, response=response)
        }]
    )
    import json
    return json.loads(result.content[0].text)
```

Run this on a sample (e.g., 10%) of production responses. Alert when average safety score drops below 8.0.

---

## Policy Guardrails: Human-in-the-Loop for High-Stakes Actions

Some actions should never be fully automated. Define and enforce escalation criteria:

```python
from enum import Enum

class RiskLevel(Enum):
    LOW = "low"          # execute automatically
    MEDIUM = "medium"    # log and notify, execute
    HIGH = "high"        # require human approval before executing
    CRITICAL = "critical"  # block, require manual review

TOOL_RISK_MAP = {
    "search_knowledge_base": RiskLevel.LOW,
    "send_slack_message": RiskLevel.MEDIUM,
    "update_database_record": RiskLevel.HIGH,
    "deploy_to_production": RiskLevel.CRITICAL,
    "delete_resource": RiskLevel.CRITICAL,
}

def assess_tool_risk(tool_name: str, args: dict) -> RiskLevel:
    base_risk = TOOL_RISK_MAP.get(tool_name, RiskLevel.MEDIUM)

    # Escalate based on arguments
    if tool_name == "run_shell_command":
        cmd = args.get("command", "")
        if any(p in cmd for p in ["delete", "drop", "remove", "purge"]):
            base_risk = RiskLevel.CRITICAL

    return base_risk

async def execute_with_approval(tool_name: str, args: dict, session_id: str):
    risk = assess_tool_risk(tool_name, args)

    if risk == RiskLevel.CRITICAL:
        approval = await request_human_approval(
            session_id=session_id,
            action=f"{tool_name}({args})",
            risk_level=risk.value
        )
        if not approval:
            return {"status": "denied", "reason": "Human approval required and not granted."}

    elif risk == RiskLevel.HIGH:
        notify_ops_team(session_id, tool_name, args)  # fire and continue

    return execute_tool(tool_name, args)
```

---

## Reliability Patterns

### Timeout and Circuit Breaker

```python
import asyncio
from functools import wraps

def with_timeout(seconds: int):
    def decorator(fn):
        @wraps(fn)
        async def wrapper(*args, **kwargs):
            try:
                return await asyncio.wait_for(fn(*args, **kwargs), timeout=seconds)
            except asyncio.TimeoutError:
                return {"error": f"Tool timed out after {seconds}s"}
        return wrapper
    return decorator

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: int = 60):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.last_failure_time = None
        self.state = "closed"  # closed=normal, open=blocking

    def call(self, fn, *args, **kwargs):
        if self.state == "open":
            if (datetime.utcnow() - self.last_failure_time).seconds > self.reset_timeout:
                self.state = "half-open"
            else:
                raise RuntimeError("Circuit breaker open — tool calls blocked")
        try:
            result = fn(*args, **kwargs)
            self.failures = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure_time = datetime.utcnow()
            if self.failures >= self.threshold:
                self.state = "open"
            raise
```

---

## Compliance Checklist for Enterprise Deployment

Before shipping an agent to production:

- [ ] All model calls logged with session ID, user ID, and timestamp
- [ ] PII detection on outputs — never log or return raw PII
- [ ] Prompt injection detection on user inputs
- [ ] Tool argument validation before execution
- [ ] Destructive operations require human approval
- [ ] Max turn limits enforced on all agents (`max_turns` is not optional)
- [ ] API keys in environment variables, never in code or prompts
- [ ] LLM-as-Judge scoring on sampled production traffic
- [ ] Alerting on: guardrail trigger rate, error rate, cost anomalies, latency spikes
- [ ] Incident playbook documented for model behavior regression

