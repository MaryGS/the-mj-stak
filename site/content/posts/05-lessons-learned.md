---
title: "What We Learned Migrating to AI-First HR Tech"
date: 2025-09-28
draft: true
description: "Key lessons, surprises, and advice from building an AI-powered CV matching system"
tags: ["lessons learned", "AI", "software development", "best practices", "retrospective"]
series: ["Candidex"]
weight: 5
ShowToc: true
TocOpen: false
---

*Part 5 of 5 in the HR Helper Blog Series*

---

## Reflecting on the Journey

Over the past four posts, we've covered the problem HR Helper solves, the technical migration from Azure to AWS, how LLMs transform CV processing, and the business case for adoption.

In this final post, we step back and share what we learned—the surprises, the mistakes, the insights that only come from building something real. We also look ahead at what's next for HR Helper and AI-powered recruitment.

---

## Technical Lessons

### Lesson 1: LLMs Are Great Unifiers

**What we expected**: Replacing each Azure service with an equivalent AWS service.

**What happened**: We replaced *five* services with *one* LLM plus storage.

The original architecture used:
- Azure Document Intelligence (CV parsing)
- Azure Cognitive Search (indexing and search)
- Azure Text Analytics (language detection)
- Azure Translator (translation)
- OpenAI API (chat completions)

The new architecture uses:
- AWS S3 (storage)
- OpenAI API (everything else)

A single, well-prompted LLM handles extraction, language detection, translation, query understanding, and response generation. This wasn't our original plan—it emerged as we realized how capable modern LLMs are at diverse tasks.

**Takeaway**: Before adding another specialized service, ask: "Could a good prompt handle this?"

---

### Lesson 2: Prompts Are Your New Codebase

We underestimated how much iteration prompts would require. Our extraction prompt went through 15+ revisions before stabilizing.

**Early prompt** (too vague):
```
Extract information from this CV and return JSON.
```

**Problem**: Inconsistent field names, missing data, hallucinated values.

**Final prompt** (explicit and constrained):
```
Extract the following information from this CV/resume and return it as a JSON object:

CV Text:
{cv_text}

Extract and return a JSON object with these fields:
- Name: Full name
- Email: Email address
[... explicit field list with descriptions ...]

If a field is not found in the CV, set it to an empty string.
Return only valid JSON, no additional text.
```

**Key improvements**:
- Explicit field names (LLM uses exactly these)
- Clear instructions for missing data
- Output format constraint

**Takeaway**: Treat prompts like code. Version control them. Test them. Review changes.

---

### Lesson 3: Temperature Matters More Than You Think

We initially used default temperature settings (0.7-1.0). Results were creative—too creative.

The same CV processed twice would yield different extractions. "Software Engineer" became "Software Developer" in one run and "Programming Professional" in another.

**The fix**: Temperature 0.1 for extraction tasks.

```python
response = await self.client.chat.completions.create(
    model=self.model,
    messages=[...],
    response_format={"type": "json_object"},
    temperature=0.1  # Low temperature = deterministic output
)
```

**When to use different temperatures**:
- **0.1-0.3**: Data extraction, classification, structured output
- **0.5-0.7**: Question answering, summarization
- **0.8-1.0**: Creative writing, brainstorming

**Takeaway**: Match temperature to task. Consistency requires low temperatures.

---

### Lesson 4: Async is Non-Negotiable for LLM Apps

Our first implementation was synchronous. Processing 100 CVs took 15+ minutes (LLM calls are slow).

Switching to async processing with `asyncio` brought that down to under 3 minutes—a 5x improvement.

```python
# Before: Sequential processing
for cv in cv_list:
    result = extract_cv_data(cv)  # Blocks for each call
    results.append(result)

# After: Concurrent processing
tasks = [extract_cv_data(cv) for cv in cv_list]
results = await asyncio.gather(*tasks)
```

**Important**: Respect API rate limits. OpenAI has limits on requests per minute. We added a semaphore:

```python
semaphore = asyncio.Semaphore(10)  # Max 10 concurrent requests

async def rate_limited_extract(cv):
    async with semaphore:
        return await extract_cv_data(cv)
```

**Takeaway**: Design for async from the start. Retrofitting is painful.

---

### Lesson 5: The Frontend Didn't Need to Change

Our biggest fear was cascading changes—new backend means new API contracts means frontend rewrites.

