---
#layout: post
title: The Platform Is the Agent. Choosing Your AI Runtime for Models, MCP, and Tools.
description: >-
    Before you write a single line of agent logic, you make a decision that constrains everything else, which platform runs your agent. That platform — your runtime — determines which models you can use, how tools connect, where data flows, who can govern it, and what you pay.
author: clyde
date: 2026-04-02
categories: [Agentic AI]
tag: [Agentic AI]
---
# The Platform Is the Agent: Choosing Your AI Runtime for Models, MCP, and Tools

## Key Takeaways

- A runtime hosts the full agent stack: model, tools, MCP servers, governance. It's not just an inference endpoint anymore.
- Managed cloud platforms (AI Foundry, Bedrock, OpenAI, Copilot Studio) give you enterprise data residency, RBAC, and audit trails — this is where production agents belong.
- Embedded application runtimes (M365, VS Code, Dynamics, Fabric, Power Platform) meet users where they work — the agent is part of the tool, not a separate system to adopt.
- MCP is the connective tissue. All major runtimes now support MCP server connections — design your tools as MCP servers and they become portable across the stack.
- Governance must be enforced at the runtime level, not just in agent code. Platform-level guardrails (Bedrock Guardrails, Azure Content Safety, Power Platform DLP) cannot be bypassed by a misconfigured prompt.

---

Before you write a single line of agent logic, you make a decision that constrains everything else: which platform runs your agent. That platform — your runtime — determines which models you can use, how tools connect, where data flows, who can govern it, and what you pay.

Get the runtime decision right and everything else builds cleanly. Get it wrong and you're re-architecting six months in.

---

## What a Runtime Actually Is

An AI agent runtime is the platform that hosts the full agent stack: the model, the tool connections, the MCP servers, the memory layer, the execution loop, and the governance controls.

```
┌─────────────────────────── RUNTIME ──────────────────────────────┐
│                                                                    │
│   ┌──────────┐   ┌─────────────┐   ┌────────────────────────┐   │
│   │  Model   │   │  MCP Server │   │  Tools & Integrations  │   │
│   │ Catalog  │   │  Registry   │   │  (APIs, DBs, systems)  │   │
│   └──────────┘   └─────────────┘   └────────────────────────┘   │
│                                                                    │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │  Governance: data residency · RBAC · audit · guardrails  │   │
│   └──────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

The runtime is not just the inference endpoint. It's the entire operating environment — and choosing it is an architectural decision, not an implementation detail.

---

## Two Runtime Categories

**Managed cloud platforms** — dedicated AI infrastructure with enterprise-grade inference, governance, and integration. You build agents here through APIs, SDKs, or visual builders. Microsoft AI Foundry, Amazon Bedrock, OpenAI Platform, and Microsoft Copilot Studio are the primary players.

**Embedded application runtimes** — AI capabilities built directly into the tools engineers and analysts already use. VS Code, Power Automate, and Power BI don't just run your work — they host agents, MCP connections, and model calls within their own execution environment.

Both are cloud-hosted. The distinction is not where they run — it's the surface they expose and the audience they serve.

---

## Managed Cloud Platforms

### Microsoft AI Foundry

Azure AI Foundry is Microsoft's unified platform for building, deploying, and governing enterprise AI agents. It is the production-grade successor to Azure ML and Azure OpenAI Service, with first-class support for the full agent stack.

**What it provides:**

- **Model catalog**: GPT-4o, o3, Phi-4, Mistral, Llama, and custom fine-tuned models — all served through a unified endpoint with Azure SLA
- **Agent SDK**: Build agents in Python with tool calling, code interpreter, file search, and MCP server connections
- **Connections**: Pre-built integrations to Azure Storage, Cognitive Search, SQL, Fabric, and third-party APIs
- **Governance**: Azure RBAC, private endpoints, VNet injection, content filtering, audit logs, Microsoft Purview integration

```python
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FileSearchTool, CodeInterpreterTool
from azure.identity import DefaultAzureCredential

client = AIProjectClient.from_connection_string(
    conn_str="eastus.api.azureml.ms;your-subscription-id;your-rg;your-project",
    credential=DefaultAzureCredential()
)

