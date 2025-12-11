---
title: "From Azure to AWS: A Practical Cloud Migration Story"
date: 2025-08-22
draft: true
description: "A detailed technical walkthrough of migrating from Azure services to AWS with Python and LLMs"
tags: ["AWS", "Azure", "cloud migration", "Python", "architecture"]
series: ["Candidex"]
weight: 2
ShowToc: true
TocOpen: false
---

## Why Migrate?

When we first built HR Helper, Azure was the natural choice. Microsoft's suite of AI services—Document Intelligence (formerly Form Recognizer), Cognitive Search, and Text Analytics—provided a coherent ecosystem for our CV processing needs.

But as the LLM landscape evolved, we found ourselves at a crossroads:

1. **LLM Flexibility**: The explosion of capable language models (GPT-4, Claude, Llama, Mistral) meant we no longer needed specialized Azure services for each task. A single, well-prompted LLM could handle extraction, translation, and understanding.

2. **Cost Optimization**: Azure's per-service pricing model was becoming expensive at scale. With LLMs, we could consolidate multiple services into fewer API calls.

3. **Architectural Simplicity**: Instead of integrating with 5+ Azure services, each with its own SDK and authentication, we could build a cleaner architecture around S3 storage and LLM APIs.

4. **Python Ecosystem**: The AI/ML ecosystem has centered around Python. Migrating from C# opened access to better libraries, faster iteration, and a larger talent pool.

This post documents our journey from a C#/Azure architecture to Python/AWS+LLM, including the challenges we faced and how we solved them.

---

## Architecture: Before and After

### The Original Azure Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        React Frontend                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     C# .NET 8.0 Backend                          │
│                     (ASP.NET Core Web API)                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Azure Blob    │   │ Azure Document  │   │ Azure Cognitive │
│ Storage /     │   │ Intelligence    │   │ Search          │
│ Data Lake     │   │ (Form Recog.)   │   │                 │
└───────────────┘   └─────────────────┘   └─────────────────┘
        │                     │                     │
        │                     ▼                     │
        │           ┌─────────────────┐             │
        │           │ Azure Text      │             │
        │           │ Analytics       │             │
        │           └─────────────────┘             │
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ OpenAI API      │
                    │ (Chat/Translate)│
                    └─────────────────┘
```

### The New Python/AWS Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    React Frontend (unchanged)                    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Python FastAPI Backend                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            │                                   │
            ▼                                   ▼
    ┌───────────────┐                   ┌─────────────────┐
    │   AWS S3      │                   │  OpenAI / LLM   │
    │   (Storage)   │                   │  (Everything)   │
    └───────────────┘                   └─────────────────┘
                                               │
                                               ▼
                                        ┌─────────────────┐
                                        │  Vector DB      │
                                        │  (ChromaDB)     │
                                        └─────────────────┘
```

The difference is striking. We went from 5+ Azure services to essentially 2 external dependencies: S3 for storage and an LLM for intelligence.

---

## Service-by-Service Migration

### 1. Azure Blob Storage → AWS S3

This was the most straightforward migration. Both services serve the same purpose: storing files in the cloud.