**What we did**: Maintained exact API compatibility.

```python
# Same endpoint paths
@router.post("/api/document/upload")
@router.post("/api/chat/answer")

# Same response structures
return {
    "success": True,
    "data": {
        "Name": extracted["Name"],
        "Email": extracted["Email"],
        # ... same fields as before
    }
}
```

The React frontend never knew the backend changed. This allowed us to:
- Test the new backend without frontend risk
- Roll back quickly if issues arose
- Deploy incrementally

**Takeaway**: API contracts are boundaries. Respect them during migrations.

---

## Business Lessons

### Lesson 6: Start with the Hardest Problem

We could have started by migrating storage (easy) or authentication (familiar). Instead, we started with CV extraction (complex, uncertain).

**Why this was right**:
- If LLM extraction didn't work, the whole project was questionable
- Early validation reduced risk
- Demonstrated value quickly to stakeholders

**Takeaway**: Attack the riskiest assumption first. Everything else is implementation detail.

---

### Lesson 7: Cost Visibility Changes Behavior

With Azure, costs were predictable but opaque—monthly bills for services.

With LLMs, costs are visible and variable—you see tokens consumed per request.

**This changes behavior**:
- Teams optimize prompts to reduce tokens
- Caching becomes a priority
- Model selection becomes a conscious choice

We built a simple cost tracking dashboard:

```python
def log_llm_usage(model: str, input_tokens: int, output_tokens: int):
    cost = calculate_cost(model, input_tokens, output_tokens)
    logger.info(f"LLM call: {model}, in:{input_tokens}, out:{output_tokens}, cost:${cost:.4f}")
```

**Takeaway**: Make costs visible. Visibility drives optimization.

---

### Lesson 8: Multi-Language Support Was Easier Than Expected

We anticipated needing:
- Separate extraction prompts per language
- Translation services for queries
- Language-specific search indexes

**Reality**: The LLM handles it all.

- Spanish CVs extract correctly with the same prompt
- Embeddings capture meaning regardless of language
- A Spanish query finds English CVs and vice versa

**The only addition**: A language field in the extracted data for filtering when needed.

**Takeaway**: Modern LLMs are inherently multilingual. Don't over-engineer language support.

---

## Surprises and Unexpected Benefits

### Surprise 1: Better Error Messages

Azure service errors were often cryptic: "BadRequest: The request is invalid."

LLM-based systems can explain what went wrong:

```python
if not extracted_data.get("Name"):
    logger.warning("Could not extract candidate name. CV may be image-based or heavily formatted.")
```

We even ask the LLM to assess extraction confidence:

```python
# Added to extraction prompt:
"- confidence: Rate your extraction confidence from 0-100. 
   Lower if the CV is poorly formatted, an image, or in an unexpected format."
```

**Benefit**: Better debugging, better user feedback.

---

### Surprise 2: Implicit Skill Inference

We didn't build skill inference—but the LLM does it naturally.

A CV mentioning "built REST APIs with FastAPI" gets extracted with TechnicalSkills including "Python" and "API development"—even if those exact words aren't present.

The LLM understands that FastAPI implies Python. That building APIs implies API development. This implicit reasoning improves search accuracy.

**Benefit**: Richer candidate profiles without additional processing.

---

### Surprise 3: Reduced Maintenance Burden

The Azure system required:
- Document Intelligence model retraining for new CV formats
- Search index schema updates for new fields
- Regular service updates and SDK upgrades

The LLM system requires:
- Prompt updates (text changes)
- Occasional model version upgrades

**Maintenance reduced by ~70%**.

---

## What We'd Do Differently

### Mistake 1: Not Testing Edge Cases Earlier

Our extraction prompt worked great for standard CVs. But then we encountered:
- Image-based PDFs (no extractable text)
- CVs with tables (confused the parser)
- Multi-page CVs with repeated headers

**We should have**: Built a diverse test set early and validated against it continuously.

---

### Mistake 2: Underestimating Token Costs for Long CVs

Some CVs are 5+ pages. At ~800 tokens per page, a long CV costs 4x more than average.

**We should have**: Implemented smart truncation earlier.

```python
def prepare_cv_for_extraction(text: str, max_tokens: int = 4000) -> str:
    """Truncate CV text intelligently, keeping important sections."""
    if estimate_tokens(text) <= max_tokens:
        return text
    
    # Keep first part (usually summary/skills) and recent experience
    sections = split_into_sections(text)
    prioritized = prioritize_sections(sections)
    return combine_within_limit(prioritized, max_tokens)
```

