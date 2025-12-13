---
title: "Building an AI-Powered CV Matcher with Azure and C#"
date: 2024-07-01
draft: false
description: "A technical overview of the Azure services and C# architecture powering our first Candidex implementation"
tags: ["Azure", "C#", ".NET", "architecture", "AI services"]
series: ["Candidex"]
weight: 2
ShowToc: true
TocOpen: false
---

## From Problem to Architecture

In the [previous post](/posts/01-hr-recruitment-challenge/), we explored why traditional keyword-based CV screening fails HR departments. Now let's dive into how we built a solution using Microsoft Azure's AI services and C#.

When designing Candidex, we needed to answer three fundamental questions:

1. **How do we extract structured data from unstructured CV documents?**
2. **How do we make that data searchable in meaningful ways?**
3. **How do we provide intelligent, conversational responses to HR queries?**

Azure's ecosystem provided answers to all three.

## The Azure Architecture

```text
  +------------------+      +------------------+      +------------------+
  |    CV Upload     |----->|   Azure Blob     |----->|    Document      |
  |   (React App)    |      |    Storage       |      |  Intelligence    |
  +------------------+      +------------------+      +--------+---------+
                                                               |
                                                               v
  +------------------+      +------------------+      +------------------+
  |     Chat UI      |<---->|   ASP.NET Core   |<---->| Azure Cognitive  |
  |     (React)      |      |     Backend      |      |      Search      |
  +------------------+      +--------+---------+      +------------------+
                                     |
                                     v
                            +------------------+
                            |    OpenAI API    |
                            |   (GPT-4 Turbo)  |
                            +------------------+
```

### The Services We Chose

| Service | Purpose | Why We Chose It |
|---------|---------|-----------------|
| **Azure Blob Storage / Data Lake** | Document storage | Scalable, integrates natively with other Azure services |
| **Azure Document Intelligence** | CV data extraction | Purpose-built for extracting structured data from documents |
| **Azure Cognitive Search** | Indexed search | Enterprise-grade search with filtering and ranking |
| **OpenAI API** | Intelligent responses | GPT-4's reasoning for natural language interactions |
| **ASP.NET Core** | Backend API | Strong typing, excellent Azure SDK support |

## The Tech Stack

Our C# implementation uses .NET 8 with the following key packages:

```xml
<PackageReference Include="Azure.AI.FormRecognizer" Version="4.1.0" />
<PackageReference Include="Azure.AI.DocumentIntelligence" Version="1.0.0-beta.2" />
<PackageReference Include="Azure.Search.Documents" Version="11.5.1" />
<PackageReference Include="Azure.Storage.Blobs" Version="12.20.0" />
<PackageReference Include="Azure.Storage.Files.DataLake" Version="12.18.0" />
<PackageReference Include="OpenAI" Version="1.11.0" />
```

Each package represents a piece of our processing pipeline:

1. **Storage SDKs** handle document upload and retrieval
2. **Document Intelligence SDK** extracts structured CV data
3. **Search SDK** enables intelligent querying
4. **OpenAI SDK** provides conversational AI capabilities

## The Processing Pipeline

### Step 1: Document Ingestion

When an HR professional uploads a CV, it lands in Azure Data Lake Storage:

```csharp
DataLakeServiceClient dataLakeServiceClient = new DataLakeServiceClient(_storageConnectionString);
DataLakeFileSystemClient fileSystemClient = dataLakeServiceClient.GetFileSystemClient("landing");
DataLakeDirectoryClient directoryClient = fileSystemClient.GetDirectoryClient("documents");
```

We chose Data Lake over simple Blob Storage for its hierarchical namespace—organizing CVs by date, department, or job opening becomes trivial.

### Step 2: Intelligent Extraction

This is where the magic happens. Azure Document Intelligence uses a trained model to extract specific fields from each CV:

```csharp
public async Task<List<Dictionary<string, object>>> ProcessDocumentFromStream(Stream stream)
{
    var operation = await _formRecognizerClient.AnalyzeDocumentAsync(
        WaitUntil.Completed, 
        _modelId, 
        stream
    );
    
    var response = await operation.WaitForCompletionAsync();
    var forms = response.Value;

    foreach (AnalyzedDocument form in forms.Documents)
    {
        Dictionary<string, object> data = new Dictionary<string, object>();
        
        data["Name"] = form.Fields["Name"].Value;
        data["Email"] = SafeGetField(form, "Email");
        data["ProfessionalSummary"] = form.Fields["ProfessionalSummary"].Value;
        data["TechnicalSkills"] = form.Fields["TechnicalSkills"].Value;
        data["JobRole"] = form.Fields["JobRole"].Value;
        data["Education"] = form.Fields["Education"].Value;
        // ... additional fields
    }
}
```

