# AI-Powered Job Application Tracker

An automated job tracking system built with n8n that scrapes live job listings, uses Claude AI to score each listing against a candidate profile, and organises results into an Airtable dashboard — running daily on a schedule with zero manual effort.

## What It Does

Most job seekers manually browse multiple platforms, copy listings into spreadsheets, and struggle to keep track of what's worth applying to. This workflow automates the entire discovery and triage process:

1. Scrapes 100 job listings daily across 5 search terms from MyCareersFuture (Singapore)
2. Removes duplicates across search results
3. Scores each job 1–10 using Claude AI based on skills match, role fit, and red flags
4. Routes high-scoring jobs (7+) to a Hot Leads table and lower scores to Archive
5. Stores structured data including AI summary, missing skills, and red flags in Airtable
6. Sends a formatted HTML email digest of today's Hot Leads automatically after each run

## System Architecture

```
Schedule Trigger (daily)
        │
        ├── HTTP Request (automation analyst)
        ├── HTTP Request (AI engineer)
        ├── HTTP Request (AI automation)
        ├── HTTP Request (process automation)
        └── HTTP Request (data operations)
                │
                ▼
          Split Out (results array → individual items)
                │
                ▼
          Merge (combine all 5 searches)
                │
                ▼
          Remove Duplicates (by uuid)
                │
                ▼
          Loop Over Items (batch size: 1)
                │
                ▼
          Build Prompt (Code node)
          - Strips HTML from job description
          - Constructs structured prompt with candidate profile
          - Prepares clean API request body
                │
                ▼
          AI Scorer (Claude API via HTTP Request)
          - Scores job 1-10 against candidate profile
          - Identifies missing skills
          - Flags red flags (presentations, wrong domain, etc.)
          - Generates 2-sentence fit summary
                │
                ▼
          Parse AI Response (Code node)
          - Strips markdown fences from Claude response
          - Parses JSON output
          - Merges AI results with original job data
                │
                ▼
          Hot Lead Filter (If node: score >= 7)
          ├── True  → Airtable Hot Leads (Create or Update)
          └── False → Airtable Archive (Create or Update)
                │
                ▼
          Loop Over Items (done output — fires once after all jobs processed)
                │
                ▼
          Fetch Hot Leads (Airtable — today's records only)
                │
                ▼
          Build Email (Code node)
          - Generates formatted HTML email table
          - Handles empty results gracefully
                │
                ▼
          Gmail (sends daily digest to candidate)
```

## Airtable Structure

### Hot Leads Table
Stores jobs scoring 7 or above — roles actively worth considering.

| Field | Type | Description |
|---|---|---|
| job_title | Text | Role title |
| company | Text | Hiring company |
| platform | Text | Source platform |
| url | URL | Direct link to listing |
| salary_range | Text | Salary from API |
| fit_score | Number | Claude's 1–10 score |
| missing_skills | Long text | Skills gaps identified |
| red_flags | Long text | Concerns flagged by AI |
| date_found | Date | Date scraped |
| status | Single select | New / Applied / Interviewing / Rejected |
| ai_summary | Long text | 2-sentence fit analysis |

### Archive Table
Same structure (minus status) for jobs scoring below 7. Kept for reference.

## AI Scoring Logic

The Claude API receives a structured prompt containing:
- Candidate profile (skills, preferences, domain, experience level)
- Job listing details (title, company, salary, description, required skills)
- Scoring instructions with explicit red flag criteria

**Red flags that lower scores:**
- Heavy stakeholder presentation requirements
- Sales or business development focus
- Hard CS degree requirements
- Test automation (Selenium/Java) rather than workflow automation
- Mismatched domain (e.g. pure finance with no tech component)

**Green flags that raise scores:**
- Python, SQL, automation mentioned
- AI/LLM integration work
- Backend, independent, or pipeline-focused work
- ESG, sustainability, or data quality domain
- n8n, workflow tools, or process improvement