---

### Mistake 3: Building Before Validating Demand

We assumed multi-language support would be heavily used. Turns out, 90% of our early users only needed English.

**We should have**: Talked to more potential users before building language features.

**Lesson**: Build the MVP, then iterate based on actual usage.

---

## Future Roadmap

### Near-Term (Next 3 Months)

**1. Improved Document Handling**
- OCR integration for image-based CVs
- Better table extraction
- Support for more formats (LinkedIn exports, Indeed profiles)

**2. Enhanced Search**
- Faceted search (filter by location, experience level, etc.)
- Saved searches and alerts
- Search history and analytics

**3. Candidate Communication**
- Email templates integrated with extracted data
- Automated status updates
- Response tracking

---

### Medium-Term (3-6 Months)

**4. Interview Scheduling**
- Calendar integration
- Automated scheduling suggestions
- Availability matching

**5. Skills Assessment**
- AI-generated screening questions based on CV
- Technical assessment recommendations
- Skill verification suggestions

**6. Reporting and Analytics**
- Pipeline metrics (time-to-hire, source effectiveness)
- Diversity analytics
- Recruiter productivity dashboards

---

### Long-Term Vision

**7. Predictive Matching**
- Learn from hiring outcomes
- Predict candidate success probability
- Suggest similar successful candidates

**8. Career Path Analysis**
- Map candidate trajectories
- Identify growth potential
- Match for future roles, not just current openings

**9. Market Intelligence**
- Salary benchmarking from aggregated data
- Skill demand trends
- Competitive hiring analysis

---

## For Others Building AI-First Applications

If you're embarking on a similar journey, here's our advice:

### Technical Advice

1. **Prompt engineering is real engineering**. Invest time in it. Document it. Test it.

2. **Start async**. LLM calls are slow. Your architecture needs to handle that.

3. **Cache everything cacheable**. Embeddings, extractions, translations—don't regenerate.

4. **Build cost observability early**. Token costs add up. Know where they're going.

5. **Keep fallbacks**. When the LLM API is down, what happens? Have an answer.

### Business Advice

1. **Validate the core value proposition first**. Everything else is details.

2. **Maintain API compatibility during migrations**. It de-risks everything.

3. **Make costs visible to the team**. Awareness drives optimization.

4. **Talk to users continuously**. Your assumptions are probably wrong.

5. **Ship early, iterate based on feedback**. Perfect is the enemy of deployed.

---

## Closing Thoughts

Building HR Helper has been a journey of discovery. We started with a simple question: *Can we help HR departments find better candidates faster?*

The answer is yes—but the path wasn't what we expected. We replaced complex service architectures with elegant LLM solutions. We found that semantic understanding beats keyword matching. We learned that modern AI doesn't just automate tasks—it fundamentally changes what's possible.

The recruitment industry is just beginning to feel the impact of these technologies. The tools we've built, and the lessons we've shared, are early steps in a larger transformation.

We hope this series has been useful. Whether you're an HR professional exploring AI solutions, a developer building similar applications, or simply someone curious about the intersection of AI and business processes—thank you for reading.

The future of recruitment is intelligent, efficient, and more human than ever. Let's build it together.

---

## Series Summary

| Post | Topic | Key Takeaway |
|------|-------|--------------|
| 1 | The HR Challenge | CVs are overwhelming; keywords aren't enough |
| 2 | Azure to AWS Migration | LLMs unify multiple services; simpler is better |
| 3 | LLM CV Processing | Semantic search finds candidates keywords miss |
| 4 | Business Impact | 75% time savings, 79% cost reduction possible |
| 5 | Lessons Learned | Start with hard problems, iterate quickly |

---

*This concludes the HR Helper Blog Series.*

*Return to [Series Index](/)*

*Previous: [The ROI of AI-Powered Recruitment](/posts/04-business-impact/)*

---

**About HR Helper**: HR Helper is an AI-powered CV matching system that helps HR departments find the best candidates using LLM-based extraction and semantic search. Built with Python, FastAPI, AWS S3, and OpenAI.

**Connect With Us**: [GitHub Repository] | [Documentation] | [Contact]

---

*Thank you for reading. If you found this series valuable, consider sharing it with others who might benefit from these insights.*

