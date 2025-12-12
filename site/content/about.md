---
title: "About Candidex"
ShowToc: false
ShowBreadCrumbs: false
---

![Candidex Team](/images/image2.png)

## What is Candidex?

**Candidex** is an AI-powered candidate matching system that transforms how HR departments find talent. Instead of relying on keyword matching that misses great candidates, Candidex uses Large Language Models and semantic search to understand what recruiters actually need.

## The Story Behind Candidex

Every HR professional knows the frustration: hundreds of CVs land in your inbox, but finding the right candidate feels like searching for a needle in a haystack. Traditional applicant tracking systems filter by keywords, missing brilliant candidates who simply used different terminology.

We built Candidex to solve this problem. This blog documents our journey—from the initial prototype built on Azure with C#, through a complete architectural migration to AWS and Python, to the semantic search system that now powers intelligent candidate discovery.

Along the way, we learned hard lessons about cloud architecture, the real capabilities of modern LLMs, and what it takes to build AI-first software that actually works in production.

## The Technology

We started with Microsoft Azure's AI services: Document Intelligence for parsing CVs, Cognitive Search for indexing, and a C# backend tying it all together. It worked, but as the LLM landscape evolved, we saw an opportunity to build something simpler and more powerful.

Today, Candidex runs on a streamlined stack: Python with FastAPI on the backend, AWS S3 for storage, and OpenAI's GPT-4 for intelligent extraction. But the real magic is in the semantic search—vector embeddings that understand meaning, not just match words.

When you search for "experienced backend developer comfortable with cloud infrastructure," Candidex finds candidates who match that description, even if their CV never uses those exact words. That's the difference between keyword matching and true understanding.

## Why This Matters

Traditional keyword search fails in predictable ways. "Python developer" doesn't match "software engineer skilled in Python." "Cloud experience" misses candidates with AWS certifications. Different languages describe identical skills completely differently.

These aren't edge cases—they're the norm. And every missed match represents a qualified candidate who never got a fair shot, and a company that never found the person they were looking for.

Candidex uses vector embeddings to bridge this gap. The result is better candidates, found faster, with less manual screening.

## Connect

Find us on [GitHub](https://github.com/MaryGS/hr-helper)

---

*Built with Hugo and hosted on AWS CloudFront*
