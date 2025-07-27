+++
title = 'What is RAG ?'
draft = false
+++

Large Language Models (LLMs) are impressively smart—they can write essays, explain complex topics, even draft code. But when it comes to answering questions based on **the latest information** or **your private data**, they run into some pretty big roadblocks.

### So, what’s the problem?

1. **They can’t access recent data**  
   Most LLMs are trained on data that cuts off at a specific point (for example, September 2021 or January 2023). If something happened after that, they have no idea.

2. **Retraining is hard (and expensive)**  
   Updating a model with new information means retraining it, which is resource-heavy, time-consuming, and not practical to do frequently—especially if your data changes every week or day.

3. **Hallucinations**  
   Sometimes LLMs "make things up"—they generate answers that sound right but aren’t based on any real data. This can be risky in areas like healthcare, law, or internal business use.

---

### This is where RAG comes in

**RAG (Retrieval-Augmented Generation)** is a smart workaround to all these issues. Instead of trying to retrain the model every time your data changes, RAG gives the model a way to **look things up on the fly**.

Here’s how it works in simple terms:

- You store your actual data (documents, notes, PDFs, internal wikis, etc.) in a way the model can search through.
- When you ask a question, the system first **retrieves** relevant pieces of that data.
- Then the language model uses those pieces to **generate** a response—grounded in real, up-to-date information.

It’s like giving the model a set of cheat sheets to refer to before answering.

---

## Chunking: Breaking Down Data the Smart Way

Before feeding your documents into a Retrieval-Augmented Generation (RAG) system, there’s one crucial step: **chunking**.

LLMs have a limit on how much text they can “see” at once—this is called the **token limit**. If your documents are too big, they can’t all fit in the model’s context window. That’s where chunking comes in: we **split large documents into smaller pieces**, or "chunks", that the model can handle.

But here’s the tricky part—how you chunk the data **really matters**.

### Why chunking needs balance

- **Large chunks** can go over the token limit or overwhelm the model. Important parts in the middle might get lost or ignored.
- **Tiny chunks** are easier to fit, but can lose context and meaning, making it harder to generate accurate answers.

The sweet spot? It depends on the **nature of your data**. A technical manual might work well split by sections or headers, while chat logs may need to be grouped by conversation turns.

### Different ways to chunk

There’s no one-size-fits-all approach. Some common methods include:

- **Character-based splitting**: Break by a set number of characters.
- **Sentence-based splitting**: Break on sentence boundaries for more natural chunks.
- **Semantic splitting**: Use meaning and structure (via embeddings or ML models) to split at logical points in the text.

