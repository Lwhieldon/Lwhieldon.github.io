---
title: "🧠 Building a Conversational Catalog Assistant with Gemini & GenAI"
date: 2025-04-10
author: "Your Name"
layout: post
tags: [GenAI, Kaggle, Gemini, PDF QA, NLP, Catalog Automation]
---

## 🔍 Making Complex Product Docs Conversational – Kaggle GenAI Capstone

In the world of commercial sales and enterprise catalogs, dense product documents often slow down business. This blog walks through my submission to the [Kaggle GenAI Intensive Capstone 2025Q1](https://www.kaggle.com/competitions/gen-ai-intensive-course-capstone-2025q1), where I built a **Generative AI-powered Question Answering Assistant** for the **Herman Miller furniture catalog**.

---

## 🚀 The Real-World Problem

Commercial furniture catalogs like Herman Miller’s are massive PDFs with hundreds of configurable parts, options, finishes, and pricing details. Sales teams and designers frequently face questions like:

- _“What is part FT123 and how much does it cost with metallic paint?”_
- _“What’s the difference between FT110 and FT111?”_
- _“Does this assembly support MicrobeCare™?”_

Manually searching for answers in the PDF takes time and invites errors. That’s where GenAI steps in.

---

## 💡 The Solution: A Gemini-Powered Product Assistant

This project creates an intelligent assistant that interprets catalog PDFs and answers natural language queries using:

- 🧾 PDF parsing and OCR (for image-based text)
- 🔍 TF-IDF-based vector search
- ✨ Context-rich Gemini Pro prompts
- 🖼️ Structured answers with pricing, specs, and image references

It transforms a static document into an interactive, grounded chatbot.

---

## 🧠 How the System Works

Here’s how the notebook solves the problem:

### 1. PDF & OCR Ingestion
- Uses `PyPDF2` and Tesseract (optional) to extract text and part numbers from both normal and image-only pages.

### 2. Chunking + TF-IDF Retrieval
- Splits content into logical chunks (by part number, description, etc.)
- Indexes with `scikit-learn` TF-IDF to find relevant sections based on user queries

### 3. Prompt Assembly
- Retrieved content (tables, features, images) is formatted into Gemini-ready prompts
- Adds product structure and configuration logic to guide the model

### 4. Gemini Pro Integration
- Uses Google’s `generativeai` Python SDK to submit structured prompts
- Returns grounded, formatted responses with pricing tables, citations, or bullet summaries

### 5. (Optional Frontend)
- In a full app version, Streamlit can visualize responses with embedded images and links

---

## 🌟 Why This Is Innovative

| Feature            | Innovation |
|--------------------|------------|
| 📄 OCR + RAG       | Links extracted image text (e.g. FT123) with price/spec data |
| 🧠 Multimodal Reasoning | Handles image + text prompts in a single QA flow |
| 🧾 Structured Answers | Gemini returns tabular pricing or comparisons, not just plain text |
| ⚙️ Product Logic | Understands dependencies like finishes, brackets, assemblies |

---

## 🎯 Why GenAI Was a Great Fit

| Need                     | GenAI Advantage |
|--------------------------|-----------------|
| Spec-heavy product docs  | Explains in human language |
| Ambiguous config options | Handles context-aware prompts |
| Dense, mixed-format PDFs | Works with OCR, tables, and prose |
| Flexible natural language | Supports varied questions like “show me finishes” or “compare X and Y” |

---

## 📌 Capstone Reflection

This project isn’t just a proof-of-concept — it’s a template for how enterprises can use GenAI to **automate knowledge access** buried in documents. From furniture catalogs to insurance policies and tech specs, this kind of assistant can change how we retrieve and interact with structured content.

---

## 🛠️ Try It Yourself

Check out the full notebook on Kaggle:
👉 [GenAI Intensive Capstone 2025Q1 Submission](https://www.kaggle.com/competitions/gen-ai-intensive-course-capstone-2025q1)

Or fork it and plug in your own PDF specs, datasheets, or pricing docs.

---

## 💬 What’s Next?

- Add multilingual support
- Replace TF-IDF with embeddings (e.g. Gemini or SentenceTransformers)
- Enable voice-based querying via Whisper or Gemini Audio

Got ideas? Feedback? Want to collab? Drop me a message!