## Tech Stack

| Tool | Purpose |
|---|---|
| n8n | Workflow orchestration and automation |
| MyCareersFuture API | Live Singapore job listings |
| Anthropic Claude API | AI job scoring and analysis |
| Airtable | Structured storage and tracking dashboard |
| Gmail | Daily HTML digest email |
| JavaScript (Code nodes) | Data transformation and prompt engineering |

## Key Design Decisions

**Why n8n instead of Python?**
n8n provides visual workflow orchestration that is easier to monitor, debug, and extend without touching code. Each node has a clear single responsibility, making the pipeline transparent and maintainable.

**Why Claude API for scoring instead of keyword matching?**
Keyword matching can't reason about context. A job titled "Automation Analyst" might require Java test automation (wrong fit) or Python workflow automation (right fit). Claude reads the full description and makes a contextual judgement — the same way a human recruiter would.

**Why Upsert instead of Create?**
The workflow runs daily. Using Create or Update (upsert) on the URL field means re-running the workflow updates existing records rather than creating duplicates. The tracker stays clean automatically.

**Why separate Hot Leads and Archive tables?**
Separating by score threshold keeps the Hot Leads view focused and actionable. Archive preserves everything for reference without cluttering the primary view.

**Why send the email digest after the loop's "done" output?**
The loop's "done" output fires exactly once after all items are processed. Connecting the email to "done" guarantees one email per run regardless of how many jobs were scored — as opposed to connecting inside the loop which would trigger an email per job.

## Setup

### Prerequisites
- n8n instance (cloud or self-hosted)
- Anthropic API key (from console.anthropic.com)
- Airtable account with the base structure above
- Airtable Personal Access Token

### Steps

1. Import the workflow JSON into n8n
2. Set up credentials:
   - Airtable Personal Access Token
   - Anthropic API key (as Header Auth: `x-api-key`)
3. Update the Airtable Base ID and Table names in both Airtable nodes
4. Activate the Schedule Trigger for daily runs
5. Run manually first to verify the pipeline end to end

### Customising the Candidate Profile

In the **Build Prompt** code node, update the candidate profile section to match your own skills and preferences:

```javascript
content: `...CANDIDATE: Your skills here, your preferences here, minimum salary SGD XXXX...`
```

The AI scoring automatically adapts to whatever profile you define.

## Sample Output

**Hot Lead example (score: 8/10)**
```
job_title:      AI Automation Developer
company:        FinTech Startup Pte Ltd
salary_range:   SGD 5000-7000
fit_score:      8
missing_skills: AWS cloud experience
red_flags:      None
ai_summary:     Strong alignment with Python automation and LLM integration
                focus. Independent backend work with minimal stakeholder
                interaction matches candidate preferences well.
```

**Archive example (score: 4/10)**
```
job_title:      Automation Test Analyst
company:        GIENTECH SINGAPORE PTE LTD
salary_range:   SGD 4000-7000
fit_score:      4
missing_skills: Selenium, TestNG, Java, CI/CD pipelines
red_flags:      Role focuses on test automation (Selenium/Java) rather than
                workflow automation. Requires software testing background
                not aligned with candidate's data/operations experience.
ai_summary:     Technical automation skills present but role requires test
                engineering expertise in a different automation domain.
                Significant skill gaps in Java and testing frameworks.
```

## Limitations and Future Improvements

- Currently scrapes MyCareersFuture only — LinkedIn and JobStreet integration planned
- AI scoring can produce false positives/negatives on ambiguous job descriptions
- No deduplication across separate daily runs for jobs that stay listed for weeks (mitigated by upsert)
- Future: Slack notification when a score 9+ job is found
- Future: Auto-generate tailored cover letter draft for Hot Leads

## Author

Yuen Wei Ling — [GitHub](https://github.com/eyyuen) | [LinkedIn](https://www.linkedin.com/in/wei-ling-y-73b88122a/) | Singapore
