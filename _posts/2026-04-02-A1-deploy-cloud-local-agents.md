---
#layout: post
title: The Execution Layer for AI Agents & Agentic Workflows **—** Managed Platforms like AI Foundry, OpenAI & Copilot Studio **—** Embedded Runtimes like Power Platform & VSCode
description: >-
  The Platform Is the Agent. Choosing Your AI Runtime for Models, MCP, and Tools.
author: clyde
date: 2026-04-02
categories: [Agentic AI]
tag: [Agentic AI]
---
## Key Takeaways

- The agent runtime hosts the full stack for deploying agents and agent workflows: model, tools, MCP servers, governance. It's not just an inference endpoint anymore.
- Managed cloud platforms (AI Foundry, Bedrock, OpenAI, Copilot Studio) give you enterprise data residency, RBAC, and audit trails — this is where production agents belong.
- Embedded application runtimes (M365, VS Code, Dynamics, Fabric, Power Platform) meet users where they work — the agent is part of the tool, not a separate system to adopt.
- Model Context Protocol is the connective tissue. All major runtimes now support MCP server connections — design your tools as MCP servers and they become portable across the stack.

---

## What a Runtime Actually Is

A agent runtime is the platform that hosts the full AI stack: the model, the tool connections, the MCP servers, the memory layer, the execution loop, and the governance controls.

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

The runtime is not just the inference endpoint. It's the entire operating environment for deploying agents with autonomy and semi or fully orchestrated agentic workflows — and choosing it is an architectural decision, not an implementation detail.

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

**Set up an agent workflow with Code Interpreter tool access in the Python SDK:**

```python
"""
The following Python sample shows how to create an agent with the code interpreter tool, upload a CSV file for analysis,
and request a bar chart based on the data. It demonstrates a complete workflow: upload a file, create an agent with Code
Interpreter enabled, request data visualization, and download the generated chart.
"""
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, CodeInterpreterTool, AutoCodeInterpreterToolParam

# Load the CSV file to be processed
asset_file_path = os.path.abspath(
    os.path.join(os.path.dirname(__file__), "../assets/synthetic_500_quarterly_results.csv")
)

# Format: "https://resource_name.ai.azure.com/api/projects/project_name"
PROJECT_ENDPOINT = "your_project_endpoint"

# Create clients to call Foundry API
project = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential(),
)
openai = project.get_openai_client()

# Upload the CSV file for the code interpreter to use
file = openai.files.create(purpose="assistants", file=open(asset_file_path, "rb"))

# Create agent with code interpreter tool
agent = project.agents.create_version(
    agent_name="MyAgent",
    definition=PromptAgentDefinition(
        model="gpt-5-mini",
        instructions="You are a helpful assistant.",
        tools=[CodeInterpreterTool(container=AutoCodeInterpreterToolParam(file_ids=[file.id]))],
    ),
    description="Code interpreter agent for data analysis and visualization.",
)

# Create a conversation for the agent interaction
conversation = openai.conversations.create()

# Send request to create a chart and generate a file
response = openai.responses.create(
    conversation=conversation.id,
    input="Could you please create bar chart in TRANSPORTATION sector for the operating profit from the uploaded csv file and provide file to me?",
    extra_body={"agent_reference": {"name": agent.name, "type": "agent_reference"}},
)

# Extract file information from response annotations
file_id = ""
filename = ""
container_id = ""

# Get the last message which should contain file citations
last_message = response.output[-1]  # ResponseOutputMessage
if (
    last_message.type == "message"
    and last_message.content
    and last_message.content[-1].type == "output_text"
    and last_message.content[-1].annotations
):
    file_citation = last_message.content[-1].annotations[-1]  # AnnotationContainerFileCitation
    if file_citation.type == "container_file_citation":
        file_id = file_citation.file_id
        filename = file_citation.filename
        container_id = file_citation.container_id
        print(f"Found generated file: {filename} (ID: {file_id})")

# Clean up resources
project.agents.delete_version(agent_name=agent.name, agent_version=agent.version)

# Download the generated file if available
if file_id and filename:
    file_content = openai.containers.files.content.retrieve(file_id=file_id, container_id=container_id)
    print(f"File ready for download: {filename}")
    file_path = os.path.join(os.path.dirname(__file__), filename)
    with open(file_path, "wb") as f:
        f.write(file_content.read())
    print(f"File downloaded successfully: {file_path}")
else:
    print("No file generated in response")
```