# Create an agent with tools
agent = client.agents.create_agent(
    model="gpt-4o",
    name="ops-analyst",
    instructions="You are an operations analyst. Use file search and code interpreter for data tasks.",
    tools=[FileSearchTool().definitions, CodeInterpreterTool().definitions],
    tool_resources=FileSearchTool(vector_store_ids=["vs_your_store_id"]).resources,
)

# Create a thread and run
thread = client.agents.create_thread()
client.agents.create_message(
    thread_id=thread.id,
    role="user",
    content="Analyze the attached incident logs and summarize recurring failure patterns."
)

run = client.agents.create_and_process_run(
    thread_id=thread.id,
    agent_id=agent.id
)
print(run.status)
```

**MCP integration in AI Foundry:**

```python
from azure.ai.projects.models import McpTool

# Connect an MCP server as a tool source
mcp_tool = McpTool(
    server_label="internal-ops",
    server_url="https://your-mcp-server.internal/sse",
    allowed_tools=["get_service_status", "tail_logs", "query_incidents"]
)

agent = client.agents.create_agent(
    model="gpt-4o",
    name="ops-agent",
    instructions="You have access to ops tools via MCP.",
    tools=mcp_tool.definitions
)
```

**Choose AI Foundry when:** You are already in Azure, have data sovereignty requirements, need enterprise RBAC, or are integrating with Microsoft Fabric, SharePoint, or Dynamics.

---

### Amazon Bedrock

Bedrock is AWS's managed AI runtime — model-agnostic by design, with first-class support for multi-model agents, guardrails, and knowledge bases.

**What it provides:**

- **Model catalog**: Claude (Anthropic), Llama (Meta), Mistral, Titan, Command (Cohere), Nova — all on-demand, no provisioning
- **Bedrock Agents**: Fully managed agent runtime with tool orchestration, knowledge base retrieval, and session management
- **Knowledge Bases**: Native RAG with S3 + OpenSearch or Aurora vector store, no pipeline to manage
- **Guardrails**: Content filtering, PII redaction, topic restrictions — applied at the platform level, not in code
- **Governance**: AWS IAM, VPC endpoints, CloudTrail, PrivateLink, HIPAA/SOC2/FedRAMP eligible

```python
import boto3, json

bedrock_agent = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

# Invoke a Bedrock Agent (model + tools + KB wired together in the console or IaC)
response = bedrock_agent.invoke_agent(
    agentId="AGENT_ID",
    agentAliasId="TSTALIASID",
    sessionId="session-001",
    inputText="Which services had the most P1 incidents last quarter? Cross-reference with deployment logs."
)

# Stream the response
for event in response["completion"]:
    if "chunk" in event:
        print(event["chunk"]["bytes"].decode(), end="", flush=True)
```

**Calling models directly (without Agents runtime):**

```python
import boto3, json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

# Claude on Bedrock
response = bedrock.invoke_model(
    modelId="anthropic.claude-opus-4-5-20241022-v2:0",
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2048,
        "tools": [...],   # same tool schema as Anthropic API
        "messages": [{"role": "user", "content": "Summarize this incident report."}]
    })
)
result = json.loads(response["body"].read())
```

**Bedrock Guardrails — applied to all traffic through the runtime:**

```python
# Guardrail applied at invocation — no code changes to agent logic needed
response = bedrock.invoke_model(
    modelId="anthropic.claude-sonnet-4-...",
    guardrailIdentifier="your-guardrail-id",
    guardrailVersion="1",
    body=json.dumps({...})
)
# Guardrail intercepts, evaluates, and redacts before your code sees the response
```

**Choose Bedrock when:** You are AWS-native, need multi-model flexibility, want managed guardrails applied infrastructure-wide, or are targeting FedRAMP/HIPAA workloads.

---

### OpenAI Platform

OpenAI's managed platform is the most capable API surface for GPT-4o and o3 models, with a growing Responses API that supports built-in tools and persistent agent state.

**What it provides:**

- **Models**: GPT-4o, o3, o3-mini — the full OpenAI frontier model lineup at lowest latency
- **Responses API**: Stateful agent runs with streaming, tool calling, and built-in tools (web search, file search, code interpreter, computer use)
- **Assistants**: Persistent agent configuration with threads, file attachments, and vector stores
- **MCP servers**: The Responses API connects to remote MCP servers directly
- **Governance**: Project-level API key scoping, spend limits, usage dashboards, SOC2 Type II

```python
from openai import OpenAI

