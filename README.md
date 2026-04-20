# Job Application AI System using n8n

An AI-powered job application workflow built with **n8n**, **Apify**, **Google Docs**, **Google Drive**, **Google Sheets**, and an **LLM** to automate job scraping, job relevance screening, resume tailoring, formatted resume generation, and application tracking.

This project is designed for **Business Analyst**, **Data Analyst**, and **Financial Analyst** job searches.

---

## Overview

This workflow does the following:

1. Takes a LinkedIn Jobs search URL as input
2. Scrapes job listings using Apify
3. Loops through jobs one by one
4. Reads the base resume from Google Docs
5. Uses AI to check whether each job is relevant
6. Filters out irrelevant jobs
7. Uses AI to tailor the resume to the selected job
8. Copies a pre-formatted Google Docs resume template
9. Replaces placeholders in the template with tailored resume content
10. Optionally exports the tailored resume as PDF
11. Logs the job and generated resume details in Google Sheets

This solves one major problem in most AI job workflows:

**Instead of dumping raw AI text into a blank Google Doc, this system uses a pre-formatted Google Docs template so the final resume keeps the same professional layout.**

---

## Architecture

### Workflow Summary

```text
Manual Trigger
→ Set Job Search Input
→ Apify: Scrape Jobs
→ Limit / Loop Over Items
→ Google Docs: Get Base Resume
→ AI: Check Relevance
→ Filter Relevant Jobs
→ AI: Tailor Resume
→ Build Placeholder Replacement Requests
→ Google Drive API: Copy Resume Template
→ Google Docs API: Fill Template
→ Google Drive API: Export PDF
→ Google Sheets: Append Job Record
```

---

## Features

- Scrapes public LinkedIn job search results
- Filters jobs based on relevance using AI
- Tailors resume content without changing factual experience
- Preserves resume formatting using Google Docs template placeholders
- Exports tailored resumes as Google Docs or PDF
- Logs all processed jobs into Google Sheets
- Easy to extend for cover letters or future automation

---

## Tech Stack

- **n8n** - workflow orchestration
- **Apify** - LinkedIn job scraping
- **Google Docs API** - formatted resume generation
- **Google Drive API** - copy template and export PDF
- **Google Sheets API** - job tracking
- **LLM / AI model** - job relevance scoring and resume tailoring

---

## Repository Structure

```text
job-application-ai-n8n/
├── README.md
├── workflows/
│   └── job-application-ai-system.json
├── docs/
│   ├── architecture.png
│   ├── workflow-screenshot.png
│   ├── placeholder-template-guide.md
│   └── troubleshooting.md
├── prompts/
│   ├── checks-relevance.txt
│   └── customized-resume.txt
├── examples/
│   ├── apify-job-output.json
│   ├── tailored-resume-output.json
│   └── google-sheet-columns.csv
└── .env.example
```

---

## Prerequisites

Before importing the workflow into n8n, set up the following:

### 1. n8n

- Self-hosted or n8n cloud instance
- Working execution environment

### 2. Apify Account

- Create an Apify account
- Get your Apify API token
- Select a LinkedIn Jobs scraper actor
- Use a **public LinkedIn Jobs search URL**, not a logged-in/private search URL

### 3. Google Cloud Setup

Enable the following APIs in Google Cloud:

- Google Docs API
- Google Drive API
- Google Sheets API

Create OAuth credentials and connect them in n8n.

### 4. LLM Provider

Set up your LLM credentials in n8n for:

- relevance check
- tailored resume generation

### 5. Google Docs Resume Template

Create a **master Google Docs template** that matches your actual resume layout.

This template should contain placeholders such as:

```text
{{NAME}}
{{CITY_STATE}}
{{PHONE}}
{{EMAIL}}
{{GITHUB}}
{{PORTFOLIO}}
{{LINKEDIN}}

{{EXP1_COMPANY}}
{{EXP1_DATES}}
{{EXP1_TITLE}}
{{EXP1_B1}}
{{EXP1_B2}}
{{EXP1_B3}}

{{EXP2_COMPANY}}
{{EXP2_DATES}}
{{EXP2_TITLE}}
{{EXP2_B1}}
{{EXP2_B2}}
{{EXP2_B3}}
```

