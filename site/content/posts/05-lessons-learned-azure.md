---
title: "Lessons Learned: Building Candidex with Azure and C#"
date: 2024-08-10
draft: false
description: "What we learned building an AI-powered CV matching system with Azure services, C#, and OpenAI"
tags: ["lessons learned", "Azure", "C#", "architecture", "retrospective"]
series: ["Candidex"]
weight: 5
ShowToc: true
TocOpen: false


## Looking Back

We set out to build an AI-powered CV matching system that would help HR departments find candidates faster and more accurately than traditional keyword searches. Four months, five Azure services, and thousands of lines of C# later, we had a working system.

Here's what we learned along the way.

## What Worked Well

### 1. Azure's Integrated Ecosystem

The promise of Azure is that services "just work" together. In our experience, this was largely true:

- **Data Lake → Document Intelligence**: Documents uploaded to storage triggered processing automatically
- **Document Intelligence → Cognitive Search**: Extracted data indexed seamlessly
- **Managed identity**: Services authenticated to each other without credential management

```csharp
// No API keys in code - managed identity handles auth
DataLakeServiceClient dataLakeServiceClient = new DataLakeServiceClient(
    new Uri("https://ourstorage.dfs.core.windows.net"),
    new DefaultAzureCredential()
);
```

**Lesson**: When building on a single cloud platform, lean into its integration patterns. The time saved on plumbing lets you focus on business logic.

### 2. Document Intelligence Custom Models

Training a custom model for CV extraction was easier than expected:

1. Upload 50 sample documents
2. Label fields in the browser-based studio
3. Click "Train"
4. Get 85%+ accuracy out of the box

The key insight: **Document Intelligence understands document structure, not just text**. It learns that "Professional Summary" usually appears near the top, that skills often appear in lists, that dates follow certain patterns. This structural understanding made it far more robust than regex-based extraction.

**Lesson**: Purpose-built AI services often outperform general-purpose solutions for specific tasks. Don't reinvent the wheel.

### 3. C# and Strong Typing

Working with structured CV data, we appreciated C#'s type system:

```csharp
public class ResultItem
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string ProfessionalSummary { get; set; }
    public string TechnicalSkills { get; set; }
    public string JobRole { get; set; }
    // ... strongly typed fields
}
```

IntelliSense caught typos, refactoring was safe, and the compiler caught errors before runtime.

**Lesson**: For data-heavy applications with well-defined schemas, static typing reduces bugs and improves developer experience.

### 4. Separation of Concerns

Our architecture kept responsibilities clear:

| Component | Responsibility |
|-----------|----------------|
| `DocumentDataExtractor` | CV parsing and field extraction |
| `SearchService` | Query execution and result retrieval |
| `ResultProcessor` | Formatting results for different contexts |
| `OpenAIService` | AI-powered summarization and insights |

Each class had a single job. Testing was straightforward. Changes were localized.

**Lesson**: Classic software engineering principles apply to AI applications. Clean architecture matters more, not less, when dealing with complex AI pipelines.

## What We'd Do Differently

### 1. Start Simpler with Document Processing

We jumped straight to Document Intelligence custom models. In hindsight, we should have started with:

1. **Pre-built CV parsing APIs**: Several vendors offer off-the-shelf CV parsing
2. **LLM-based extraction**: GPT-4 can extract structured data from text with a well-crafted prompt

Custom training made sense eventually, but we spent weeks labeling documents before validating the core product concept.

**Lesson**: Validate the business case with the simplest solution that could work. Optimize later.

### 2. Better Error Handling for AI Services

AI services fail in ways traditional APIs don't:

```csharp
// What we had
var response = await _httpClient.SendAsync(httpRequestMessage);
response.EnsureSuccessStatusCode();

// What we needed
try 
{
    var response = await _httpClient.SendAsync(httpRequestMessage);
    if (!response.IsSuccessStatusCode)
    {
        // Log, retry with backoff, fall back to cached response
    }
}
catch (HttpRequestException ex)
{
    // Handle rate limits, timeouts, model overload
}
```

OpenAI rate limits, Document Intelligence processing delays, Cognitive Search indexing lag—all required more robust handling than we initially built.

**Lesson**: AI services need defensive programming. Build retry logic, fallbacks, and graceful degradation from day one.

### 3. Observability from the Start

Debugging AI pipelines is hard. When a candidate didn't appear in search results, was it because:
- The CV wasn't uploaded?
- Document Intelligence failed to extract data?
- The search index wasn't updated?
- The query didn't match?

We added logging reactively, chasing bugs. We should have instrumented proactively.

**Lesson**: For AI pipelines, logging isn't optional. Log every stage: input received, processing started, fields extracted, index updated, search executed, results returned.

