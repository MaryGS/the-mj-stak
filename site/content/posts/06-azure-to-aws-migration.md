---
title: "From Azure to AWS: Migrating to an LLM-First Architecture"
date: 2025-09-25
draft: false
description: "A detailed technical walkthrough of migrating from Azure services to AWS with Python and LLMs"
tags: ["AWS", "Azure", "cloud migration", "Python", "architecture", "LLM"]
series: ["Candidex"]
weight: 6
ShowToc: true
TocOpen: false
---

*Part 6 of the Candidex Blog Series*

---

## Time for a Change

In the [previous post](/posts/05-lessons-learned-azure/), we reflected on what we learned building Candidex with Azure and C#. The system worked, but we identified key limitations:

- **Keyword search didn't understand meaning**
- **Per-document pricing didn't scale**
- **Five services meant five failure modes**

As we watched the LLM landscape evolve—GPT-4, Claude, Llama, Mistral—we realized something: tasks that required 4+ specialized services could now be handled by a single, well-prompted LLM.

It was time to rethink our architecture.

---

## The Migration Decision

### Why Migrate at All?

1. **LLM Flexibility**: The explosion of capable language models meant we no longer needed specialized Azure services for each task. One LLM could handle extraction, translation, and understanding.

2. **Semantic Search**: Vector embeddings could find candidates by *meaning*, not just keywords. "Cloud experience" would finally match "AWS certified."

3. **Architectural Simplicity**: Instead of integrating with 5+ Azure services, each with its own SDK and authentication, we could build a cleaner architecture around S3 and LLMs.

4. **Python Ecosystem**: The AI/ML ecosystem centers on Python. Migrating from C# opened access to better libraries, faster iteration, and a larger talent pool.

### Why Python + AWS?

| Decision | Reasoning |
|----------|-----------|
| **Python over C#** | Better AI/ML library ecosystem, faster prototyping |
| **FastAPI over ASP.NET** | Async-first, automatic OpenAPI docs, simpler |
| **AWS S3 over Azure Blob** | Simpler API, better pricing, wider tooling |
| **LLM over Document Intelligence** | No training needed, instant schema changes |
| **Vector embeddings over Cognitive Search** | Semantic understanding, not just keywords |

---

## Architecture: Before and After

### The Azure Architecture (Before)

```
┌─────────────────────────────────────────────────────────────────┐
│                        React Frontend                            │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     C# .NET 8.0 Backend                          │
│                     (ASP.NET Core Web API)                       │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────┐
        │                         │                     │
        ▼                         ▼                     ▼
┌───────────────┐       ┌─────────────────┐   ┌─────────────────┐
│ Azure Blob    │       │ Azure Document  │   │ Azure Cognitive │
│ Storage /     │       │ Intelligence    │   │ Search          │
│ Data Lake     │       └─────────────────┘   └─────────────────┘
└───────────────┘               │
                                ▼
                      ┌─────────────────┐
                      │ Azure Text      │
                      │ Analytics       │
                      └─────────────────┘
                                │
                                ▼
                      ┌─────────────────┐
                      │ OpenAI API      │
                      └─────────────────┘
```

**5+ services**, each with its own SDK, credentials, and failure modes.

### The New Architecture (After)

```
┌─────────────────────────────────────────────────────────────────┐
│                    React Frontend (unchanged)                    │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Python FastAPI Backend                        │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
            ┌─────────────────────┴─────────────────┐
            │                                       │
            ▼                                       ▼
    ┌───────────────┐                     ┌─────────────────┐
    │   AWS S3      │                     │  LLM API        │
    │   (Storage)   │                     │  (Everything)   │
    └───────────────┘                     └─────────────────┘
                                                  │
                                                  ▼
                                          ┌─────────────────┐
                                          │  Vector Store   │
                                          │  (Embeddings)   │
                                          └─────────────────┘
```

**2-3 services**. Storage, intelligence, and search—unified and simple.

---

## Service-by-Service Migration

### 1. Azure Blob Storage → AWS S3

The most straightforward migration. Both services store files; S3's API is simpler.

