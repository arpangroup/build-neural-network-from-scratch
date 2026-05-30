# A2A Protocol (Agent-to-Agent Protocol) — Complete Guide with Real-World Examples

# Table of Contents

1. [Introduction](#1-introduction)
2. [Why A2A Exists](#2-why-a2a-exists)
3. [What Problem Does A2A Solve?](#3-what-problem-does-a2a-solve)
4. [Core Architecture](#4-core-architecture)
5. [Real-World E-Commerce Example](#5-real-world-e-commerce-example)
6. [Core A2A Concepts](#6-core-a2a-concepts)
   - [Agent Card](#61-agent-card)
   - [Task](#62-task)
   - [Message](#63-message)
   - [Artifact](#64-artifact)
7. [End-to-End Workflow](#7-end-to-end-workflow)
8. [**Python Implementation**](#8-python-implementation)
   - [Research Agent](#research-agent)
   - [Summary Agent](#summary-agent)
   - [Orchestrator Agent](#orchestrator-agent)
9. [Async Multi-Agent Execution](#9-async-multi-agent-execution)
10. [Travel Booking Example](#10-travel-booking-example)
11. [A2A vs MCP](#11-a2a-vs-mcp)
12. [Production Architecture](#12-production-architecture)
13. [Benefits of A2A](#13-benefits-of-a2a)
14. [Challenges](#14-challenges)
15. [Security Considerations](#15-security-considerations)
16. [Enterprise Use Cases](#16-enterprise-use-cases)
17. [Best Practices](#17-best-practices)
18. [Complete Workflow Diagram](#18-complete-workflow-diagram)
19. [**Complete A2A Implementation**](#19-complete-a2a-implementation)
20. [Conclusion](#20-conclusion)

---

# 1. Introduction

The **A2A (Agent-to-Agent) Protocol** is an open communication protocol that enables AI agents to communicate, collaborate, and exchange work regardless of the framework, language, or vendor they are built on.

Think of A2A as:

> HTTP for AI Agents

Just as HTTP allows web applications to communicate, A2A allows AI agents to communicate and cooperate.

Instead of one giant AI attempting to perform every task, multiple specialized AI agents can work together.

Examples:

* Research Agent → gathers information
* Coding Agent → writes software
* Finance Agent → performs financial analysis
* Travel Agent → books flights and hotels
* Customer Support Agent → handles user issues

Each agent specializes in a specific domain and collaborates with other agents through A2A.

---

# 2. Why A2A Exists

Modern AI systems are becoming increasingly complex.

A single AI assistant may need to:

* Search information
* Access databases
* Analyze documents
* Generate reports
* Book services
* Execute business workflows

Building one massive agent for all tasks creates problems:

* Difficult maintenance
* Limited scalability
* Poor specialization
* Vendor lock-in
* Reduced flexibility

A2A solves this by enabling:

* Agent specialization
* Independent deployment
* Distributed execution
* Cross-platform interoperability

---

# 3. What Problem Does A2A Solve?

Without A2A:

```text
User
  |
  v
Huge Monolithic AI Agent
  |
  +--> Search
  +--> Coding
  +--> Finance
  +--> Travel
  +--> Reporting
```

Problems:

* Large codebase
* Difficult updates
* Single point of failure
* Hard to scale

With A2A:

```text
User
  |
  v
Coordinator Agent
  |
  +--> Research Agent
  |
  +--> Coding Agent
  |
  +--> Finance Agent
  |
  +--> Reporting Agent
```

Benefits:

* Modular design
* Easier maintenance
* Better scalability
* Independent deployment

---

# 4. Core Architecture

A2A systems generally follow a distributed architecture.

```text
+----------------+
|     User       |
+----------------+
         |
         v
+----------------+
| Coordinator    |
| Agent          |
+----------------+
      / | \
     /  |  \
    v   v   v

+------+ +------+ +------+
|Agent | |Agent | |Agent |
|  A   | |  B   | |  C   |
+------+ +------+ +------+
```

The coordinator:

1. Receives user request
2. Breaks work into tasks
3. Delegates tasks
4. Collects responses
5. Produces final result

---

# 5. Real-World E-Commerce Example

Customer Request:

```text
Find the best laptop under $1000 and generate a recommendation report.
```

### Agent Network

```text
Customer
   |
   v
Shopping Agent
   |
   +--> Pricing Agent
   |
   +--> Review Agent
   |
   +--> Report Agent
```

### Workflow

#### Step 1

Shopping Agent receives request.

```json
{
  "budget": 1000,
  "product": "laptop"
}
```

#### Step 2

Pricing Agent searches products.

Response:

```json
{
  "products": [
    {
      "name": "Dell Inspiron",
      "price": 899
    },
    {
      "name": "HP Pavilion",
      "price": 949
    }
  ]
}
```

#### Step 3

Review Agent analyzes reviews.

```json
{
  "dell_score": 4.5,
  "hp_score": 4.3
}
```

#### Step 4

Report Agent generates report.

```json
{
  "recommendation": "Dell Inspiron"
}
```

#### Step 5

Shopping Agent combines everything.

```json
{
  "recommended_product": "Dell Inspiron",
  "price": 899,
  "rating": 4.5
}
```

Returned to customer.

---

# 6. Core A2A Concepts

---

## 6.1 Agent Card

An Agent Card is a capability advertisement.

It tells other agents:

* Who am I?
* What can I do?
* How can you contact me?

Example:

```json
{
  "name": "ReviewAgent",
  "description": "Analyzes customer reviews",
  "skills": [
    "review_analysis",
    "sentiment_analysis"
  ],
  "endpoint": "http://localhost:8001"
}
```

Purpose:

* Agent discovery
* Capability matching
* Dynamic collaboration

---

## 6.2 Task

A Task is a unit of work sent between agents.

Example:

```json
{
  "task": "review_analysis",
  "input": {
    "product": "Dell Inspiron"
  }
}
```

Tasks define:

* Required action
* Input data
* Expected outcome

---

## 6.3 Message

Messages carry communication.

Example:

```json
{
  "from": "ShoppingAgent",
  "to": "ReviewAgent",
  "message": "Analyze Dell Inspiron reviews"
}
```

Messages can contain:

* Instructions
* Metadata
* Status updates
* Results

---

## 6.4 Artifact

Artifacts are outputs generated by agents.

Example:

```json
{
  "artifact": {
    "summary": "Average rating 4.5/5"
  }
}
```

Artifacts may include:

* Reports
* PDFs
* Images
* JSON results
* Summaries
* Code files

---

# 7. End-to-End Workflow

Consider:

```text
Research topic:
Artificial Intelligence
```

Workflow:

```text
User
 |
 v
Orchestrator Agent
 |
 +--> Research Agent
 |
 +--> Summary Agent
 |
 v
Final Response
```

Sequence:

1. User submits topic
2. Research Agent gathers information
3. Summary Agent summarizes
4. Orchestrator combines output
5. User receives result

---

# 8. Python Implementation

---

## Research Agent

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/task")
async def receive_task(data: dict):

    topic = data["topic"]

    result = {
        "topic": topic,
        "content": f"Research results about {topic}"
    }

    return result
```

Run:

```bash
uvicorn research_agent:app --port 8001
```

---

## Summary Agent

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/task")
async def receive_task(data: dict):

    content = data["content"]

    summary = content[:50] + "..."

    return {
        "summary": summary
    }
```

Run:

```bash
uvicorn summary_agent:app --port 8002
```

---

## Orchestrator Agent

```python
import requests

topic = "Artificial Intelligence"

research = requests.post(
    "http://localhost:8001/task",
    json={
        "topic": topic
    }
).json()

summary = requests.post(
    "http://localhost:8002/task",
    json={
        "content": research["content"]
    }
).json()

print(summary)
```

Output:

```json
{
  "summary": "Research results about Artificial Intelligence..."
}
```

This demonstrates a simple A2A workflow.

---

# 9. Async Multi-Agent Execution

Production systems execute tasks in parallel.

Example:

```python
import asyncio
import httpx

async def get_flights():

    async with httpx.AsyncClient() as client:

        r = await client.post(
            "http://localhost:8001/task",
            json={
                "task": "flight_search"
            }
        )

        return r.json()

async def get_hotels():

    async with httpx.AsyncClient() as client:

        r = await client.post(
            "http://localhost:8002/task",
            json={
                "task": "hotel_search"
            }
        )

        return r.json()

async def main():

    flights, hotels = await asyncio.gather(
        get_flights(),
        get_hotels()
    )

    print({
        "flights": flights,
        "hotels": hotels
    })

asyncio.run(main())
```

Benefits:

* Faster execution
* Better resource utilization
* Improved user experience

---

# 10. Travel Booking Example

User says:

```text
Book a trip from Hyderabad to Bangalore next weekend.
```

---

## Travel Agent

Receives:

```json
{
  "intent": "trip_booking"
}
```

---

## Flight Agent

Receives:

```json
{
  "task": "find_flights",
  "from": "Hyderabad",
  "to": "Bangalore"
}
```

Returns:

```json
{
  "airline": "IndiGo",
  "price": 4500
}
```

---

## Hotel Agent

Receives:

```json
{
  "task": "find_hotel",
  "city": "Bangalore"
}
```

Returns:

```json
{
  "hotel": "ABC Residency",
  "price": 2500
}
```

---

## Final Aggregation

Travel Agent combines:

```json
{
  "flight": {
    "airline": "IndiGo",
    "price": 4500
  },
  "hotel": {
    "hotel": "ABC Residency",
    "price": 2500
  },
  "total_cost": 7000
}
```

User sees one response.

Multiple agents collaborated behind the scenes.

---

# 11. A2A vs MCP

| Feature       | A2A                                  | MCP                    |
| ------------- | ------------------------------------ | ---------------------- |
| Full Form     | Agent-to-Agent Protocol              | Model Context Protocol |
| Purpose       | Agent communication                  | Tool communication     |
| Focus         | Multi-agent collaboration            | Tool integration       |
| Communication | Agent ↔ Agent                        | Model ↔ Tool           |
| Example       | Research Agent talks to Coding Agent | LLM queries database   |
| Scope         | Distributed agents                   | External tools         |

---

# 12. Production Architecture

A typical enterprise architecture:

```text
                    User
                      |
                      v
              +----------------+
              | Support Agent  |
              +----------------+
                /      |      \
               /       |       \
              v        v        v

          MCP      MCP       A2A
           |         |         |
           v         v         v

      Database     CRM    Billing Agent
                             |
                             v
                        Shipping Agent
```

Explanation:

Support Agent:

* Uses MCP for database access
* Uses MCP for CRM access
* Uses A2A for billing
* Uses A2A for shipping

Combines all responses.

Returns one answer.

---

# 13. Benefits of A2A

### Specialization

Agents become experts.

### Scalability

Deploy independently.

### Reusability

One agent serves many systems.

### Interoperability

Works across vendors.

### Fault Isolation

One failure doesn't break everything.

### Parallel Processing

Multiple tasks execute simultaneously.

---

# 14. Challenges

### Discovery

Finding the right agent.

### Authentication

Verifying agent identity.

### Trust

Ensuring agent reliability.

### Latency

Network communication adds delay.

### Coordination

Complex workflows require orchestration.

### Error Handling

Failures must be managed gracefully.

---

# 15. Security Considerations

Production A2A systems require:

## Authentication

```text
OAuth
JWT
API Keys
mTLS
```

## Authorization

Control which agents can call others.

## Encryption

```text
HTTPS
TLS
```

## Auditing

Track:

* Requests
* Responses
* Decisions
* Artifacts

---

# 16. Enterprise Use Cases

## Customer Support

Support Agent

Calls:

* Billing Agent
* Shipping Agent
* Returns Agent

---

## Healthcare

Medical Coordinator

Calls:

* Diagnosis Agent
* Insurance Agent
* Appointment Agent

---

## Finance

Finance Coordinator

Calls:

* Risk Agent
* Fraud Agent
* Portfolio Agent

---

## Software Development

Engineering Agent

Calls:

* Research Agent
* Coding Agent
* Testing Agent
* Documentation Agent

---

# 17. Best Practices

### Keep Agents Small

Single responsibility.

### Define Clear Contracts

Use consistent schemas.

### Use Versioning

```json
{
  "version": "1.0"
}
```

### Implement Retries

Handle transient failures.

### Use Async Execution

Improve performance.

### Monitor Everything

Track latency and failures.

### Secure Communications

Always use authentication and encryption.

---

# 18. Complete Workflow Diagram

```text
User
 |
 v
+------------------+
| Coordinator      |
| Agent            |
+------------------+
      |
      |
      +----------------+
      |                |
      v                v

+-------------+   +-------------+
| Research    |   | Review      |
| Agent       |   | Agent       |
+-------------+   +-------------+
      |                |
      +--------+-------+
               |
               v

       +---------------+
       | Report Agent  |
       +---------------+
               |
               v

       Final Artifact
               |
               v

            User
```

---

# 19. Complete A2A Implementation
a2a_demo.py
```python
import asyncio
import threading
import uvicorn
import httpx

from fastapi import FastAPI

# =====================================================
# Research Agent
# =====================================================

research_agent = FastAPI()


@research_agent.get("/.well-known/agent.json")
async def research_card():
    return {
        "name": "ResearchAgent",
        "description": "Researches topics",
        "skills": [
            {
                "id": "research",
                "description": "Research any topic"
            }
        ],
        "endpoint": "http://localhost:8001/tasks"
    }


@research_agent.post("/tasks")
async def research_task(task: dict):
    topic = task["input"]["topic"]

    await asyncio.sleep(2)

    return {
        "artifact": {
            "type": "research_result",
            "content": f"{topic} is growing rapidly in enterprise AI systems."
        }
    }


# =====================================================
# Summary Agent
# =====================================================

summary_agent = FastAPI()


@summary_agent.get("/.well-known/agent.json")
async def summary_card():
    return {
        "name": "SummaryAgent",
        "description": "Summarizes text",
        "skills": [
            {
                "id": "summarize",
                "description": "Summarize documents"
            }
        ],
        "endpoint": "http://localhost:8002/tasks"
    }


@summary_agent.post("/tasks")
async def summary_task(task: dict):
    text = task["input"]["text"]

    await asyncio.sleep(1)

    return {
        "artifact": {
            "type": "summary",
            "content": text[:50] + "..."
        }
    }


# =====================================================
# Helper to run servers
# =====================================================

def run_server(app, port):
    uvicorn.run(app, host="0.0.0.0", port=port)


# =====================================================
# A2A Orchestrator
# =====================================================

async def discover_agent(base_url):
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"{base_url}/.well-known/agent.json"
        )

        return response.json()


async def execute_task(endpoint, payload):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            endpoint,
            json=payload
        )

        return response.json()


async def orchestrator():
    print("\n=== AGENT DISCOVERY ===\n")

    research_card = await discover_agent(
        "http://localhost:8001"
    )

    summary_card = await discover_agent(
        "http://localhost:8002"
    )

    print(research_card)
    print(summary_card)

    print("\n=== SEND TASK TO RESEARCH AGENT ===\n")

    research_result = await execute_task(
        research_card["endpoint"],
        {
            "task_id": "task-1",
            "skill": "research",
            "input": {
                "topic": "A2A Protocol"
            }
        }
    )

    print(research_result)

    research_text = research_result["artifact"]["content"]

    print("\n=== SEND RESULT TO SUMMARY AGENT ===\n")

    summary_result = await execute_task(
        summary_card["endpoint"],
        {
            "task_id": "task-2",
            "skill": "summarize",
            "input": {
                "text": research_text
            }
        }
    )

    print(summary_result)

    print("\n=== FINAL OUTPUT ===\n")

    print(summary_result["artifact"]["content"])


# =====================================================
# MAIN
# =====================================================

if __name__ == "__main__":

    threading.Thread(
        target=run_server,
        args=(research_agent, 8001),
        daemon=True
    ).start()

    threading.Thread(
        target=run_server,
        args=(summary_agent, 8002),
        daemon=True
    ).start()

    import time
    time.sleep(2)

    asyncio.run(orchestrator())
```

### What happens?
### 1. Agent Discovery

Orchestrator calls:
```http
GET /.well-known/agent.json
```
Research Agent returns:
```json
{
  "name": "ResearchAgent",
  "skills": [
    {
      "id": "research"
    }
  ]
}
```

Summary Agent returns:
```json
{
  "name": "SummaryAgent",
  "skills": [
    {
      "id": "summarize"
    }
  ]
}
```

### 2. Task Exchange
Research task:
```json
{
  "task_id": "task-1",
  "skill": "research",
  "input": {
    "topic": "A2A Protocol"
  }
}
```
Response:
```json
{
  "artifact": {
    "type": "research_result",
    "content": "A2A Protocol is growing rapidly..."
  }
}
```

### 3. Agent-to-Agent Handoff
Output of Agent 1 becomes input of Agent 2:
```json
{
  "task_id": "task-2",
  "skill": "summarize",
  "input": {
    "text": "A2A Protocol is growing rapidly..."
  }
}
```

---

# 20. Conclusion

A2A (Agent-to-Agent Protocol) is a foundational technology for building collaborative AI ecosystems.

Instead of creating one massive AI system, organizations can build specialized agents that communicate through a common protocol.

A2A enables:

* Agent discovery
* Task delegation
* Message exchange
* Artifact sharing
* Multi-agent workflows
* Cross-platform interoperability



In modern AI architectures:

* **A2A connects agents to agents**
* **MCP connects models to tools**

Together, they form the basis of scalable, enterprise-grade AI systems where specialized agents collaborate to solve complex real-world problems efficiently.

**Skills:** In A2A, skills are the capabilities an agent advertises to other agents.
- **Agent Card** = "Here's who I am."
- **Skills** = "Here's what I can do."
- **Task** = "Please do this specific thing."
- **Artifact** = "Here's the result."