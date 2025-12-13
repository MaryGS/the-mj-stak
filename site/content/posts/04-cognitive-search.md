---
title: "Azure Cognitive Search: Building Intelligent Candidate Discovery"
date: 2024-07-20
draft: false
description: "How we built a searchable candidate index using Azure Cognitive Search and enhanced it with OpenAI"
tags: ["Azure", "Cognitive Search", "search", "OpenAI", "C#"]
series: ["Candidex"]
weight: 4
ShowToc: true
TocOpen: false
---

## From Data to Discovery

In the [previous post](/posts/03-document-intelligence/), we extracted structured data from CVs using Azure Document Intelligence. Now we had clean, organized candidate information—but data sitting in storage isn't useful. HR professionals need to *find* candidates, and they need to find them fast.

Enter Azure Cognitive Search.

## Why Azure Cognitive Search?

We evaluated several options:

| Option | Pros | Cons |
|--------|------|------|
| **SQL Database** | Familiar, transactional | Poor full-text search, no ranking |
| **Elasticsearch** | Powerful, flexible | Operational overhead, self-managed |
| **Azure Cognitive Search** | Managed, Azure-native, AI features | Vendor lock-in, cost |

For our use case, Azure Cognitive Search won because:

1. **Zero infrastructure management**: No clusters to size, no nodes to balance
2. **Native Azure integration**: Direct connections to Blob Storage, Cosmos DB, and other sources
3. **Built-in AI enrichment**: Entity extraction, language detection, sentiment (though we handled these elsewhere)
4. **Relevance tuning**: Scoring profiles to rank candidates by match quality

## Designing the Search Index

A search index is like a highly optimized database table designed for fast text retrieval. We defined our schema to match our extracted CV fields:

```json
{
  "name": "candidates-index",
  "fields": [
    { "name": "id", "type": "Edm.String", "key": true },
    { "name": "Name", "type": "Edm.String", "searchable": true },
    { "name": "Email", "type": "Edm.String", "filterable": true },
    { "name": "Phone", "type": "Edm.String" },
    { "name": "ProfessionalSummary", "type": "Edm.String", "searchable": true },
    { "name": "TechnicalSkills", "type": "Edm.String", "searchable": true, "filterable": true },
    { "name": "JobRole", "type": "Edm.String", "searchable": true, "facetable": true },
    { "name": "Company", "type": "Edm.String", "searchable": true },
    { "name": "WorkPeriod", "type": "Edm.String" },
    { "name": "JobResponsibilities", "type": "Edm.String", "searchable": true },
    { "name": "Education", "type": "Edm.String", "searchable": true },
    { "name": "CurrentRole", "type": "Edm.String", "searchable": true, "facetable": true },
    { "name": "Languages", "type": "Edm.String", "filterable": true },
    { "name": "Language", "type": "Edm.String", "filterable": true },
    { "name": "Certifications", "type": "Edm.String", "searchable": true }
  ]
}
```

Key decisions:
- **Searchable fields**: Text that should match user queries (skills, summaries, responsibilities)
- **Filterable fields**: Fields for exact matching (language, email)
- **Facetable fields**: Fields for aggregation/grouping (job roles, current positions)

## The Search Service Implementation

Our `SearchService` class wraps the Azure SDK:

```csharp
public class SearchService
{
    private readonly SearchClient _searchClient;
    private readonly ResultProcessor _resultProcessor;
    private readonly OpenAIService _openAIService;

    public SearchService(IConfiguration configuration, OpenAIService openAIService)
    {
        var config = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("local.settings.json", optional: true, reloadOnChange: true)
            .Build();

        var searchServiceEndpoint = config["Values:SearchServiceEndpoint"];
        var searchServiceApiKey = config["Values:SearchServiceApiKey"];
        var indexName = config["Values:IndexName"];

        AzureKeyCredential credential = new AzureKeyCredential(searchServiceApiKey);
        _searchClient = new SearchClient(
            new Uri(searchServiceEndpoint), 
            indexName, 
            credential
        );

        _resultProcessor = new ResultProcessor();
        _openAIService = openAIService;
    }

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
}
```

The `ExecuteSearch` method supports:
- **Full-text queries**: "python developer with AWS experience"
- **Language filtering**: Show only Spanish CVs to Spanish-speaking recruiters
- **Result counting**: Know how many total matches exist

## Processing Search Results

Raw search results need transformation for the UI. Our `ResultProcessor` handles this:

