---

# 1.5 Chains in LangChain: Composing LLM Workflows

## 📌 Overview

Chains are the core workflow abstraction in LangChain.
They connect:

- **Prompts**
- **LLMs**
- **Memory**
- **Output Parsers**
- **Other Chains**

Instead of just:

```
Prompt → Model → Output
```

Chains allow:

```
Prompt → Model → Transform → Model → Route → Retrieve → Output
```

They enable structured, multi-step AI applications.

---

# 1️⃣ LLMChain – The Fundamental Building Block

## 🔹 What It Is

`LLMChain` = PromptTemplate + LLM + (optional) OutputParser

It:

1. Formats the prompt
2. Calls the model
3. Parses output (if parser provided)
4. Returns result

---

## 🔹 When to Use

- Simple prompt → model tasks
- Reusable prompt-model logic
- Component inside larger chains
- When you want built-in memory support

---

## 🔹 Example

```python
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI
from langchain.output_parsers import CommaSeparatedListOutputParser

prompt = PromptTemplate(
    template="List three {adjective} fruits.\n{format_instructions}",
    input_variables=["adjective"],
    partial_variables={
        "format_instructions": CommaSeparatedListOutputParser().get_format_instructions()
    }
)

chain = LLMChain(
    llm=ChatOpenAI(model="gpt-3.5-turbo"),
    prompt=prompt,
    output_parser=CommaSeparatedListOutputParser(),
    verbose=True
)

result = chain.run(adjective="tropical")
print(result)
```

---

## 🔹 Key Notes

- Implements `Runnable` interface.
- Works with LCEL (`prompt | model | parser`).
- Supports memory integration.
- Good for small, modular tasks.

---

# 2️⃣ Sequential Chains – Multi-Step Pipelines

Sequential chains execute multiple chains in order.

---

## 2.1 SimpleSequentialChain

### 🔹 Concept

- Each step has:
  - 1 input
  - 1 output

- Output of one step → Input of next

### 🔹 Use Case

- Linear transformations
- Generate → Improve → Polish
- Simple content pipelines

---

### 🔹 Example

```python
from langchain.chains import SimpleSequentialChain, LLMChain
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI

first_prompt = PromptTemplate(
    input_variables=["product"],
    template="What is a good name for a company that makes {product}?"
)

chain_one = LLMChain(llm=ChatOpenAI(), prompt=first_prompt)

second_prompt = PromptTemplate(
    input_variables=["company_name"],
    template="Write a catchy tagline for a company called {company_name}."
)

chain_two = LLMChain(llm=ChatOpenAI(), prompt=second_prompt)

overall_chain = SimpleSequentialChain(
    chains=[chain_one, chain_two],
    verbose=True
)

result = overall_chain.run("eco-friendly water bottles")
print(result)
```

---

## 2.2 SequentialChain (Advanced Version)

### 🔹 Concept

- Supports multiple named inputs and outputs
- Explicit variable mapping
- More flexible than SimpleSequentialChain

---

### 🔹 When to Use

- Complex pipelines
- Multiple outputs required
- Later steps depend on multiple earlier outputs

---

### 🔹 Example

```python
from langchain.chains import SequentialChain, LLMChain
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI

prompt1 = PromptTemplate(
    input_variables=["product"],
    template="Write a detailed review of {product}."
)

chain1 = LLMChain(llm=ChatOpenAI(), prompt=prompt1, output_key="review")

prompt2 = PromptTemplate(
    input_variables=["review"],
    template="Summarize the following review in one sentence: {review}"
)

chain2 = LLMChain(llm=ChatOpenAI(), prompt=prompt2, output_key="summary")

prompt3 = PromptTemplate(
    input_variables=["review", "summary"],
    template="""Based on the review and summary, suggest a follow-up question.
Review: {review}
Summary: {summary}"""
)

chain3 = LLMChain(llm=ChatOpenAI(), prompt=prompt3, output_key="question")

sequential_chain = SequentialChain(
    chains=[chain1, chain2, chain3],
    input_variables=["product"],
    output_variables=["review", "summary", "question"]
)

result = sequential_chain({"product": "wireless headphones"})
```

---

# 3️⃣ Router Chains – Intelligent Routing

## 🔹 Concept

Router chains dynamically select which sub-chain to execute.

Flow:

```
User Input → Router LLM → Destination Chain → Output
```

---

## 🔹 When to Use

