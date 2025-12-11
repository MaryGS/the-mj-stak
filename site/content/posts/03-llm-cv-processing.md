---
title: "Beyond Keywords: Using LLMs to Actually Understand Resumes"
date: 2025-09-05
draft: true
description: "Deep dive into how Large Language Models and semantic search transform CV processing"
tags: ["LLM", "AI", "semantic search", "embeddings", "NLP"]
series: ["Candidex"]
weight: 3
ShowToc: true
TocOpen: false
---

## The Intelligence Gap

Traditional CV screening systems are sophisticated pattern matchers. They excel at finding exact strings, Boolean combinations, and proximity matches. What they can't do is *understand*.

Consider this search: **"Senior developer comfortable with modern cloud infrastructure"**

A keyword system would look for:
- "Senior" OR "Sr." OR "Lead"
- "Developer" OR "Engineer" OR "Programmer"
- "Cloud" AND ("AWS" OR "Azure" OR "GCP")

But what about a candidate whose CV says:

> *"Led the migration of our monolithic application to a microservices architecture on AWS, implementing CI/CD pipelines and infrastructure-as-code using Terraform."*

This candidate never used the word "cloud." They never said "modern." But they clearly possess exactly what the search describes. A keyword system might miss them. An LLM won't.

This post explores how HR Helper uses Large Language Models and semantic search to bridge the intelligence gap.

---

## How LLMs "Understand" CVs

### From Unstructured Text to Structured Data

CVs are messy. They come in different formats, layouts, languages, and styles. One candidate lists skills in a sidebar; another buries them in job descriptions. One uses bullet points; another writes paragraphs.

Traditional parsing requires rigid templates. LLMs need only instructions:

```python
EXTRACTION_PROMPT = """Extract the following information from this CV/resume 
and return it as a JSON object:

CV Text:
{cv_text}

Extract and return a JSON object with these fields:
- Name: Full name
- Email: Email address
- Phone: Phone number
- ProfessionalSummary: Professional summary or objective
- TechnicalSkills: Technical skills (comma-separated)
- JobRole: Current or most recent job role
- Company: Current or most recent company
- WorkPeriod: Work period or dates
- JobResponsibilities: Job responsibilities
- Education: Education details
- Languages: Languages spoken
- Certifications: Certifications
- Language: Detected language code (e.g., 'en', 'es')

If a field is not found in the CV, set it to an empty string.
Return only valid JSON, no additional text."""
```

The LLM receives this prompt along with raw CV text and returns structured JSON. No templates. No training data. No regex rules. Just natural language instructions.

### The Extraction Pipeline

Here's how a CV flows through our system:

```
┌─────────────────┐
│  CV Upload      │
│  (PDF/DOCX)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Text           │
│  Extraction     │  ← PyPDF2, pdfplumber, python-docx
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM            │
│  Processing     │  ← OpenAI GPT-4 / Claude
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Structured     │
│  JSON Data      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Vector         │
│  Embedding      │  ← OpenAI text-embedding-3-small
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Storage        │
│  (S3 + VectorDB)│
└─────────────────┘
```

### Code Walkthrough: The Extractor

```python
class LLMExtractor:
    """Service for extracting structured CV data using LLM"""
    
    def __init__(self):
        self.client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = settings.OPENAI_MODEL  # e.g., "gpt-4"
    
    async def extract_cv_data(self, cv_text: str) -> Optional[Dict]:
        """
        Extract structured data from CV text using LLM
        
        Args:
            cv_text: Raw text extracted from CV document
        
        Returns:
            Dictionary with extracted CV fields, or None if error
        """
        if not cv_text or not cv_text.strip():
            logger.warning("Empty CV text provided")
            return None
            
        try:
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {
                        "role": "system", 
                        "content": "You are an expert at extracting structured data from CVs and resumes."
                    },
                    {
                        "role": "user", 
                        "content": EXTRACTION_PROMPT.format(cv_text=cv_text)
                    }
                ],
                response_format={"type": "json_object"},  # Ensures valid JSON
                temperature=0.1  # Low temperature for consistency
            )
            
            content = response.choices[0].message.content
            extracted_data = json.loads(content)
            return self._normalize_extracted_data(extracted_data)
            
        except Exception as e:
            logger.error(f"Error extracting CV data: {str(e)}")
            return None
```

**Key design decisions:**

1. **`response_format={"type": "json_object"}`**: Forces the model to return valid JSON, eliminating parsing errors.

2. **`temperature=0.1`**: Low temperature reduces creativity and increases consistency. For data extraction, we want deterministic results.

3. **Async processing**: Using `AsyncOpenAI` allows concurrent processing of multiple CVs.

4. **Normalization**: The `_normalize_extracted_data` method ensures all expected fields exist, even if the LLM omitted some.