**Before (C# with Azure Data Lake):**

```csharp
DataLakeServiceClient dataLakeServiceClient = new DataLakeServiceClient(_storageConnectionString);
DataLakeFileSystemClient fileSystemClient = dataLakeServiceClient.GetFileSystemClient("landing");
DataLakeDirectoryClient directoryClient = fileSystemClient.GetDirectoryClient("documents");

await foreach (PathItem pathItem in directoryClient.GetPathsAsync(recursive: false))
{
    DataLakeFileClient fileClient = directoryClient.GetFileClient(fileName);
    using var stream = await fileClient.OpenReadAsync();
    // Process stream...
}
```

**After (Python with boto3):**

```python
import boto3
from botocore.exceptions import ClientError

class S3Service:
    def __init__(self):
        self.s3_client = boto3.client(
            's3',
            aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
            aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
            region_name=settings.AWS_REGION
        )
        self.bucket_name = settings.S3_BUCKET_NAME

    def upload_file(self, file_content: BinaryIO, s3_key: str) -> bool:
        try:
            self.s3_client.upload_fileobj(file_content, self.bucket_name, s3_key)
            return True
        except ClientError as e:
            logger.error(f"Error uploading to S3: {e}")
            return False

    def download_file(self, s3_key: str) -> Optional[bytes]:
        try:
            response = self.s3_client.get_object(Bucket=self.bucket_name, Key=s3_key)
            return response['Body'].read()
        except ClientError as e:
            logger.error(f"Error downloading from S3: {e}")
            return None
```

**Key differences:**
- boto3 is more Pythonic and concise
- S3's flat namespace is simpler than Data Lake's hierarchy
- Presigned URLs work similarly in both

---

### 2. Azure Document Intelligence → LLM Extraction

This was the biggest change. Azure Document Intelligence required training a custom model. With LLMs, we just describe what we want.

**Before (C# with Form Recognizer):**

```csharp
public async Task<List<Dictionary<string, object>>> ProcessDocumentFromStream(Stream stream)
{
    // Requires a pre-trained custom model
    var operation = await _formRecognizerClient.AnalyzeDocumentAsync(
        WaitUntil.Completed, 
        _modelId,  // Custom trained model ID
        stream
    );
    
    var response = await operation.WaitForCompletionAsync();
    
    foreach (AnalyzedDocument form in forms.Documents)
    {
        Dictionary<string, object> data = new Dictionary<string, object>();
        data["Email"] = SafeGetField(form, "Email");
        data["Name"] = form.Fields["Name"].Value;
        // ... 15+ fields manually mapped
    }
}
```

**After (Python with LLM):**

```python
EXTRACTION_PROMPT = """Extract the following information from this CV/resume.
Return a JSON object with these fields:

CV Text:
{cv_text}

Fields to extract:
- Name: Full name
- Email: Email address  
- Phone: Phone number
- ProfessionalSummary: Professional summary or objective
- TechnicalSkills: List of technical skills
- JobRole: Current or most recent job role
- Company: Current or most recent employer
- Education: Educational background
- Languages: Languages spoken
- Certifications: Professional certifications

If a field is not found, set it to empty string.
Return only valid JSON."""

class LLMExtractor:
    def __init__(self):
        self.client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
    
    async def extract_cv_data(self, cv_text: str) -> Optional[Dict]:
        response = await self.client.chat.completions.create(
            model="gpt-4-turbo",
            messages=[
                {"role": "system", "content": "You extract structured data from CVs."},
                {"role": "user", "content": EXTRACTION_PROMPT.format(cv_text=cv_text)}
            ],
            response_format={"type": "json_object"},
            temperature=0.1
        )
        
        return json.loads(response.choices[0].message.content)
```

**Why LLM extraction wins:**

| Aspect | Document Intelligence | LLM Extraction |
|--------|----------------------|----------------|
| Setup time | Days (training) | Minutes (prompt) |
| Schema changes | Retrain model | Update prompt |
| Multi-language | Train per language | Works automatically |
| New CV formats | May need retraining | Handles naturally |
| Cost model | Per page | Per token |

---

### 3. Azure Cognitive Search → Vector Embeddings

This was the game-changer for search quality. Instead of keyword matching, we now search by meaning.

**The conceptual shift:**

```
Keyword Search:   "Find 'Python' AND 'AWS' in documents"
                  → Exact text matching
                  
Vector Search:    "Find documents semantically similar to this query"
                  → Meaning-based matching
```

**New embedding-based search:**

```python
from openai import OpenAI

class EmbeddingService:
    def __init__(self):
        self.client = OpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = "text-embedding-3-small"
    
    def get_embedding(self, text: str) -> list[float]:
        response = self.client.embeddings.create(
            input=text,
            model=self.model
        )
        return response.data[0].embedding
    
    def search_similar(self, query: str, documents: list, top_k: int = 10):
        query_embedding = self.get_embedding(query)
        
        similarities = []
        for doc in documents:
            similarity = cosine_similarity(query_embedding, doc['embedding'])
            similarities.append((doc, similarity))
        
        return sorted(similarities, key=lambda x: x[1], reverse=True)[:top_k]
```

**The magic moment**: When we searched for "cloud experience" and it returned candidates with AWS, Azure, and GCP certifications—even though "cloud" wasn't in their CVs. That's semantic understanding.

---

### 4. Azure Text Analytics → LLM

Language detection and sentiment analysis? Now just part of the prompt.

**Before:** Separate API call to Azure Text Analytics
**After:** One line in the extraction prompt

```python
# Language detection is now just a field:
"- Language: Detected language code (e.g., 'en', 'es')"

# Or a lightweight separate call:
async def detect_language(self, text: str) -> str:
    response = await self.client.chat.completions.create(
        model="gpt-3.5-turbo",  # Cheap model works fine
        messages=[{
            "role": "user", 
            "content": f"What language is this? Reply with ISO 639-1 code only.\n\n{text[:500]}"
        }],
        max_tokens=5
    )
    return response.choices[0].message.content.strip()
```

---

### 5. Azure AD → JWT Tokens

Azure AD is enterprise-grade but complex. For our use case, JWT tokens provide sufficient security with much less configuration.

```python
from jose import jwt
from passlib.context import CryptContext

class AuthService:
    def __init__(self):
        self.secret_key = settings.JWT_SECRET_KEY
        self.algorithm = "HS256"
    
    def create_access_token(self, data: dict) -> str:
        to_encode = data.copy()
        expire = datetime.utcnow() + timedelta(hours=24)
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
    
    def verify_token(self, token: str) -> dict:
        return jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
```

---

## The New Project Structure

```
hr-helper-python/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI entry point
│   ├── config.py                  # Configuration
│   ├── api/
│   │   └── routes/
│   │       ├── document.py        # CV upload endpoints
│   │       ├── chat.py            # Chat/search endpoints
│   │       └── health.py          # Health check
│   ├── services/
│   │   ├── s3_service.py          # AWS S3 operations
│   │   ├── document_reader.py     # PDF/DOCX text extraction
│   │   ├── llm_extractor.py       # LLM-based CV extraction
│   │   ├── embedding_service.py   # Vector embeddings
│   │   ├── search_service.py      # Semantic search
│   │   ├── answer_generator.py    # Response formatting
│   │   ├── language_detector.py   # Language detection
│   │   ├── translator.py          # Translation
│   │   └── auth_service.py        # JWT authentication
│   └── utils/
│       └── logger.py
├── tests/
├── requirements.txt
└── README.md
```

Clean, organized, and each service has a single responsibility.

---

## Challenges We Faced

### Challenge 1: PDF Text Extraction

Azure Document Intelligence handled PDF parsing automatically. Without it, we needed a solution.

**Solution:** PyPDF2 for simple PDFs, pdfplumber for complex layouts.

```python
import pdfplumber

def extract_text_from_pdf(file_content: bytes) -> str:
    with pdfplumber.open(io.BytesIO(file_content)) as pdf:
        text = ""
        for page in pdf.pages:
            text += page.extract_text() or ""
    return text
```

### Challenge 2: Maintaining API Compatibility

The React frontend expected certain API contracts. We needed to maintain compatibility during migration.

**Solution:** Keep the same endpoint paths and response structures.

```python
# Match the original API contract exactly
@router.post("/api/document/upload")
async def upload_document(file: UploadFile):
    # Process and return same structure frontend expects
    return {"success": True, "data": extracted_data}
```

The frontend never knew the backend changed.

### Challenge 3: Rate Limits and Costs

LLM APIs have rate limits and per-token costs.

**Solutions:**
- **Caching**: Store extracted data to avoid re-processing
- **Model selection**: GPT-3.5-turbo for simple tasks, GPT-4 only when needed
- **Token optimization**: Truncate irrelevant text before sending
- **Batching**: Process multiple CVs in parallel where possible

---

## Migration Summary

| Component | Azure (Before) | Python/AWS (After) |
|-----------|---------------|-------------------|
| Language | C# .NET 8.0 | Python 3.10+ |
| Framework | ASP.NET Core | FastAPI |
| Storage | Azure Data Lake | AWS S3 |
| CV Extraction | Document Intelligence | LLM + prompts |
| Search | Cognitive Search | Vector embeddings |
| Language Detection | Text Analytics | LLM |
| Translation | Azure Translator | LLM |
| Authentication | Azure AD | JWT |

---

## Results

### Simplicity Wins

- **Services**: 5+ → 3
- **SDKs to manage**: 5 → 2
- **Lines of code**: Reduced by ~40%
- **Configuration complexity**: Significantly simpler

### Search Quality Improved

The semantic search was transformative. Queries like "experienced backend developer comfortable with cloud" now find relevant candidates even when they use different terminology.

### Cost Structure Changed

- **Fixed costs**: Lower (fewer managed services)
- **Variable costs**: Higher per-query (LLM tokens)
- **Net result**: Cheaper at moderate scale, needs monitoring at high scale

---

## What We Learned

1. **LLMs are great unifiers**: Tasks that required 4+ specialized services can often be handled by one well-prompted LLM.

2. **Keep the frontend stable**: Migrating the backend while maintaining API compatibility let us iterate without frontend changes.

3. **Test extraction quality**: LLM extraction is flexible but requires validation. We built tests comparing extraction results.

4. **Monitor costs differently**: Token usage is less predictable than per-service pricing. Implement logging and alerts early.

5. **Semantic search is magic**: The jump from keyword to semantic search was the single biggest quality improvement.

---

## What's Next

With the migration complete, we have a cleaner, more powerful system. In future posts, we'll explore:

- Deep dive into semantic search and embeddings
- How we optimized LLM costs at scale
- Building the chat interface for natural language queries

The foundation is set for the next evolution of Candidex.

---

*Previous: [Lessons Learned: Building with Azure and C#](/posts/05-lessons-learned-azure/)*

---

**About This Series**: This blog series documents the development of Candidex, from the initial Azure-based implementation through its migration to Python and AWS. We share technical decisions, code examples, and honest lessons learned.

