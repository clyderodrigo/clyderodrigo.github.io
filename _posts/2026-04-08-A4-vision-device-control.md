# See and Act: Building Agents with Vision and Interface Control

## Key Takeaways

- Vision turns any UI or document into an API. Stop waiting for one to exist.
- Use `detail: "high"` for text-heavy screenshots, `"low"` for scene classification. The cost difference is 10x.
- Playwright scopes interface control to web content: SPAs, authenticated sessions, JS-rendered apps.
- The Computer Use API generalizes from browser to any rendered surface — desktop apps, ERPs, legacy thick clients. Same loop, broader reach.
- Build retry logic into every interface agent. UI surfaces change; your agent needs to recover.

---

Most enterprise systems were built for humans, not APIs. Dashboards, legacy portals, knowledge bases, forms, and document workflows have no programmatic interface — and no one is building one. Vision and browser control are how agents operate in that world: reading what's on screen, navigating what can't be scraped, and extracting what can't be parsed directly.

This article covers how to integrate visual understanding and browser automation into Claude and GPT agents for real task completion — not demos, not toy scripts.

---

## The Capability Stack

Vision and browser control are separate capabilities that compose into something powerful:

- **Vision**: The model interprets screenshots, images, documents, diagrams
- **Interface control**: The agent navigates, clicks, types, and extracts across web and desktop surfaces — operating any rendered interface programmatically
- **Combined**: The agent sees what's on screen, decides what to do, acts on it, and observes the result

This is the loop that powers computer use agents.

---

## Vision: What the Models Can See

Both GPT-4o and Claude 3+ support image input natively. Pass images as base64 or URL.

### OpenAI Vision

```python
import base64
from openai import OpenAI

client = OpenAI()

def encode_image(path: str) -> str:
    with open(path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/png;base64,{encode_image('dashboard.png')}",
                        "detail": "high"  # "low" for speed/cost, "high" for precision
                    }
                },
                {
                    "type": "text",
                    "text": "Identify all error indicators in this monitoring dashboard. List them with their values."
                }
            ]
        }
    ]
)
```

### Anthropic Vision

```python
import anthropic, base64

client = anthropic.Anthropic()

with open("architecture_diagram.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": image_data
                    }
                },
                {
                    "type": "text",
                    "text": "Map the data flow in this architecture. Identify any single points of failure."
                }
            ]
        }
    ]
)
```

**Image detail levels (OpenAI):**
- `"low"`: 85 tokens per image, fast, suitable for classification and general scene understanding
- `"high"`: 765–1105 tokens per image, required for reading text, fine-grained UI analysis

---

## Document Intelligence with Vision

PDFs, invoices, forms, reports — anything that was designed for human eyes can now be parsed by an agent.

```python
import fitz  # PyMuPDF
import anthropic, base64

client = anthropic.Anthropic()

def pdf_page_to_base64(pdf_path: str, page_num: int = 0) -> str:
    doc = fitz.open(pdf_path)
    page = doc[page_num]
    pix = page.get_pixmap(dpi=150)
    return base64.b64encode(pix.tobytes("png")).decode()

def extract_invoice_data(pdf_path: str) -> dict:
    image_b64 = pdf_page_to_base64(pdf_path)

    response = client.messages.create(
        model="claude-sonnet-4",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {"type": "base64", "media_type": "image/png", "data": image_b64}
                },
                {
                    "type": "text",
                    "text": "Extract: invoice number, date, vendor name, line items, total. Return JSON only."
                }
            ]
        }]
    )
    import json
    return json.loads(response.content[0].text)
```

---

## Browser Control: Playwright Integration

For web-based interfaces — SaaS tools, portals, and JS-rendered apps — Playwright gives your agent a real browser, not a scraper. A full Chromium instance where SPAs, authenticated sessions, and dynamically loaded content all work. For interfaces that live outside the browser, the Computer Use API (below) operates at the display level directly.

### Setup

```bash
pip install playwright
playwright install chromium
```

### Synchronous Browser Agent

```python
from playwright.sync_api import sync_playwright
from openai import OpenAI
import base64

client = OpenAI()

def capture_screenshot(page) -> str:
    screenshot_bytes = page.screenshot(full_page=False)
    return base64.b64encode(screenshot_bytes).decode()

def browser_agent(task: str, start_url: str):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page(viewport={"width": 1280, "height": 800})
        page.goto(start_url)

        for step in range(20):  # max steps guard
            screenshot_b64 = capture_screenshot(page)

            response = client.chat.completions.create(
                model="gpt-4o",
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "You control a browser. At each step, return a JSON action: "
                            '{"action": "click"|"type"|"navigate"|"done", '
                            '"selector": "css selector", "text": "text to type", "url": "url", '
                            '"result": "task result if done"}'
                        )
                    },
                    {
                        "role": "user",
                        "content": [
                            {
                                "type": "image_url",
                                "image_url": {"url": f"data:image/png;base64,{screenshot_b64}"}
                            },
                            {"type": "text", "text": f"Task: {task}\nCurrent URL: {page.url}"}
                        ]
                    }
                ],
                response_format={"type": "json_object"}
            )

            import json
            action = json.loads(response.choices[0].message.content)

            if action["action"] == "done":
                browser.close()
                return action.get("result")

            elif action["action"] == "click":
                page.click(action["selector"])

            elif action["action"] == "type":
                page.fill(action["selector"], action["text"])

            elif action["action"] == "navigate":
                page.goto(action["url"])

            page.wait_for_load_state("networkidle", timeout=5000)

        browser.close()
        return "Max steps reached"
```