Do not use a blank document.\
That is the main reason formatted resumes break.

---

## Google Docs Template Design

To make the final resume look like the original formatted resume:

### Use a structured template

Recommended layout:

- **Header**

  - 3-column borderless table
  - Left: City / Location
  - Center: Full Name
  - Right: Phone, Email, GitHub, Portfolio, LinkedIn

- **Experience section**

  - 2-column borderless table per experience
  - Left: Company + Title
  - Right: Dates

- **Bullet points**

  - Pre-created bullet lines with placeholders inside them

### Why this matters

If you create a blank document and insert plain text, Google Docs will not automatically recreate:

- alignment
- bold headings
- bullet spacing
- section layout
- right-aligned dates

Using a template fixes that.

---

## n8n Workflow Setup

### Step 1: Trigger Node

Use:

- **When clicking “Execute workflow”**

This is useful for testing.

---

### Step 2: Job Search URL / Config Node

Create a **Set** node to define workflow inputs.

Example fields:

```json
{
  "search_url": "PASTE_PUBLIC_LINKEDIN_JOBS_URL",
  "template_doc_id": "YOUR_GOOGLE_DOC_TEMPLATE_ID",
  "base_resume_doc_id": "YOUR_BASE_RESUME_DOC_ID",
  "sheet_id": "YOUR_GOOGLE_SHEET_ID",
  "target_role": "Business Analyst / Data Analyst / Financial Analyst",
  "max_jobs": 20
}
```

---

### Step 3: Scrape Jobs with Apify

Use the **Apify** node:

- Resource: `Actor`
- Operation: `Run Actor and Get Dataset`

Pass the LinkedIn Jobs search URL into the actor input.

#### Important

Use a **public LinkedIn Jobs search URL**.\
Do not use a logged-in search page URL copied from an authenticated LinkedIn session.

---

### Step 4: Limit Number of Jobs

Use either:

- **Code node**
- **Limit node**

Example Code node:

```javascript
const maxJobs = $json.max_jobs ?? 10;
return items.slice(0, maxJobs);
```

This prevents wasting tokens and API calls.

---

### Step 5: Loop Over Jobs

Use **Loop Over Items** with:

- Batch Size = `1`

This processes one job at a time and makes debugging easier.

---

### Step 6: Get Base Resume

Use the **Google Docs** node to fetch your base resume text from Google Docs.

Purpose:

- anchor the tailoring to your actual resume
- avoid hallucinated experience
- keep tailoring truthful

---

### Step 7: AI Node - Check Relevance

Use an AI node to evaluate whether the job is worth tailoring for.

#### Input to AI

- job title
- company
- location
- job description
- base resume text
- target role

#### Recommended prompt

```text
You are screening whether a job is worth tailoring my resume for.

Inputs:
- Target roles: {{$node["Job Search URL"].json["target_role"]}}
- Resume: {{$node["Get Resume"].json["text"] || $json["text"]}}
- Job title: {{$json.title}}
- Company: {{$json.companyName}}
- Location: {{$json.location}}
- Description: {{$json.descriptionText}}

Rules:
- Prefer Business Analyst, Data Analyst, Financial Analyst, BI, Reporting, Operations Analyst, Process Improvement, Analytics, PMO-adjacent roles.
- Reject irrelevant engineering-heavy roles, senior-only roles, and roles requiring certifications I clearly do not have.
- Be strict.

Return JSON only:
{
  "verdict": true,
  "score": 0,
  "reason": "one sentence"
}
```

#### Critical rule

Do not return plain English paragraphs.\
Return strict JSON only.

---

### Step 8: Filter Relevant Jobs

Use the **Filter** node to keep only relevant jobs.

#### Example condition

Left side:

```javascript
{{ $json.verdict }}
```

Operator:

- `is equal to`

Right side:

- `true`

If your AI node returns text-wrapped JSON instead of proper JSON, parse it first before filtering.

---

### Step 9: AI Node - Tailor Resume

