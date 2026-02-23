# 1.2 Prompts in LangChain – The Art of Talking to Machines

> Prompts are the interface between your intent and the model’s output.
> Mastering prompts = mastering control over LLM behavior.

This section covers:

1. PromptTemplate
2. ChatPromptTemplate
3. Few-shot prompting
4. MessagesPlaceholder

---

# 1️⃣ PromptTemplate

`PromptTemplate` is string templating for LLM prompts.

Think of it as **Python f-strings with structure + reusability**.

---

## 🔹 Why Use PromptTemplate?

- Reusability (prompt blueprint)
- Separation of structure and data
- Safer variable injection
- Supports partial variables
- Works seamlessly in chains

---

## 🔹 When to Use

- Simple text generation tasks
- Single-string prompts
- Summarization
- Translation
- Q&A (non-chat models)
- LLM-based pipelines

---

## 🔹 Basic Example

```python
from langchain.prompts import PromptTemplate

template = "What is a good name for a company that makes {product}?"
prompt = PromptTemplate.from_template(template)

formatted_prompt = prompt.format(product="colorful socks")
print(formatted_prompt)
```

Output:

```
What is a good name for a company that makes colorful socks?
```

---

## 🔹 Multiple Variables

```python
template = "Tell me a {adjective} joke about {topic}."
prompt = PromptTemplate(
    template=template,
    input_variables=["adjective", "topic"]
)

print(prompt.format(adjective="funny", topic="programmers"))
```

---

## 🔹 Partial Variables (Very Important)

Pre-fill common values.

```python
partial_prompt = PromptTemplate(
    template="Provide a {adjective} summary of the following text: {text}",
    partial_variables={"adjective": "concise"}
)

formatted = partial_prompt.format(text="Long article content...")
print(formatted)
```

### Why Partial Variables Matter

- Avoid repetition
- Keep shared system context
- Great for production pipelines

You can also use:

```python
prompt = prompt.partial(adjective="concise")
```

---

# 2️⃣ ChatPromptTemplate

Used for chat-based models (GPT-4, Claude, Gemini).

Chat models expect **role-based messages**:

- system
- human
- ai

---

## 🔹 Why Use ChatPromptTemplate?

- Role-based prompting
- Multi-turn conversations
- Persona definition
- Few-shot within chat format
- Production chatbots

---

## 🔹 When to Use

- Chatbots
- Virtual assistants
- Agents
- Multi-turn workflows
- Any modern LLM application

> 🔥 Rule: If using ChatModel → Use ChatPromptTemplate.

---

## 🔹 Basic Example (Recommended Way)

```python
from langchain.prompts import ChatPromptTemplate

chat_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant that translates English to French."),
    ("human", "Translate this: {text}")
])

messages = chat_prompt.format_messages(text="Hello, how are you?")
print(messages)
```

---

## 🔹 Explicit Message Templates

```python
from langchain.prompts import (
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)

system_template = "You are a {language} translator."
system_message = SystemMessagePromptTemplate.from_template(system_template)

human_template = "{text}"
human_message = HumanMessagePromptTemplate.from_template(human_template)

chat_prompt = ChatPromptTemplate.from_messages([
    system_message,
    human_message
])

formatted = chat_prompt.format_messages(
    language="Spanish",
    text="Good morning"
)
```

---

## 🧠 Key Insight

- System message → controls behavior
- Human message → user input
- AI message → examples or history

---

# 3️⃣ Few-Shot Prompting (Dynamic Example Selection)

Few-shot prompting = Give the model examples before real input.

Instead of:

```
Input: glad
Output:
```

We provide examples first.

---

## 🔹 Why Use Few-Shot?

- Improves accuracy
- Guides output format
- Reduces hallucinations
- Helps structured generation
- Boosts performance on complex tasks

---

## 🔹 When to Use

- Classification
- Extraction
- Code generation
- Synonym tasks
- Structured outputs
- Format-sensitive tasks

---

## 🔹 Components

1. Example dataset
2. Example selector
3. Example prompt template
4. Final few-shot template

---

## 🔹 Example (Semantic Similarity Based)

```python
from langchain.prompts import FewShotPromptTemplate, PromptTemplate
from langchain.prompts.example_selector import SemanticSimilarityExampleSelector
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma

examples = [
    {"input": "happy", "output": "joyful"},
    {"input": "sad", "output": "unhappy"},
    {"input": "big", "output": "large"},
    {"input": "small", "output": "tiny"},
]

example_selector = SemanticSimilarityExampleSelector.from_examples(
    examples,
    OpenAIEmbeddings(),
    Chroma,
    k=2
)

example_prompt = PromptTemplate(
    input_variables=["input", "output"],
    template="Input: {input}\nOutput: {output}"
)

few_shot_prompt = FewShotPromptTemplate(
    example_selector=example_selector,
    example_prompt=example_prompt,
    prefix="Give the synonym for the following word:",
    suffix="Input: {word}\nOutput:",
    input_variables=["word"]
)

query = "glad"
prompt = few_shot_prompt.format(word=query)
print(prompt)
```

---

## 🔥 Why Dynamic Selection Is Powerful

- Saves tokens
- Keeps prompts small
- Uses relevant examples only
- Scales to large example datasets

---

# 4️⃣ MessagesPlaceholder

Used inside `ChatPromptTemplate`.

It acts as a placeholder for a list of messages.

---

## 🔹 Why Use MessagesPlaceholder?

- Inject conversation history
- Inject dynamic examples
- Insert variable-length message lists
- Required for memory systems

---

## 🔹 When to Use

- Chatbots with memory
- RAG chat systems
- Agents
- Multi-turn conversation

---

## 🔹 Example

```python
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.schema import HumanMessage, AIMessage

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

history_messages = [
    HumanMessage(content="What is the capital of France?"),
    AIMessage(content="Paris.")
]

formatted = prompt.format_messages(
    history=history_messages,
    input="What is its population?"
)

print(formatted)
```

---

## ⚠ Important Rule

`MessagesPlaceholder` expects:

- A list of `BaseMessage` objects
- NOT plain strings

---

# 🧠 Revision Comparison Table

| Component             | Used For                | Model Type | Supports Roles | Supports History |
| --------------------- | ----------------------- | ---------- | -------------- | ---------------- |
| PromptTemplate        | Simple string prompts   | LLM        | ❌             | ❌               |
| ChatPromptTemplate    | Structured chat prompts | ChatModel  | ✅             | ✅               |
| FewShotPromptTemplate | Example-based prompting | Both       | ❌             | ❌               |
| MessagesPlaceholder   | Inject dynamic messages | ChatModel  | ✅             | ✅               |

---

# 🚀 Architecture Insight

Production flow usually looks like:

```
ChatPromptTemplate
    ↓
Model
    ↓
OutputParser
    ↓
Application Logic
```

Few-shot and placeholders enhance the prompt layer.

---

# 🔥 Final Revision Summary

- PromptTemplate = string templating
- ChatPromptTemplate = role-based structured prompting
- Few-shot = guide model via examples
- Semantic selector = dynamic relevant examples
- MessagesPlaceholder = inject history dynamically

---