client = OpenAI()

# Responses API — stateful, multi-turn, with built-in tool use
response = client.responses.create(
    model="gpt-4o",
    instructions="You are an expert infrastructure analyst. Be precise. Cite evidence.",
    tools=[
        {"type": "web_search_preview"},
        {"type": "file_search", "vector_store_ids": ["vs_your_store_id"]},
        {"type": "code_interpreter"}
    ],
    input="Compare our current Kubernetes resource limits against the incident data in the attached runbook.",
)

print(response.output_text)
```

**MCP connection in Responses API:**

```python
response = client.responses.create(
    model="gpt-4o",
    tools=[
        {
            "type": "mcp",
            "server_label": "deepwiki",
            "server_url": "https://mcp.deepwiki.com/mcp",
            "allowed_tools": ["read_wiki_structure", "search_documentation"]
        }
    ],
    input="What changed in the deployment pipeline last week?",
)
```

**Choose OpenAI Platform when:** You need the highest-capability GPT-4o/o3 models, want the simplest integration path, or are building on top of ChatGPT's ecosystem.

---

### Microsoft Copilot Studio

Copilot Studio is the low-code/pro-code runtime for building enterprise agents on Microsoft's infrastructure — without managing any of the underlying platform.

**What it provides:**

- **Visual agent builder**: Drag-and-drop topics, actions, and branching — no SDK required for common patterns
- **Model selection**: GPT-4o via Azure OpenAI, with content safety and responsible AI controls applied by default
- **Connectors**: 1,000+ Power Platform connectors to Microsoft 365, Dynamics, Salesforce, ServiceNow, and custom APIs
- **MCP support**: Declare MCP tools as agent actions — Copilot Studio handles tool calling protocol
- **Deployment**: Publish to Teams, SharePoint, web chat, or as a standalone app with one click
- **Governance**: Tenant-level DLP policies, admin center oversight, audit logs, data residency in your Microsoft tenant

**Pro-code extension — adding custom logic:**

```python
# Custom connector as a Copilot Studio action (exposed via Power Platform custom connector)
# This Python function runs in Azure Functions, registered as an action in Copilot Studio

import azure.functions as func
import json

def main(req: func.HttpRequest) -> func.HttpResponse:
    data = req.get_json()
    service_name = data.get("service_name")

    # Your business logic here
    status = get_service_health(service_name)

    return func.HttpResponse(
        json.dumps({"status": status["state"], "last_checked": status["timestamp"]}),
        mimetype="application/json"
    )
```

**Choose Copilot Studio when:** Business users need to author or maintain agent logic, you're deploying within Microsoft 365, or you need governance enforced by IT at the tenant level rather than by developers in code.

---

## Embedded Application Runtimes

These are the tools where engineers and analysts work every day. Each one ships with an agent runtime built in — and increasingly supports MCP and custom tool connections.

### VS Code — The Developer Agent Runtime

With GitHub Copilot's agent mode, VS Code becomes a full agent runtime: the model, tool use, multi-step reasoning, and MCP server connections all operate inside the editor.

**What it provides:**

- **Models**: Claude, GPT-4o, o3, Gemini — switchable per session
- **Agent mode**: Multi-step autonomous task completion across files, terminal, and browser preview
- **MCP integration**: Connect any MCP server directly in VS Code settings — tools appear natively in agent mode
- **Built-in tools**: File read/write, terminal execution, web fetch, test runner integration
- **Context**: `@workspace`, `@file`, `#selection` — deep codebase awareness at every call

**Connecting an MCP server in VS Code:**

```json
// .vscode/mcp.json  (workspace-scoped MCP config)
{
  "servers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${env:GITHUB_TOKEN}" }
    },
    "postgres": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "POSTGRES_CONNECTION_STRING": "${env:DB_CONN}" }
    },
    "internal-ops": {
      "type": "sse",
      "url": "https://your-mcp-server.internal/sse"
    }
  }
}
```

