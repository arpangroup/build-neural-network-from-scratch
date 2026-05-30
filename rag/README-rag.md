# Retrieval-Augmented Generation (RAG) — Complete Step-by-Step Guide with Real-Time Example and Full Python Implementation


## Vector Database
- FAISS (Facebook AI Similarity Search)
- Pinecone
- Weaviate
- Milvus
- ChromaDB


## Complete Single-File Python Code

```python
pip install sentence-transformers
pip install faiss-cpu
pip install openai
pip install numpy
```

```python
import os
import faiss
import numpy as np
from sentence_transformers import SentenceTransformer
from openai import OpenAI


# =====================================================
# CONFIGURATION
# =====================================================

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
client = OpenAI(api_key=OPENAI_API_KEY)
EMBEDDING_MODEL = "all-MiniLM-L6-v2"
GENERATION_MODEL = "gpt-4o-mini"


# =====================================================
# KNOWLEDGE BASE
# =====================================================

documents = [
    "Employees receive 20 annual leave days per year.",
    "Employees may work remotely up to 3 days per week.",
    "Business travel expenses are reimbursed within 30 days.",
    "Health insurance coverage begins after 30 days of employment.",
    "Employees are eligible for performance bonuses every year."
]


# =====================================================
# LOAD EMBEDDING MODEL
# =====================================================

print("Loading embedding model...")

embedder = SentenceTransformer(EMBEDDING_MODEL)


# =====================================================
# CREATE EMBEDDINGS
# =====================================================

print("Generating document embeddings...")

doc_embeddings = embedder.encode(
    documents,
    convert_to_numpy=True
)

dimension = doc_embeddings.shape[1]


# =====================================================
# BUILD FAISS INDEX
# =====================================================

index = faiss.IndexFlatL2(dimension)

index.add(doc_embeddings)

print(f"Indexed {len(documents)} documents")


# =====================================================
# RETRIEVAL FUNCTION
# =====================================================

def retrieve(query, top_k=3):
    """
    Retrieve top-k relevant documents.
    """

    query_embedding = embedder.encode(
        [query],
        convert_to_numpy=True
    )

    distances, indices = index.search(
        query_embedding,
        top_k
    )

    retrieved_docs = []

    for idx in indices[0]:
        retrieved_docs.append(documents[idx])

    return retrieved_docs


# =====================================================
# PROMPT BUILDER
# =====================================================

def build_prompt(query, context_docs):
    context = "\n".join(context_docs)

    prompt = f"""
You are a helpful assistant.

Answer ONLY from the provided context.

Context:
{context}

Question:
{query}

Answer:
"""

    return prompt


# =====================================================
# GENERATION FUNCTION
# =====================================================

def generate_answer(query):

    retrieved_docs = retrieve(query)

    prompt = build_prompt(
        query,
        retrieved_docs
    )

    response = client.chat.completions.create(
        model=GENERATION_MODEL,
        messages=[
            {
                "role": "user",
                "content": prompt
            }
        ],
        temperature=0
    )

    answer = response.choices[0].message.content

    return answer, retrieved_docs


# =====================================================
# MAIN
# =====================================================

if __name__ == "__main__":

    print("\n===== SIMPLE RAG DEMO =====\n")

    while True:

        question = input(
            "\nAsk a question (or type exit): "
        )

        if question.lower() == "exit":
            break

        answer, docs = generate_answer(question)

        print("\nRetrieved Context:")
        print("-" * 60)

        for d in docs:
            print(f"- {d}")

        print("\nGenerated Answer:")
        print("-" * 60)
        print(answer)
```


## faiss-cpu
> Facebook AI Similarity Search

It is a library made by Meta (Facebook) for fast similarity search over vectors.

It finds the most similar vectors quickly.

```python
import faiss
import numpy as np

d = 384  # embedding size
index = faiss.IndexFlatL2(d)

vectors = np.random.random((10, d)).astype('float32')
index.add(vectors)

query = np.random.random((1, d)).astype('float32')

distances, indices = index.search(query, k=3)

print(indices)
```

## sentence-transformers
**Sentence-Transformers** is a Python library that converts text into dense vector embeddings using transformer-based models like BERT, RoBERTa, etc.

Example:
```
"The cat sat on the mat"
```
Output:
```
[0.12, -0.44, 0.87, ...]  # high-dimensional vector
```

Python Code:
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

emb = model.encode("What is RAG?")
print(len(emb))  # e.g., 384-dimensional vector
```

---

### How the Code Works

### Step1: Index Creation
```
doc_embeddings = embedder.encode(documents)
index.add(doc_embeddings)
```
Creates vector representations and stores them in FAISS.

### Step2: Retrieval
```python
retrieve(query)
```
Returns most similar documents.

### Step3: Prompt Construction
```python
Context:
<retrieved docs>

Question:
<user query>
```
Grounds the model.

### Step4: Generation
```python
client.chat.completions.create(...)
```
