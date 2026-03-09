# Meitheal User Guide

## 1. Introduction
Meitheal is a self-hosted, automated research paper cataloging system designed to streamline how researchers manage, read, and analyze their libraries. By simply providing a PDF or an arXiv identifier, Meitheal automatically extracts the text, generates a summary, identifies key topics (tags), parses cited references, and attempts to resolve forward citations. It organizes your library into an easily searchable, unified interface, saving you hours of manual metadata entry and cross-referencing.

## 2. Main Features
- **Automated PDF Processing:** Upload a PDF and let the system extract text, metadata, and references.
- **arXiv Integration:** Import papers directly by providing their arXiv ID.
- **AI-Assisted Summarization & Tagging:** Automatically generates a TL;DR and relevant topic tags for each paper.
- **Reference & Citation Extraction:** Identifies papers cited by your document and attempts to find papers that cite your document using external scholarly APIs.
- **Asynchronous Processing:** Background jobs handle heavy extraction tasks without freezing the user interface.
- **Full-Text Search:** Query your entire library by keyword, author, year, or tag.
- **Self-Hosted Data:** All files and databases are stored locally on your machine for complete privacy.

## 3. Step-by-Step Guides

### Uploading a PDF paper
1. Navigate to the **Add Paper** page.
2. Select the **Upload PDF** option.
3. Choose the PDF file from your computer and click **Upload**.
4. The system will immediately accept the file and assign it a background job. You will be redirected to the job status page where you can monitor its progress through the pipeline stages (parsing, summarizing, etc.).

### Importing a paper from arXiv
1. Navigate to the **Add Paper** page.
2. Select the **Import from arXiv** option.
3. Enter the standard arXiv identifier (e.g., `2303.08774`).
4. Click **Import**. The system will fetch the PDF and metadata directly from arXiv and start the background processing job.

### Browsing your library
1. Navigate to the **Papers** (or Library) page.
2. You will see a paginated list of all successfully processed papers.
3. Each item displays the title, authors, year, and a brief snippet.
4. Use the pagination controls at the bottom to navigate through large collections.

### Searching papers
1. On the **Papers** page, locate the search bar.
2. Enter keywords (e.g., `neural networks`) into the `q` search field to perform a full-text search across titles, abstracts, and extracted content.
3. You can combine your text search with filters:
   - **Tag:** Filter by specific automatically generated topics.
   - **Author:** Find papers written by a specific person.
   - **Year:** Restrict results to a specific publication year.
4. Press Enter or click **Search** to view the filtered results.

### Viewing a paper's detail
Click on any paper in the library to view its full details. The detail page includes:
- **Metadata:** Title, authors, publication date, and source (e.g., arXiv).
- **Summary / TL;DR:** An automatically generated summary of the paper's core contributions.
- **Tags:** A list of topics extracted from the text.
- **Full Text:** The raw text extracted from the PDF, accessible via the expandable Full Text section.
- **References:** A list of papers cited by this document. Resolved entries include a DOI link.
- **Cited By:** A list of other papers in your library that cite this document.

### Understanding processing status
When a paper is added, it goes through a multi-stage background pipeline. Here is what each status means:

| Status | Meaning |
|---|---|
| **QUEUED** | The job is waiting for an available worker thread. |
| **RUNNING** | The job has started. |
| **PARSE** | Extracting raw text and metadata from the PDF. |
| **CHUNK** | Breaking the extracted text into overlapping segments. |
| **SUMMARIZE** | Generating the TL;DR summary. |
| **TAG** | Identifying core keywords and topics. |
| **REF_EXTRACT** | Scanning the document for cited references. |
| **REF_RESOLVE** | Looking up references against external databases (OpenAlex, CrossRef). |
| **PROCESSED** | The paper is fully cataloged and available in your library. |
| **FAILED** | An error occurred. The job history will show the failure reason. |

### Viewing job history
1. Navigate to the **Jobs** page.
2. This page lists all historical and active processing jobs.
3. You can see the status of each job, the associated paper, and any error message if the job failed.

### Using the metrics endpoint
For administrators or power users, Meitheal provides a health and statistics endpoint.

```
GET http://localhost:8000/api/metrics
```

The response is a JSON object showing:
- `papers_total` — how many papers are in your library
- `jobs` — counts of jobs by status (queued, running, processed, failed)
- `resolution` — breakdown of reference resolution results (resolved, unresolved, and the resolution rate as a fraction from 0.0 to 1.0)

## 4. Data & Privacy
Meitheal is designed to be a local, self-hosted application.
- All uploaded PDFs, extracted text, and generated metadata are stored on your local machine's disk and SQLite database.
- External calls are only made to public scholarly databases (arXiv, CrossRef, OpenAlex) to fetch metadata or resolve citations.
- No personal data, reading habits, or proprietary documents are sent to the cloud.

## 5. Limitations (MVP)
As an MVP, Meitheal has a few known limitations:

- **Extractive summaries only:** The summarizer uses sentence extraction rather than a generative LLM. Summaries are pulled from the text's abstract rather than rewritten from scratch.
- **Reference extraction requires a structured header:** The system looks for a "References" or "Bibliography" section heading. Papers with non-standard layouts may not have their references extracted.
- **Citation resolution depends on external APIs:** Resolving references relies on OpenAlex and CrossRef. Papers not indexed in those databases will appear as unresolved.
- **No user authentication:** The system is a single-user local tool. There is no login system — anyone with access to the network port can view the library.
