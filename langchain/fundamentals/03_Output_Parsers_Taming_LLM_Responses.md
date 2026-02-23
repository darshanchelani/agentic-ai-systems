# 1.3 Output Parsers – Taming the Chaos of LLM Responses

> LLMs generate unstructured text.  
> Output Parsers convert that chaos into structured, validated, production-ready data.

Output parsers sit at the **end of a chain**:

```
Prompt → Model → OutputParser
```

They implement the `Runnable` interface, so they can be chained using:

```python
chain = prompt | model | parser
```

---

# 🔥 Why Output Parsers Matter

### ✅ Reliability

- Enforces schema
- Raises errors if format is invalid

### ✅ Integration

- Structured output (JSON, dict, Pydantic model)
- Directly usable in APIs, databases, pipelines

### ✅ Prompt Guidance

- Many parsers auto-generate format instructions
- Improves output consistency

---

# 1️⃣ StrOutputParser

The simplest parser.

Extracts plain string content from model output.

---

## 🔹 When to Use

- You just need raw text
- Passing output into another prompt
- No structure required

---

## 🔹 Example

```python
from langchain.chat_models import ChatOpenAI
from langchain.schema import HumanMessage
from langchain.output_parsers import StrOutputParser

model = ChatOpenAI(model="gpt-3.5-turbo")
parser = StrOutputParser()

chain = model | parser

result = chain.invoke([
    HumanMessage(content="Say hello in French.")
])

print(result)  # "Bonjour !" (string)
```

---

## 🧠 What It Does Internally

- If output is `AIMessage` → returns `.content`
- If output is `str` → returns as-is

---

# 2️⃣ PydanticOutputParser (Most Important)

The gold standard for structured output.

You define a **Pydantic model** → parser ensures output matches it.

---

## 🔹 What It Does

1. Generates format instructions (JSON schema)
2. Parses model output into a Pydantic object
3. Validates fields and types

---

## 🔹 When to Use

- Entity extraction
- API payload generation
- Structured data generation
- Information extraction
- Production systems

---

## 🔹 Example

```python
from langchain.output_parsers import PydanticOutputParser
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI
from pydantic import BaseModel, Field

class Person(BaseModel):
    name: str = Field(description="person's full name")
    age: int = Field(description="person's age in years")
    occupation: str = Field(description="person's current job")

parser = PydanticOutputParser(pydantic_object=Person)

prompt = PromptTemplate(
    template="""
Extract the person's details from the following text.

{format_instructions}

Text: {query}
""",
    input_variables=["query"],
    partial_variables={
        "format_instructions": parser.get_format_instructions()
    }
)

model = ChatOpenAI(model="gpt-3.5-turbo")

chain = prompt | model | parser

input_text = "John Doe is a 32-year-old software engineer living in New York."

person = chain.invoke({"query": input_text})

print(person)
print(person.name)
```

---

## 🧠 Why This Is Powerful

- Enforces type validation
- Reduces hallucination
- Safer production usage
- Schema-driven prompting

---

# 3️⃣ Specialized Parsers

---

# 🔹 CommaSeparatedListOutputParser

Expects comma-separated output.

### Example Output:

```
apple, banana, cherry
```

Returns:

```python
["apple", "banana", "cherry"]
```

---

## 🔹 When to Use

- Keywords
- Tags
- To-do lists
- Ingredients

---

## 🔹 Example

```python
from langchain.output_parsers import CommaSeparatedListOutputParser
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI

parser = CommaSeparatedListOutputParser()

prompt = PromptTemplate(
    template="List five fruits.\n{format_instructions}",
    input_variables=[],
    partial_variables={
        "format_instructions": parser.get_format_instructions()
    }
)

model = ChatOpenAI(model="gpt-3.5-turbo")

chain = prompt | model | parser

result = chain.invoke({})
print(result)
```

---

# 🔹 DatetimeOutputParser