**Before (C# with Azure Data Lake):**

```csharp
// Azure Data Lake Client setup
DataLakeServiceClient dataLakeServiceClient = new DataLakeServiceClient(_storageConnectionString);
DataLakeFileSystemClient fileSystemClient = dataLakeServiceClient.GetFileSystemClient("landing");
DataLakeDirectoryClient directoryClient = fileSystemClient.GetDirectoryClient("documents");

// Reading files
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
            logger.error(f"Error uploading file to S3: {str(e)}")
            return False

    def download_file(self, s3_key: str) -> Optional[bytes]:
        try:
            response = self.s3_client.get_object(Bucket=self.bucket_name, Key=s3_key)
            return response['Body'].read()
        except ClientError as e:
            logger.error(f"Error downloading file from S3: {str(e)}")
            return None
```

**Key differences:**
- boto3 is more Pythonic and concise
- S3's API is simpler than Data Lake's hierarchical structure
- Presigned URLs work similarly in both

---

### 2. Azure Document Intelligence → LLM-Based Extraction

This was the most significant change. Azure Document Intelligence required training a custom model to recognize CV fields. With LLMs, we just describe what we want.

**Before (C# with Azure Form Recognizer):**

```csharp
public async Task<List<Dictionary<string, object>>> ProcessDocumentFromStream(Stream stream)
{
    var extractedData = new List<Dictionary<string, object>>();
    
    // Requires a pre-trained custom model
    var operation = await _formRecognizerClient.AnalyzeDocumentAsync(
        WaitUntil.Completed, 
        _modelId,  // Custom trained model ID
        stream
    );
    
    var response = await operation.WaitForCompletionAsync();
    var forms = response.Value;

    foreach (AnalyzedDocument form in forms.Documents)
    {
        Dictionary<string, object> data = new Dictionary<string, object>();
        data["id"] = Guid.NewGuid().ToString();
        data["Email"] = SafeGetField(form, "Email");
        data["Name"] = form.Fields["Name"].Value;
        data["Phone"] = form.Fields["Phone"].Value;
        // ... 15+ more fields manually mapped
    }
    return extractedData;
}
```

**After (Python with LLM):**

```python
EXTRACTION_PROMPT = """Extract the following information from this CV/resume 
and return it as a JSON object:

CV Text:
{cv_text}

Extract and return a JSON object with these fields:
- id: Generate a unique UUID
- Email: Email address
- Name: Full name
- Phone: Phone number
- ProfessionalSummary: Professional summary or objective
- TechnicalSkills: Technical skills (comma-separated or as array)
- JobRole: Current or most recent job role
... (additional fields)

If a field is not found in the CV, set it to an empty string.
Return only valid JSON, no additional text."""

class LLMExtractor:
    def __init__(self):
        self.client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
        self.model = settings.OPENAI_MODEL
    
    async def extract_cv_data(self, cv_text: str) -> Optional[Dict]:
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "You are an expert at extracting structured data from CVs."},
                {"role": "user", "content": EXTRACTION_PROMPT.format(cv_text=cv_text)}
            ],
            response_format={"type": "json_object"},
            temperature=0.1
        )
        
        return json.loads(response.choices[0].message.content)
```

**Why LLM extraction is better:**

| Aspect | Azure Document Intelligence | LLM Extraction |
|--------|----------------------------|----------------|
| Setup | Requires training data, model training | Just write a prompt |
| Flexibility | Fixed schema from training | Change schema instantly |
| Languages | Needs separate training per language | Handles any language automatically |
| Maintenance | Retrain for new formats | Update prompt |
| Cost | Per-page pricing + training costs | Per-token pricing |

---

### 3. Azure Cognitive Search → Vector Embeddings

Azure Cognitive Search is powerful but complex. It requires defining schemas, managing indexes, and learning a query DSL. Vector search with embeddings is conceptually simpler.

**The concept shift:**

```
Traditional Search: "Find documents containing 'Python' AND 'AWS'"
                    ↓
                    Exact keyword matching
                    
Vector Search:      "Find documents similar to this meaning"
                    ↓
                    Semantic similarity via embeddings
```

**New approach with embeddings:**

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
        
        # Calculate cosine similarity with all documents
        similarities = []
        for doc in documents:
            similarity = cosine_similarity(query_embedding, doc['embedding'])
            similarities.append((doc, similarity))
        
        # Return top matches
        return sorted(similarities, key=lambda x: x[1], reverse=True)[:top_k]
```

---

### 4. Azure Text Analytics → LLM Language Detection

**Before:** Dedicated API call to Azure Text Analytics
**After:** Add a field to our extraction prompt

```python
# Language detection is now just part of the extraction prompt:
"- Language: Detected language code (e.g., 'en', 'es')"

# Or as a separate lightweight call:
async def detect_language(self, text: str) -> str:
    response = await self.client.chat.completions.create(
        model="gpt-3.5-turbo",  # Cheaper model sufficient for this
        messages=[{
            "role": "user", 
            "content": f"What language is this text? Reply with just the ISO 639-1 code.\n\n{text[:500]}"
        }],
        max_tokens=5
    )
    return response.choices[0].message.content.strip()
```

---

### 5. Azure AD → JWT Authentication

Azure AD is enterprise-grade but complex. For our use case, simple JWT tokens provide sufficient security with much less configuration.

```python
from jose import jwt
from passlib.context import CryptContext

class AuthService:
    def __init__(self):
        self.secret_key = settings.JWT_SECRET_KEY
        self.algorithm = "HS256"
        self.pwd_context = CryptContext(schemes=["bcrypt"])
    
    def create_access_token(self, data: dict) -> str:
        to_encode = data.copy()
        expire = datetime.utcnow() + timedelta(hours=24)
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
    
    def verify_token(self, token: str) -> dict:
        return jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
```

---

## Challenges We Faced

### Challenge 1: PDF Text Extraction

Azure Document Intelligence handles PDF parsing automatically. Without it, we needed a solution.

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

### Challenge 2: Rate Limits and Costs

LLM APIs have rate limits and per-token costs that can add up.

**Solutions:**
- Caching: Store extracted data to avoid re-processing
- Batching: Process multiple CVs in parallel where possible
- Model selection: Use GPT-3.5-turbo for simple tasks, GPT-4 only when needed
- Token optimization: Truncate irrelevant text before sending to LLM

### Challenge 3: Maintaining API Compatibility

The React frontend expected certain API contracts. We needed to maintain compatibility.

**Solution:** Keep the same endpoint paths and response structures.

```python
# Match the original API contract
@router.post("/api/document/upload")  # Same path as before
async def upload_document(file: UploadFile):
    # Process and return same structure frontend expects
    return {"success": True, "data": extracted_data}
```

---

## Migration Summary Table

| Component | Azure (Before) | AWS/Python (After) | Benefit |
|-----------|---------------|-------------------|---------|
| Language | C# .NET 8.0 | Python 3.10+ | Better AI/ML ecosystem |
| Framework | ASP.NET Core | FastAPI | Simpler, async-first |
| Storage | Azure Data Lake | AWS S3 | Cost, simplicity |
| CV Extraction | Document Intelligence | LLM + prompts | No training needed |
| Search | Cognitive Search | Vector embeddings | Semantic understanding |
| Language Detection | Text Analytics | LLM | Unified solution |
| Translation | Azure Translator | LLM | Unified solution |
| Authentication | Azure AD | JWT | Simpler setup |

---

## Lessons Learned

1. **LLMs are great unifiers**: Tasks that required 4+ specialized services can often be handled by a single, well-prompted LLM.

2. **Don't over-engineer auth**: Unless you need enterprise SSO, simple JWT tokens are sufficient and much easier to manage.

3. **Keep the frontend stable**: Migrating the backend while maintaining API compatibility let us iterate without frontend changes.

4. **Test extraction quality**: LLM extraction is more flexible but requires validation. We built a test suite comparing extraction results.

5. **Monitor costs**: LLM token usage can be unpredictable. Implement logging and alerts early.

---

## What's Next

In the next post, we'll dive deep into the AI innovation: how LLMs actually understand CVs, how vector embeddings enable semantic search, and why this approach produces better candidate matches than traditional methods.

---

*Next up: [Beyond Keywords: Using LLMs to Actually Understand Resumes](/posts/03-llm-cv-processing/)*

*Previous: [Why HR Departments Are Drowning in CVs](/posts/01-hr-recruitment-challenge/)*

---

**About This Series**: This blog series documents the development of HR Helper, an AI-powered CV matching system. We share our technical decisions, business learnings, and vision for the future of recruitment technology.