```csharp
public class ResultProcessor
{
    public IEnumerable<Dictionary<string, object>> ProcessResults(
        IEnumerable<SearchResult<Dictionary<string, object>>> searchResults, 
        string context)
    {
        return searchResults.Select(result => result.Document);
    }

    public string GenerateAnswer(
        IEnumerable<Dictionary<string, object>> processedResults, 
        string context)
    {
        switch (context.ToLower())
        {
            case "offer":
                return GenerateOfferAnswer(processedResults);
            case "role":
                return GenerateRoleAnswer(processedResults);
            default:
                return "I couldn't find relevant information. Could you rephrase your question?";
        }
    }

    private string GenerateOfferAnswer(IEnumerable<Dictionary<string, object>> processedResults)
    {
        var candidates = processedResults
            .Select(result => new
            {
                Name = GetValueOrDefault(result, "Name"),
                JobRole = GetValueOrDefault(result, "JobRole"),
                Company = GetValueOrDefault(result, "Company"),
            })
            .ToList();

        if (!candidates.Any())
        {
            return "No candidates match the specified job offer. Please provide more details.";
        }

        var sb = new StringBuilder();
        sb.AppendLine("Here are the top candidates for this position:");
        
        foreach (var candidate in candidates)
        {
            sb.AppendLine($"- {candidate.Name} currently working as {candidate.JobRole} at {candidate.Company}");
        }
        
        sb.AppendLine("\nLet me know if you need more details about any candidate.");

        return sb.ToString();
    }

    private string GenerateRoleAnswer(IEnumerable<Dictionary<string, object>> processedResults)
    {
        var candidates = processedResults
            .Select(result => new
            {
                Name = GetValueOrDefault(result, "Name"),
                JobRole = GetValueOrDefault(result, "JobRole"),
                ProfessionalSummary = GetValueOrDefault(result, "ProfessionalSummary"),
                TechnicalSkills = GetValueOrDefault(result, "TechnicalSkills"),
            })
            .ToList();

        if (!candidates.Any())
        {
            return "No candidates match the specified role. Please provide more details.";
        }

        var sb = new StringBuilder();
        sb.AppendLine("Candidates matching this role:");
        
        foreach (var candidate in candidates)
        {
            sb.AppendLine($"- {candidate.Name} ({candidate.JobRole})");
            sb.AppendLine($"  Summary: {candidate.ProfessionalSummary}");
            sb.AppendLine($"  Skills: {candidate.TechnicalSkills}");
        }

        return sb.ToString();
    }

    private string GetValueOrDefault(Dictionary<string, object> document, string key)
    {
        return document.TryGetValue(key, out var value) 
            ? value?.ToString() ?? string.Empty 
            : string.Empty;
    }
}
```

This gives us **context-aware responses**: searching for a job offer generates different output than searching for a specific role.

## Enhancing with OpenAI

Search alone returns matches. But HR professionals want *insights*. That's where OpenAI comes in.

We feed search results to GPT-4 for intelligent summarization:

```csharp
public class OpenAIService
{
    private readonly HttpClient _httpClient;
    private readonly string _openAIApiKey;

    public OpenAIService(IConfiguration configuration)
    {
        _httpClient = new HttpClient();
        _openAIApiKey = config["Values:OpenAIKey"] ?? "";
    }

    public async Task<string> CallOpenAIApi(object[] messages)
    {
        var request = new 
        { 
            model = "gpt-4-turbo", 
            messages = messages, 
            max_tokens = 2048 
        };
        
        var jsonContent = JsonSerializer.Serialize(request);

        var httpRequestMessage = new HttpRequestMessage(
            HttpMethod.Post, 
            "https://api.openai.com/v1/chat/completions")
        {
            Headers =
            {
                { "Authorization", $"Bearer {_openAIApiKey}" },
                { "Accept", "application/json" }
            },
            Content = new StringContent(jsonContent, Encoding.UTF8, "application/json")
        };

        var response = await _httpClient.SendAsync(httpRequestMessage);
        response.EnsureSuccessStatusCode();

        var jsonResponse = await response.Content.ReadAsStringAsync();
        var openAIAPIResponse = JsonSerializer.Deserialize<OpenAIChatResponse>(jsonResponse);
        
        return openAIAPIResponse?.choices?.FirstOrDefault()?.message?.content;
    }
}
```

A typical flow:

1. HR asks: "Who would be best for a senior backend role requiring Python and cloud?"
2. Cognitive Search returns 15 matching candidates
3. We send those results to GPT-4 with context
4. GPT-4 returns: "Based on the candidates, I recommend Maria Garcia and James Chen. Maria has 8 years of Python experience and AWS certifications. James led cloud migrations at his previous company..."

## The Chat Interface

Our React frontend provides a conversational experience:

