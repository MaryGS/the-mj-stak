---
title: "Training Azure Document Intelligence for CV Extraction"
date: 2024-07-15
draft: false
description: "How we trained a custom Azure Document Intelligence model to extract structured data from unstructured CVs"
tags: ["Azure", "Document Intelligence", "Form Recognizer", "AI", "data extraction"]
series: ["Candidex"]
weight: 3
ShowToc: true
TocOpen: false
---

## The Challenge of Unstructured Documents

CVs are notoriously inconsistent. One candidate uses a sleek single-column template. Another prefers a creative two-column layout. A third submits a plain text document with minimal formatting. Yet somewhere in each document lies the same core information: name, contact details, experience, skills, education.

Traditional approaches to this problem involve:
- **Regular expressions**: Brittle, break with format changes
- **Template matching**: Works only for exact layouts
- **Manual data entry**: Accurate but doesn't scale

Azure Document Intelligence (formerly Form Recognizer) offers a different approach: train a model to *understand* document structure and extract fields regardless of layout.

## How Document Intelligence Works

At its core, Document Intelligence combines:

1. **Optical Character Recognition (OCR)**: Reading text from documents
2. **Layout Analysis**: Understanding document structure (tables, sections, headers)
3. **Custom Field Extraction**: Learning which content maps to which fields

When you train a custom model, you're teaching Azure which parts of your documents correspond to which data fields. Once trained, the model generalizes to new documents with similar content but different layouts.

## Our CV Data Model

Before training, we defined what we wanted to extract. Based on typical HR requirements, we identified these fields:

| Field | Description | Example |
|-------|-------------|---------|
| `Name` | Full name | "Maria Garcia" |
| `Email` | Email address | "maria@email.com" |
| `Phone` | Phone number | "+1 555-0123" |
| `LinkedIn` | LinkedIn profile URL | "linkedin.com/in/mariagarcia" |
| `GitHub` | GitHub profile URL | "github.com/mgarcia" |
| `CurrentRole` | Current job title | "Senior Software Engineer" |
| `Company` | Current employer | "TechCorp Inc." |
| `ProfessionalSummary` | Career summary/objective | "10+ years building scalable systems..." |
| `TechnicalSkills` | Technical skills list | "Python, AWS, Docker, Kubernetes" |
| `JobRole` | Primary job category | "Software Engineer" |
| `WorkPeriod` | Employment dates | "2019 - Present" |
| `JobResponsibilities` | Job duties description | "Led team of 5 developers..." |
| `Education` | Degrees and institutions | "MS Computer Science, Stanford" |
| `Certifications` | Professional certifications | "AWS Solutions Architect" |
| `Languages` | Languages spoken | "English, Spanish, French" |

## The Training Process

### Step 1: Gather Training Data

We collected 50+ sample CVs representing the variety we expected:
- Different layouts (single column, two column, creative designs)
- Different formats (PDF, DOCX, images)
- Different languages (English and Spanish)
- Different career levels (entry level to executive)

**Pro tip**: Include edge cases early. That creative CV with the sidebar? The one-page minimalist design? Train on those from the start.

### Step 2: Label the Documents

Using Azure's Document Intelligence Studio, we manually labeled each training document:

1. Upload documents to the studio
2. Draw bounding boxes around each field
3. Assign field names to each region
4. Repeat for all training documents

This is time-consuming but critical. The quality of your labels directly impacts model accuracy.

### Step 3: Train the Model

With labeled documents ready, training is straightforward:

```bash
# Using Azure CLI (or through the portal)
az cognitiveservices account deployment create \
  --name my-form-recognizer \
  --resource-group hr-helper-rg \
  --model-name cv-extraction-model \
  --model-version 1.0
```

Training typically takes 15-30 minutes depending on document complexity and volume.

### Step 4: Test and Iterate

We tested the model against a held-out set of CVs and measured extraction accuracy:

| Field | Accuracy (v1) | After Iteration | 
|-------|---------------|-----------------|
| Name | 98% | 99% |
| Email | 95% | 98% |
| Phone | 92% | 96% |
| TechnicalSkills | 85% | 92% |
| ProfessionalSummary | 78% | 88% |
| JobResponsibilities | 72% | 85% |

Free-form text fields (summaries, responsibilities) were harder than structured fields (email, phone). We improved accuracy by:
- Adding more training examples for problematic layouts
- Refining field boundaries to include complete sections
- Separating compound fields (e.g., splitting "JobResponsibilities2" for multi-role CVs)

## Integrating with C#

Once trained, integrating the model into our application was clean:

