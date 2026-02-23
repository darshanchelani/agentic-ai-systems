# 1.6 Tools & Agents: Empowering LLMs to Take Action

## 📌 Overview

**Agents** are the next evolutionary step beyond chains.

While chains follow a fixed pipeline:

```
Input → Step1 → Step2 → Output
```

Agents are dynamic:

```
User Input
   ↓
LLM reasons
   ↓
Chooses Tool
   ↓
Executes Tool
   ↓
Observes Result
   ↓
Repeats until Final Answer
```

They combine:

1. **Tools** → External actions (APIs, DBs, calculators, etc.)
2. **LLM** → Reasoning engine
3. **Prompt** → Behavior instructions
4. **AgentExecutor** → Runtime loop controller

---

# 🎯 Why Agents?

- ✅ Dynamic decision-making
- ✅ Multi-step reasoning
- ✅ Extensible architecture
- ✅ Error recovery
- ✅ Can interact with external systems

---

# 🏗 Agent Architecture

```
                ┌────────────┐
User Input ───▶ │    LLM     │
                └─────┬──────┘
                      │
              Decides which tool
                      │
                ┌─────▼──────┐
                │    Tool    │
                └─────┬──────┘
                      │
                Observation
                      │
                (Back to LLM)
                      │
               Final Answer
```

---

# 1️⃣ Tool Creation

Tools are functions the agent can call.

Each tool has:

- `name`
- `description` (VERY important)
- optional structured schema
- execution logic

The **description guides the LLM** on when to use the tool.

---

## 🔹 A) `@tool` Decorator (Simple Tools)

Best for:

- Single string input
- Quick utilities
- Rapid prototyping

```python
from langchain.tools import tool

@tool
def get_current_weather(location: str) -> str:
    """Get the current weather for a specified location."""
    return f"The weather in {location} is sunny and 72°F."

print(get_current_weather.name)
print(get_current_weather.description)
```

### ✅ Use When:

- Simple tool
- Minimal schema
- Quick development

---

## 🔹 B) `StructuredTool` (Typed Inputs)

Best for:

- Multiple parameters
- Validation
- Function-calling APIs

```python
from langchain.tools import StructuredTool
from pydantic import BaseModel, Field

class WeatherInput(BaseModel):
    location: str = Field(description="City and country")
    unit: str = Field(description="Celsius or Fahrenheit")

def get_weather_with_unit(location: str, unit: str) -> str:
    return f"The weather in {location} is 20°{unit[0].upper()}."

weather_tool = StructuredTool.from_function(
    func=get_weather_with_unit,
    name="get_weather_with_unit",
    description="Get current weather with specified unit",
    args_schema=WeatherInput,
    return_direct=False
)
```

### 🔥 `return_direct`

- `False` → LLM processes tool result before final answer
- `True` → Tool output is sent directly to user

### ✅ Use When:

- Multiple inputs
- Integration with OpenAI function calling
- Strong schema validation needed

---

## 🔹 C) `BaseTool` (Full Control)

Best for:

- Async support
- Custom validation
- Rate limiting
- Advanced logic

```python
from langchain.tools import BaseTool

class CustomSearchTool(BaseTool):
    name = "custom_search"
    description = "Search the web for information."

    def _run(self, query: str) -> str:
        return f"Search results for '{query}'..."

    async def _arun(self, query: str) -> str:
        return await some_async_search(query)
```

### ✅ Use When:

- Production-grade tools
- Async execution required
- Custom error handling needed

---

# 🧠 Choosing Tool Type

| Tool Type        | When to Use           |
| ---------------- | --------------------- |
| `@tool`          | Simple single-input   |
| `StructuredTool` | Multiple typed inputs |
| `BaseTool`       | Full customization    |

---

# 2️⃣ Agent Types

Different agents exist depending on LLM capabilities.

---

## 🔹 A) `create_openai_tools_agent` (Recommended)

Best for:

- OpenAI models
- Native tool calling
- Highest reliability

Uses **function calling API** instead of parsing text.

