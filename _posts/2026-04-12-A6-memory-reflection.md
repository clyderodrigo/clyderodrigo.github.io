# Agents That Recall, Experiment and Learn: Persistent Memory and Reflective Learning

## Key Takeaways

- A stateless agent is a liability for recurring work. Invest in memory proportional to the task's recurrence.
- Vector-based semantic memory beats keyword lookup for relevance. `text-embedding-3-small` is cheap enough to embed everything.
- Run reflection after task completion, not during. Use a fast/cheap model (Haiku, GPT-4o-mini) — it's a background operation.
- Inject memory selectively into context. Dumping all history into every prompt defeats the purpose and inflates cost.
- Procedural memory compounds. An agent that learns from 100 deployments is not the same agent as one that's done 1.

---

A stateless agent repeats the same mistakes. It re-asks questions it was already answered. It re-discovers facts it learned last week. Every session starts cold.

Memory changes this. A well-designed memory system lets an agent accumulate knowledge across sessions, refine its approach based on past outcomes, and provide continuity that makes it genuinely useful over time — not just in a single conversation.

This article covers the memory architecture patterns that matter, how to implement them with Claude and GPT agents, and how to build reflection loops that improve performance through experience.

---

## Memory Taxonomy

Not all memory is the same. Map your use case to the right type.

| Type | What It Stores | Scope | Example |
|---|---|---|---|
| **Working memory** | Current task state, active context | Single session | Tool call results, current step |
| **Episodic memory** | Past interactions, task history | Cross-session | "Last Tuesday you asked me to..." |
| **Semantic memory** | Facts, domain knowledge, preferences | Persistent | User preferences, system docs |
| **Procedural memory** | Learned patterns, refined workflows | Persistent | "When X fails, try Y first" |

Most production agents need all four — working memory is handled by the context window, the others require explicit implementation.

---

## Working Memory: Context Window Management

The context window is your agent's working memory. Engineer it deliberately.

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class WorkingMemory:
    task_goal: str
    completed_steps: list[str] = field(default_factory=list)
    tool_results: dict[str, Any] = field(default_factory=dict)
    observations: list[str] = field(default_factory=list)
    pending_questions: list[str] = field(default_factory=list)

    def to_system_context(self) -> str:
        lines = [f"CURRENT GOAL: {self.task_goal}"]
        if self.completed_steps:
            lines.append("COMPLETED STEPS:\n" + "\n".join(f"  - {s}" for s in self.completed_steps))
        if self.observations:
            lines.append("OBSERVATIONS:\n" + "\n".join(f"  - {o}" for o in self.observations))
        return "\n".join(lines)
```

Embedding structured working memory in the system prompt gives the model reliable orientation at every step.

---

## Episodic Memory: Remembering Past Sessions

Store interaction summaries and retrieve them by relevance at session start.

### Storage Layer

```python
import json
from datetime import datetime
from pathlib import Path

class EpisodicMemoryStore:
    def __init__(self, storage_path: str = "agent_memory/episodes"):
        self.path = Path(storage_path)
        self.path.mkdir(parents=True, exist_ok=True)

    def save_episode(self, session_id: str, summary: str, metadata: dict):
        episode = {
            "session_id": session_id,
            "timestamp": datetime.utcnow().isoformat(),
            "summary": summary,
            "outcome": metadata.get("outcome"),
            "tags": metadata.get("tags", []),
            "user_id": metadata.get("user_id")
        }
        file = self.path / f"{session_id}.json"
        file.write_text(json.dumps(episode, indent=2))

    def get_recent_episodes(self, user_id: str, limit: int = 5) -> list[dict]:
        episodes = []
        for f in sorted(self.path.glob("*.json"), key=lambda x: x.stat().st_mtime, reverse=True):
            ep = json.loads(f.read_text())
            if ep.get("user_id") == user_id:
                episodes.append(ep)
            if len(episodes) >= limit:
                break
        return episodes
```

### Generating Episode Summaries

After each session, use a cheap model to summarize it:

```python
from openai import OpenAI

client = OpenAI()

def summarize_session(messages: list[dict]) -> str:
    transcript = "\n".join(
        f"{m['role'].upper()}: {m['content']}"
        for m in messages
        if isinstance(m.get('content'), str)
    )

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": (
                    "Summarize this agent session. Include: "
                    "1) what was asked, 2) what actions were taken, "
                    "3) outcome/resolution, 4) any unresolved items. "
                    "Be factual and concise. Max 200 words."
                )
            },
            {"role": "user", "content": transcript}
        ],
        max_tokens=300
    )
    return response.choices[0].message.content
```

---

## Semantic Memory: Vector-Based Long-Term Knowledge

Facts, user preferences, domain knowledge — stored in a vector database and retrieved by semantic similarity.

```python
# Using ChromaDB for lightweight persistent vector storage
import chromadb
from openai import OpenAI

client = OpenAI()
chroma = chromadb.PersistentClient(path="agent_memory/semantic")
collection = chroma.get_or_create_collection("agent_knowledge")

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def remember(fact: str, tags: list[str] = None, doc_id: str = None):
    import uuid
    doc_id = doc_id or str(uuid.uuid4())
    collection.add(
        documents=[fact],
        embeddings=[embed(fact)],
        metadatas=[{"tags": ",".join(tags or [])}],
        ids=[doc_id]
    )

