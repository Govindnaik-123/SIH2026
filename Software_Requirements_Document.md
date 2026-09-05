# Software Requirements Document (SRD) — v2.0
## Gen AI Platform for Automated Content Transformation (SIH 2026 - Problem Statement 26154)

---

## 1. Problem Framing & Real-World Context

Organizations such as Computer Emergency Response Teams (CERTs), government media departments, PR/policy cells, and corporate crisis groups face a constant deluge of unstructured raw information. In high-stakes environments, this data must be digested rapidly and repurposed into audience-appropriate materials:
*   **Public Advisories** (clear, structured warnings)
*   **Executive Briefings** (summaries focused on risk and action)
*   **Media Communications** (LinkedIn posts, Twitter threads)
*   **Educational Materials** (Infographics, Slide decks, Video storyboards)

Currently, the process of manually reading sources, summarizing, checking facts, and writing different formats is time-consuming and highly prone to **factual drift** (where dates, numbers, and technical details diverge between media pieces). The objective of this platform is to automate the parsing, extraction, and generation flows, ensuring that **one source** yields **multiple consistent outputs** in seconds, eliminating manual bottlenecks.

---

## 2. Goals & Success Criteria

The system's performance and usability are measured by the following targets:

| Metric | Target / Success Criteria | Verification Method |
|---|---|---|
| **Consistency** | 100% agreement on critical entities (dates, severities, names) across all formats from a single source. | Verification script comparing `key_facts` extraction with output entities. |
| **Throughput** | Generate 7 parallel output formats from a 10-page document in under 30 seconds. | API response timer logs during end-to-end integration tests. |
| **Fault Tolerance** | A failure in one generator (e.g., presentation LLM call timeout) must not block or corrupt other outputs. | Mock injection testing during CI build checks. |
| **Ingestion Support** | Handle plain text, standard `.docx` formatting, multi-column `.pdf` extraction, and OCR on screenshot files. | File parsing unit test suite. |
| **Usability** | Operators can select target audience, tone, detail level, and language via a visual dashboard without writing prompts. | Frontend UI verification. |

---

## 3. System Scope

### 3.1 In Scope (MVP for SIH 2026)
*   **Multi-Format Ingestion**: Support for direct copy-paste text, `.docx`, `.pdf` documents, and `.png`/`.jpg` images (with text-based OCR and visual-context description).
*   **Unified Context Extraction**: Intermediate JSON-based `ContentBrief` summarizing core facts, entities, domain category, urgency, and recommended steps.
*   **7 Core Output Channels**:
    1.  *LinkedIn*: Engagement-focused formatting, hashtags, visual recommendations.
    2.  *Twitter/X*: Thread layout structure with character counts and engagement prompts.
    3.  *Advisory*: Strict, formal, action-first security layout.
    4.  *Video Script*: Narrative structure, scene descriptions, voiceovers, and subtitles.
    5.  *Infographic Content*: Structured layout blocks, suggestions for icons and visual emphasis.
    6.  *Executive Summary*: Strategic impact assessments, highlights, and operational recommendations.
    7.  *Presentation*: Multi-slide JSON containing title, bullets, notes, and visual layout guides.
*   **Direct File Exports**: Automated generation of `.docx` (Advisories, Exec Summaries), `.pptx` (Presentation Slides), and `.srt` (Subtitles).
*   **Operator Dashboard**: Modern responsive UI with parameter overrides (Tone, Audience, Language, Detail Level, Objective) and per-format editing/regeneration.

### 3.2 Out of Scope
*   **Direct Video Rendering**: The platform outputs video script/storyboard, audio subtitles, and instructions for production, but does not render MP4 files.
*   **Vector Graphic (SVG/PNG) Rendering of Infographics**: Renders structural layouts, copy, and icons, but does not compile the visual image itself (layout schema only).
*   **Multi-tenant Roles**: The MVP runs as a single-operator deployment without database-enforced role access.

---

## 4. Functional Modules & Checklist