**Expected output:**

```Console
Found generated file: transportation_operating_profit_bar_chart.png (ID: file-xxxxxxxxxxxxxxxxxxxx)
File ready for download: transportation_operating_profit_bar_chart.png
File downloaded successfully: transportation_operating_profit_bar_chart.png
```

The agent uploads your CSV file to Azure storage, creates a sandboxed Python environment, analyzes the data to filter transportation sector records, generates a PNG bar chart showing operating profit by quarter, and downloads the chart to your local directory. The file annotations in the response provide the file ID and container information needed to retrieve the generated chart.

**Set up MCP integration with agents in AI Foundry:**

```python
"""
Shows how to connect an MCP server as a tool source. The sample code connects to GitHub MCP Server.
"""
from azure.ai.projects.models import PromptAgentDefinition, MCPTool

# [START tool_declaration]
tool = MCPTool(
    server_label="api-specs",
    server_url="https://api.githubcopilot.com/mcp",
    require_approval="always",
    project_connection_id=MCP_CONNECTION_NAME,
)
# [END tool_declaration]

# Create a prompt agent with MCP tool capabilities
agent = project.agents.create_version(
    agent_name="MyAgent7",
    definition=PromptAgentDefinition(
        model="gpt-5-mini",
        instructions="Use MCP tools as needed",
        tools=[tool],
    ),
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

**Set up a multi-turn agent workflow in Bedrock Agent Runtime with Python SDK (Boto3):**

```python
"""
Shows how to run a Bedrock flow with InvokeFlow and handle muli-turn interaction for a single conversation.
"""
import logging
import boto3
import botocore

import botocore.exceptions

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def invoke_flow(client, flow_id, flow_alias_id, input_data, execution_id):
    """
    Invoke an Amazon Bedrock flow and handle the response stream.

    Args:
        client: Boto3 client for Amazon Bedrock agent runtime.
        flow_id: The ID of the flow to invoke.
        flow_alias_id: The alias ID of the flow.
        input_data: Input data for the flow.
        execution_id: Execution ID for continuing a flow. Use the value None on first run.

    Returns:
        Dict containing flow_complete status, input_required info, and execution_id
    """

    response = None
    request_params = None

    if execution_id is None:
        # Don't pass execution ID for first run.
        request_params = {
            "flowIdentifier": flow_id,
            "flowAliasIdentifier": flow_alias_id,
            "inputs": [input_data],
            "enableTrace": True
        }
    else:
        request_params = {
            "flowIdentifier": flow_id,
            "flowAliasIdentifier": flow_alias_id,
            "executionId": execution_id,
            "inputs": [input_data],
            "enableTrace": True
        }

    response = client.invoke_flow(**request_params)

    if "executionId" not in request_params:
        execution_id = response['executionId']

    input_required = None
    flow_status = ""

    # Process the streaming response
    for event in response['responseStream']:

        # Check if flow is complete.
        if 'flowCompletionEvent' in event:
            flow_status = event['flowCompletionEvent']['completionReason']

        # Check if more input us needed from user.
        elif 'flowMultiTurnInputRequestEvent' in event:
            input_required = event

        # Print the model output.
        elif 'flowOutputEvent' in event:
            print(event['flowOutputEvent']['content']['document'])

        # Log trace events.
        elif 'flowTraceEvent' in event:
            logger.info("Flow trace:  %s", event['flowTraceEvent'])

    return {
        "flow_status": flow_status,
        "input_required": input_required,
        "execution_id": execution_id
    }