def recall(query: str, top_k: int = 5) -> list[str]:
    results = collection.query(
        query_embeddings=[embed(query)],
        n_results=top_k
    )
    return results["documents"][0] if results["documents"] else []
```

### Injecting Semantic Memory at Session Start

```python
def build_system_prompt(user_query: str, user_id: str) -> str:
    relevant_facts = recall(user_query, top_k=5)
    recent_episodes = memory_store.get_recent_episodes(user_id, limit=3)

    memory_context = ""
    if relevant_facts:
        memory_context += "RELEVANT KNOWLEDGE:\n" + "\n".join(f"- {f}" for f in relevant_facts)
    if recent_episodes:
        memory_context += "\n\nRECENT HISTORY:\n"
        for ep in recent_episodes:
            memory_context += f"- [{ep['timestamp'][:10]}] {ep['summary']}\n"

    base_prompt = "You are an expert operations agent with persistent memory across sessions."
    return f"{base_prompt}\n\n{memory_context}" if memory_context else base_prompt
```

---

## Reflection: Learning from Task Repetition

Reflection is the mechanism by which an agent improves. After completing a task — especially a recurring one — it reviews what happened and extracts generalizable lessons.

### Reflection Loop

```python
import anthropic

client = anthropic.Anthropic()

REFLECTION_PROMPT = """
You just completed a task. Review what happened and extract learnings.

TASK: {task}
STEPS TAKEN: {steps}
OUTCOME: {outcome}
ERRORS ENCOUNTERED: {errors}

Identify:
1. What worked well and should be repeated
2. What failed and should be avoided or done differently
3. Any patterns or heuristics that would help future instances of this task

Return a JSON object:
{{
  "lessons": ["lesson 1", "lesson 2", ...],
  "patterns": {{"trigger": "action", ...}},
  "avoid": ["anti-pattern 1", ...]
}}
"""

def reflect_on_task(task: str, steps: list, outcome: str, errors: list) -> dict:
    response = client.messages.create(
        model="claude-haiku-3-5",  # cheap — this runs after every task
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": REFLECTION_PROMPT.format(
                task=task,
                steps="\n".join(f"  {i+1}. {s}" for i, s in enumerate(steps)),
                outcome=outcome,
                errors="\n".join(errors) if errors else "None"
            )
        }]
    )
    import json
    return json.loads(response.content[0].text)
```

### Storing and Retrieving Procedural Memory

```python
class ProceduralMemory:
    def __init__(self, storage_path: str = "agent_memory/procedural"):
        self.path = Path(storage_path)
        self.path.mkdir(parents=True, exist_ok=True)
        self.lessons_file = self.path / "lessons.jsonl"

    def store_reflection(self, task_type: str, reflection: dict):
        entry = {
            "task_type": task_type,
            "timestamp": datetime.utcnow().isoformat(),
            **reflection
        }
        with open(self.lessons_file, "a") as f:
            f.write(json.dumps(entry) + "\n")

        # Also embed lessons into semantic memory for retrieval
        for lesson in reflection.get("lessons", []):
            remember(
                f"For {task_type}: {lesson}",
                tags=["lesson", task_type]
            )

    def get_lessons_for_task(self, task_type: str) -> list[str]:
        if not self.lessons_file.exists():
            return []
        lessons = []
        with open(self.lessons_file) as f:
            for line in f:
                entry = json.loads(line)
                if entry["task_type"] == task_type:
                    lessons.extend(entry.get("lessons", []))
        return list(set(lessons))[-20:]  # deduplicate, keep recent
```

---

## The Full Memory-Augmented Agent

Wiring all memory types together:

```python
class MemoryAugmentedAgent:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.episodic = EpisodicMemoryStore()
        self.procedural = ProceduralMemory()
        self.session_messages = []
        self.working_memory = None

    def start_task(self, task: str):
        self.working_memory = WorkingMemory(task_goal=task)
        prior_lessons = self.procedural.get_lessons_for_task(task[:50])

        system_prompt = build_system_prompt(task, self.user_id)
        if prior_lessons:
            system_prompt += "\n\nPRIOR LESSONS FOR THIS TASK TYPE:\n"
            system_prompt += "\n".join(f"- {l}" for l in prior_lessons)

        self.session_messages = [{"role": "system", "content": system_prompt}]

    def run_step(self, user_input: str) -> str:
        self.session_messages.append({"role": "user", "content": user_input})
        # ... model call + tool execution loop ...
        return response_text

    def end_task(self, outcome: str, errors: list = None):
        # Summarize and save episode
        summary = summarize_session(self.session_messages)
        self.episodic.save_episode(
            session_id=str(uuid.uuid4()),
            summary=summary,
            metadata={"outcome": outcome, "user_id": self.user_id}
        )

        # Reflect and store procedural knowledge
        if self.working_memory:
            reflection = reflect_on_task(
                task=self.working_memory.task_goal,
                steps=self.working_memory.completed_steps,
                outcome=outcome,
                errors=errors or []
            )
            self.procedural.store_reflection(
                task_type=self.working_memory.task_goal[:50],
                reflection=reflection
            )
```