### 4.1 Ingestion Module
1.  **Text Clean-Up**: Sanitize control characters, remove trailing whitespace, and reject inputs under 50 characters.
2.  **DOCX Extraction**: Parse structural files using XML extraction via `python-docx` to preserve tables, headers, and bullet structures.
3.  **PDF Extraction**: Parse files page-by-page using `pdfplumber`. Maintain columnar text continuity to avoid layout scrambling.
4.  **OCR & Vision Engine**:
    *   Detect if an uploaded image has text (screenshots, document pictures) and run Tesseract OCR.
    *   If no text, or if there is visual information (diagrams, architecture charts), run a Gemini multimodal vision check to describe the content.
5.  **Unified Ingestion Router**: Compile all parsed details into a single `IngestionResult` schema.

### 4.2 Context Extraction Module
1.  **Context Construction**: Input raw text to a highly analytical LLM model (Gemini Pro/Flash / Claude).
2.  **Structured Extraction**: Restrict LLM to output a verified JSON matching the `ContentBrief` schema.
3.  **Domain & Severity Categorization**: Automatically classify the source topic into domains (e.g., cybersecurity, health, general) and assign an urgency indicator (low, medium, high, critical).
4.  **Self-Correction**: Implement Pydantic schema verification. If the LLM generates bad JSON structure, immediately retry once with the error details.

### 4.3 Format Generation Modules
1.  **Modular Pipeline**: Execute format generation in parallel using async operations (`asyncio.gather`).
2.  **Input Isolations**: Every generator receives only the `ContentBrief` and `GenerationParams`. No direct access to raw input files is allowed, preventing format-specific hallucinations.
3.  **Channel Schemas**: Define, validate, and output Pydantic objects for each channel.
4.  **Style Overrides**: Map parameter settings (audience, tone, language) directly into system prompts.

### 4.4 Assembly & Export Module
1.  **Word Generation**: Write structured JSON variables into styled DOCX templates using `python-docx`.
2.  **Presentation Construction**: Map slide JSON arrays onto PowerPoint slide templates via `python-pptx`, setting title, body, and speaker notes.
3.  **Subtitle Formatting**: Convert voiceover subtitle timings into clean, standard SRT file layouts.

### 4.5 Operator Dashboard (Frontend)
1.  **Input Controls**: Drag-and-drop file upload zone + tabbed copy-paste interface.
2.  **Parameter Settings**: Selectors for tone, audience, language, detail, and objectives.
3.  **Card-Based Results Grid**: Each active channel displays its own visual status card (loading, error, success).
4.  **Interaction Panels**: Copy-to-clipboard buttons, file downloads, and per-format parameters with isolated regeneration controls.

---

## 5. API Specification (Contracts)

### 5.1 POST `/api/jobs`
Initiate ingestion, extraction, and generation.
*   **Request Schema**:
    ```json
    {
      "source_text": "string (optional if file is uploaded)",
      "output_types": ["linkedin", "twitter", "advisory", "video", "infographic", "exec_summary", "presentation"],
      "parameters": {
        "audience": "general_public | technical | executive | policy_maker",
        "tone": "formal | urgent | neutral | persuasive",
        "language": "en | es | fr | de | hi",
        "detail_level": "brief | standard | detailed",
        "objective": "inform | alert | persuade | educate"
      }
    }
    ```
*   **Multipart Form Data Support**: Optionally upload files (PDF, DOCX, Image) under field name `file`.
*   **Response Schema**:
    ```json
    {
      "job_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "status": "processing"
    }
    ```