The custom model knows where to find:
- Contact information (name, email, phone, LinkedIn, GitHub)
- Professional summary and current role
- Technical skills and certifications
- Work history with responsibilities
- Education and academic projects
- Languages spoken

### Step 3: Searchable Index

Extracted data feeds into Azure Cognitive Search, making it queryable:

```csharp
public async Task<IEnumerable<SearchResult<Dictionary<string, object>>>> ExecuteSearch(
    string query, 
    string language)
{
    var searchOptions = new SearchOptions 
    { 
        IncludeTotalCount = true, 
        Filter = $"Language eq '{language}'" 
    };
    
    var searchResults = await _searchClient.SearchAsync<Dictionary<string, object>>(
        query, 
        options: searchOptions
    );

    return searchResults.Value.GetResults();
}
```

The search index supports:
- **Full-text search** across all CV fields
- **Filtering** by language, skills, or experience level
- **Ranking** by relevance to the query

### Step 4: Intelligent Responses

When an HR professional asks a question, we combine search results with GPT-4:

```csharp
public async Task<string> CallOpenAIApi(object[] messages)
{
    var request = new 
    { 
        model = "gpt-4-turbo", 
        messages = messages, 
        max_tokens = 2048 
    };
    
    var response = await _httpClient.SendAsync(httpRequestMessage);
    var openAIAPIResponse = JsonSerializer.Deserialize<OpenAIChatResponse>(jsonResponse);
    
    return openAIAPIResponse?.choices?.FirstOrDefault()?.message?.content;
}
```

The result? Natural conversations like:

> **HR**: "Find me senior developers with Python and cloud experience"
> 
> **Candidex**: "Based on your query, here are the top candidates:
> - Maria Garcia, currently working as Senior Software Engineer at TechCorp
> - James Chen, Lead Developer at CloudSystems Inc.
> 
> Both have 5+ years of Python experience and AWS/Azure certifications. Would you like more details about their specific projects?"

## The Frontend Experience

We built the frontend in React with a clean, professional interface:

```jsx
// components/SearchResults.js
const SearchResults = ({ results }) => {
    return (
        <div className="results-container">
            {results.map(candidate => (
                <CandidateCard 
                    key={candidate.id}
                    name={candidate.Name}
                    role={candidate.CurrentRole}
                    skills={candidate.TechnicalSkills}
                    summary={candidate.ProfessionalSummary}
                />
            ))}
        </div>
    );
};
```

The chat interface allows natural language queries while displaying structured candidate information—the best of both worlds.

## Multi-Language Support

A key feature: our system handles CVs in multiple languages. The Document Intelligence model was trained on both English and Spanish documents, and we filter search results by language:

```csharp
var searchOptions = new SearchOptions 
{ 
    Filter = $"Language eq '{language}'" 
};
```

This ensures Spanish-speaking recruiters see Spanish CVs first, while maintaining the ability to search across all candidates when needed.

## What Worked Well

**Azure's Integration Story**: The services connect seamlessly. Documents flow from Storage to Document Intelligence to Search without complex glue code.

**C# and Strong Typing**: When dealing with structured data extraction, compile-time type checking prevents entire categories of bugs.

**Document Intelligence Training**: Once we trained the custom model with sample CVs, extraction accuracy exceeded 90% for most fields.

## What We'd Change

**Service Complexity**: Managing five Azure services means five sets of credentials, five monitoring dashboards, five potential failure points.

**Cost at Scale**: Per-document pricing for Document Intelligence adds up quickly with high CV volumes.

**Flexibility**: The trained model is specific to our CV format. New fields require retraining.

These observations would eventually lead us to explore a different architecture—but that's a story for a future post.

## Coming Up Next

In the next post, we'll dive deep into **Azure Document Intelligence**: how we trained our custom CV model, handled edge cases, and achieved reliable data extraction from wildly varied resume formats.

---

*Next up: [Training Azure Document Intelligence for CV Extraction](/posts/03-document-intelligence/)*

---

**About This Series**: This blog series documents the development of Candidex, exploring both the original C# Azure implementation and its evolution. We share technical decisions, challenges faced, and lessons learned along the way.