---

## Anthropic Computer Use API: Interface Control Beyond the Browser

Playwright is scoped to web content. The Computer Use API operates on pixels — it doesn't care whether the interface is a browser, a desktop ERP, a legacy thick client, or a terminal. Any rendered surface becomes operable.

Three tools compose into full workflow execution:

| Tool | What it does |
|---|---|
| `computer` | Captures a screenshot, then acts: click, type, scroll, key combination. Works on anything visible on screen. |
| `text_editor` | Reads and writes file content directly, without going through the display. |
| `bash` | Executes shell commands — for workflow steps that don't need UI interaction. |

A workflow that logs into a desktop ERP, copies a report to a file, transforms it with a script, and sends a notification uses all three — with no API required.

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "type": "computer_20241022",
        "name": "computer",
        "display_width_px": 1280,
        "display_height_px": 800,
        "display_number": 1
    },
    {
        "type": "text_editor_20241022",
        "name": "str_replace_editor"
    },
    {
        "type": "bash_20241022",
        "name": "bash"
    }
]
```

### The Workflow Execution Loop

The Computer Use API works through a screenshot-act-observe loop: the model sees the screen, calls a tool to act, you execute the action and feed back the result — including a fresh screenshot — and repeat until the task is done. This is the same loop as the Playwright agent above, generalized to any surface.

```python
def run_interface_workflow(task: str, sandbox) -> str:
    """
    sandbox: object with execute(tool_name, tool_input) -> result
    Options: Docker + VNC, xvfb virtual display, or hosted sandbox (E2B, Browserbase)
    """
    messages = [{"role": "user", "content": task}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            return next(b.text for b in response.content if hasattr(b, "text"))

        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue

            raw = sandbox.execute(block.name, block.input)

            # computer actions return a screenshot so the model observes the new state
            if block.name == "computer":
                content = [{"type": "image", "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": raw["screenshot_b64"]
                }}]
            else:
                content = raw.get("output", "")  # text output for bash / text_editor

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": content
            })

        messages.append({"role": "user", "content": tool_results})
```

The sandbox abstracts the execution environment. In development, this is typically a Docker container running a VNC server with `xdotool` for mouse and keyboard input. In production, managed options like E2B or Browserbase provide sandboxes with screenshot and input APIs and eliminate the infrastructure overhead.

---

## Production Considerations

### Screenshot Fidelity vs. Cost

Higher DPI = better accuracy = more tokens = more cost. Find the minimum resolution where the model reliably reads UI elements.

```python
# Start with 96 DPI, increase only if accuracy is insufficient
page.screenshot(path="screenshot.png", scale="device")
```

### Anti-Detection for Web Agents

Many sites block automated browsers. Use realistic browser profiles:

```python
browser = p.chromium.launch(
    headless=False,  # visible browser is harder to detect
    args=["--disable-blink-features=AutomationControlled"]
)
context = browser.new_context(
    user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    viewport={"width": 1280, "height": 800}
)
```

### Reliability Patterns

Interface agents fail. Pages change, elements shift, desktop layouts update, timeouts happen. Build explicit retry and fallback:

```python
def safe_click(page, selector: str, retries: int = 3):
    for attempt in range(retries):
        try:
            page.wait_for_selector(selector, timeout=3000)
            page.click(selector)
            return True
        except Exception as e:
            if attempt == retries - 1:
                raise
            page.reload()
    return False
```

---

## Use Cases Worth Building

| Use Case | Vision | UI Control | Model |
|---|---|---|---|
| Invoice/receipt extraction | ✓ | | Claude Sonnet |
| Dashboard monitoring & alerting | ✓ | ✓ | GPT-4o |
| Form filling automation | | ✓ | GPT-4o-mini |
| Competitor price monitoring | ✓ | ✓ | Claude Haiku |
| Legacy system data extraction | ✓ | ✓ | Claude Sonnet |
| Document classification at scale | ✓ | | Claude Haiku |
| Desktop ERP / CRM data workflow | ✓ | ✓ | Claude Opus |
| Cross-app workflow automation | ✓ | ✓ | Claude Opus |

