---
title: "About Candidex"
ShowToc: false
ShowBreadCrumbs: false
---

## What is Candidex?

**Candidex** is an AI-powered candidate matching system that transforms how HR departments find talent. Instead of relying on keyword matching that misses great candidates, Candidex uses Large Language Models (LLMs) and semantic search to understand what recruiters actually need.

## The Story

This blog documents our journey building Candidex:

- 🏗️ **The Challenge**: Why traditional CV screening fails at scale
- ☁️ **Azure to AWS**: Migrating from C# and Azure to Python and AWS
- 🤖 **The AI Core**: How LLMs and vector embeddings power semantic search
- 📊 **Real Results**: Actual ROI numbers and cost analysis
- 💡 **Lessons Learned**: What we discovered building AI-first software

## The Technology

### Original Stack (Azure + C#)
- Azure Document Intelligence for CV extraction
- Azure Cognitive Search for indexing
- C# .NET 8.0 backend
- React frontend

### Current Stack (AWS + Python)
- LLM-based extraction (OpenAI GPT-4)
- Vector embeddings for semantic search
- Python FastAPI backend
- AWS S3 for storage
- React frontend (unchanged)

## Why Semantic Search?

Traditional keyword search fails because:
- "Python developer" doesn't match "software engineer skilled in Python"
- "Cloud experience" doesn't find candidates with "AWS certified"
- Different languages describe the same skills differently

Candidex uses **vector embeddings** to understand meaning, not just match words. The result? Better candidates, faster.

## Connect

- [GitHub](https://github.com/your-org)

---

*Built with Hugo • Hosted on AWS CloudFront*