```python
from langchain.agents import create_openai_tools_agent
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.agents import AgentExecutor

tools = [get_current_weather, weather_tool]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant that can use tools."),
    ("user", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

agent = create_openai_tools_agent(llm, tools, prompt)

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True
)

result = agent_executor.invoke({"input": "What's the weather in Paris?"})
print(result["output"])
```

### ⚠️ Important

You MUST include:

```
MessagesPlaceholder(variable_name="agent_scratchpad")
```

This stores intermediate reasoning steps.

---

## 🔹 B) `create_react_agent`

Based on **ReAct (Reason + Act)** pattern.

Works with:

- Any LLM
- Text-based models
- Open-source models

Agent outputs:

```
Thought:
Action:
Observation:
```

```python
from langchain.agents import create_react_agent
from langchain.llms import OpenAI

llm = OpenAI(model="text-davinci-003", temperature=0)
agent = create_react_agent(llm, tools, prompt)

executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
executor.invoke({"input": "What's the weather in London?"})
```

### ✅ Use When:

- No native tool calling
- Using open-source models
- Need full reasoning trace

---

## 🔹 C) `create_structured_chat_agent`

Middle ground.

- Uses JSON action format
- More reliable than free text
- Works with models good at structured output

```python
from langchain.agents import create_structured_chat_agent

agent = create_structured_chat_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

### ✅ Use When:

- Model handles JSON well
- No native function calling
- Need more structure than ReAct

---

# 🧠 Choosing the Right Agent

| Agent                          | Best For            | Requirement         |
| ------------------------------ | ------------------- | ------------------- |
| `create_openai_tools_agent`    | OpenAI GPT models   | Native tool calling |
| `create_react_agent`           | Any LLM             | Text reasoning      |
| `create_structured_chat_agent` | JSON-capable models | Structured output   |

---

# 3️⃣ AgentExecutor – The Runtime Loop

The **AgentExecutor** controls execution.

Loop logic:

```
1. Call LLM
2. LLM decides:
      → Final Answer → STOP
      → Tool Call → Execute Tool
3. Add observation
4. Repeat
```

---

## 🔹 Important Parameters

### `max_iterations`

Prevents infinite loops.

```python
max_iterations=5
```

---

### `early_stopping_method`

If max iterations reached:

- `"force"` → Return last observation
- `"generate"` → Ask LLM to produce final answer

---

### `max_execution_time`

```python
max_execution_time=30
```

Stops execution after time limit.

---

## 🔹 Example Configuration

```python
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=5,
    early_stopping_method="generate",
    max_execution_time=30
)
```

---

# 4️⃣ Handling Errors

Agents must be resilient.

---

## 🔹 A) Parsing Errors

```python
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True
)
```

Custom message:

```python
handle_parsing_errors="Check output format and try again."
```

---

## 🔹 B) Tool Exceptions

```python
@tool
def risky_operation(param: str) -> str:
    try:
        return "Success"
    except Exception as e:
        return f"Error: {e}"
```

The agent can retry or choose another strategy.

---

## 🔹 C) Timeout Protection

```python
max_execution_time=30
```

Prevents runaway costs.

---

# 🏁 Complete Example

```python
from langchain.agents import create_openai_tools_agent, AgentExecutor
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.tools import tool

@tool
def search_web(query: str) -> str:
    """Search the web for recent information."""
    return f"Top result for '{query}': LangChain is awesome."

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"Error: {e}"

tools = [search_web, calculate]

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant with tools."),
    ("user", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

agent = create_openai_tools_agent(llm, tools, prompt)

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=3,
    max_execution_time=30,
    handle_parsing_errors=True
)

response = agent_executor.invoke(
    {"input": "What's 123 * 456? Also search for AI news."}
)

print(response["output"])
```

---

# 🧠 Mental Model

Agents = LLM + Tools + Loop

```
LLM thinks
→ chooses tool
→ observes result
→ thinks again
→ produces final answer
```

---

# 🔥 Key Takeaways

- Agents are dynamic decision-makers
- Tool descriptions are critical
- OpenAI tools agent is most robust
- Always configure iteration limits
- Handle parsing & tool errors
- Use structured tools for production systems

---