This node should generate **structured resume fields**, not one giant resume paragraph.

#### Recommended output format

```json
{
  "NAME": "______",
  "CITY_STATE": "______",
  "PHONE": "_______",
  "EMAIL": "_______",
  "GITHUB": "Github",
  "PORTFOLIO": "Portfolio",
  "LINKEDIN": "LinkedIn",

  "EXP1_COMPANY": "",
  "EXP1_DATES": "",
  "EXP1_TITLE": "",
  "EXP1_B1": "",
  "EXP1_B2": "",
  "EXP1_B3": "",

  "EXP2_COMPANY": "",
  "EXP2_DATES": "",
  "EXP2_TITLE": "",
  "EXP2_B1": "",
  "EXP2_B2": "",
  "EXP2_B3": ""
}
```

#### Recommended prompt

```text
You are tailoring my resume to a job.

Use ONLY the experience and projects already present in my resume.
Do not invent employers, achievements, dates, tools, or metrics.
Rewrite bullets to align with the job, but stay truthful.
Keep the style concise and ATS-friendly.

Return JSON only with the exact placeholder keys required by the resume template.
```

---

### Step 10: Build Google Docs Replace Requests

Use a **Code** node to convert the tailored JSON into Google Docs placeholder replacement requests.

#### Code

```javascript
const data = $json;

const requests = Object.entries(data).map(([key, value]) => ({
  replaceAllText: {
    containsText: {
      text: `{{${key}}}`,
      matchCase: true
    },
    replaceText: value ?? ""
  }
}));

return [{ json: { requests } }];
```

---

### Step 11: Copy Resume Template

Use an **HTTP Request** node to copy the master Google Docs template.

#### Node settings

- Method: `POST`
- Authentication: `Predefined Credential Type`
- Credential Type: `Google OAuth2 API`

#### URL

```javascript
{{ 'https://www.googleapis.com/drive/v3/files/' + $node["Job Search URL"].json["template_doc_id"] + '/copy' }}
```

#### Body

```json
{
  "name": "CV - {{$json.companyName}} - {{$json.title}}"
}
```

This creates a fresh Google Doc for each job.

---

### Step 12: Fill Template with Tailored Content

Use another **HTTP Request** node.

#### Node settings

- Method: `POST`
- Authentication: `Predefined Credential Type`
- Credential Type: `Google OAuth2 API`
- Send Body: `true`
- Body Content Type: `JSON`

#### URL

```javascript
{{ 'https://docs.googleapis.com/v1/documents/' + $node["Copy Resume Template"].json["id"] + ':batchUpdate' }}
```

#### Body

```javascript
{{ { requests: $json.requests } }}
```

This replaces placeholders inside the copied resume template.

---

### Step 13: Export Resume as PDF

Optional but useful.

Use another **HTTP Request** node.

#### Node settings

- Method: `GET`
- Authentication: `Predefined Credential Type`
- Credential Type: `Google OAuth2 API`
- Response Format: `File`
- Binary Property: `data`

#### URL

```javascript
{{ 'https://www.googleapis.com/drive/v3/files/' + $node["Copy Resume Template"].json["id"] + '/export?mimeType=application/pdf' }}
```

This exports the tailored Google Doc as PDF.

---

### Step 14: Append Job Record in Google Sheets

Use the **Google Sheets** node to log each processed job.

#### Suggested sheet columns

```text
run_time
job_id
job_title
company
location
posted_at
job_url
apply_url
relevance_score
relevance_reason
resume_doc_id
resume_doc_url
workflow_status
```

#### Suggested mappings

- `run_time` → `{{$now}}`
- `job_id` → `{{$json.id}}`
- `job_title` → `{{$json.title}}`
- `company` → `{{$json.companyName}}`
- `location` → `{{$json.location}}`
- `posted_at` → `{{$json.postedAt}}`
- `job_url` → `{{$json.link}}`
- `apply_url` → `{{$json.applyUrl}}`
- `workflow_status` → `tailored`

---

## Recommended Final Workflow

