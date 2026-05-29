# Agent Memory in LLM Systems

> How AI agents store and retrieve context.

## Table of Contents

* [Why Memory Matters](#why-memory-matters)
* [Overview of Agent Memory Types](#overview-of-agent-memory-types)
* [1. In-Memory Conversation Memory](#1-in-memory-conversation-memory)
* [2. Persistent Memory with SQLite](#2-persistent-memory-with-sqlite)
* [3. Tool-Based Memory](#3-tool-based-memory)
* [4. Semantic Memory (Vector Database)](#4-semantic-memory-vector-database)
* [5. LangChain Memory](#5-langchain-memory)
* [6. CrewAI Memory](#6-crewai-memory)
* [7. AutoGen Memory](#7-autogen-memory)
* [8. Production Memory Architecture](#8-production-memory-architecture)
* [Key Takeaways](#key-takeaways)

---


# Overview of Agent Memory Types

Agent memory is not a single concept—it depends on what the system needs to remember.

There are four primary types:

* **Conversation memory** → remembers messages within a session
* **Persistent memory** → stores facts across sessions
* **Semantic memory** → retrieves knowledge via embeddings
* **Working memory** → tracks intermediate reasoning and tool outputs

---

# Why Memory Matters
Without memory:

- Agents forget context
- Conversations restart every time

Types of Memory

- Short-term memory (session)
- Long-term memory (vector DB)
- Tool memory (state per tool)

---

# 1. In-Memory Conversation Memory

This is the simplest form of memory: storing chat history in a list and sending it back to the model each time.

## Implementation

```python id="mem1"
from openai import OpenAI

client = OpenAI(api_key="...")

messages = [
    {
        "role": "system",
        "content": "You are a helpful assistant"
    }
]

while True:

    user_input = input("User: ")

    messages.append({
        "role": "user",
        "content": user_input
    })

    response = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=messages
    )

    answer = response.choices[0].message.content

    print("Assistant:", answer)

    messages.append({
        "role": "assistant",
        "content": answer
    })
```

## Key Idea

Memory exists only while the process is running.

---

# 2. Persistent Memory with SQLite

Persistent memory allows agents to store facts across sessions.

## Create Database

```python id="mem2"
import sqlite3

conn = sqlite3.connect("memory.db")

conn.execute("""
CREATE TABLE IF NOT EXISTS memory(
    key TEXT PRIMARY KEY,
    value TEXT
)
""")

conn.commit()
```

---

## Save Memory

```python id="mem3"
def save_memory(key, value):

    conn.execute(
        """
        INSERT OR REPLACE INTO memory
        VALUES (?,?)
        """,
        (key, value)
    )

    conn.commit()
```

---

## Read Memory

```python id="mem4"
def get_memory(key):

    cursor = conn.execute(
        """
        SELECT value
        FROM memory
        WHERE key=?
        """,
        (key,)
    )

    row = cursor.fetchone()

    return row[0] if row else None
```

---

## Example Usage

```python id="mem5"
save_memory("favorite_city", "Hyderabad")

city = get_memory("favorite_city")
print(city)
```

---

# 3. Tool-Based Memory

In this approach, memory is exposed as tools that the LLM can call.

## Concept

Instead of manually storing data, the model decides when to store or retrieve memory.

---

## Example Tools

```python id="mem6"
def save_memory(key, value):
    ...

def recall_memory(key):
    ...
```

---

## Tool Schema (Conceptual)

```python id="mem7"
tools = [
    {
        "type": "function",
        "function": {
            "name": "save_memory"
        }
    },
    {
        "type": "function",
        "function": {
            "name": "recall_memory"
        }
    }
]
```

---

## How It Works

User:

```
My favorite stock is Nimbus
```

Model decides:

```
save_memory(key="favorite_stock", value="Nimbus")
```

Later:

User:

```
What's my favorite stock?
```

Model decides:

```
recall_memory(key="favorite_stock")
```

---

# 4. Semantic Memory (Vector Database)

Semantic memory stores meaning instead of exact keys.

## Key Idea

Instead of exact matching, use embeddings + similarity search.

---

## Example Memories

* I like Hyderabad
* I work in finance
* I own Apple stock

---

## Using Chroma

```python id="mem8"
from chromadb import PersistentClient

client = PersistentClient("memory")

collection = client.get_or_create_collection(
    "agent_memory"
)

collection.add(
    ids=["1"],
    documents=[
        "User likes Hyderabad"
    ]
)
```

---

## Querying Memory

```python id="mem9"
results = collection.query(
    query_texts=[
        "preferred city"
    ],
    n_results=3
)
```

---

## Key Benefit

Works even when user phrasing changes:

* "Which city do I prefer?"
* "Where do I like living?"

Both retrieve: *Hyderabad*

---

# 5. LangChain Memory

LangChain provides built-in memory abstractions.

## Conversation Memory

```python id="mem10"
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(
    return_messages=True
)
```

---

## Using with Agent

```python id="mem11"
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    memory=memory
)
```

---

## What Happens

LangChain automatically:

* Stores conversation history
* Appends it to prompts
* Maintains context continuity

---

# 6. CrewAI Memory

CrewAI supports memory natively for agents.

## Enable Memory

```python id="mem12"
crew = Crew(
    agents=[agent],
    tasks=[task],
    memory=True
)
```

---

## With Embeddings

```python id="mem13"
crew = Crew(
    agents=[agent],
    tasks=[task],
    memory=True,
    embedder={
        "provider": "openai"
    }
)
```

---

## Behavior

CrewAI automatically:

* Stores interactions
* Retrieves relevant past context
* Injects memory into tasks

---

# 7. AutoGen Memory

AutoGen uses tool-based memory patterns.

## Example

```python id="mem14"
memory_store = []

def save_memory(text):
    memory_store.append(text)

def search_memory(query):
    ...
```

---

## Register as Tools

Agents can call these functions just like any other tool:

* Weather tool
* Stock tool
* Memory tool

---

## Key Idea

Memory becomes just another callable capability.

---

# 8. Production Memory Architecture

A real-world AI system typically uses layered memory.

```text id="mem15"
                User
                  │
                  ▼
             Agent (LLM)
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
 Short-Term   Long-Term     Tools
 Memory       Memory
      │           │
      ▼           ▼
 Conversation  Vector DB
 History       (Chroma,
               Pinecone,
               Weaviate)
```

Architecture Diagrams
```text id="mem150"
User
  ↓
Orchestrator Agent
  ↓
┌───────────────┬───────────────┐
│ Short-term     │ Long-term     │
│ memory         │ memory        │
└───────────────┴───────────────┘
        ↓
   Vector DB + Cache
```

---

## Flow

1. User asks a question
2. Retrieve relevant memories (vector DB)
3. Inject into prompt
4. Call LLM
5. Extract new facts
6. Store updated memory

---

## Why This Works

This is called:

> **Retrieval-Augmented Memory (RAM)**

It scales better than:

* Sending full chat history
* Storing everything in context windows

---

# Key Takeaways

## Memory Types Summary

| Type         | Purpose                 |
| ------------ | ----------------------- |
| Conversation | Short-term context      |
| Persistent   | Cross-session facts     |
| Semantic     | Meaning-based retrieval |
| Working      | Intermediate reasoning  |

---

## When to Use What

* Simple chatbot → in-memory history
* Personal assistant → SQLite or DB memory
* Smart assistant → vector DB memory
* Production system → hybrid (all layers combined)

---

## Final Insight

Modern AI agents don’t rely on a single memory system.

They combine:

* Short-term context
* Long-term structured memory
* Semantic retrieval
* Tool-based memory operations

to behave more like persistent intelligent systems rather than stateless chat models.