### 5.2 GET `/api/jobs/{job_id}`
Query job progress, return completed formats, or show error codes for individual failed channels.
*   **Response Schema**:
    ```json
    {
      "job_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "status": "processing | completed | partial_failure | failed",
      "content_brief": {
        "topic": "Sophisticated Phishing Campaign targets Government Agencies",
        "summary": "Threat actors are targeting staff using realistic spear-phishing templates.",
        "key_facts": ["Launched on August 15, 2026", "Targets HR departments", "Uses spoofed gov domains"],
        "entities": ["HR staff", "Government Agencies"],
        "domain": "cybersecurity",
        "urgency_level": "critical",
        "recommended_actions": ["Enable MFA", "Disable external macros", "Report suspicious emails"],
        "sentiment": "alarming"
      },
      "outputs": {
        "linkedin": {
          "status": "success",
          "content": {
            "post_body": "Critical Alert: A targeted spear-phishing campaign has been detected targeting government HR personnel...",
            "hashtags": ["Cybersecurity", "PhishingAlert", "GovSec"],
            "recommended_visual_concept": "Graphic depicting email warning overlay on an official folder"
          }
        },
        "advisory": {
          "status": "success",
          "content": {
            "title": "Threat Advisory: Targeted HR Spear-Phishing Campaign",
            "summary": "Sophisticated actors targeting government staff using spoofed administrative emails.",
            "threat_details": "Incident began mid-August 2026. Attackers send PDF templates containing macro payloads...",
            "impact_analysis": "Potential exposure of staff credentials and internal networks.",
            "recommended_actions": ["Mandate multi-factor authentication", "Block macros", "Deploy email filters"],
            "contact_info": "Reach out to the security desk at cert-alert@gov.in"
          },
          "file_url": "/exports/9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d/advisory.docx"
        },
        "presentation": {
          "status": "failed",
          "error": "LLM API Call timed out during slide layout creation."
        }
      }
    }
    ```

### 5.3 POST `/api/jobs/{job_id}/outputs/{type}/regenerate`
Re-run a specific format generator with modified parameters.
*   **Request Schema**:
    ```json
    {
      "parameters": {
        "audience": "executive",
        "tone": "formal",
        "language": "en",
        "detail_level": "brief",
        "objective": "inform"
      }
    }
    ```
*   **Response Schema**:
    ```json
    {
      "status": "success",
      "output_type": "linkedin",
      "content": {
        "post_body": "Executive Brief: New threat directives highlight targeting of HR channels...",
        "hashtags": ["SecurityBriefing", "RiskManagement"],
        "recommended_visual_concept": "Clean executive slide style with risk indicators"
      }
    }
    ```

### 5.4 GET `/api/jobs/{job_id}/outputs/{type}/download`
Download the compiled export file (e.g. PPTX, DOCX, SRT) for the specified format.
*   **Response**: File download stream matching content-type `application/vnd.openxmlformats-officedocument.wordprocessingml.document` or similar.

---

## 6. Non-Functional Requirements

### 6.1 Performance & Latency
*   **Parallelization**: Backend must gather generator promises asynchronously to keep processing times low.
*   **Under 30s Pipeline**: Ingestion + Context Extraction + 7 formats + Document Assembly must execute in under 30 seconds for standard documents.

### 6.2 Reliability & Fault Tolerance
*   **Graceful Degradation**: If one generator fails due to token limits, validation errors, or API hiccups, it must write its error string to the database and let other pipelines continue.
*   **Retry Mechanisms**: Single self-correcting logic loop for prompt-to-JSON validation.

### 6.3 Security & Infrastructure
*   **Secure API Keys**: Secrets are managed through `.env` configurations and are never committed to code repositories.
*   **Safe File Deletion**: Jobs exports are stored in localized temporary folders on the server disk, allowing clean scheduling of cleanup scripts.

---

## 7. Comprehensive Test Matrix

| Category | Goal | Test Input | Expected Result |
|---|---|---|---|
| **Ingestion** | Verify PDF text parsing | Multi-column PDF threat advisory | Text matches source layout, columns extracted in reading order. |
| **Ingestion** | Verify DOCX text parsing | DOCX with embedded tables | Flat text with table rows mapped as CSV-like strings. |
| **Ingestion** | OCR validation | Screenshot of security alert | Extract clean text from image matching 95%+ of screenshot characters. |
| **Context Extraction** | Schema validation | Long raw news article | Produces a clean Pydantic ContentBrief containing valid JSON. |
| **Context Extraction** | Error recovery | Structured prompt with forced JSON parsing error | Validator catches format error, orchestrator retries, and extracts correct brief. |
| **Generation** | Tone mapping | Brief + tone: 'urgent' | LinkedIn post uses high-action vocabulary, warnings, and warning emojis. |
| **Export** | DOCX creation | Valid Advisory JSON | Generates download file opening cleanly in MS Word, containing headers. |
| **Export** | PPTX creation | Valid Slide Deck JSON | Generates PPTX, opening cleanly in PowerPoint, each slide structured correctly. |
| **Integration** | Parallel run check | Multi-format execution | All selected formats generated, file URLs resolved. |
