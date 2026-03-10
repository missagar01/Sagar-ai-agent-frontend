# 🤖 Intelligent Multi-Database Assistant

**Version 2.1 — Deep Schema Aware | Bilingual Support**

An advanced AI-powered system designed to interact with **multiple disparate databases** (PostgreSQL) using natural language. Unlike standard Text-to-SQL bots, this system uses a **Router-Validator Architecture** to understand context, handle ambiguity across different business domains, and self-correct SQL errors.

> 🆕 **v2.2 Updates**: Replaced `plant_visitor` with new `visitors` table for Gate Pass tracking, including inline HTML image rendering for visitor photos in the chat UI. Bilingual queries and negation intent handling intact.

---

## 🌟 Key Features

### 🧠 1. Intelligent "Deep Router"

* **Schema-Aware Routing**: The router analyzes the **actual table & column names** of every registered database, not just keywords.
* **Ambiguity Protocol**: If a user asks "Show me the status" and multiple databases have a "Status" column, the agent **pauses** and asks: *"Did you mean Lead Status or Leave Request Status?"*
* **Clarification Memory**: Once you clarify (e.g., "The leave one"), it merges this with your original question to execute the correct query.

### 🛡️ 2. Self-Correcting SQL Agents

* **Generate-Validate-Regenerate Loop**:
  1. **Generator (LLM 1)**: Writes the initial SQL query using semantic schema + business rules.
  2. **Validator (LLM 2)**: Checks the query against column restrictions, intent matching, and LOWER() enforcement.
  3. **Refiner**: If the Validator rejects it, the Generator automatically rewrites with specific feedback (max 3 attempts).
* **Ghost Record Filtering**: Automatically ignores incomplete or "test" data (rows with NULL names/dates).

### 🌐 3. Hindi / Hinglish Bilingual Support

* Understands mixed Hindi-English queries like:
  - `"aaj ke pending tasks dikhao"` → Show today's pending tasks
  - `"jinka leave approve nhi hua unka naam batao"` → Names of employees whose leave was not approved
  - `"sabka travel data do"` → Show everyone's travel data
* Built-in glossary prevents Hindi words ("datta", "kitne", "dikhao") from being misinterpreted as person names or filter values.

### 📊 4. Supported Integrations

The system integrates **3 distinct business databases** with a total of **13 tables**:

---

#### 📋 Database 1: **Checklist DB** — Employee & HR Operations

A comprehensive employee management system covering task tracking, leave management, travel, hiring, and visitor approvals.

| Table              | Purpose                          | Key Columns                                                        |
| :----------------- | :------------------------------- | :----------------------------------------------------------------- |
| `checklist`        | Daily/routine tasks              | name, department, task_description, frequency, task_start_date, submission_date, status |
| `delegation`       | One-time assigned tasks          | name, given_by, department, task_description, planned_date, submission_date |
| `users`            | Employee info & login            | user_name, department, role, email_id, number, status              |
| `ticket_book`      | Ticket bookings & travel bills   | person_name, type_of_bill, status, bill_number, total_amount, charges |
| `leave_request`    | Leave management & approvals     | employee_name, from_date, to_date, reason, request_status, approved_by, hr_approval |
| `visitors`         | Visitor gate pass tracking       | visitor_name, purpose_of_visit, person_to_meet, date_of_visit, time_of_entry, approval_status |
| `request`          | Travel requests                  | person_name, type_of_travel, from_city, to_city, departure_date, reason_for_travel |
| `resume_request`   | Hiring / candidate pipeline      | candidate_name, applied_for_designation, experience, interviewer_status, joined_status |

**Example queries:**
- `"aaj ke pending tasks dikhao"` — Today's pending tasks
- `"jinka leave approve nhi hua unka naam batao"` — Employees with unapproved leave
- `"total kitne candidates ne apply kiya"` — Total job applicants

---

#### ⚙️ Database 2: **Sagar DB** — Machine Maintenance & Facility Management

A specialized system for tracking machine repairs, maintenance schedules, and technician assignments.