---

## Semantic Search: Finding Meaning, Not Words

### The Problem with Keyword Search

Keyword search operates on a simple principle: match strings. This creates several problems:

**Vocabulary mismatch**: "Full-Stack Developer" vs. "Full Stack Engineer" vs. "Web Developer" might all describe the same role.

**Implicit skills**: A candidate who "built REST APIs with FastAPI" clearly knows Python—but might not have listed "Python" explicitly.

**Context blindness**: "5 years of Python experience" and "used Python in a weekend project" both contain "Python" but represent vastly different expertise levels.

### Vector Embeddings: A Brief Primer

Vector embeddings transform text into numerical representations where semantic similarity corresponds to mathematical proximity.

```
"Python developer with AWS experience"  →  [0.23, -0.15, 0.87, ..., 0.42]
"Software engineer skilled in Python and cloud" → [0.25, -0.12, 0.84, ..., 0.39]
"Java programmer" → [-0.31, 0.44, -0.22, ..., 0.18]
```

Notice how the first two vectors (semantically similar) would be mathematically close, while the third (semantically different) would be distant.

### Generating Embeddings

```python
class EmbeddingService:
    """Service for generating vector embeddings"""
    
    def __init__(self):
        self.openai_client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
        self.embedding_model = "text-embedding-3-small"  # 1536 dimensions
    
    async def generate_embedding(self, text: str) -> List[float]:
        """Generate embedding vector for text"""
        if not text or not text.strip():
            raise ValueError("Text cannot be empty")
        
        response = await self.openai_client.embeddings.create(
            model=self.embedding_model,
            input=text
        )
        return response.data[0].embedding
    
    async def generate_embeddings_batch(self, texts: List[str]) -> List[List[float]]:
        """Generate embeddings for multiple texts efficiently"""
        # OpenAI supports batch embeddings natively
        response = await self.openai_client.embeddings.create(
            model=self.embedding_model,
            input=texts
        )
        # Sort by index to maintain order
        sorted_data = sorted(response.data, key=lambda x: x.index)
        return [item.embedding for item in sorted_data]
```

### Similarity Search

With embeddings stored, search becomes a similarity calculation:

```python
import numpy as np

def cosine_similarity(vec1: List[float], vec2: List[float]) -> float:
    """Calculate cosine similarity between two vectors"""
    a = np.array(vec1)
    b = np.array(vec2)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

async def search_candidates(query: str, candidate_embeddings: List[dict], top_k: int = 10):
    """
    Search for candidates semantically similar to query
    
    Args:
        query: Natural language search query
        candidate_embeddings: List of {id, embedding, data} dicts
        top_k: Number of results to return
    
    Returns:
        Top k matching candidates with similarity scores
    """
    # Generate query embedding
    query_embedding = await embedding_service.generate_embedding(query)
    
    # Calculate similarity with all candidates
    results = []
    for candidate in candidate_embeddings:
        similarity = cosine_similarity(query_embedding, candidate['embedding'])
        results.append({
            'candidate': candidate['data'],
            'score': similarity
        })
    
    # Sort by similarity and return top k
    results.sort(key=lambda x: x['score'], reverse=True)
    return results[:top_k]
```

---

## Natural Language Queries: Talking to Your Database

HR Helper doesn't just accept search queries—it understands questions. This is where the question processor comes in.

### Understanding Intent

```python
class QuestionProcessor:
    """Service for understanding user questions"""
    
    def __init__(self):
        self.client = OpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = settings.OPENAI_MODEL
    
    def process_question(self, question: str):
        """
        Analyze a user question to extract query parameters
        
        Returns:
            Tuple of (search_query, context_type, language)
        """
        system_message = {
            "role": "system",
            "content": """You are an HR assistant helping with recruitment queries. 
            Analyze the user input and provide a JSON response with:
            - 'type': either 'role', 'offer', or 'jobDescription'
            - 'language': either 'en' or 'es'
            - 'keywords': array of relevant skills or keywords"""
        }
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[system_message, {"role": "user", "content": question}],
            response_format={"type": "json_object"},
            temperature=0.3
        )
        
        result = json.loads(response.choices[0].message.content)
        return result
```

### Example Queries in Action

**Query**: "Find me Python developers with machine learning experience who can work remotely"

**LLM Analysis**:
```json
{
    "type": "role",
    "language": "en",
    "keywords": ["Python", "machine learning", "ML", "remote", "data science"]
}
```

**Query**: "Busco candidatos con experiencia en desarrollo web y español fluido"
*(Looking for candidates with web development experience and fluent Spanish)*

**LLM Analysis**:
```json
{
    "type": "role",
    "language": "es",
    "keywords": ["desarrollo web", "web development", "frontend", "backend", "español"]
}
```