def converse_with_flow(bedrock_agent_client, flow_id, flow_alias_id):
    """
    Run a conversation with the supplied flow.

    Args:
        bedrock_agent_client: Boto3 client for Amazon Bedrock agent runtime.
        flow_id: The ID of the flow to run.
        flow_alias_id: The alias ID of the flow.

    """

    flow_execution_id = None
    finished = False

    # Get the intial prompt from the user.
    user_input = input("Enter input: ")

    # Use prompt to create input data.
    flow_input_data = {
        "content": {
            "document": user_input
        },
        "nodeName": "FlowInputNode",
        "nodeOutputName": "document"
    }

    try:
        while not finished:
            # Invoke the flow until successfully finished.

            result = invoke_flow(
                bedrock_agent_client, flow_id, flow_alias_id, flow_input_data, flow_execution_id)

            status = result['flow_status']
            flow_execution_id = result['execution_id']
            more_input = result['input_required']
            if status == "INPUT_REQUIRED":
                # The flow needs more information from the user.
                logger.info("The flow %s requires more input", flow_id)
                user_input = input(
                    more_input['flowMultiTurnInputRequestEvent']['content']['document'] + ": ")
                flow_input_data = {
                    "content": {
                        "document": user_input
                    },
                    "nodeName": more_input['flowMultiTurnInputRequestEvent']['nodeName'],
                    "nodeInputName": "agentInputText"

                }
            elif status == "SUCCESS":
                # The flow completed successfully.
                finished = True
                logger.info("The flow %s successfully completed.", flow_id)

    except botocore.exceptions.ClientError as e:
        print(f"Client error: {str(e)}")
        logger.error("Client error: %s", {str(e)})

    except Exception as e:
        print(f"An error occurred: {str(e)}")
        logger.error("An error occurred: %s", {str(e)})
        logger.error("Error type: %s", {type(e)})


def main():
    """
    Main entry point for the script.
    """

    # Replace these with your actual flow ID and flow alias ID.
    FLOW_ID = 'YOUR_FLOW_ID'
    FLOW_ALIAS_ID = 'YOUR_FLOW_ALIAS_ID'

    logger.info("Starting conversation with FLOW: %s ID: %s",
                FLOW_ID, FLOW_ALIAS_ID)

    # Get the Bedrock agent runtime client.
    session = boto3.Session(profile_name='default')
    bedrock_agent_client = session.client('bedrock-agent-runtime')

    # Start the conversation.
    converse_with_flow(bedrock_agent_client, FLOW_ID, FLOW_ALIAS_ID)

    logger.info("Conversation with FLOW: %s ID: %s finished",
                FLOW_ID, FLOW_ALIAS_ID)


if __name__ == "__main__":
    main()
```

**Choose Bedrock when:** You are AWS-native, need multi-model flexibility, want managed guardrails applied infrastructure-wide, or are targeting FedRAMP/HIPAA workloads.

---

### OpenAI Platform

OpenAI's managed platform is the most capable API surface for GPT-4 and GPT-5 series models, with a growing Responses API that supports built-in tools and persistent agent state.

**What it provides:**

- **Models**: GPT-4, GPT-5 series — the full OpenAI frontier model lineup at lowest latency
- **Responses API**: Stateful agent runs with streaming, tool calling, and built-in tools (web search, file search, code interpreter, computer use)
- **Assistants**: Persistent agent configuration with threads, file attachments, and vector stores
- **MCP servers**: The Responses API connects to remote MCP servers directly
- **Governance**: Project-level API key scoping, spend limits, usage dashboards, SOC2 Type II

**Configure agent with File search in Responses API:**

```python
from openai import OpenAI
client = OpenAI()