Parses datetime strings into Python `datetime` objects.

---

## 🔹 When to Use

- Event scheduling
- Extracting dates
- Timeline generation
- Historical data parsing

---

## 🔹 Example

```python
from langchain.output_parsers import DatetimeOutputParser
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI

parser = DatetimeOutputParser()

prompt = PromptTemplate(
    template="What is the date of the next Christmas?\n{format_instructions}",
    input_variables=[],
    partial_variables={
        "format_instructions": parser.get_format_instructions()
    }
)

model = ChatOpenAI(model="gpt-3.5-turbo")

chain = prompt | model | parser

result = chain.invoke({})

print(result)
print(type(result))
```

---

# 4️⃣ Custom Output Parsers

When built-in parsers are insufficient.

Create your own by subclassing `BaseOutputParser`.

---

## 🔹 When to Build Custom Parser

- Custom formats (XML, YAML, CSV dialect)
- Proprietary formats
- Line-by-line parsing
- Advanced validation logic
- Error handling + retries

---

## 🔹 Example – Key-Value Parser

Expected format:

```
name: Alice
age: 30
city: Paris
```

---

```python
from langchain.schema import BaseOutputParser
from typing import Dict

class KeyValueParser(BaseOutputParser[Dict[str, str]]):

    def parse(self, text: str) -> Dict[str, str]:
        lines = text.strip().split("\n")
        result = {}

        for line in lines:
            if ": " in line:
                key, value = line.split(": ", 1)
                result[key.strip()] = value.strip()

        return result

    @property
    def _type(self) -> str:
        return "key_value"
```

---

## 🔹 Usage

```python
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI

parser = KeyValueParser()

prompt = PromptTemplate.from_template(
    "Extract name, age, and city as key-value pairs.\nText: {input}"
)

model = ChatOpenAI()

chain = prompt | model | parser

result = chain.invoke({
    "input": "Alice is 30 years old and lives in Paris."
})

print(result)
```

---

## 🧠 Custom Parser Rules

- Must implement:
  ```python
  def parse(self, text: str) -> T
  ```
- Optional:
  ```python
  def get_format_instructions()
  ```
- `_type` property recommended for serialization

---

# 🔥 Combining Parsers with Chains

Because parsers are `Runnable`, you can do:

```python
chain = prompt | model | parser
```

You can also:

```python
chain.with_fallbacks([...])
```

For:

- Retry logic
- Model switching
- Graceful degradation

---

# 🧠 Revision Comparison Table

| Parser                         | Output Type    | Best For              | Validation   |
| ------------------------------ | -------------- | --------------------- | ------------ |
| StrOutputParser                | string         | Raw text              | ❌           |
| PydanticOutputParser           | Pydantic model | Structured extraction | ✅           |
| CommaSeparatedListOutputParser | list[str]      | Simple lists          | Basic        |
| DatetimeOutputParser           | datetime       | Date extraction       | Format-based |
| Custom Parser                  | Any            | Special formats       | Custom       |

---

# 🏗 Production Architecture Pattern

```
User Input
   ↓
PromptTemplate / ChatPromptTemplate
   ↓
Model
   ↓
OutputParser (validation layer)
   ↓
Application Logic / DB / API
```

---

# 🚀 Key Production Insight

Without output parsers:

- Your app = brittle
- Parsing = manual
- Bugs = frequent

With output parsers:

- Schema-driven AI
- Reliable pipelines
- Strong typing
- Clean integrations

---

# 🎯 Final Takeaways

- Always use `StrOutputParser` if you don't need structure.
- Prefer `PydanticOutputParser` for serious applications.
- Specialized parsers reduce prompt engineering effort.
- Custom parsers unlock full control.
- Parsers = bridge between LLMs and real systems.

---

# 📌 What You’ve Mastered So Far

✅ Models  
✅ Prompts  
✅ Output Parsers

Next logical topic:

---

End of Notes – 1.3 Output Parsers
