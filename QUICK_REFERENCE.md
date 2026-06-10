# Guardrails API Quick Reference

## PII Types & Strategies

### Email Redaction
```python
PIIMiddleware("email", strategy="redact", apply_to_input=True)
# abc@gmail.com → [REDACTED_EMAIL]
```

### Credit Card Masking
```python
PIIMiddleware("credit_card", strategy="mask", apply_to_input=True)
# 4242-4242-4242-4242 → 4242-****-****-****
```

### API Key Blocking
```python
PIIMiddleware(
    "api_key",
    detector=r"sk-[a-zA-Z0-9]{32,}",
    strategy="block",
    apply_to_input=True
)
# Raises exception if detected ❌
```

### Other PII Types
- `phone`: Phone numbers
- `ssn`: Social Security Numbers
- `ip`: IP addresses
- `mac_address`: MAC addresses
- `url`: URLs

---

## Guardrail Methods

### 1. Deterministic Guardrail
```python
def deterministic_guardrail(text: str) -> bool:
    banned_keywords = ["hack", "exploit", "malware", "bomb", "attack", "phishing"]
    return any(kw in text.lower() for kw in banned_keywords)
```
**Returns**: `True` (blocked), `False` (allowed)

### 2. Model-Based Guardrail
```python
def model_based_guardrail(text: str) -> str:
    prompt = f"Is this safe? {text}"
    result = model.invoke([{"role": "user", "content": prompt}])
    return result.content.strip()
```
**Returns**: "safe" or "unsafe"

### 3. PII Middleware
```python
PIIMiddleware(
    pii_type: str,
    strategy: "redact" | "mask" | "block",
    apply_to_input: bool = True,
    apply_to_output: bool = False,
    detector: str | None = None  # Custom regex
)
```

### 4. Human-in-the-Loop Middleware
```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "tool_name": bool,  # True = require approval
    }
)
```

### 5. Custom Hook Middleware
```python
class CustomMiddleware(AgentMiddleware):
    @hook_config(can_jump_to=["end"])
    def before_agent(self, state: AgentState, runtime: Runtime):
        # Return None to continue
        # Return dict to modify state or jump
        return None
```

---

## Common Code Snippets

### Create Agent with Multiple Guardrails
```python
agent = create_agent(
    model=ChatGroq(model="llama-3.3-70b-versatile", temperature=0),
    tools=[tool1, tool2],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
        ContentFilterMiddleware(banned_keywords=[...]),
        HumanInTheLoopMiddleware(interrupt_on={"sensitive_tool": True}),
    ],
    checkpointer=InMemorySaver(),
)
```

### Invoke Agent with HITL
```python
# Step 1: Initial invocation (pauses)
config = {"configurable": {"thread_id": "session_001"}}
result = agent.invoke({"messages": [...]}, config=config)

# Step 2: Approve
approved = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config
)

# OR: Reject
rejected = agent.invoke(
    Command(resume={"decisions": [{"type": "reject", "reason": "..."}]}),
    config=config
)
```

### Invoke Agent with Before-Agent Hook
```python
result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "Safe or unsafe input"
    }]
})
# Hook filters automatically before processing
```

---

## Middleware Stack Execution Order

```
Middleware added FIRST executes LAST
Middleware added LAST executes FIRST

middleware=[
    Middleware1(),  # 3rd execution
    Middleware2(),  # 2nd execution
    Middleware3(),  # 1st execution
]
```

---

## Decision Types for HITL

### Approve
```python
Command(resume={"decisions": [{"type": "approve"}]})
```

### Reject
```python
Command(resume={"decisions": [{"type": "reject", "reason": "Too risky"}]})
```

### Modify
```python
Command(resume={"decisions": [{"type": "modify", "new_params": {...}}]})
```

---

## Hook Decorators & Config

### Before-Agent Hook
```python
@hook_config(can_jump_to=["end"])
def before_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
    if should_block:
        return {
            "messages": [...],
            "jump_to": "end"
        }
    return None  # Continue
```

### Available Jump Targets
- `"end"`: Skip to final response
- `"agent"`: Continue to agent processing
- `"tools"`: Execute tools
- Custom node names in graph

---

## Key Imports