```text
When clicking "Execute workflow"
→ Job Search URL
→ Scrape Jobs
→ Loop Over Items
→ Get Resume
→ Checks Relevance
→ Filter
→ Customized Resume
→ Build Replace Requests
→ Copy Resume Template
→ Fill Template
→ Export PDF
→ Append row in sheet
```

---

## What to Remove from the Old Workflow

Do not use this old formatting logic:

```text
Customized Resume
→ Markdown
→ Create a document
→ Add Resume Text
```

Why it fails:

- it creates a blank doc
- it inserts raw AI text
- it destroys your original resume formatting

Replace it with:

```text
Customized Resume
→ Build Replace Requests
→ Copy Resume Template
→ Fill Template
```

That is the correct architecture.

---

## Testing Strategy

Do not test everything at once.

### Recommended order

1. Test the config node
2. Test Apify scraping
3. Limit to 3 jobs
4. Test relevance output
5. Test filter behavior
6. Test tailored resume JSON
7. Test template copy
8. Test 2 placeholder replacements first
9. Test full placeholder replacement
10. Test PDF export
11. Test Google Sheets append
12. Run full workflow

---

## Common Errors and Fixes

### 1. Scrape returns 0 jobs

**Cause**

- bad LinkedIn URL
- logged-in/private LinkedIn search URL
- wrong Apify actor input

**Fix**

- use a public LinkedIn Jobs search URL
- re-copy the search URL from an incognito or public jobs search page

---

### 2. Filter discards all jobs

**Cause**

- relevance node output format does not match filter condition

**Fix**

- force the AI node to return valid JSON
- check whether `verdict` is boolean `true` instead of string `"true"`

---

### 3. Customized Resume node does not run

**Cause**

- Filter did not pass any items

**Fix**

- inspect Filter output
- inspect relevance JSON
- simplify filter logic

---

### 4. Google Docs formatting looks broken

**Cause**

- blank document + text insertion
- no fixed template structure

**Fix**

- use a preformatted Google Docs template
- replace placeholders instead of inserting full raw text

---

### 5. HTTP Request credential error

**Cause**

- selected credential is blocked from HTTP Request usage

**Fix**

- use **Google OAuth2 API (generic)** credential
- ensure it is usable inside HTTP Request nodes

---

### 6. Docs API batchUpdate fails

**Cause**

- one or more `requests[]` items are malformed

**Fix**

- test with 2 placeholders first
- validate request payload
- expand gradually

---

## Security Notes

Do not commit:

- API tokens
- OAuth credentials
- Google client secrets
- personal resume documents
- exported resumes
- private job tracking data

Use a `.env.example` file for sample configuration only.

Example:

```env
APIFY_API_TOKEN=your_apify_token_here
GOOGLE_TEMPLATE_DOC_ID=your_template_doc_id_here
GOOGLE_BASE_RESUME_DOC_ID=your_base_resume_doc_id_here
GOOGLE_SHEET_ID=your_sheet_id_here
LLM_PROVIDER_API_KEY=your_llm_key_here
```

---

## Known Limitations

This system is strong for:

- job scraping
- AI relevance screening
- AI resume tailoring
- formatted resume generation
- job tracking

This system does **not** fully automate job applications across all ATS platforms.

That is a different problem involving:

- browser automation
- captcha handling
- site-specific selectors
- anti-bot defenses
- login/session management
- frequent UI breakage

This project focuses on the stable half of the workflow:\
**finding jobs, screening them, tailoring resumes, and generating application-ready documents.**

---

## Future Improvements

- cover letter generation
- company-specific keyword extraction
- score-based job ranking
- email alerts for top matches
- PDF upload to Drive folder
- browser automation for selected job portals
- duplicate job detection
- retry/error handling branch



## How to Use

1. Import the n8n workflow JSON
2. Configure Apify, Google, and AI credentials
3. Create the Google Docs resume template with placeholders
4. Update your config node with:
   - LinkedIn Jobs search URL
   - template doc ID
   - base resume doc ID
   - Google Sheet ID
5. Execute the workflow
6. Review generated docs and logs in Google Sheets

---

## Author

**Ashish Shinde**\
Business Analyst

---

##