| Table                      | Purpose                     | Key Columns                                              |
| :------------------------- | :-------------------------- | :------------------------------------------------------- |
| `maintenance_task_assign`  | Machine maintenance tasks   | Machine_Name, Doer_Name, Task_Start_Date, Actual_Date    |

**Business rules:**
- `Actual_Date IS NULL` → Task is **PENDING/OPEN**
- `Actual_Date IS NOT NULL` → Task is **COMPLETED**
- ⚠️ Column names use **Mixed-Case** (e.g., `"Machine_Name"`) — requires quoting in SQL.

**Example queries:**
- `"Show pending machine repairs"`
- `"Which technician completed the most tasks this month?"`
- `"What machines are overdue for maintenance?"`

---

#### 💼 Database 3: **Lead-To-Order** — Sales CRM & Pipeline Management

A full-cycle sales system tracking the journey from Lead → Enquiry → Quotation → Order.

| Table              | Purpose                       | Key Columns                                                    |
| :----------------- | :---------------------------- | :------------------------------------------------------------- |
| `fms_leads`        | Lead generation & tracking    | created_at, planned, actual, lead_source, status, enquiry_received_status, is_order_received |
| `enquiry_to_order` | Enquiry-to-order conversion   | timestamp, planned, actual, is_order_received                  |
| `make_quotation`   | Quotation management          | quotation_no, quotation_date, prepared_by, company_name, contact_name, grand_total, items |
| `login`            | CRM user accounts             | username, password, usertype                                   |

**Business rules:**
- `status` values: `'Hot'`, `'Warm'`, `'Cold'` (lead temperature)
- `lead_source` values: `'Indiamart'`, `'Direct Visit'`, `'Telephonic'`, `'Email'`
- `is_order_received = 'yes'` → Lead converted to order
- `grand_total` — Total quotation value (NUMERIC)

**Example queries:**
- `"How many hot leads came from Indiamart this month?"`
- `"Show total quotation value by prepared_by"`
- `"What is the lead conversion rate?"`

---

## 🛠️ System Architecture

```mermaid
flowchart TD
    classDef userNode fill:#1e1b4b,stroke:#6366f1,stroke-width:2px,color:#c7d2fe
    classDef gatewayNode fill:#0c1e2e,stroke:#06b6d4,stroke-width:2px,color:#a5f3fc
    classDef routerNode fill:#0c1e2e,stroke:#22d3ee,stroke-width:2.5px,color:#67e8f9
    classDef dbNode fill:#1c1410,stroke:#fb923c,stroke-width:2px,color:#fdba74
    classDef agentNode fill:#0f1f1a,stroke:#10b981,stroke-width:2px,color:#6ee7b7
    classDef errorNode fill:#1f1010,stroke:#ef4444,stroke-width:2px,color:#fca5a5
    classDef outputNode fill:#1e1b3a,stroke:#8b5cf6,stroke-width:2px,color:#c4b5fd
    classDef coreNode fill:#0d1a14,stroke:#34d399,stroke-width:2.5px,color:#a7f3d0

    U("👤 User"):::userNode
    AG("📡 API Gateway"):::gatewayNode
    RT{"🔀 Deep Schema Router<br/>Intent Classification"}:::routerNode
    AH["❓ Ambiguity Handler<br/>Clarify & Re-route"]:::errorNode

    CE["🔄 Context Engine<br/>Reformulate Query"]:::agentNode
    SG["🤖 SQL Generator<br/>LLM 1: Build Query"]:::agentNode
    SV{"🛡️ Validator<br/>LLM 2: Check & Approve"}:::coreNode
    EX["⚡ SQL Executor<br/>Run Query"]:::agentNode

    DB1[("📋 Checklist DB<br/>8 Tables")]:::dbNode
    DB2[("⚙️ Sagar DB<br/>Machine Data")]:::dbNode
    DB3[("💼 Sales DB<br/>Sales Data")]:::dbNode

    SY["📝 Answer Synthesizer<br/>LLM 3: Generate Response"]:::outputNode
    UR("👤 User<br/>Receives Result"):::userNode

    U --> AG
    AG --> RT

    RT -.->|"Ambiguous?"| AH
    AH -.->|"Clarify"| U

    RT -->|"Employee / HR Intent"| DB1
    RT -->|"Machine Intent"| DB2
    RT -->|"Sales Intent"| DB3

    subgraph CORE ["🤖 Autonomous SQL Agent Loop"]
        direction TB
        CE --> SG
        SG --> SV
        SV -->|"❌ Reject"| SG
        SV -->|"✅ Approve"| EX
    end

    DB1 --> CE
    DB2 --> CE
    DB3 --> CE

    EX --> SY
    SY --> UR
```