Once registered, these tools are available to the agent in natural language — no additional code. The agent can query your database, search GitHub issues, or call your internal MCP server as part of a single task.

**Custom instructions for the workspace agent:**

```markdown
<!-- .github/copilot-instructions.md -->
You are an operations engineer agent for the payments platform.
- Always validate schema before suggesting DB changes
- Reference internal runbooks via the ops-mcp server before proposing fixes
- Tag all code suggestions with the relevant JIRA ticket format: PAY-XXXX
```

**Choose VS Code when:** The agent is serving developers, task completion requires direct code manipulation, or you want tight integration between agent actions and the development workflow.

---

### Power Automate — The Workflow Agent Runtime

Power Automate is Microsoft's automation platform — and with AI Builder and Copilot integration, it's also a production agent runtime for business process automation.

**What it provides:**

- **AI Builder**: Pre-built models (document extraction, classification, sentiment) and custom model integration
- **Copilot in flows**: Natural language to flow — describe the automation, Copilot builds it
- **Agent flows**: Agentic multi-step automation that can branch, retry, and call external APIs based on AI reasoning
- **MCP connectors**: Connect MCP tools as custom actions within flows
- **Governance**: Power Platform DLP policies, connection governance, audit logs — IT-managed

**Agent flow pattern — AI-driven document routing:**

```
Trigger: Email arrives with attachment
  │
  ▼
AI Builder: Extract invoice fields (vendor, amount, PO number)
  │
  ▼
Condition: Amount > $50,000?
  ├── Yes → Approval workflow → Finance system update
  └── No  → Auto-approve → ERP write → Confirmation email
```

Custom connectors expose any API as a step, including MCP servers wrapped in an HTTP endpoint.

**Choose Power Automate when:** The process owner is a business analyst, the workflow needs to span Microsoft 365 and enterprise systems, or the agent output needs to trigger transactional systems (ERP, CRM, ITSM).

---

### Power BI — The Analytics Agent Runtime

Power BI's Copilot features turn the analytics platform into an agent runtime for data exploration, report generation, and insight extraction at scale.

**What it provides:**

- **Copilot for reports**: Natural language queries generate DAX, build visuals, and summarize insights
- **Narrative summaries**: Agent-generated prose summaries embedded directly in reports, refreshed automatically
- **Q&A agent**: Semantic layer awareness — users ask questions, the agent queries the model, returns a visual
- **Data agent (preview)**: Agentic access to Fabric data — multi-step query planning and execution against lakehouses and warehouses
- **Governance**: Sensitivity labels, row-level security enforced through the agent path — not bypassable

**The key capability:** Agents in Power BI operate against the **semantic model** — business logic, hierarchies, and security rules are already encoded. The agent doesn't bypass them; it reasons within them.

**Choose Power BI when:** The agent's role is analysis, reporting, or insight delivery — not system action. The audience is business users, and data governance through the semantic layer is non-negotiable.

---

## Choosing Your Runtime

The runtime decision maps directly to use case, audience, and governance requirements:

| Runtime                   | Best For                                            | Audience          | Governance Model               |
| ------------------------- | --------------------------------------------------- | ----------------- | ------------------------------ |
| **AI Foundry**      | Custom agents, fine-tuning, complex pipelines       | Engineers         | Azure RBAC + Purview           |
| **Bedrock**         | Multi-model, AWS-native, regulated workloads        | Engineers         | IAM + Guardrails               |
| **OpenAI Platform** | Highest-capability GPT/o3 agents, fastest iteration | Engineers         | Project scoping + spend limits |
| **Copilot Studio**  | Enterprise deployment, business-user authoring      | Analysts + IT     | Tenant DLP + admin center      |
| **VS Code**         | Developer productivity, code-adjacent tasks         | Developers        | Workspace + org policy         |
| **Power Automate**  | Business process automation, cross-system workflows | Business analysts | DLP + connection governance    |
| **Power BI**        | Data analysis, reporting, semantic layer queries    | Business users    | Sensitivity labels + RLS       |

**The runtime is not a deployment detail. It defines your model access, tool connectivity, governance posture, and the audience you can realistically serve.** Pick the platform that matches your target outcome — then build the agent to fit.