- Multi-domain assistants
- Task-based routing (math, physics, coding)
- Pre-agent systems
- Specialized prompt selection

---

## 🔹 Components

1. Router Prompt (classification logic)
2. Destination Chains
3. Default Chain
4. RouterOutputParser

---

## 🔹 Example Concept

```python
chain = MultiPromptChain(
    router_chain=router_chain,
    destination_chains=destination_chains,
    default_chain=destination_chains["default"],
    verbose=True
)

print(chain.run("What is the speed of light?"))
print(chain.run("Solve x^2 + 2x + 1 = 0"))
```

---

## 🔹 Important

Router must output:

```json
{
  "destination": "chain_name",
  "next_inputs": { ... }
}
```

`RouterOutputParser` handles parsing.

---

# 4️⃣ TransformChain – Non-LLM Processing

## 🔹 What It Is

A chain that:

- Does NOT call an LLM
- Transforms input dictionary → output dictionary
- Uses Python functions

---

## 🔹 Why Use It?

- Pre-processing text
- Post-processing model output
- Data extraction
- Field mapping
- Structured pipelines

---

## 🔹 Example

```python
from langchain.chains import TransformChain

def extract_first_sentence(inputs: dict) -> dict:
    text = inputs["text"]
    first_sentence = text.split(".")[0] + "."
    return {"first_sentence": first_sentence}

transform_chain = TransformChain(
    input_variables=["text"],
    output_variables=["first_sentence"],
    transform=extract_first_sentence
)
```

---

## 🔹 Key Insight

Wrap Python logic as chain to:

- Maintain consistent I/O format
- Integrate into SequentialChain
- Enable composition

---

# 5️⃣ Built-In Chains (High-Level Utilities)

LangChain provides ready-made chains for common workflows.

---

# 5.1 load_summarize_chain

## 🔹 Purpose

Summarize long documents.

Handles:

- Text splitting
- Chunk summarization
- Combining summaries

---

## 🔹 chain_type Options

| Type         | Description                  |
| ------------ | ---------------------------- |
| `stuff`      | Put everything in one prompt |
| `map_reduce` | Summarize chunks → combine   |
| `refine`     | Iteratively improve summary  |

---

## 🔹 Example

```python
from langchain.chains.summarize import load_summarize_chain
from langchain.chat_models import ChatOpenAI

llm = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)
chain = load_summarize_chain(llm, chain_type="map_reduce")

summary = chain.run(docs)
```

---

# 5.2 create_retrieval_chain (RAG)

## 🔹 Purpose

Build Retrieval-Augmented Generation pipeline.

Flow:

```
User Question
   ↓
Retriever (Vector DB)
   ↓
Context + Question
   ↓
LLM
   ↓
Answer
```

---

## 🔹 When to Use

- QA over documents
- Private knowledge bases
- Enterprise search
- Custom knowledge assistants

---

## 🔹 Example

```python
from langchain.chains import create_retrieval_chain

rag_chain = create_retrieval_chain(retriever, combine_docs_chain)

response = rag_chain.invoke({"input": "What is the capital of France?"})

print(response["answer"])
print(response["context"])
```

---

# 🔥 Chain Selection Guide

| Chain Type             | Best For                    |
| ---------------------- | --------------------------- |
| LLMChain               | Basic prompt → model        |
| SimpleSequentialChain  | Linear pipeline             |
| SequentialChain        | Multi-input/output pipeline |
| RouterChain            | Dynamic chain selection     |
| TransformChain         | Python logic in pipeline    |
| load_summarize_chain   | Long document summarization |
| create_retrieval_chain | RAG applications            |

---

# 🧠 Conceptual Understanding

## Chains = Deterministic Workflows

Chains:

- Follow predefined structure
- Do not decide which tools to use dynamically
- Execute defined logic

Agents (next topic) are different:

- Agents decide what to do
- Use tools dynamically
- Think step-by-step

---

# 🚀 Key Takeaways

- Chains are composable building blocks.
- `LLMChain` is the foundation.
- Sequential chains build pipelines.
- Router chains enable intent-based branching.
- Transform chains insert pure Python logic.
- Built-in chains accelerate development.
- LCEL is the modern composition method.

---

# 📌 Mental Model

Think of Chains like:

```
Backend microservices for LLM workflows
```

Each chain:

- Accepts structured input
- Produces structured output
- Can be combined with others

---

✅ This section prepares you for **Agents**, where workflows become dynamic and decision-based.

---
