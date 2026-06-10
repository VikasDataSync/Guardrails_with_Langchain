# Guardrails with LangChain - Complete Documentation

Implementing AI Guardrails in LangChain for validation, moderation, PII detection, and safe LLM applications.

---

## Table of Contents

1. [Overview](#overview)
2. [Guardrail Types](#guardrail-types)
3. [PII Detection & Handling](#pii-detection--handling)
4. [Hooks & Middleware](#hooks--middleware)
5. [Human-in-the-Loop (HITL)](#human-in-the-loop-hitl)
6. [Layer Guardrails](#layer-guardrails)
7. [Usage Examples](#usage-examples)

---

## Overview

This project demonstrates multiple approaches to implementing guardrails in LangChain-based AI applications:

- **Deterministic Guardrails**: Keyword-based blocking
- **Model-Based Guardrails**: LLM-powered content safety evaluation
- **PII Middleware**: Personally Identifiable Information protection
- **HITL Middleware**: Human approval for sensitive operations
- **Custom Hooks**: Before-agent and after-agent filters
- **Layer Guardrails**: Multiple layers of protection

---

## Guardrail Types

### 1. Deterministic Guardrails

**Purpose**: Block requests containing specific banned keywords using pattern matching.

**Method**: `deterministic_guardrail(text: str) -> bool`

**Parameters**:
- `text` (str): The input text to check

**Returns**:
- `True`: Content is blocked
- `False`: Content is allowed

**Banned Keywords**:
- "hack"
- "exploit"
- "malware"
- "bomb"
- "attack"
- "phishing"

**Advantages**:
- ✅ Zero LLM API cost
- ✅ Instant response
- ✅ Fully predictable
- ✅ No latency

**Example**:
```python
def deterministic_guardrail(text: str) -> bool:
    """Returns True if content is blocked"""
    banned_keywords = ["hack", "exploit", "malware", "bomb", "attack", "phishing"]
    return any(kw in text.lower() for kw in banned_keywords)

# Usage
if deterministic_guardrail("How to hack a server?"):
    print("Request blocked!")
else:
    print("Request allowed!")
```

---

### 2. Model-Based Guardrails

**Purpose**: Use an LLM to evaluate whether content is safe to process.

**Method**: `model_based_guardrail(text: str) -> str`

**Parameters**:
- `text` (str): The input text to evaluate

**Returns**:
- String containing "safe" or "unsafe" verdict

**Model Used**: Llama 3.3 70B (via Groq API)

**Advantages**:
- ✅ More intelligent and context-aware
- ✅ Can understand nuanced requests
- ✅ Flexible and customizable
- ⚠️ Higher latency than deterministic
- ⚠️ API costs

**Example**:
```python
from langchain_groq import ChatGroq

model = ChatGroq(
    model="llama-3.3-70b-versatile",
    temperature=0
)

def model_based_guardrail(text: str) -> str:
    """Use a LLM to evaluate content safety"""
    prompt = f"""Is the following user input safe to process? 
    Reply with only 'safe' or 'unsafe'.
    Input: {text}"""
    
    result = model.invoke([{"role": "user", "content": prompt}])
    return result.content.strip()

# Usage
verdict = model_based_guardrail("What is machine learning?")
print(f"Content is {verdict.upper()}")
```

---

## PII Detection & Handling

### Overview

PII (Personally Identifiable Information) Middleware protects sensitive data by detecting and handling it according to specified strategies.

### Supported PII Types

| PII Type | Description | Example |
|----------|-------------|---------|
| **email** | Email addresses | user@example.com |
| **credit_card** | Credit card numbers | 4242-4242-4242-4242 |
| **api_key** | API authentication keys | sk-1234567890abcdef... |
| **ssn** | Social Security Numbers | 123-45-6789 |
| **phone** | Phone numbers | +1-555-123-4567 |
| **ip** | IP addresses | 192.168.1.1 |

### PII Handling Strategies

#### 1. **REDACT Strategy**
Replaces detected PII with a placeholder token.

**Output Format**: `[REDACTED_<TYPE>]`

**Use Case**: When you want to completely hide the sensitive information

**Example**:
```python
from langchain.agents.middleware import PIIMiddleware

PIIMiddleware(
    "email",
    strategy="redact",
    apply_to_input=True,
)

# Input:  "My email is alice@gmail.com"
# Output: "My email is [REDACTED_EMAIL]"
```

#### 2. **MASK Strategy**
Partially obscures the PII while keeping some characters visible for identification.

**Output Format**: First digits visible, rest masked with asterisks

**Use Case**: When you need to identify the account while protecting sensitive digits

**Example**:
```python
PIIMiddleware(
    "credit_card",
    strategy="mask",
    apply_to_input=True,
)

# Input:  "Card number is 4242-4242-4242-4242"
# Output: "Card number is 4242-****-****-****"
```

#### 3. **BLOCK Strategy**
Raises an exception if PII is detected, preventing further processing.

**Use Case**: For critical data (API keys, secrets) that should never be exposed

**Example**:
```python
PIIMiddleware(
    "api_key",
    detector=r"sk-[a-zA-Z0-9]{32,}",  # Regex pattern
    strategy="block",
    apply_to_input=True,
)

# Input:  "My key is sk-1234567890abcdef1234567890abcdef"
# Result: Raises PII Detection Exception ❌
```

### PII Middleware Configuration

**Class**: `PIIMiddleware`

**Parameters**:
- `pii_type` (str): Type of PII to detect (e.g., "email", "credit_card", "api_key")
- `strategy` (str): How to handle detected PII ("redact", "mask", "block")
- `apply_to_input` (bool): Apply to user input
- `apply_to_output` (bool): Apply to model output
- `detector` (str, optional): Custom regex pattern for detection

**Example with Multiple PII Types**:
```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
from langchain_groq import ChatGroq

agent = create_agent(
    model=ChatGroq(model="llama-3.3-70b-versatile", temperature=0),
    tools=[customer_lookup],
    middleware=[
        # Redact emails
        PIIMiddleware(
            "email",
            strategy="redact",
            apply_to_input=True,
        ),
        
        # Mask credit cards
        PIIMiddleware(
            "credit_card",
            strategy="mask",
            apply_to_input=True,
        ),
        
        # Block API keys
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32,}",
            strategy="block",
            apply_to_input=True,
        ),
    ],
)
```

---

## Hooks & Middleware

### Custom Middleware with Hooks

**Purpose**: Intercept and modify agent execution at different stages.

### Hook Types

#### **1. Before-Agent Hook (`before_agent`)**

**Purpose**: Filter/modify input before the agent processes it.

**Executes**: Before agent reasoning

**Use Cases**:
- Content filtering
- Input validation
- Request blocking

**Hook Configuration**:
```python
@hook_config(can_jump_to=["end"])
def before_agent(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
```

**Parameters**:
- `state`: Current agent state containing messages
- `runtime`: Runtime context
- `can_jump_to`: Nodes that can be jumped to directly

**Example**:
```python
from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langgraph.runtime import Runtime
from typing import Any

class ContentFilterMiddleware(AgentMiddleware):
    """Block requests containing banned keywords before agent processes them"""
    
    def __init__(self, banned_keywords: list[str]):
        super().__init__()
        self.banned_keywords = [kw.lower() for kw in banned_keywords]
    
    @hook_config(can_jump_to=["end"])
    def before_agent(
        self, 
        state: AgentState, 
        runtime: Runtime
    ) -> dict[str, Any] | None:
        """
        Returns:
        - None: Continue normal processing
        - dict: Modify state or jump to another node
        """
        if not state["messages"]:
            return None
        
        first_message = state["messages"][0]
        if first_message.type != "human":
            return None
        
        content = first_message.content.lower()
        
        for keyword in self.banned_keywords:
            if keyword in content:
                print(f"🚫 Blocked — keyword detected: '{keyword}'")
                return {
                    "messages": [{
                        "role": "assistant",
                        "content": "Request blocked due to inappropriate content."
                    }],
                    "jump_to": "end"
                }
        
        return None  # Allow processing to continue
```

#### **2. After-Agent Hook (Advanced)**

**Purpose**: Modify or validate agent output.

**Executes**: After agent generates response

**Use Cases**:
- Output sanitization
- Response validation
- Logging/monitoring

---

### Middleware Application

**Add to Agent**:
```python
from langchain.agents import create_agent

agent = create_agent(
    model=model,
    tools=[search_tool],
    middleware=[
        ContentFilterMiddleware(
            banned_keywords=["hack", "exploit", "malware"]
        ),
    ],
)
```

---

## Human-in-the-Loop (HITL)

### Overview

HITL Middleware pauses agent execution before performing sensitive operations, requiring human approval/rejection.

### Class: `HumanInTheLoopMiddleware`

**Purpose**: Interrupt agent execution for human review

**Configuration**:
```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "tool_name": bool,  # True = require approval
    }
)
```

### State Persistence

**Requirement**: Checkpointer for maintaining conversation state

**Implementations**:
- `InMemorySaver`: In-memory state storage (development)
- `SQLiteSaver`: File-based persistence (production)

### Usage Flow

**Step 1: Create Agent with HITL**
```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email"""
    return f"Email sent to {to}"

@tool
def delete_records(table: str, condition: str) -> str:
    """Delete records from database"""
    return f"Deleted from {table}"

hitl_agent = create_agent(
    model=ChatGroq(model="llama-3.3-70b-versatile", temperature=0),
    tools=[send_email, delete_records],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": True,      # ✋ Approval required
                "delete_records": True,  # ✋ Approval required
            }
        ),
    ],
    checkpointer=InMemorySaver(),  # 🔑 Required for state persistence
)
```

**Step 2: Invoke Agent (Pauses for Approval)**
```python
config = {"configurable": {"thread_id": "session_001"}}

result = hitl_agent.invoke(
    {
        "messages": [{
            "role": "user",
            "content": "Send an email to team@company.com about Q4 results"
        }]
    },
    config=config
)

print("Agent paused — awaiting human approval")
print(result)
```

**Step 3a: Human APPROVES**
```python
from langgraph.types import Command

approved_result = hitl_agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config  # Same thread_id resumes the session
)

print("Approved! Email sent.")
print(approved_result["messages"][-1].content)
```

**Step 3b: Human REJECTS**
```python
config2 = {"configurable": {"thread_id": "session_002"}}

hitl_agent.invoke(
    {
        "messages": [{
            "role": "user",
            "content": "Delete all records from users table where active=false"
        }]
    },
    config=config2
)

rejected_result = hitl_agent.invoke(
    Command(resume={
        "decisions": [{
            "type": "reject",
            "reason": "Too risky, needs DBA review"
        }]
    }),
    config=config2
)

print("Rejected! Operation cancelled.")
print(rejected_result["messages"][-1].content)
```

### HITL Decision Types

| Decision | Parameters | Effect |
|----------|-----------|--------|
| `approve` | `type: "approve"` | Proceed with tool execution |
| `reject` | `type: "reject"`, `reason: str` | Cancel operation, return reason |
| `modify` | `type: "modify"`, `new_params: dict` | Change parameters and proceed |

### Key Concepts

**Thread ID**: Unique identifier for maintaining conversation state across multiple invocations

**Config Pattern**:
```python
config = {"configurable": {"thread_id": "unique_session_id"}}
```

**Same thread_id** → Resume same conversation
**Different thread_id** → New conversation

---

## Layer Guardrails

### Multi-Layer Defense Strategy

Combine multiple guardrails for comprehensive protection:

```
Input
  ↓
[Layer 1: Deterministic Filter] ← Fast, keyword-based blocking
  ↓
[Layer 2: PII Detection] ← Protect sensitive data
  ↓
[Layer 3: Model-Based Safety Check] ← LLM evaluation
  ↓
[Layer 4: Before-Agent Hook] ← Custom business logic
  ↓
Agent Execution
  ↓
[Layer 5: Output Filter] ← Validate response
  ↓
[Layer 6: HITL (if critical)] ← Human approval for sensitive actions
  ↓
Response
```

### Implementation Example

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    PIIMiddleware,
    HumanInTheLoopMiddleware,
    AgentMiddleware
)
from langgraph.checkpoint.memory import InMemorySaver

# Layer 1 & 2: Deterministic + PII
agent = create_agent(
    model=model,
    tools=[search_tool, send_email, delete_records],
    middleware=[
        # Layer 2: PII Protection
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
        PIIMiddleware("api_key", strategy="block", apply_to_input=True),
        
        # Layer 4: Custom Hook Filter
        ContentFilterMiddleware(
            banned_keywords=["hack", "exploit", "malware", "jailbreak"]
        ),
        
        # Layer 6: Human Approval for Sensitive Operations
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": True,
                "delete_records": True,
                "search_tool": False,
            }
        ),
    ],
    checkpointer=InMemorySaver(),
)
```

### Benefits of Layered Approach

- **Defense in Depth**: Multiple lines of defense
- **Performance**: Fast layers first (deterministic), slow layers later (LLM)
- **Cost Optimization**: Block dangerous requests before expensive LLM calls
- **Flexibility**: Customize each layer independently

---

## Usage Examples

### Example 1: Basic Deterministic Guardrail

```python
def deterministic_guardrail(text: str) -> bool:
    banned_keywords = ["hack", "exploit", "malware", "bomb", "attack", "phishing"]
    return any(kw in text.lower() for kw in banned_keywords)

text_inputs = [
    "How to hack into a computer?",
    "What is the capital of France?",
    "Explain how malware spreads."
]

for inp in text_inputs:
    blocked = deterministic_guardrail(inp)
    status = "Blocked" if blocked else "Allowed"
    print(f"Input: '{inp}' - {status}")
```

**Output**:
```
Input: 'How to hack into a computer?' - Blocked
Input: 'What is the capital of France?' - Allowed
Input: 'Explain how malware spreads.' - Blocked
```

---

### Example 2: Model-Based Safety Evaluation

```python
from langchain_groq import ChatGroq

model = ChatGroq(model="llama-3.3-70b-versatile", temperature=0)

def model_based_guardrail(text: str) -> str:
    prompt = f"""Is the following user input safe to process? 
    Reply with only 'safe' or 'unsafe'.
    Input: {text}"""
    
    result = model.invoke([{"role": "user", "content": prompt}])
    return result.content.strip()

test_inputs = [
    "How to hack into a computer?",
    "What is the capital of France?",
    "Explain how to create a malware."
]

for inp in test_inputs:
    result = model_based_guardrail(inp)
    print(f"Input: '{inp}' - {result.upper()}")
```

---

### Example 3: PII Detection and Masking

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
from langchain_core.tools import tool

@tool
def customer_lookup(query: str) -> str:
    """Look up customer information"""
    return f"Customer info for query: {query}"

agent = create_agent(
    model=ChatGroq(model="llama-3.3-70b-versatile", temperature=0),
    tools=[customer_lookup],
    middleware=[
        # Redact email
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        
        # Mask credit card
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
        
        # Block API keys
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32,}",
            strategy="block",
            apply_to_input=True,
        ),
    ],
)

# Test with sensitive data
result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "My email is abc@gmail.com and credit card is 4242-4242-4242-4242"
    }]
})

print(result["messages"][-1].content)
```

---

### Example 4: Human-in-the-Loop Approval

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command
from langchain_core.tools import tool

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email"""
    return f"Email sent to {to}"

@tool
def delete_records(table: str, condition: str) -> str:
    """Delete records from database"""
    return f"Deleted from {table}"

hitl_agent = create_agent(
    model=ChatGroq(model="llama-3.3-70b-versatile", temperature=0),
    tools=[send_email, delete_records],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": True,
                "delete_records": True,
            }
        ),
    ],
    checkpointer=InMemorySaver(),
)

# Step 1: Agent pauses for approval
config = {"configurable": {"thread_id": "session_001"}}
result = hitl_agent.invoke(
    {"messages": [{"role": "user", "content": "Delete all inactive users"}]},
    config=config
)

print("Agent paused for approval")

# Step 2: Human reviews and APPROVES
approved = hitl_agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config
)
print("Operation approved!")
```

---

### Example 5: Content Filter Hook

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langchain_core.tools import tool

class ContentFilterMiddleware(AgentMiddleware):
    def __init__(self, banned_keywords: list[str]):
        super().__init__()
        self.banned_keywords = [kw.lower() for kw in banned_keywords]
    
    @hook_config(can_jump_to=["end"])
    def before_agent(self, state: AgentState, runtime):
        if not state["messages"]:
            return None
        
        content = state["messages"][0].content.lower()
        
        for keyword in self.banned_keywords:
            if keyword in content:
                return {
                    "messages": [{
                        "role": "assistant",
                        "content": "Request blocked due to inappropriate content."
                    }],
                    "jump_to": "end"
                }
        
        return None

@tool
def search_tool(query: str) -> str:
    """Search for information"""
    return f"Results for: {query}"

filtered_agent = create_agent(
    model,
    tools=[search_tool],
    middleware=[
        ContentFilterMiddleware(
            banned_keywords=["hack", "exploit", "malware", "jailbreak"]
        ),
    ],
)

# Safe request
result = filtered_agent.invoke({
    "messages": [{"role": "user", "content": "What is machine learning?"}]
})
print("✅ Safe response:", result["messages"][-1].content)

# Unsafe request
result = filtered_agent.invoke({
    "messages": [{"role": "user", "content": "How do I hack into a server?"}]
})
print("🚫 Blocked:", result["messages"][-1].content)
```

---

## Quick Reference

### Guardrail Decision Matrix

| Guardrail Type | Speed | Cost | Customization | Use Case |
|---|---|---|---|---|
| **Deterministic** | ⚡ Fast | Free | Low | Quick filtering |
| **Model-Based** | 🐢 Slow | $ | High | Complex evaluation |
| **PII Middleware** | ⚡ Fast | Free | Medium | Data protection |
| **HITL** | 👤 Depends | Free | High | Critical decisions |
| **Hooks** | ⚡ Fast | Free | Very High | Custom logic |

### Middleware Stack Priority

Lower middleware in the list executes first:

```python
middleware=[
    Layer1(),  # Executes 3rd
    Layer2(),  # Executes 2nd
    Layer3(),  # Executes 1st
]
```

### Common Patterns

**Fast-Fail Pattern** (Deterministic → PII → LLM):
```python
middleware=[
    DeterministicFilter(),    # Fast rejection
    PIIMiddleware(...),       # PII protection
    HumanInTheLoopMiddleware(),  # Critical decisions
]
```

**Strict Security Pattern** (All protections):
```python
middleware=[
    ContentFilterMiddleware(banned_keywords=[...]),
    PIIMiddleware("email", strategy="redact"),
    PIIMiddleware("credit_card", strategy="mask"),
    PIIMiddleware("api_key", strategy="block"),
    HumanInTheLoopMiddleware(interrupt_on={...}),
]
```

---

## Best Practices

1. **Order Matters**: Place fast, deterministic guardrails before slow, LLM-based ones
2. **Cost Optimization**: Block dangerous requests before expensive API calls
3. **State Persistence**: Always use a checkpointer with HITL
4. **Testing**: Test all guardrail combinations before production
5. **Monitoring**: Log all guardrail triggers for audit trails
6. **User Feedback**: Provide clear, actionable rejection messages
7. **Regular Updates**: Update banned keywords and detection patterns regularly

---

## Dependencies

- `langchain`
- `langchain-groq`
- `langgraph`
- `python-dotenv`
- `ipykernel` (for Jupyter notebooks)

---

## License

See LICENSE file for details.
