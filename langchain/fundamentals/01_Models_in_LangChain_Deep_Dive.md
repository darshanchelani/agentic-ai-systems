# 1.1 Models in LangChain – Deep Dive

> Structured notes for understanding Models in LangChain (LLM vs ChatModel, Runnable Interface, Model IO, Multimodality)

---

# 1️⃣ LLM vs ChatModel

LangChain abstracts two primary types of models:

- **LLM (Legacy Completion Models)**
- **ChatModel (Conversational Models)**

Understanding the difference is fundamental.

---

## 🔹 LLM (Language Model)

### Interface

- Input → `string`
- Output → `string`

### Best For

- Simple text generation
- Legacy models
- Text-in → text-out tasks
- No conversational memory required

### Example Models

- `text-davinci-003`
- Completion-style APIs

### Example

```python
from langchain.llms import OpenAI

llm = OpenAI(model="text-davinci-003", temperature=0)
response = llm("What is the capital of France?")
print(response)
```

---

## 🔹 ChatModel

### Interface

- Input → List of messages:
  - `SystemMessage`
  - `HumanMessage`
  - `AIMessage`
- Output → `AIMessage`

### Best For

- Multi-turn conversations
- System role instructions
- Tool calling
- Modern production apps

### Example Models

- `gpt-3.5-turbo`
- `gpt-4`
- `claude-3`
- `gemini`

### Example

```python
from langchain.chat_models import ChatOpenAI
from langchain.schema import HumanMessage, SystemMessage

chat = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

messages = [
    SystemMessage(content="You are a helpful assistant that speaks like a pirate."),
    HumanMessage(content="What is the capital of France?")
]

response = chat(messages)
print(response.content)
```

---

## 🧠 When to Use Which?

| Situation               | Use LLM | Use ChatModel |
| ----------------------- | ------- | ------------- |
| Simple text completion  | ✅      |               |
| Multi-turn conversation |         | ✅            |
| Tool calling            |         | ✅            |
| System instructions     |         | ✅            |
| Legacy model support    | ✅      |               |
| 95% of modern apps      |         | ✅            |

> 🔥 Rule of Thumb: Use **ChatModel** for almost everything today.

---

# 2️⃣ Runnable Interface

LangChain standardizes model interaction using the **Runnable protocol**.

This makes models, chains, retrievers interchangeable.

---

## Core Methods

| Method      | Description                        | Use Case             |
| ----------- | ---------------------------------- | -------------------- |
| `invoke()`  | Single input → Single output       | Default sync calls   |
| `stream()`  | Streams output chunks              | Real-time UI         |
| `batch()`   | Multiple inputs → Multiple outputs | Bulk processing      |
| `ainvoke()` | Async invoke                       | FastAPI / async apps |
| `astream()` | Async stream                       | Websocket streaming  |
| `abatch()`  | Async batch                        | Parallel async jobs  |

---

## Example

```python
from langchain.chat_models import ChatOpenAI
from langchain.schema import HumanMessage
import asyncio

chat = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

# 1️⃣ invoke
result = chat.invoke([HumanMessage(content="Tell me a joke")])
print(result.content)

# 2️⃣ stream
for chunk in chat.stream([HumanMessage(content="Tell me a long story")]):
    print(chunk.content, end="", flush=True)

# 3️⃣ batch
inputs = [
    [HumanMessage(content="What is 2+2?")],
    [HumanMessage(content="What is 3+3?")]
]
results = chat.batch(inputs)
for res in results:
    print(res.content)

# 4️⃣ async
async def async_example():
    result = await chat.ainvoke([HumanMessage(content="Hello")])
    print(result.content)

asyncio.run(async_example())
```

---

## 🧠 When to Use What?

- Use `invoke()` → Default
- Use `stream()` → Chat UI, real-time generation
- Use `batch()` → Summarizing multiple docs
- Use async → High-performance APIs

---

# 3️⃣ Model IO – Parameter Tuning, Tool Binding, Fallbacks

---

# 🔹 Parameter Tuning

### Important Parameters

| Parameter           | Purpose               |
| ------------------- | --------------------- |
| `temperature`       | Creativity/randomness |
| `max_tokens`        | Output length limit   |
| `top_p`             | Nucleus sampling      |
| `presence_penalty`  | Encourage new topics  |
| `frequency_penalty` | Reduce repetition     |

### Example

```python
chat = ChatOpenAI(
    model="gpt-3.5-turbo",
    temperature=0.2,
    max_tokens=100
)
```

---

# 🔹 Tool Binding (Function Calling)

Modern chat models can call tools (functions).

Use cases:

- API calls
- DB queries
- Structured extraction
- External computation

### Example

```python
from langchain.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get current weather for a location."""
    return f"Weather in {location} is sunny."

chat_with_tools = chat.bind_tools([get_weather])

response = chat_with_tools.invoke(
    [HumanMessage("What's the weather in Paris?")]
)

if response.tool_calls:
    for tool_call in response.tool_calls:
        print(tool_call["name"], tool_call["args"])
```

---

# 🔹 Fallbacks (Production Safety)

Use when:

- Rate limits
- Model downtime
- Provider failure

### Example

```python
from langchain.llms import OpenAI

llm_fallback = OpenAI(model="text-davinci-003")

chain = chat | (lambda msg: msg.content)
chain_with_fallback = chain.with_fallbacks([llm_fallback])

result = chain_with_fallback.invoke("Hello")
print(result)
```

> 💡 Production Tip: Always configure fallbacks in real-world systems.

---

# 4️⃣ Multimodality (Vision Models)

Multimodal models accept:

- Text
- Images (URL or base64)

---

## Supported Models

- GPT-4 Vision
- Claude 3
- Gemini Vision

---

## Use Cases

- Image captioning
- OCR via prompting
- Chart understanding
- Invoice parsing
- Screenshot debugging

---

## Example – Image URL

```python
from langchain.chat_models import ChatOpenAI
from langchain.schema import HumanMessage

chat = ChatOpenAI(model="gpt-4-vision-preview", max_tokens=300)

message = HumanMessage(
    content=[
        {"type": "text", "text": "What's in this image?"},
        {
            "type": "image_url",
            "image_url": {
                "url": "https://example.com/cat.jpg"
            }
        }
    ]
)

response = chat.invoke([message])
print(response.content)
```

---

## Example – Base64 Image

```python
import base64

with open("cat.jpg", "rb") as f:
    image_data = base64.b64encode(f.read()).decode("utf-8")

message = HumanMessage(
    content=[
        {"type": "text", "text": "Describe this image"},
        {
            "type": "image_url",
            "image_url": {
                "url": f"data:image/jpeg;base64,{image_data}"
            }
        }
    ]
)

response = chat.invoke([message])
```

---

# 🔥 Final Revision Summary

### Core Learnings

- ChatModel > LLM for modern apps
- Runnable interface = standardized execution
- Tune parameters for cost + quality
- Bind tools for real-world actions
- Always configure fallbacks
- Use multimodal models for image + text tasks

---