If you're curious to explore different chunking strategies in practice, here's a fantastic tutorial showing five levels of text splitting:  
[5 Levels of Text Splitting - RetrievalTutorials on GitHub](https://github.com/FullStackRetrieval-com/RetrievalTutorials/blob/main/tutorials/LevelsOfTextSplitting/5_Levels_Of_Text_Splitting.ipynb)

---

Chunking may seem like a small step, but it plays a huge role in the accuracy and efficiency of your RAG pipeline.

---


## What Are Embeddings?

Once your data is chunked, the next step is to make it **searchable** for your RAG system. This is where **embeddings** come in.

Think of embeddings as a way to turn words, sentences, or entire documents into numbers—specifically, **vectors** (which are just long lists of numbers). These numbers capture the **meaning** of the text, so that similar chunks end up with similar embeddings.

### Why embeddings matter

You can’t compare two pieces of text directly in raw form (e.g., “How to cook rice” vs. “Steps to boil white rice”), but once they’re turned into embeddings, the system can measure how **similar** they are—kind of like measuring distance between points on a map.

So when you ask a question like _“How do I cook rice?”_, the RAG system can search through your embedded chunks and find the ones closest in meaning—even if the wording is totally different.

---

## Models Used to Create Embeddings

There are several popular models that generate embeddings, and each comes with trade-offs in terms of **quality, speed, and cost**.

Here are a few common ones:

### 🔹 OpenAI’s `text-embedding-3-small` or `text-embedding-ada-002`
- **Pros**: High-quality, fast, easy to integrate.
- **Cons**: Paid API, so cost can add up for large-scale use.

### 🔹 HuggingFace models (e.g. `all-MiniLM-L6-v2`)
- **Pros**: Open-source, free to run locally or on your own servers.
- **Cons**: Slightly lower performance than OpenAI for some tasks; requires some setup.

### 🔹 Cohere, Google, and others
- **Pros**: Strong performance; often tuned for different use cases (e.g., longer texts).
- **Cons**: Varies by provider—some are paid, some have limitations.

---

## Indexing Embeddings and FAISS

So now we’ve got our text chunks turned into embeddings—cool! But what happens when a user asks a question? How does the system **quickly find** the most relevant chunks from possibly thousands (or millions) of embeddings?

That’s where **indexing** comes in.

### Why Indexing Matters

Imagine you’ve got a massive list of coordinates (your embeddings), and someone asks: “Which of these is closest to this new point?”  
You *could* check every single one, but that would be **slow and inefficient**—especially as your dataset grows.

Instead, we use **special data structures** to speed this up. These structures are called **indexes**, and they’re built to make similarity search fast and scalable.

### Measuring Similarity: Distance Matters

To find similar embeddings, we calculate the **distance** between them. The closer two vectors are, the more similar the texts they represent.

There are a few common distance types:

- **Cosine Similarity**: Measures the angle between two vectors (ignores length). Often used when vectors are normalized.
- **Euclidean Distance**: Measures the straight-line distance between two points.
- **Dot Product**: Used in some models to capture both similarity and magnitude.

> **Normalization** (scaling vectors to unit length) helps make comparisons more stable—especially when using cosine similarity. It ensures you're comparing *direction*, not *size*.

---

### Types of Indexes

Depending on your needs (speed vs. accuracy), you can choose different types of indexes:

- **Flat Index**: Brute-force, checks every vector. Very accurate, but slow for large datasets.
- **IVF (Inverted File Index)**: Groups vectors into clusters—faster but slightly less accurate.
- **HNSW (Hierarchical Navigable Small World)**: Graph-based structure, great for high speed and high accuracy.
- **PQ (Product Quantization)**: Compresses vectors to save space—useful for massive datasets.

---

### Enter FAISS: Facebook AI Similarity Search

[FAISS](https://github.com/facebookresearch/faiss) is an open-source library by Facebook AI that makes all of this practical.

- Built for **fast vector search**, even with millions of embeddings.
- Supports multiple index types (Flat, IVF, HNSW, PQ, and combos).
- Can run on **CPU or GPU**, and works great with Python.
- Easy to integrate into your RAG pipeline.

With FAISS, your system can **instantly retrieve the most relevant text chunks**, even from a massive collection, making your RAG responses snappy and accurate.

---

## Vector Databases: Managing Embeddings at Scale

Once you've got embeddings and an index (like FAISS), you need a way to **store, manage, and search** them effectively—especially in real-world applications where your data changes, grows, or needs to be queried alongside metadata.

This is where **vector databases** come in.

---

### Wait—Can’t We Just Use a Regular Database?

Traditional databases (like PostgreSQL, MongoDB, MySQL) are amazing for structured data—think rows, columns, and filters like `age > 30`. But they weren’t designed for **vector math** or similarity search.

However, things are changing. Many traditional databases are now adding **vector support**:

- **PostgreSQL** has extensions like `pgvector`, which allow you to store and search vector embeddings.
- **MongoDB** introduced native vector search in recent versions.

**Pros:**
- Familiar ecosystem and tools
- Combine structured filters (e.g., tags, categories) with vector search
- Easier for teams already using these databases

**Cons:**
- May not scale well for *very* large vector datasets
- Slower or less optimized compared to dedicated vector DBs

---

### Dedicated Vector Databases

These are built from the ground up to handle vector data—fast search, scaling, and features like hybrid search (vector + keyword). Here are some of the most popular ones:

#### 🔹 **Pinecone**
- **Pros:** Fully managed, fast, supports metadata filtering, scales well
- **Cons:** Paid service; less control if you need full self-hosting

#### 🔹 **Weaviate**
- **Pros:** Open-source or managed, supports semantic and hybrid search, auto chunking, metadata-aware
- **Cons:** Heavier footprint for self-hosting; steeper learning curve

#### 🔹 **Milvus**
- **Pros:** High performance, GPU acceleration, large-scale production-ready
- **Cons:** Needs more infra to set up; more suitable for enterprise use

#### 🔹 **Qdrant**
- **Pros:** Lightweight, easy API, fast, open-source
- **Cons:** Fewer integrations than some competitors (but improving fast)

#### 🔹 **Vespa**, **Vald**, **Zilliz** (Milvus-backed) — also great choices depending on your use case and infra preferences

---

### So… Which One Should You Use?

- **Just getting started?** Try `pgvector` with PostgreSQL or `Qdrant` for a simple, open-source experience.
- **Scaling up?** Look at `Pinecone`, `Weaviate`, or `Milvus`.
- **Need full control?** Go for open-source + FAISS or a self-hosted vector DB.

---

In short, vector databases are the backbone of RAG at scale. They make it possible to **store thousands or millions of embeddings** and search them in milliseconds—often combined with filters like document type, source, or time.

---

## Conclusion: RAG Is Just the Beginning

RAG—Retrieval-Augmented Generation—is a powerful way to make LLMs smarter, more reliable, and grounded in real data. By combining a model’s ability to generate text with the ability to **look things up**, we overcome a lot of the core limitations of standalone LLMs.

We’ve walked through the core steps:

- **Chunking** your data the right way to balance context and size
- **Embedding** those chunks into a form machines can understand
- **Indexing** them for fast and efficient search
- **Using vector databases** to store and manage it all at scale

But that’s not the whole story.

There are plenty of **advanced techniques** that can push RAG even further:

-  **Re-ranking**: After the initial retrieval, you can use another model to sort or refine the top results for better relevance.
-  **Task Decomposition**: You can use one LLM to break down a complex question into smaller tasks, and have another LLM (or the same one) answer each with more precision.
-  **Agents & Recursive Retrieval**: Intelligent agents can dig deeper—asking follow-up questions, retrieving more info in multiple rounds, and stitching it all together to give better answers.
> These strategies make RAG systems more dynamic, more accurate, and more adaptable—especially in complex domains like legal, medical, or enterprise use cases.
---
#### References : <br>
- [Advance RAG - Huggingface cookbook](https://huggingface.co/learn/cookbook/advanced_rag)<br>
- [Augmented Language Models](https://fullstackdeeplearning.com/llm-bootcamp/spring-2023/)
---