The system automatically:
- Detects the query language
- Expands keywords to related terms
- Understands the intent (finding candidates vs. generating job descriptions)

---

## Multi-Language Support: No Translation Required

One of LLM's superpowers is language agnosticism. The same model that understands English CVs also understands Spanish, French, German, and dozens of other languages.

### Handling Multi-Language CVs

When a Spanish CV is uploaded:

1. The LLM extracts structured data in the CV's original language
2. It detects the language and stores it as metadata
3. The embedding captures the *meaning*, not the specific words
4. Searches work across languages because embeddings are semantic

**A Spanish CV describing "desarrollo de aplicaciones web con React y Node.js"** will have an embedding similar to an English CV describing "web application development with React and Node.js"—because they mean the same thing.

### Optional Translation

For display purposes, we can also translate extracted data:

```python
async def translate_cv_data(self, cv_data: dict, target_language: str) -> dict:
    """Translate CV data to target language while preserving structure"""
    
    response = await self.client.chat.completions.create(
        model=self.model,
        messages=[
            {
                "role": "system",
                "content": f"Translate the following CV data to {target_language}. "
                           f"Preserve the JSON structure and field names (in English). "
                           f"Only translate the values."
            },
            {
                "role": "user",
                "content": json.dumps(cv_data)
            }
        ],
        response_format={"type": "json_object"}
    )
    
    return json.loads(response.choices[0].message.content)
```

---

## Why This Approach Beats Traditional Methods

| Aspect | Keyword Search | LLM + Semantic Search |
|--------|---------------|----------------------|
| **Setup** | Requires schema design, indexing rules | Just write prompts |
| **Vocabulary** | Exact matches only | Understands synonyms and related concepts |
| **Languages** | Separate indexes per language | Single system handles all languages |
| **Queries** | Boolean logic (AND/OR/NOT) | Natural language |
| **Flexibility** | Rigid, predefined fields | Adapts to any CV format |
| **Maintenance** | Update rules for new patterns | Update prompts |
| **Context** | None | Understands context and nuance |

### Real-World Example

**Searching for a "DevOps engineer with container experience"**

**Keyword approach** might require:
```
("DevOps" OR "Dev Ops" OR "SRE" OR "Site Reliability") 
AND 
("Docker" OR "Kubernetes" OR "container" OR "K8s" OR "containerization")
```

**Semantic approach** needs only:
```
"DevOps engineer with container experience"
```

And the semantic search will also find candidates who:
- Describe "orchestrating microservices deployments" (implies containers)
- Mention "maintaining CI/CD pipelines with container-based builds"
- Have experience with "ECS" or "Fargate" (AWS container services)
- Used "Podman" instead of Docker

The LLM understands that these all relate to the core query, even without exact keyword matches.

---

## Trade-offs and Considerations

### Latency

LLM calls add latency (100ms-2s depending on model and prompt). We mitigate this through:
- Async processing
- Caching embeddings
- Using faster models for simple tasks (GPT-3.5-turbo vs GPT-4)

### Cost

LLM APIs charge per token. For CV extraction:
- Average CV: ~2,000 tokens input, ~500 tokens output
- With GPT-4: ~$0.08 per CV
- With GPT-3.5-turbo: ~$0.004 per CV

We optimize by:
- Truncating irrelevant text before sending
- Caching extracted data
- Using cheaper models where quality allows

### Accuracy

LLMs can occasionally hallucinate or misinterpret. We handle this through:
- Low temperature settings (0.1-0.3)
- Validation of extracted fields
- Human review flags for low-confidence extractions

---

## The Future: Where This Is Heading

The capabilities of LLMs are evolving rapidly. Features on our roadmap:

**Skill inference**: Not just extracting stated skills, but inferring likely skills from experience descriptions.

**Career trajectory analysis**: Understanding career progression and predicting fit for senior roles.

**Cultural fit indicators**: Analyzing communication style and values alignment from CV text.

**Interactive refinement**: "Show me similar candidates but with more enterprise experience"—refining searches through conversation.

---

## Conclusion

The shift from keyword search to semantic understanding isn't just an incremental improvement—it's a fundamental change in how we match candidates to roles.

Keywords ask: "Does this document contain these strings?"
Semantic search asks: "Does this candidate match what we're looking for?"

That's the difference between scanning and understanding. And it's why LLMs are transforming recruitment technology.

---

*Next up: [The ROI of AI-Powered Recruitment: Real Numbers, Real Results](/posts/04-business-impact/)*

*Previous: [From Azure to AWS: A Practical Cloud Migration Story](/posts/02-azure-to-aws-migration/)*

---

**About This Series**: This blog series documents the development of HR Helper, an AI-powered CV matching system. We share our technical decisions, business learnings, and vision for the future of recruitment technology.