```csharp
public class DocumentDataExtractor
{
    private IDocumentAnalysisWrapper _formRecognizerClient;
    private string _modelId;

    public DocumentDataExtractor(
        string storageConnectionString, 
        string formRecognizerEndpoint, 
        string formRecognizerApiKey, 
        string modelId)
    {
        _modelId = modelId;
        _formRecognizerClient = new DocumentAnalysisWrapper(
            formRecognizerEndpoint, 
            formRecognizerApiKey
        );
    }

    public async Task<List<Dictionary<string, object>>> ProcessDocumentFromStream(Stream stream)
    {
        var extractedData = new List<Dictionary<string, object>>();
        
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

            data["id"] = Guid.NewGuid().ToString();
            data["Name"] = form.Fields["Name"].Value;
            data["Email"] = SafeGetField(form, "Email");
            data["Phone"] = form.Fields["Phone"].Value;
            data["ProfessionalSummary"] = form.Fields["ProfessionalSummary"].Value;
            data["TechnicalSkills"] = form.Fields["TechnicalSkills"].Value;
            // ... additional fields

            extractedData.Add(data);
        }
        
        return extractedData;
    }

    private string SafeGetField(AnalyzedDocument form, string fieldName, string defaultValue = "")
    {
        if (form.Fields.Any(t => t.Key == fieldName)) 
        {
            return form.Fields[fieldName].Value.ToString();
        }
        return defaultValue;
    }
}
```

The `SafeGetField` helper handles optional fields gracefully—not every CV includes a GitHub profile or certifications.

## Processing Documents at Scale

For batch processing, we iterate through documents in Azure Data Lake:

```csharp
public async Task<List<Dictionary<string, object>>> ExtractDataFromDocumentsAsync()
{
    var extractedData = new List<Dictionary<string, object>>();
    
    DataLakeServiceClient dataLakeServiceClient = new DataLakeServiceClient(_storageConnectionString);
    DataLakeFileSystemClient fileSystemClient = dataLakeServiceClient.GetFileSystemClient("landing");
    DataLakeDirectoryClient directoryClient = fileSystemClient.GetDirectoryClient("documents");

    await foreach (PathItem pathItem in directoryClient.GetPathsAsync(recursive: false))
    {
        if (pathItem.IsDirectory ?? false)
            continue;

        try
        {
            var fileName = pathItem.Name.Substring(pathItem.Name.IndexOf('/'));
            DataLakeFileClient fileClient = directoryClient.GetFileClient(fileName);

            using var stream = await fileClient.OpenReadAsync();
            var extractedDataLocal = await ProcessDocumentFromStream(stream);
            extractedData.AddRange(extractedDataLocal);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error processing blob: {pathItem.Name}");
            Console.WriteLine($"Exception: {ex.Message}");
        }
    }

    return extractedData;
}
```

Error handling is crucial here—one malformed document shouldn't stop the entire batch.

## Lessons Learned

### What Worked

**Pre-built analysis for common patterns**: Document Intelligence automatically detects emails, phone numbers, and URLs even without specific training.

**Confidence scores**: Each extracted field includes a confidence score, letting us flag low-confidence extractions for human review.

**Incremental training**: Adding new training documents to improve accuracy without starting from scratch.

### Challenges We Faced

**Multi-page CVs**: Some candidates submit 3+ page documents. Ensuring continuity across pages required careful field boundary management.

**Tables vs. prose**: Work experience in a table format extracted differently than narrative descriptions. We trained on both.

**Handwritten notes**: Occasionally, scanned documents included handwritten annotations. OCR quality dropped significantly for these.

**Language mixing**: Some candidates mixed English and Spanish in the same CV. Field extraction remained accurate, but downstream translation became necessary.

## Cost Considerations

Document Intelligence pricing is per-document:
- **Custom model analysis**: ~$10 per 1,000 pages
- **Training**: Included with the service
- **Storage**: Separate Azure Storage costs

For our use case (hundreds of CVs per month), costs were reasonable. At enterprise scale (tens of thousands of documents), costs would require careful budgeting.

## The Testing Strategy

We wrapped the Document Intelligence client for testability:

```csharp
public interface IDocumentAnalysisWrapper 
{
    Task<AnalyzeDocumentOperation> AnalyzeDocumentAsync(
        WaitUntil waitUntil, 
        string modelId, 
        Stream document, 
        AnalyzeDocumentOptions options = default,
        CancellationToken cancellationToken = default
    );
}
```

This allowed unit testing with mocked responses:

```csharp
[Test]
public async Task ProcessDocument_ExtractsNameAndEmail()
{
    var mockWrapper = new Mock<IDocumentAnalysisWrapper>();
    // Setup mock to return test data
    
    var extractor = new DocumentDataExtractor(mockWrapper.Object);
    var result = await extractor.ProcessDocumentFromStream(testStream);
    
    Assert.AreEqual("Test Candidate", result[0]["Name"]);
    Assert.AreEqual("test@email.com", result[0]["Email"]);
}
```

## What's Next

With structured data extracted from CVs, the next challenge was making it searchable. In the next post, we'll explore how Azure Cognitive Search indexes candidate data and enables intelligent queries—from simple filters to semantic search.

---

*Next up: [Azure Cognitive Search: Building Intelligent Candidate Discovery](/posts/04-cognitive-search/)*

---

**About This Series**: This blog series documents the development of Candidex. We share technical decisions, code examples, and lessons learned from building AI-powered recruitment tools.