```javascript
// components/ChatInterface.js
const ChatInterface = () => {
    const [messages, setMessages] = useState([]);
    const [input, setInput] = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        
        // Add user message
        setMessages([...messages, { role: 'user', content: input }]);
        
        // Call backend
        const response = await searchService.query(input);
        
        // Add assistant response
        setMessages([
            ...messages, 
            { role: 'user', content: input },
            { role: 'assistant', content: response }
        ]);
        
        setInput('');
    };

    return (
        <div className="chat-container">
            <MessageList messages={messages} />
            <form onSubmit={handleSubmit}>
                <input 
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    placeholder="Ask about candidates..."
                />
                <button type="submit">Search</button>
            </form>
        </div>
    );
};
```

HR professionals can ask natural questions and receive intelligent, contextualized responses.

## Performance Considerations

### Search Latency

Azure Cognitive Search is fast—typical queries return in 50-200ms. For our candidate pool (hundreds of CVs), performance was never an issue.

### OpenAI Latency

GPT-4 calls add 2-5 seconds. We mitigated this with:
- **Streaming responses**: Show partial results as they arrive
- **Caching**: Identical queries return cached results
- **Progressive enhancement**: Show search results immediately, then enhance with AI summary

### Index Updates

When new CVs are processed, we push updates to the search index:

```csharp
public async Task IndexDocuments(List<Dictionary<string, object>> documents)
{
    var batch = IndexDocumentsBatch.Upload(documents);
    await _searchClient.IndexDocumentsAsync(batch);
}
```

Near-real-time indexing means newly processed CVs are searchable within seconds.

## What We Learned

### Successes

**Relevance was good out of the box**: Azure's default scoring algorithm produced reasonable rankings without custom tuning.

**Filtering + search combo**: "Python developers who speak Spanish" translated cleanly to a search query with filters.

**Facets for exploration**: Showing counts by job role helped HR understand their candidate pool.

### Challenges

**Synonym handling**: "Software Engineer" and "Software Developer" are the same role but searched differently. We added a synonym map to handle common variations.

**Skills normalization**: "JS", "JavaScript", "ECMAScript" should all match. This required preprocessing at indexing time.

**No semantic understanding**: Traditional search matches words, not meaning. "Cloud experience" doesn't automatically match "AWS certified."

This last point would become a major driver for architectural evolution—but that's a topic for future exploration.

## The Complete Flow

Putting it all together:

```text
  +------------------------------------------------------------------+
  |                         User Query                               |
  |            "Find Python developers with 5+ years"                |
  +------------------------------------------------------------------+
                                |
                                v
  +------------------------------------------------------------------+
  |                    Azure Cognitive Search                        |
  |   - Full-text search across TechnicalSkills, ProfessionalSummary |
  |   - Filter by Language if specified                              |
  |   - Return top 10 ranked matches                                 |
  +------------------------------------------------------------------+
                                |
                                v
  +------------------------------------------------------------------+
  |                       Result Processor                           |
  |   - Extract relevant fields                                      |
  |   - Format for display                                           |
  |   - Prepare context for OpenAI                                   |
  +------------------------------------------------------------------+
                                |
                                v
  +------------------------------------------------------------------+
  |                         OpenAI GPT-4                             |
  |   - Analyze candidate profiles                                   |
  |   - Generate intelligent summary                                 |
  |   - Recommend best matches with reasoning                        |
  +------------------------------------------------------------------+
                                |
                                v
  +------------------------------------------------------------------+
  |                        User Response                             |
  |   "Based on your criteria, here are the top candidates:          |
  |    1. Maria Garcia - 7 years Python, AWS certified               |
  |    2. James Chen - 6 years Python, led backend team              |
  |    Would you like more details about either candidate?"          |
  +------------------------------------------------------------------+
```

## Wrapping Up the Azure Implementation

Over these posts, we've built a complete CV matching system:

1. **Storage**: Azure Data Lake for document management
2. **Extraction**: Document Intelligence for structured data
3. **Search**: Cognitive Search for intelligent queries
4. **AI**: OpenAI for natural language interaction

The system works. HR teams can upload CVs, search candidates naturally, and get intelligent recommendations.

But as we'll explore in future posts, this architecture has limitations—and the rapid evolution of LLM technology opened new possibilities we couldn't ignore.

---

*Next up: [Lessons Learned: Building Candidex with Azure and C#](/posts/05-lessons-learned-azure/)*

---

**About This Series**: This blog series documents the development of Candidex, an AI-powered CV matching system. We share technical decisions, code examples, and lessons learned building with Azure, C#, and modern AI services.

