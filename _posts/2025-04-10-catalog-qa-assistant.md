---
title: "🧠 Building a Conversational Catalog Assistant with Gemini & GenAI"
date: 2025-04-10
author: "Lee Whieldon"
layout: post
tags: [GenAI, Kaggle, Gemini, PDF QA, NLP, Catalog Automation]
---

## 🔍 Making Complex Product Docs Conversational – Kaggle GenAI Capstone

In the world of commercial sales and enterprise catalogs, dense product documents often slow down business. This blog walks through my submission to the [Kaggle GenAI Intensive Capstone 2025Q1](https://www.kaggle.com/competitions/gen-ai-intensive-course-capstone-2025q1), where I built a **Generative AI-powered Question Answering Assistant** for the [**Herman Miller Canvas Office Landscape Wall & Private Office catalog**](https://www.hermanmiller.com/content/dam/hermanmiller/documents/pricing/PB_CWB.pdf).

---

## 🚀 The Real-World Problem

Commercial office furniture catalogs like Herman Miller’s are massive PDFs with hundreds of configurable parts, options, finishes, and pricing details. Sales teams and designers frequently face questions like:

- _“What is part FT123 and how much does it cost with metallic paint?”_
- _“What’s the difference between FT110 and FT111?”_
- _“Does this assembly support MicrobeCare™?”_

Manually searching for answers in the PDF takes time and invites errors. That’s where GenAI steps in.

---

## 💡 The Solution: A Gemini-Powered Product Assistant

This project creates an intelligent assistant that interprets catalog PDFs and answers natural language queries using:

- 🧾 PDF parsing
- 🔍 TF-IDF-based vector search
- ✨ Context-rich Gemini Pro prompts
- 🖼️ Structured answers with pricing, specs, and references

It transforms a static document into an interactive, grounded chatbot.

---

## 🧠 How the System Works

Below is a detailed breakdown of the system with code snippets and explanations for each step:

### 1. PDF Ingestion & Text Extraction

```python
from PyPDF2 

# Extract text from a standard (non-image) page of the PDF.
reader = PyPDF2.PdfReader("PB_CWB.pdf")
text = reader.pages[10].extract_text()

```

*Explanation:*  
- **PDF Text Extraction:** The code uses `PyPDF2` to extract text from page 10 of the PDF.  

---

### 2. Chunking + TF-IDF Retrieval

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

# Build a TF-IDF matrix from the text chunks.
vectorizer = TfidfVectorizer().fit_transform(text)

# Prepare a sample query.
query = "FT123 metallic paint price"
query_vec = vectorizer.transform([query])

# Calculate similarity scores between the query and each text chunk.
scores = cosine_similarity(query_vec, vectorizer).flatten()

# Retrieve the chunk with the highest similarity score.
top_chunk = chunks[scores.argmax()]
```

*Explanation:*  
- **TF-IDF Vectorization:** `TfidfVectorizer` converts these text chunks into vectors.  
- **Similarity Scoring:** By comparing the query vector against the indexed chunks using cosine similarity, the most relevant chunk is identified.

---

### 3. Prompt Assembly

```python
# Construct a Gemini Pro prompt using the most relevant content chunk.
prompt = f"""
You are a helpful assistant for Herman Miller. Using the following context, answer the user’s question clearly and accurately.

Context:
{top_chunk}

Question:
What is the price of FT123 with metallic paint?

Answer:
"""
```

*Explanation:*  
- **Context and Query Integration:** The code assembles a prompt that includes the retrieved context (the most relevant chunk), along with the user's question.  
- **Structured Prompt:** This format helps guide Gemini Pro to provide a grounded, well-structured answer based on the extracted content.

---

### 4. Gemini Pro Integration

```python
import google.generativeai as genai

# Configure the Gemini Pro client with your API key.
genai.configure(api_key="YOUR_API_KEY")
model = genai.GenerativeModel("gemini-exp-1206")

# Submit the prompt to Gemini Pro to generate an answer.
response = model.generate_content(prompt)
print(response.text)
```

*Explanation:*  
- **API Configuration:** This code initializes the Gemini Pro client using the Google Generative AI SDK.  
- **Prompt Submission:** The preassembled prompt is sent to Gemini Pro (model name, "gemini-exp-1206"), and the generated answer is printed.  
- **Seamless Integration:** This step demonstrates how to connect the backend processing with a generative model to obtain a structured answer.

---

## 🌟 Why This Is Innovative

| Feature               | Innovation |
|-----------------------|------------|
| 🧠 Multimodal Reasoning | Handles image + text prompts in a single QA flow |
| 🧾 Structured Answers  | Gemini returns tabular pricing or comparisons, not just plain text |
| ⚙️ Product Logic       | Understands dependencies like finishes, brackets, assemblies |

---

## 🎯 Why GenAI Was a Great Fit

| Need                     | GenAI Advantage |
|--------------------------|-----------------|
| Spec-heavy product docs  | Explains in human language |
| Ambiguous config options | Handles context-aware prompts |
| Dense, mixed-format PDFs | Works with text & tables |
| Flexible natural language | Supports varied questions like “show me finishes” or “compare X and Y” |

---

## 📌 Capstone Reflection

This project isn’t just a proof-of-concept — it’s a template for how enterprises can use GenAI to **automate knowledge access** buried in documents. From furniture catalogs to insurance policies and tech specs, this kind of assistant can change how we retrieve and interact with structured content.

---

## 🛠️ Try It Yourself

Check out the competition on Kaggle:  
👉 [Q1 2025 GenAI Intensive Capstone Competition](https://www.kaggle.com/competitions/gen-ai-intensive-course-capstone-2025q1)

Or fork my notebook and plug in your own PDF specs, datasheets, or pricing docs:  
👉 [My GenAI Intensive Capstone 2025Q1 Submission](https://www.kaggle.com/code/leewhieldon/genai-intensive)

---

## 💬 What’s Next?

- Add multilingual support  
- Replace TF-IDF with embeddings (e.g. Gemini or SentenceTransformers)  
- Enable voice-based querying via Whisper or Gemini Audio  

Got ideas? Feedback? Want to collab? Drop me a message!