response = client.responses.create(
    model="gpt-4.1",
    input="What is deep research by OpenAI?",
    tools=[{
        "type": "file_search",
        "vector_store_ids": ["<vector_store_id>"]
    }]
)
print(response)
```

**Configure agent with MCP connection in Responses API:**

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4.1",
    tools=[{
        "type": "mcp",
        "server_label": "shopify",
        "server_url": "https://pitchskin.com/api/mcp",
    }],
    input="Add the Blemish Toner Pads to my cart"
)

print(response.output_text)
```

**Choose OpenAI Platform when:** You need the highest-capability GPT models, want the simplest integration path, or are building on top of ChatGPT's ecosystem.

---

### Microsoft Copilot Studio

Copilot Studio is the low-code/pro-code runtime for buiding enterprise agents on Microsoft's infrastructure — without managing any of the underlying platform.

**What it provides:**

- **Visual agent builder**: Drag-and-drop topics, actions, and branching — no SDK required for common patterns
- **Model selection**: GPT models via Azure OpenAI, with content safety and responsible AI controls applied by default
- **Connectors**: 1,000+ Power Platform connectors to Microsoft 365, Dynamics, Salesforce, ServiceNow, and custom APIs
- **MCP support**: Declare MCP tools as agent actions — Copilot Studio handles tool calling protocol
- **Deployment**: Publish to Teams, SharePoint, web chat, or as a standalone app with one click
- **Governance**: Tenant-level DLP policies, admin center oversight, audit logs, data residency in your Microsoft tenant

**Pro-code extension — Set up custom MCP tool with Azure Functions MCP Extension:**

```python
"""
Implementation pattern for custom logic in Copilot Studio exposed as a **remote MCP server** using the Azure Functions MCP Extension. The Function App becomes the MCP server — Copilot Studio connects to it directly as a tool source, no intermediate flow or adapter needed.
"""
import json
import azure.functions as func
from typing import Any, Dict

app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)

@app.mcp_tool(description="Returns the current health status of a named service")
@app.mcp_tool_property(
    arg_name="service_name",
    description="Name of the service to check (e.g. payments-api, auth-service)"
)
def get_service_health(service_name: str) -> str:
    status = check_health(service_name)
    return json.dumps({
        "service": service_name,
        "state": status["state"],
        "last_checked": status["timestamp"]
    })

def check_health(service_name: str) -> dict:
    # Replace with your actual logic (Azure Monitor, DB, internal API, etc.)
    return {"state": "healthy", "timestamp": "2025-01-01T00:00:00Z"}
```

**Configure the MCP server in `host.json`:**

```json
{
  "version": "2.0",
  "extensions": {
    "mcp": {
      "serverName": "ops-tools",
      "serverVersion": "1.0.0",
      "instructions": "Use these tools to query service health and operational status."
    }
  }
}
```

**Register in Copilot Studio:**

```
Copilot Studio → Tools → Add tool → New tool → Model Context Protocol
  Server URL:  https://<function-app>.azurewebsites.net/runtime/webhooks/mcp/
  Auth:        API key  (x-functions-key header)
               — or —
               OAuth 2.0  (Authorization Code flow, Entra ID)
```

Once registered, every `@app.mcp_tool` function in your Function App appears as a callable tool in the Copilot Studio agent — no custom connector, no Power Automate flow, and no separate HTTP wrapper required.

**Choose Copilot Studio when:** Business users need to author or maintain conversational logic, you're deploying within Microsoft 365 or Teams, or you need governance enforced by IT at the tenant level — while your custom tools are exposed as a governed MCP server on Azure Functions.

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

Before you write a single line of agent logic, you make a decision that constrains everything else: which platform runs your agent. That platform — your runtime — determines which models you can use, how tools connect, where data flows, who can govern it, and what you pay.

Get the runtime decision right and everything else builds cleanly. Get it wrong and you're re-architecting six months in.

**The runtime is not a deployment detail. It defines your model access, tool connectivity, governance posture, and the audience you can realistically serve.** Pick the platform that matches your target outcome — then build the agent to fit.
