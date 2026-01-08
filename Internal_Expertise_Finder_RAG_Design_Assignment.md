# Internal Expertise Finder (RAG Chatbot) — Design Assignment (MacBook Local)

> **Goal:** Build a local, **Internal Expertise Finder** chatbot that recommends the **right people** for a topic (e.g., “Redis TLS”, “RAG”, “ECS Fargate”, “dbt”) using **RAG (Retrieval‑Augmented Generation)** over a provided JSON dataset.  
> **Key requirement:** Responses must be **grounded** in the dataset with **evidence snippets** (no hallucinations).

---

## Table of Contents
1. [Objective](#1-objective)
2. [What the Company Will Provide](#2-what-the-company-will-provide)
3. [Allowed Local Stack (MacBook‑Friendly)](#3-allowed-local-stack-macbookfriendly)
4. [Models (Local)](#4-models-local)
5. [Dataset Format (Expected JSON Schema)](#5-dataset-format-expected-json-schema)
6. [Expected Chatbot Behavior (Functional Requirements)](#6-expected-chatbot-behavior-functional-requirements)
7. [Output Requirements (Suggested Format)](#7-output-requirements-suggested-format)
8. [Scoring Rubric — 50 Points Total](#8-scoring-rubric--50-points-total)
9. [Bonus Tasks — +10 Points (Max)](#9-bonus-tasks--10-points-max)
10. [Review Test Prompts (Examples)](#10-review-test-prompts-examples)
11. [Submission Requirements](#11-submission-requirements)

---

## 1) Objective
Build an **Internal Expertise Finder** chatbot that helps employees find the **right person** for a topic (e.g., “Redis TLS”, “RAG”, “ECS Fargate”, “dbt”).  
The chatbot must use **RAG (Retrieval‑Augmented Generation)** over a **provided JSON dataset of employee profiles** and produce **grounded answers with evidence**.

### Success Criteria
- Returns **top 1–3 best matches**
- Explains **why** they match (skills/projects/bio)
- Includes **evidence** (short snippets from the dataset)
- **Does not hallucinate** (no made‑up skills/people/projects)
- Asks **clarifying questions** when the request is vague
- Supports **constraints/filters** (e.g., team, location, experience)

---

## 2) What the Company Will Provide
- `data/profiles.json` containing ~20–25 employee profiles (synthetic/anonymized)

### Assumptions About Real‑World Dataset Quality
The dataset may include:
- inconsistent skill naming (e.g., “PostgreSQL” vs “Postgres”)
- missing fields for some profiles
- short bios for some people, long bios for others

**Your solution must handle these cases gracefully** (e.g., safe defaults, robust parsing, “no strong match” when needed).

---

## 3) Allowed Local Stack (MacBook‑Friendly)
You may choose any stack that runs locally. Below is a recommended option.

### Local Model Runtime
- **Ollama** (local LLM + local embeddings)

### Python Packages (Recommended)
- **LlamaIndex** (RAG orchestration)
- **ChromaDB** (local persistent vector database)
- **Streamlit** (simple chatbot UI)

### Install Packages (Reference)
```bash
pip install -U streamlit chromadb llama-index
pip install -U llama-index-llms-ollama llama-index-embeddings-ollama
pip install -U llama-index-vector-stores-chroma
```

---

## 4) Models (Local)
You must document the model(s) you used.

### Recommended Models (via Ollama)
**LLM options (pick one):**
- `llama3.2:3b`
- `qwen2.5:3b`

**Embedding model:**
- `nomic-embed-text`

### Example Pull Commands (Reference)
```bash
ollama pull llama3.2:3b
ollama pull qwen2.5:3b
ollama pull nomic-embed-text
```

> **Note:** You only need to pull **one** LLM (choose either `llama3.2:3b` or `qwen2.5:3b`).

---

## 5) Dataset Format (Expected JSON Schema)
Your system must read `data/profiles.json` which contains an **array of profiles**.

### Minimum Fields Expected
- `id` (string)
- `name` (string)
- `title` (string)
- `team` (string)
- `location` (string)
- `email` (string)
- `skills` (array of strings)
- `projects` (array of objects with at least `name`, `desc`)
- `bio` (string)

### Example Profile
```json
{
  "id": "u007",
  "name": "Aditi Shah",
  "title": "ML Engineer",
  "team": "ML",
  "location": "Bangalore",
  "email": "aditi.shah@company.com",
  "skills": ["RAG", "Embeddings", "Chroma", "Python"],
  "projects": [
    {
      "name": "Expertise Finder",
      "desc": "Built internal expertise search using embeddings + vector database; grounded answers with citations."
    }
  ],
  "bio": "Builds LLM-powered internal tools and semantic search systems."
}
```

### Robustness Expectations
Your ingestion should not crash if:
- a profile is missing `projects`
- `skills` is empty or missing
- `bio` is missing or null
- unexpected extra fields exist

Recommended behavior:
- Treat missing arrays as `[]`
- Treat missing strings as `""`
- Still index what exists (e.g., title/team/location)

---

## 6) Expected Chatbot Behavior (Functional Requirements)

### 6.1 Grounded Matching (No Hallucination)
**Meaning:** The chatbot must only recommend people when there is support in the dataset.

**Example:**  
If asked: “Who knows Snowflake?” but no profile mentions Snowflake, the chatbot should not claim someone knows it.

**Good response style:**
- “I couldn’t find a strong Snowflake match in the retrieved profiles.”
- “Closest matches mention BigQuery/Spark.”
- “Do you want a data warehouse expert in general?”

### 6.2 Evidence for Every Recommendation
**Meaning:** Each recommended person must include at least **1–2 evidence snippets** taken from their profile (skills/projects/bio).

**Example evidence:**
- Evidence: “Built internal expertise search using embeddings + vector database…”
- Evidence: Skills include “RAG, Embeddings, Chroma”

### 6.3 Clarifying Questions for Vague Queries
**Meaning:** If the query is broad, ask **1–2 clarifying questions** instead of returning random matches.

**Example ambiguous query:**  
“Need someone good at analytics.”

**Good clarifying questions:**
- “Analytics for what: marketing, product, finance, or data engineering?”
- “Any preference for team/location?”

### 6.4 Constraint / Filter Support
**Meaning:** If the user mentions constraints, the chatbot should respect them (via UI filters or query parsing).

Constraints may include:
- `team` (e.g., “Growth”)
- `location` (e.g., “Bangalore”)
- `experience_years` (e.g., “5+ years”, if present in dataset)

**Example query:**  
“Who can help with RAG in Bangalore?”

**Expected behavior:**
- Prefer (or restrict to) profiles where `location == "Bangalore"`
- If filters remove all strong matches, say so and offer alternatives:
  - “No Bangalore matches; want remote/any location?”

---

## 7) Output Requirements (Suggested Format)
You may choose your own format, but the output must be decision‑friendly.

### Suggested Answer Format
Return **Top 1–3 matches**.

For each match:
- **Name, Title, Team, Location, Email**
- **Why they match** (bullets)
- **Evidence** (short quotes/snippets)

### Example Answer (Illustrative)
**Aditi Shah** — ML Engineer (ML, Bangalore) — aditi.shah@company.com  
- **Why:** Built internal RAG expertise search; has embeddings + vector DB experience  
- **Evidence:** “Built internal expertise search using embeddings + vector database…”  
- **Evidence:** Skills: “RAG, Embeddings, Chroma”

**If not enough evidence:**
- Ask **1–2 clarifying questions**, or
- Say “no strong match found” and suggest closest related areas.

---

## 8) Scoring Rubric — 50 Points Total

### A) Core RAG Functionality — 18 Points
**A1) JSON Ingestion & Robustness — 5 pts**  
Meaning: Correctly reads the JSON dataset and handles missing fields without crashing.  
Example: If a profile has no projects, it is still searchable via skills + bio.

**A2) Semantic Retrieval Over Profiles — 7 pts**  
Meaning: For a user query, retrieves relevant profiles using embeddings/vector search (semantic retrieval).  
Example: “Who knows Redis TLS on ElastiCache?” → profiles mentioning “Redis TLS” / “ElastiCache” / “TLS” rank near top.

**A3) RAG Response Generation — 6 pts**  
Meaning: Uses retrieved profile content as the source of truth; does not answer from general knowledge alone.  
Example: If no one matches “SAP ABAP”, the system says no strong match and may offer closest alternatives.

### B) Grounding, Evidence, and Safety — 14 Points
**B1) Evidence/Citations for Each Recommendation — 6 pts**  
Meaning: Each recommendation includes evidence snippets from dataset (skills/projects/bio).  

**B2) No Hallucinations — 4 pts**  
Meaning: Never invent skills, projects, or people not present in dataset.  

**B3) Clear Output Formatting — 4 pts**  
Meaning: Short, actionable, easy to scan (top 1–3 only).

### C) Query Intelligence & User Experience — 10 Points
**C1) Clarifying Questions When Needed — 4 pts**  
Meaning: Detect vague queries and ask 1–2 clarifying questions.  

**C2) Filters/Constraints Support — 6 pts**  
Meaning: Supports filtering by at least **team** and **location** (UI controls or query parsing).  
Example: “RAG expert in Bangalore” should prioritize `location == Bangalore` and explain if results are limited.

### D) Documentation & Demo Readiness — 8 Points
**D1) README with Setup, Run Instructions, and Demo Queries — 4 pts**  
Expected in README:
- prerequisites (Mac, Python, Ollama or equivalent)
- install commands
- model names used
- how to start the app
- 5–8 sample queries

**D2) Limitations & Edge Cases Documented — 4 pts**  
Explain behavior for:
- no matches
- multiple close matches
- missing fields
- unusual acronyms
- low confidence retrieval

Example:  
“If similarity is low or no strong match is found, the system asks a clarification question instead of forcing a weak recommendation.”

---

## 9) Bonus Tasks — +10 Points (Max)
You may implement any combination up to 10 bonus points. Clearly list implemented bonus items in your README.

### Bonus 1) Hybrid Retrieval (Semantic + Keyword) — 4 pts
Meaning: Combine semantic search with keyword matching so exact terms like “Redis TLS” work well.  
Example: A profile containing the exact phrase “Redis TLS” should rank high even if embeddings are noisy.

### Bonus 2) Reranking — 4 pts
Meaning: After retrieving top‑K candidates, rerank them using a reranker (LLM‑based or model‑based).  
Example: Among multiple “Redis” profiles, prioritize the one mentioning “TLS” and “ElastiCache”.

### Bonus 3) Confidence Label (High/Med/Low) — 2 pts
Meaning: Provide a confidence indicator per recommendation based on similarity score and evidence strength.  
Examples:
- “Confidence: High (explicit skill + project match)”
- “Confidence: Low (only indirect match on related skills)”

### Bonus 4) Privacy Controls / Redaction — 3 pts
Meaning: Support dataset fields like `visibility` / `contact_visibility` and hide email when restricted.  
Example: If `contact_visibility = "restricted"`, show:  
“Contact details are restricted; please reach out via team lead.”


---

## 10) Review Test Prompts (Examples)
During evaluation, reviewers may ask:

- “Who knows Redis TLS on ElastiCache?”
- “Need someone for RAG using Chroma + LlamaIndex”
- “Find a Terraform + Kubernetes expert in Bangalore”
- “Need help in analytics” *(expects clarifying questions)*
- “Who knows SAP ABAP?” *(expects no‑match behavior or honest closest‑match explanation)*

---

## 11) Submission Requirements
- Working local application (MacBook runnable)
- Uses the provided dataset (`profiles.json`)
- Produces grounded recommendations with evidence
- README includes:
  - packages used
  - model names and how to install/pull them
  - how to run the application
  - demo queries
  - bonus items (if any) clearly listed and labeled as **“Bonus”**