---

## 📂 Project Structure

```text
DB_Assistant/
├── Backend_New/                    # FastAPI Python Backend
│   ├── main.py                     # Application entry point
│   ├── .env                        # Environment variables
│   ├── app/
│   │   ├── core/
│   │   │   ├── router.py           # Deep Schema Router (The Brain)
│   │   │   ├── config.py           # Global settings (ALLOWED_TABLES, etc.)
│   │   │   ├── security.py         # SQL injection prevention
│   │   │   └── column_restrictions.py  # Per-table allowed columns
│   │   ├── databases/              # 🔌 Modular Database Agents
│   │   │   ├── checklist/          # Employee DB Module (8 tables)
│   │   │   │   ├── config.py       # Schema, metadata, column restrictions
│   │   │   │   ├── connection.py   # DB connection & table whitelist
│   │   │   │   ├── prompts.py      # Generator, Validator, Answer prompts
│   │   │   │   └── workflow.py     # LangGraph workflow (6 nodes)
│   │   │   ├── sagar_db/           # Maintenance DB Module
│   │   │   └── lead_to_order/      # Sales DB Module
│   │   ├── services/               # Shared Utilities
│   │   │   ├── sql_agent.py        # Legacy standalone agent
│   │   │   ├── agent_nodes.py      # Legacy fallback nodes
│   │   │   ├── db_service.py       # Database operations
│   │   │   ├── cache_service.py    # Semantic query cache (ChromaDB)
│   │   │   ├── context_manager.py  # Conversation context
│   │   │   └── session_manager.py  # Multi-user sessions
│   │   ├── api/
│   │   │   └── routes/
│   │   │       └── chat.py         # Chat streaming endpoint (SSE)
│   │   └── tools/
│   │       └── db_inspector.py     # Schema inspection utility
├── Frontend/                       # Chat UI (HTML/JS)
│   └── index.html
├── Database_Schemas/               # 📊 Auto-Generated Schema Reports
│   ├── checklist/
│   │   ├── metadata_analysis.json
│   │   └── schema_report.md
│   └── ...
├── BACKEND_NEW_WORKFLOW.md         # 📘 Detailed workflow documentation
├── DATABASE_INTEGRATION_GUIDE.md   # 📘 How to add new databases
└── README.md                       # This file
```

---

## 🚀 Getting Started

### Prerequisites

* Python 3.10+
* PostgreSQL Database(s)
* OpenAI API Key
* pip (Python package manager)

### Installation

1. **Clone & Setup**:

   ```bash
   git clone <repo_url>
   cd DB_Assistant
   ```

2. **Create Virtual Environment**:

   ```bash
   python -m venv .venv
   # Windows
   .venv\Scripts\activate
   # Linux/Mac
   source .venv/bin/activate
   ```

3. **Install Dependencies**:

   ```bash
   cd Backend_New
   pip install -r requirements.txt
   ```

   Key dependencies:
   - `fastapi` + `uvicorn` — API server
   - `langchain-community` + `langchain-openai` — LLM framework
   - `langgraph` — Agent state machine
   - `psycopg2-binary` — PostgreSQL driver
   - `chromadb` — Semantic cache
   - `python-dotenv` + `pydantic-settings` — Configuration

