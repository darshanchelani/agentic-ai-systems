# 1.4 Memory in LangChain – Giving Your LLM a Brain That Remembers

> By default, LLMs are stateless.  
> Memory in LangChain enables context retention across interactions.

Without memory:

User: My name is Alice.  
Assistant: Nice to meet you, Alice!  
User: What’s my name?  
Assistant: I don’t know. 😞

With memory → the assistant remembers.

---

# 🔥 Why Memory Matters

- Enables multi-turn conversations
- Maintains context across interactions
- Allows persistent user preferences
- Required for intelligent agents
- Essential for production chat systems

---

# 🧠 Core Memory Concept

All memory classes implement:

```python
load_memory_variables(inputs)
save_context(inputs, outputs)
```

The most important integration concept:

## 🔑 `memory_key`

This is the variable name in your prompt where memory will be injected.

Example:

```
Conversation history:
{history}
```

Then your memory must use:

```python
memory_key="history"
```

If names don’t match → memory will NOT work.

---

# 1️⃣ ConversationBufferMemory (Raw Storage)

Stores entire conversation history.

---

## 🔹 How It Works

- Appends every message
- Returns full conversation
- Can return:
  - String (`return_messages=False`)
  - Message list (`return_messages=True`)

---

## 🔹 When to Use

- Short chats
- Debugging
- Exact recall needed
- Token size not an issue

---

## 🔹 Example

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain
from langchain.chat_models import ChatOpenAI

memory = ConversationBufferMemory(return_messages=True)

llm = ChatOpenAI(model="gpt-3.5-turbo")
conversation = ConversationChain(llm=llm, memory=memory)

response = conversation.predict(input="Hi, I'm Alice.")
print(response)

response = conversation.predict(input="What's my name?")
print(response)

print(memory.chat_memory.messages)
```

---

## ⚠ Trade-off

- History grows indefinitely
- Token usage increases
- Not scalable for long sessions

---

# 2️⃣ ConversationBufferWindowMemory (Sliding Window)

Keeps only last `k` exchanges.

---

## 🔹 How It Works

- Stores only recent `k` message pairs
- Old messages removed automatically
- Controls token growth

---

## 🔹 When to Use

- Long-running chats
- Only recent context matters
- Token efficiency required

---

## 🔹 Example

```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(k=2, return_messages=True)

memory.save_context({"input": "Hi"}, {"output": "Hello"})
memory.save_context({"input": "I'm Alice"}, {"output": "Nice to meet you"})
memory.save_context({"input": "I like pizza"}, {"output": "Pizza is great!"})

vars = memory.load_memory_variables({})
print(vars["history"])
```

---

## ⚠ Trade-off

- Older information permanently lost
- Not suitable for long-term recall

---

# 3️⃣ ConversationSummaryMemory (Summarized Memory)

Summarizes older messages using an LLM.

---

## 🔹 How It Works

1. Stores conversation
2. Periodically summarizes using LLM
3. Injects:
   - Summary
   - Recent messages

---

## 🔹 When to Use

- Very long conversations
- Need gist of earlier context
- Personal assistants
- Therapy/chat companions

---

## 🔹 Example

```python
from langchain.memory import ConversationSummaryMemory
from langchain.chat_models import ChatOpenAI

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

memory = ConversationSummaryMemory(
    llm=llm,
    return_messages=True
)

memory.save_context(
    {"input": "I'm a software engineer from Seattle."},
    {"output": "Cool! I love Seattle."}
)

memory.save_context(
    {"input": "I work on AI projects."},
    {"output": "Fascinating field."}
)

vars = memory.load_memory_variables({})
print(vars["history"])
```

---

## ⚠ Trade-off

- Additional LLM calls
- Higher latency
- Increased cost

---

# 4️⃣ VectorStoreRetrieverMemory (Semantic Memory)

Stores conversation in vector database.

Retrieves relevant past memories based on similarity.

---

## 🔹 How It Works

1. Embed messages
2. Store in vector DB
3. On new input:
   - Embed query
   - Retrieve similar past messages
   - Inject relevant ones only

---

## 🔹 When to Use

- Long-term memory
- Cross-session recall
- Personal AI assistants
- Large-scale history
- Fact retrieval from past

---

## 🔹 Example

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma

embeddings = OpenAIEmbeddings()

vectorstore = Chroma(
    embedding_function=embeddings,
    collection_name="memory"
)

retriever = vectorstore.as_retriever(
    search_kwargs={"k": 2}
)

memory = VectorStoreRetrieverMemory(
    retriever=retriever,
    memory_key="history"
)

memory.save_context(
    {"input": "My favorite color is blue."},
    {"output": "Noted."}
)

query = "What is my favorite color?"
relevant = memory.load_memory_variables({"input": query})

print(relevant["history"])
```

---

## ⚠ Trade-off

- Requires embeddings
- Requires vector database
- Slight retrieval overhead

---

# 🔑 How Memory Works in Chains

## Step-by-Step Flow

1. User input received
2. Chain calls:

```python
memory.load_memory_variables(inputs)
```

3. Memory returns:

```python
{"history": "..."}
```

4. Chain merges inputs + history
5. Prompt formatted
6. Model invoked
7. After response:

```python
memory.save_context(inputs, outputs)
```

---

# 🧩 Custom Chain Example with memory_key

```python
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain
from langchain.memory import ConversationBufferMemory
from langchain.chat_models import ChatOpenAI

template = """
You are a helpful assistant.

Conversation history:
{history}

Human: {input}
Assistant:
"""

prompt = PromptTemplate(
    input_variables=["history", "input"],
    template=template
)

memory = ConversationBufferMemory(
    memory_key="history",
    return_messages=False
)

llm = ChatOpenAI()

chain = LLMChain(
    llm=llm,
    prompt=prompt,
    memory=memory
)

response = chain.run("My name is Alice.")
print(response)

response = chain.run("What's my name?")
print(response)
```

---

# 🤖 Memory in Agents

Agents also support memory.

Difference:

- Agents may use additional placeholders like:
  - `{agent_scratchpad}`
  - `{history}`

Memory injection works the same way via `memory_key`.

Used with:

```python
AgentExecutor(memory=memory)
```

---

# 📊 Memory Comparison Table

| Memory Type                    | Best For                  | Trade-Off               |
| ------------------------------ | ------------------------- | ----------------------- |
| ConversationBufferMemory       | Short chats               | Unlimited growth        |
| ConversationBufferWindowMemory | Recent context            | Loses old info          |
| ConversationSummaryMemory      | Long conversations        | Extra LLM cost          |
| VectorStoreRetrieverMemory     | Long-term semantic memory | Embedding + DB overhead |

---

# 🏗 Advanced: Combined Memory

You can combine multiple memory types:

- Window memory for recent context
- Summary memory for long-term context
- Vector memory for semantic recall

Using:

```python
CombinedMemory(...)
```

Advanced production pattern.

---

# 🚀 Production Strategy

For most apps:

- Small chatbot → BufferWindowMemory
- Long assistant → SummaryMemory
- Personal AI → VectorStoreRetrieverMemory
- Enterprise system → CombinedMemory

---

# 🎯 Final Takeaways

- LLMs are stateless by default.
- Memory injects context via `memory_key`.
- Choose memory type based on:
  - Conversation length
  - Token constraints
  - Cost sensitivity
  - Need for long-term recall
- Memory transforms chatbots into assistants.

---

# 📌 You’ve Mastered So Far

✅ Models  
✅ Prompts  
✅ Output Parsers  
✅ Memory

---

End of Notes – 1.4 Memory in LangChain