```python
# Core
from langchain.agents import create_agent
from langchain_groq import ChatGroq
from langchain_core.tools import tool

# Middleware
from langchain.agents.middleware import (
    PIIMiddleware,
    HumanInTheLoopMiddleware,
    AgentMiddleware,
    hook_config,
    AgentState,
)

# State Management
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command
from langgraph.runtime import Runtime

# Tools
from langchain_core.tools import tool

# Other
from typing import Any
```

---

## PII Strategy Comparison

| Strategy | Input | Output | Exception | Use Case |
|----------|-------|--------|-----------|----------|
| **redact** | abc@email.com | [REDACTED_EMAIL] | No | Hide completely |
| **mask** | 4242-4242-4242-4242 | 4242-****-****-**** | No | Partial visibility |
| **block** | sk-abcd1234... | - | YES ❌ | Critical data |

---

## Common Errors & Solutions

### Error: "No checkpointer provided"
**Problem**: HITL without state persistence
**Solution**: 
```python
checkpointer=InMemorySaver()  # Add this parameter
```

### Error: "Thread ID not found"
**Problem**: Using different thread_id in resume
**Solution**: Use same `thread_id` in config:
```python
config = {"configurable": {"thread_id": "same_id"}}
# Use this config in both invoke calls
```

### Error: "Middleware not applying"
**Problem**: Incorrect parameter names
**Solution**: Check exact parameter names:
- `apply_to_input` ✅
- `apply_to_output` ✅
- `apply_to_agent_input` ❌

### PII Not Detected
**Problem**: Incorrect regex pattern
**Solution**: Test regex pattern:
```python
import re
pattern = r"sk-[a-zA-Z0-9]{32,}"
text = "sk-1234567890abcdef1234567890abcdef"
if re.search(pattern, text):
    print("Match found!")
```

---

## Performance Tips

1. **Order**: Deterministic filters → PII → LLM-based
2. **Cost**: Block early, avoid expensive API calls
3. **Latency**: Use in-memory checkpointer for HITL
4. **Parallelization**: Run independent checks in parallel

---

## Configuration Examples

### Strict Security
```python
middleware=[
    ContentFilterMiddleware(banned_keywords=[...]),
    PIIMiddleware("email", strategy="redact", apply_to_input=True),
    PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
    PIIMiddleware("api_key", strategy="block", apply_to_input=True),
    HumanInTheLoopMiddleware(interrupt_on={"delete": True, "modify": True}),
]
```

### Balanced Security
```python
middleware=[
    ContentFilterMiddleware(banned_keywords=[...]),
    PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
    HumanInTheLoopMiddleware(interrupt_on={"critical_tool": True}),
]
```

### Permissive
```python
middleware=[
    ContentFilterMiddleware(banned_keywords=["bomb", "attack"]),
    PIIMiddleware("email", strategy="redact", apply_to_input=True),
]
```

---

## Tools Definition

### Basic Tool
```python
@tool
def search_web(query: str) -> str:
    """Search the web for information."""
    return f"Search results for: {query}"
```

### Tool with Multiple Parameters
```python
@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email to a recipient."""
    return f"Email sent to {to} with subject: {subject}"
```

---

## Testing Your Guardrails

```python
# Safe input
result = agent.invoke({
    "messages": [{"role": "user", "content": "What is AI?"}]
})
print("✅ Safe:", result["messages"][-1].content)

# Blocked input
result = agent.invoke({
    "messages": [{"role": "user", "content": "How to hack?"}]
})
print("🚫 Blocked:", result["messages"][-1].content)

# PII input
result = agent.invoke({
    "messages": [{"role": "user", "content": "My email is test@example.com"}]
})
print("🔒 PII Protected:", result["messages"][-1].content)
```

---

## Troubleshooting Checklist

- [ ] Imports are correct
- [ ] API keys configured in `.env`
- [ ] Checkpointer added for HITL
- [ ] Thread IDs are consistent
- [ ] Middleware order is optimized
- [ ] PII detector regex is valid
- [ ] Tools are properly decorated with `@tool`
- [ ] Model is initialized with correct parameters
- [ ] State is persisted between invocations

---

## Resources

- LangChain Docs: https://python.langchain.com/
- Groq API: https://console.groq.com/
- LangGraph: https://langchain-ai.github.io/langgraph/