4. **Environment Variables**:
   Create a `.env` file in `Backend_New/`:

   ```properties
   # LLM
   OPENAI_API_KEY=sk-...
   LLM_MODEL=gpt-4o

   # Database Connections
   DATABASE_URL=postgresql://user:pass@host:5432/checklist_db
   DB_SAGAR_URL=postgresql://user:pass@host:5432/sagar_db
   DB_L2O_URL=postgresql://user:pass@host:5432/lead_to_order

   # Optional
   MAX_VALIDATION_ATTEMPTS=3
   CONFIDENCE_THRESHOLD=70
   CACHE_SIMILARITY_THRESHOLD=0.92
   ```

5. **Run the Backend**:

   ```bash
   cd Backend_New
   uvicorn main:app --reload
   ```

   The API will start at `http://127.0.0.1:8000`.

6. **Open the Frontend**:

   Open `Frontend/index.html` in your browser or serve it with any static file server.

---

## 🔧 Developer Guide

### How to Add a New Database

1. **Generate Schema**: Use `app/tools/db_inspector.py` to inspect your new database.
2. **Create Module**: Create a folder under `app/databases/your_new_db/` with:
   - `config.py` — `ROUTER_METADATA`, `ALLOWED_COLUMNS`, `SEMANTIC_SCHEMA`
   - `connection.py` — DB connection + `RestrictedSQLDatabase`
   - `prompts.py` — Generator, Validator, and Answer Synthesis prompts
   - `workflow.py` — LangGraph agent workflow
3. **Register**: Import your metadata in `app/core/router.py`.
4. **Update Global Config**: Add tables to `app/core/config.py` → `ALLOWED_TABLES` if needed.

See `DATABASE_INTEGRATION_GUIDE.md` for a full step-by-step walkthrough.

### Adding New Tables to an Existing Database

1. Add to `ALLOWED_TABLES` in `app/core/config.py`
2. Add to `ALLOWED_COLUMNS` in both `app/databases/<db>/config.py` and `app/core/column_restrictions.py`
3. Add to `SEMANTIC_SCHEMA` in `app/databases/<db>/config.py`
4. Update `target_tables` in `app/databases/<db>/workflow.py`
5. Update `connection.py` to include in `RestrictedSQLDatabase`
6. Update prompts in `app/databases/<db>/prompts.py`
7. Update legacy code in `app/services/sql_agent.py` and `app/services/agent_nodes.py`

### Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| "Ambiguous Query" loop | Router metadata descriptions too similar | Make `ROUTER_METADATA` descriptions more distinct |
| "Column does not exist" | PostgreSQL mixed-case columns | Add quotes in config (e.g., `"TaskID"`) |
| "No result returned" | Empty query results or graph error | Check debug logs; empty results now handled gracefully |
| Hindi words used as filter | Glossary incomplete | Add new words to HINDI GLOSSARY in prompts.py |
| Excessive validation loops | Validator too strict | Review validator prompt strictness settings |

---

## 🆕 Recent Changes (v2.1)

### New Tables Integrated
- `ticket_book` — Ticket bookings & travel bills
- `leave_request` — Employee leave management with multi-level approval
- `visitors` — Visitor gate pass tracking with inline UI image rendering!
- `request` — Employee travel requests
- `resume_request` — Candidate resume intake & hiring pipeline

### Improvements Applied
1. **Empty Result Handling** — Empty SQL results now generate AI-powered friendly messages instead of raw errors.
2. **Hindi/Hinglish Glossary** — Bilingual word recognition in prompts prevents misinterpretation of Hindi words as database values.
3. **Negation-Aware Intent Rules** — Explicit rules for "not approved", "not completed", "not joined" mapped to correct SQL logic.
4. **Column-Level Security** — Each table has explicitly defined allowed and forbidden columns.
5. **LOWER() Enforcement** — All string comparisons use case-insensitive matching across all 8 tables.

---

### 📞 Support

For bugs or feature requests, check the debug logs in the terminal output or contact the development team.