### 4. Cost Monitoring

Azure's per-service pricing surprised us:

| Service | What We Expected | What We Got |
|---------|------------------|-------------|
| Document Intelligence | Cheap for small volumes | $1.50 per 1000 pages adds up fast |
| Cognitive Search | Basic tier is fine | Needed Standard for features |
| OpenAI API | Pay per token | GPT-4 costs 10x GPT-3.5 |

We didn't set up cost alerts early enough. A testing bug that called OpenAI in a loop generated an unexpected bill.

**Lesson**: Set up billing alerts before writing code. Monitor costs weekly during development.

## Technical Debt We Accumulated

### Configuration Sprawl

```csharp
var config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("local.settings.json", optional: true, reloadOnChange: true)
    .Build();

var searchServiceEndpoint = config["Values:SearchServiceEndpoint"];
var searchServiceApiKey = config["Values:SearchServiceApiKey"];
var indexName = config["Values:IndexName"];
```

Every service needed its own configuration. We ended up with:
- `local.settings.json` for local development
- Environment variables for staging
- Azure Key Vault references for production
- Inconsistent naming across services

**What we'd do differently**: Define a single `AppConfiguration` class early, with clear naming conventions and validation on startup.

### Inconsistent Async Patterns

Some code was fully async:

```csharp
public async Task<List<Dictionary<string, object>>> ExtractDataFromDocumentsAsync()
```

Other code blocked on async operations:

```csharp
var response = await operation.WaitForCompletionAsync();
```

This led to occasional deadlocks in the web application context.

**What we'd do differently**: Establish async/await conventions from the start. Never block on async code. Use `ConfigureAwait(false)` consistently in library code.

### Magic Strings

```csharp
data["ProfessionalSummary"] = form.Fields["ProfessionalSummary"].Value;
data["TechnicalSkills"] = form.Fields["TechnicalSkills"].Value;
```

Field names repeated throughout the codebase. A typo would only surface at runtime.

**What we'd do differently**: Define constants or an enum for field names. Use nameof() where possible.

## The Limitations We Discovered

### Keyword Search Isn't Enough

Azure Cognitive Search is powerful, but it's still fundamentally keyword-based. When an HR professional searches for "cloud experience," they expect to find candidates who mention AWS, Azure, GCP, Kubernetes, or Docker—even if they never use the word "cloud."

We added synonym maps:
```json
{
  "cloud": "AWS, Azure, GCP, kubernetes, docker, serverless"
}
```

But maintaining these became tedious, and we always missed edge cases.

**The realization**: What we really needed was **semantic search**—understanding meaning, not matching keywords. This would eventually drive us toward a different architecture.

### Per-Document Processing Doesn't Scale

Document Intelligence charges per document. For an HR department processing 100 CVs per week, costs are manageable. For 10,000 CVs per week? The math stops working.

**The realization**: At scale, we needed a different approach—one where the marginal cost of processing an additional CV approached zero.

### Multiple Services = Multiple Failure Modes

Our architecture depended on five Azure services working correctly:
1. Blob Storage / Data Lake
2. Document Intelligence
3. Cognitive Search
4. Azure App Service
5. OpenAI API

Each service had its own:
- Authentication mechanism
- Rate limits
- Error responses
- SDK quirks

When something broke, pinpointing the cause required checking all five.

**The realization**: Fewer moving parts means fewer things that can break. Could we simplify?

## What's Next

These lessons planted seeds for the next evolution of Candidex. The questions we started asking:

- **What if LLMs could handle extraction, search, and summarization?**
- **What if we simplified from five services to two or three?**
- **What if we used semantic search instead of keyword matching?**

Those questions led us to explore a different architecture—one built around embeddings, vector search, and modern LLM capabilities.

---

*Next up: [From Azure to AWS: Migrating to an LLM-First Architecture](/posts/06-azure-to-aws-migration/)*

---

## Key Takeaways

1. **Azure integration is real** — Services work well together when you use them as intended
2. **Purpose-built AI beats general solutions** — Document Intelligence for documents, Cognitive Search for search
3. **Start simple** — Validate with the minimum viable AI before building complex pipelines
4. **Observe everything** — AI systems need more logging, not less
5. **Watch costs** — Per-call pricing adds up quickly
6. **Clean architecture matters** — Classic software engineering principles apply to AI applications
7. **Know your limitations** — Keyword search has ceilings that semantic approaches can break through

---

**About This Series**: This blog series documents the development of Candidex, from the initial Azure-based implementation to its ongoing evolution. We share technical decisions, code examples, and honest lessons learned building AI-powered recruitment tools.

