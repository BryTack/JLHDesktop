# JLH Desktop — Working Specification

**Version:** 0.0.8
**Date:** 2026-02-21
**Status:** Draft

---

## 1. Project Overview

**JLH Desktop** is a standalone Windows desktop application (.exe) that acts as a personal task library for a non-technical user (Joanna). It allows her to define reusable tasks once, in plain English, through a guided AI conversation. Defined tasks persist permanently and can be returned to days, weeks, or months later — fully self-explanatory and ready to use.

### Core Philosophy

> *Define a task once. Come back months later. Understand it immediately. Use it. Get results.*

- The AI does the thinking — the AI handles housekeeping, suggestions, categorisation, and layout decisions
- Joanna makes decisions, not configurations
- Every task is self-documenting — no external knowledge required to use it
- Non-technical user experience throughout — complexity is hidden or flagged for specialist help

### AI Provider Philosophy

The application is **AI agnostic**. It is not coupled to any specific AI provider or model. The active provider and model are configured in one place and can be swapped without code changes. Claude (Anthropic) is the initial default, but this is a configuration choice, not an architectural dependency.

---

## 2. Research Findings

Prior to technical design, research was conducted into existing GitHub projects, commercial products, open source libraries, and known production patterns for Electron + AI applications. Key findings are summarised here.

### 2.1 Originality Assessment

The concept is **genuinely novel** as a combined product. The distinguishing three-stage loop does not exist as a cohesive, non-technical, locally-running desktop application anywhere in the market:

1. **Define** — a guided AI conversation extracts what a task does and what inputs it needs, then persists this as a typed schema with a human-readable summary
2. **Parameterise** — the saved schema is rendered as a clean, purpose-built form — no chat required, no re-explaining
3. **Execute** — the task executes using the filled-in values and the appropriate engine (AI or local)

### 2.2 Closest Existing Products

| Product | What It Shares | What It Lacks |
|---|---|---|
| Raycast AI PromptLab | Saved AI commands with variable placeholders | macOS only. No guided definition conversation. No tool execution. Prompts are static templates, not typed schemas. |
| Zapier Copilot | Natural language workflow creation, saves reusable flows | Cloud/SaaS only. Requires app integrations. Not a personal local library. Cannot run free-form tasks. |
| AnythingLLM | Desktop app, local storage, agent builder | Agent builder is node-graph-based, not conversational. No form generation from schema. Skews technical. |
| PyGPT | Desktop, presets, plugin system | Presets are prompt templates, not parameterised schemas. No form generation. Task definition and execution are not separated. |
| Lindy AI | Conversational agent definition, recurring tasks | Cloud-only. No local storage. No form generation. Runs on schedules, not on-demand form execution. |
| Flowise / LangFlow | Visual workflow builder, local, open source | Requires technical drag-and-drop node editing. Not accessible to a non-technical user. |

### 2.3 Confirmed Gaps This Project Fills

- **Conversationally-defined, schema-first task persistence** — no existing tool cleanly separates "define a task with AI" from "run a task with AI"
- **AI-generated form UI as the daily interaction surface** — all AI products lead with chat; the form is the novel UX pattern here
- **Local-first, Windows desktop, non-technical user** — no Windows desktop product targets this combination
- **Personal library metaphor** — tasks are owned, named, reusable assets that accumulate over time

### 2.4 Relevant Open Source Projects Reviewed

| Project | URL | Relevance |
|---|---|---|
| AnythingLLM | github.com/Mintplex-Labs/anything-llm | Closest desktop AI competitor. Node.js + React. No form generation. |
| PyGPT | github.com/szczyglis-dev/py-gpt | Feature-rich desktop AI app. Python/Qt. Useful reference for plugin architecture. |
| electron-vite-react | github.com/electron-vite/electron-vite-react | Selected as the build scaffold. Official electron-vite React + TypeScript template. |
| react-jsonschema-form | github.com/rjsf-team/react-jsonschema-form | Selected for form rendering layer. |
| electron-react-boilerplate-sqlite3 | github.com/wds4/electron-react-boilerplate-sqlite3 | Reference for Electron IPC + SQLite bridge pattern. |

### 2.5 Key Technical Learnings from Research

**IPC boundary security**
All SQLite and AI API calls must run in Electron's main process. The renderer communicates through a typed, named `contextBridge` (preload script). Exposing raw `ipcRenderer` to the renderer is a common security mistake in amateur Electron projects and must be avoided.

**ASAR archive exclusion**
Electron packages app code into a read-only ASAR archive. The SQLite database must live in `app.getPath('userData')` — outside the archive — so it can be written to at runtime. First launch runs migration SQL to create the empty schema.

**JSON schema generation reliability**
Prompt-only JSON generation from the AI fails to produce valid output approximately 5% of the time. Using AI structured outputs with a Zod schema reduces this to effectively zero. This is the single most important reliability decision in the application.

**Drizzle-kit in packaged apps**
The `drizzle-kit` migration CLI must not run inside the packaged application. The correct approach: generate migrations during development, bundle the resulting SQL files into the app, and execute them programmatically at startup using `better-sqlite3`'s `exec()` method.

**Streaming on task execution**
When "Go" is pressed, tasks may take 10–60 seconds. Streaming intermediate progress back to the UI is expected UX. A static loading spinner with no feedback is perceived as broken.

**Execution reliability**
Production AI agent hallucination rates are reported at 17–33% even in commercial tools. Mitigation: keep individual tasks narrow in scope, show intermediate results where possible, and store execution logs per run for debugging.

---

## 3. Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| **Build tooling** | electron-vite + TypeScript | Official 2025 Electron build tool. Vite-based — fast hot reload, smaller bundles. Scaffolds Electron + React + TypeScript cleanly. |
| **Desktop shell** | Electron | Bundles its own Chromium — no WebView2 dependency. Produces a self-contained Windows `.exe`. |
| **UI framework** | React | Component-based, works naturally with dynamically rendered JSON schemas. |
| **UI components** | Shadcn/UI + TailwindCSS | Ships source code into the project. Fully controllable styling. Consistent, clean, non-technical appearance. |
| **Form rendering** | react-jsonschema-form (RJSF) | Standard library for rendering validated forms from JSON Schema in React. The AI generates the schema; RJSF renders the form. |
| **Database driver** | better-sqlite3 | Fastest synchronous SQLite driver for Node.js. Runs in Electron main process. |
| **ORM / migrations** | Drizzle ORM | Lightweight TypeScript-first ORM. Migration SQL files bundled into the app and run at startup. |
| **LLM orchestration** | Vercel AI SDK | TypeScript-first, AI-provider-agnostic. `generateObject` with Zod schemas for reliable JSON output. `streamText` with tool definitions for task execution with real-time progress. |
| **Structured outputs** | Provider structured outputs + Zod | Eliminates the ~5% malformed-output failure rate of prompt-only JSON generation. |
| **AI provider** | Configurable — Claude (Anthropic) by default | Provider and model configured via admin task. Can be changed without code changes. |
| **Web search tool** | Tavily (`@tavily/core`) | Purpose-built search API for AI agents. Structured, AI-ready results. 1,000 free calls/month. |
| **API key storage** | Electron `safeStorage` | OS-level encryption (Windows DPAPI). Keys never in source code or config files. |
| **Local script execution** | Node.js (JS, compiled from TypeScript) | Local tool scripts authored in TypeScript, compiled to JS, executed at runtime. Consistent with the codebase. No additional runtime required. |
| **Knowledge base** | Obsidian (local markdown files) | Free, local-first. JLH writes `.md` files directly to the vault folder — no API needed. |
| **Database location** | `app.getPath('userData')` | Persistent storage outside the read-only ASAR archive. Persists across app updates. |

---

## 4. Application Structure

### 4.1 Navigation

The app uses a **persistent left sidebar** for navigation. The main area renders the selected view. The sidebar shows `user` category tasks in user mode; admin mode additionally reveals `admin` and `agent` category tasks.

```
┌─────────────────────┬──────────────────────────────────┐
│  SIDEBAR            │  MAIN AREA                       │
│                     │                                  │
│  + New Task         │  (Selected task or Home view)    │
│                     │                                  │
│  ▼ Travel           │                                  │
│      Find a Hotel   │                                  │
│      Book a Train   │                                  │
│                     │                                  │
│  ▼ Work             │                                  │
│      Weekly Expenses│                                  │
│      Meeting Notes  │                                  │
│                     │                                  │
│  ▼ Home             │                                  │
│      Utility Prices │                                  │
└─────────────────────┴──────────────────────────────────┘
```

### 4.2 Application Modes

The app operates in one of two modes at any time:

| Mode | Who | Access |
|---|---|---|
| **User mode** | Anyone — default on launch | `user` category tasks. User Wizard Design/Review Scripts apply. |
| **Admin mode** | Anyone with the shortcut — `Ctrl+Shift+A` | All task categories. Admin Wizard Design/Review Scripts apply. Can overrule the Wizard Review Script. |

Joanna is aware admin mode exists. The app is designed so she could grow into using it herself over time.

### 4.3 Views

| View | Description |
|---|---|
| **Home** | Shown on launch. Displays task library as cards. Entry point for new task creation. |
| **Task View** | Full view of a selected task — summary, fields, Go button, results area |
| **Admin UI** | Minimal — mode indicator and API key management only. Everything else is admin tasks. |

---

## 5. Home View

Displayed on application launch. Contains:

- A grid/list of **task cards**, each showing:
  - Task name
  - One-line summary
  - Display category
  - Last used date
  - Draft badge (if `status = 'draft'`)
- A **"+ New Task"** button that begins the task definition conversation

Cards are grouped by display category matching the sidebar. Clicking any card opens the Task View for that task.

---

## 6. Task View

### 6.1 Always Visible

Every task view permanently displays:

- **Task name** (top of view)
- **Draft banner** — shown prominently if `status = 'draft'`, explaining the task is not yet ready to run and why
- **Plain English summary** — what this task does, written for the user
- **Field descriptions** — each input field has a plain English label and explanation
- **Any restrictions or notes**
- **Last used date**

### 6.2 Input Form

Dynamically rendered from the task's stored JSON schema. Field types include:

- Text input
- Number input
- Dropdown / select
- Date picker
- File picker (for tasks that require a local file as input)
- *(Extensible — new field types added as needed)*

### 6.3 Buttons

| Button | Action | Availability |
|---|---|---|
| **Go** | Validates inputs, runs the task, displays results | Disabled if `status = 'draft'` |
| **New** | Clears last session inputs and results | Only shown if `retain_last_session = true` |
| **⚙ (Cog icon)** | Opens the task refinement dialog | Always visible |

### 6.4 Results Area

Displayed below the form after "Go" is pressed. Format is defined per task during setup. A follow-up text input remains active after results are shown — the user can type responses or further questions in context.

### 6.5 Session Persistence

Each task has a `retain_last_session` flag. When true:
- Last inputs and results are shown when the task is reopened
- "New" button is visible to clear and start fresh

When false:
- Task always opens clean

---

## 7. Task Definition — The Wizard

### 7.1 Starting a New Task

Triggered by "+ New Task". A conversation view opens. The AI asks:

> *"How can I help?"*

The user describes their task in plain English. The mode at the time of creation determines the task's `access_category`:

| Mode | Task built as |
|---|---|
| User mode | `access_category = 'user'` |
| Admin mode | `access_category = 'admin'` |

### 7.2 The Wizard Design Script

During the conversation, the Wizard regularly consults the **Wizard Design Script** — a document that governs how the Wizard approaches task design. The correct script is loaded based on the current mode:

- **User mode** → User Wizard Design Script
- **Admin mode** → Admin Wizard Design Script

The script is advisory — its findings surface as warnings or steers in the conversation, never as hard blocks. It includes guidance such as:

- **Can this be done locally, without AI?** — If yes, the Wizard steers toward `ai_required = false` and a local script approach. Example: *"Which documents changed today"* is better served by a local file scan than an AI call.
- **Does this meet safety standards?** — Surfaces a warning in the conversation if a concern is found, but the conversation continues.
- **Is this task within scope?** — If the task requires tools or integrations beyond what is available, the Wizard responds: *"This task would need some technical setup — you might want to flag it for later."*

Both Wizard Design Scripts are editable via admin tasks.

### 7.3 What the Wizard Determines

Through the conversation, the Wizard:

1. Asks clarifying questions to fully understand the task
2. Consults the appropriate Wizard Design Script at key decision points
3. Determines whether the task requires AI (`ai_required`) or can run locally
4. Checks whether the required local tools exist in the library (see Section 10)
5. Identifies what fields/data are needed and decides field types
6. Suggests a task name and display category
7. Proposes a plain English summary
8. Proposes session persistence behaviour

### 7.4 The Review Button and Wizard Review Script

When the user is happy, they click **"Review"**. The Wizard passes the complete task definition through the **Wizard Review Script** — the correct script loaded by mode:

- **User mode** → User Wizard Review Script
- **Admin mode** → Admin Wizard Review Script

This is a **hard gate**:

- **Passes** → task saved as `status = 'active'`, form generated, task appears in sidebar
- **Fails** → user sees a plain English explanation and may **Save as Draft**
- **Admin only** → can overrule a failed review and save the task as active, accepting the risk

Both Wizard Review Scripts are editable via admin tasks.

### 7.5 Local Tool Gap Detection

If the Wizard determines the task requires a local tool that does not exist in the library:

1. Wizard tells the user plainly: *"I can almost do this, but I'm missing one capability. I'll flag it for review. You can save this task as a draft and come back to it once it's been sorted."*
2. Wizard generates a **tool specification** including AI-generated JS code
3. The tool spec lands in the admin tool approval queue (see Section 10)
4. Task is saved as `status = 'draft'`
5. Once admin approves the tool, the user returns to the Draft task and tries again

### 7.6 Category Management

The Wizard suggests a display category during task creation. The user confirms or suggests an alternative. New categories are created automatically. Tasks can be moved via the cog at any time.

---

## 8. Task Refinement — The Cog

Accessed via the ⚙ icon on any task view. Opens a dialog containing:

- The current plain English summary of the task
- A conversation interface
- An **"Update"** button

The user converses with the AI to request changes — logic, UI, or both. The AI updates the summary draft in real time. When satisfied, clicking Update saves the revised task definition, form schema, and execution instructions to SQLite. The task view re-renders immediately.

---

## 9. Task Types and Execution Engines

### 9.1 Access Categories

Every task has an `access_category` field that controls who can see and invoke it. This is separate from the `display_category` used for sidebar grouping.

| Access Category | Visible In UI | Who Can Run It |
|---|---|---|
| `user` | Yes — in user mode and admin mode | User, admin, AI agents |
| `admin` | Admin mode only | Admin, AI agents |
| `agent` | Not shown in UI | AI agents and the Non-AI Agent runner only |

Agents can only invoke `agent` category tasks. If an agent needs to perform something that exists only as a `user` or `admin` task, an agent-equivalent version is created and approved separately.

### 9.2 Execution Engines

Every task has an `ai_required` boolean field that determines which execution engine runs it.

| Engine | `ai_required` | How It Runs |
|---|---|---|
| **AI Agent** | `true` | Calls the configured AI provider. Uses registered AI tools and agent tasks. Streams progress and results. |
| **Non-AI Agent** | `false` | Runs a local Node.js script from the local tool library. No AI provider call. No internet required. Returns result directly. |

### 9.3 The Four Combinations

| | `ai_required = true` | `ai_required = false` |
|---|---|---|
| **`user`** | User-facing task calling the AI provider. E.g. *"What's the weather in London"*, *"Draft a letter"* | User-facing task running entirely locally. E.g. *"Which documents changed today"* |
| **`admin`** | Admin-only AI task. E.g. testing a provider, AI-powered diagnostics | Admin-only local task. E.g. export library, view event log |
| **`agent`** | Agent sub-task calling the AI provider. E.g. suggest a category, summarise notes | Agent sub-task running locally. E.g. *add_note*, *search_kb*, *db_read* |

### 9.4 Two-Layer Pattern

Agent tasks are the engine room — primitives that do one thing well. User tasks are the dashboard — friendly wrappers with good forms and plain summaries, built on top of agent tasks.

```
User Task: "Add a Note"          ← what the user sees and runs
    └── Agent Task: add_note      ← the implementation primitive
```

This pattern applies everywhere. Admin builds agent tasks first, then builds user tasks that wrap them, then exports the finished user tasks. The user never sees the agent layer.

---

## 10. Local Tool Library

### 10.1 Overview

The local tool library is the registry of all tools available to the Non-AI Agent execution engine. Each tool is a Node.js function stored in the `local_tools` database table. The library grows organically — tools are added only when a real task requires them.

The Wizard reads the tool library at task definition time to understand what local capabilities are available. The Non-AI Agent reads the tool library at execution time to run the appropriate tool.

### 10.2 Tool Definition Format

Each tool record is readable by both humans and AI:

- **`name`** — snake_case identifier (e.g. `scan_folder`)
- **`description`** — plain English. What the Wizard reads to decide if a tool is appropriate
- **`parameters`** — JSON schema of inputs: name, type, description for each
- **`returns`** — plain English description of what the tool returns
- **`code`** — executable Node.js JS code, stored as text

### 10.3 Built-in Tools

These ship with the application, authored as TypeScript modules during development:

| Tool | Description |
|---|---|
| `scan_folder` | Recursively scans a folder for files matching criteria (date, type, name pattern) |
| `read_file` | Reads the text content of a local file |
| `write_file` | Writes or appends text to a local file |
| `obsidian_add` | Writes a new markdown note to the configured Obsidian vault folder |
| `obsidian_search` | Searches file contents across the Obsidian vault folder |
| `calculate` | Performs arithmetic and unit conversion |
| `db_read` | Safe read-only queries against the SQLite database |
| `approve_tool` | Sets a local tool's status to `approved` in the `local_tools` table |
| `set_config` | Updates a key/value pair in `app_config` |
| `set_api_key` | Stores an API key securely via Electron `safeStorage` |
| `open_file_dialog` | Opens a native Windows file picker and returns the selected file path |
| `export_file` | Opens a native Windows save dialog and writes content to the chosen path |
| `read_json_file` | Reads and parses a JSON file from a given path |

DB access tools (`db_read`, `approve_tool`, `set_config`, `set_api_key`) are available only to `admin` and `agent` category tasks.

### 10.4 Tool Gap Detection and Approval Flow

When the Wizard identifies that a task requires a local tool that does not exist:

1. Wizard informs the user a capability is missing and saves the task as Draft
2. Wizard generates a complete tool spec including AI-generated JS code
3. Tool spec placed in the **Admin Tool Approval Queue** — an admin task that queries `local_tools WHERE status = 'pending_review'`
4. Admin reviews the name, description, parameters, return value, and code — edits if needed
5. Admin runs the approve task → tool inserted into `local_tools` with `status = 'approved'`
6. User returns to the Draft task, re-opens Wizard, tries again — tool now exists, task completes

### 10.5 Agent Task Library

Agent tasks (tasks with `access_category = 'agent'`) form a parallel library to local tools. Like local tools, they:

- Are reviewed and approved before use
- Grow organically based on real need
- Are built through the same Wizard flow (in admin mode)
- Are visible in admin mode in the sidebar

The agent task library and the local tool library together form the full capability set available to the execution engines.

---

## 11. Agent Task Invocation

### 11.1 Dynamic Tool Registration

Before any AI interaction begins, the app reads all `approved` agent tasks from the tasks table and registers each one as a tool with the Vercel AI SDK alongside the built-in tools (e.g. `web_search`):

| Field | Source |
|---|---|
| `name` | Task name (snake_case) |
| `description` | Task intent / plain_summary — what the AI reads to decide when to use it |
| `parameters` | Derived from the task's `form_schema` fields |
| `execute()` | Function that runs the agent task and returns the result |

The AI then calls agent tasks naturally as part of its reasoning — no special invocation syntax.

### 11.2 The Execute Function

| Agent task type | What `execute()` does |
|---|---|
| `ai_required = true` | Makes a nested AI call using the agent task's execution instructions and provided inputs |
| `ai_required = false` | Runs the appropriate local tool script and returns the result |

From the calling AI's perspective, both types behave identically — inputs in, result out.

### 11.3 Dynamic Library

Newly approved agent tasks are automatically available at the next AI interaction — no code changes required. The library expands transparently as tasks are approved.

### 11.4 Depth Limit

To prevent circular calls or infinite loops, agent task invocation is limited to a maximum depth of **3 levels**. An agent task that attempts to call beyond this limit fails gracefully with an internal log entry.

---

## 12. Knowledge Base Integration

### 12.1 Obsidian as the Knowledge Base

Obsidian is the designated personal knowledge base. It is free, local-first, and stores notes as plain markdown (`.md`) files in a vault folder on disk. The user browses, searches, and reads notes using the Obsidian desktop application.

JLH Desktop writes to the knowledge base by creating `.md` files in the vault folder directly — no Obsidian API, no dependency on Obsidian being open.

### 12.2 Configuration

The Obsidian vault folder path is set via an admin task and stored in `app_config` under `obsidian_vault_path`. All knowledge base tools read this path at runtime.

### 12.3 Two-Layer Knowledge Base Tasks

Knowledge base functionality follows the standard two-layer pattern:

**Agent tasks (primitives — `access_category = 'agent'`, `ai_required = false`):**

| Agent Task | What It Does |
|---|---|
| `add_note` | Writes a new `.md` file to the vault |
| `search_kb` | Searches file contents across the vault |
| `read_note` | Reads and returns the content of a named note |

**User tasks (friendly wrappers — `access_category = 'user'`):**

| User Task | Underlying Agent Task | `ai_required` |
|---|---|---|
| Add a Note | `add_note` | `false` |
| Search My Notes | `search_kb` | `false` |
| Read a Note | `read_note` | `false` |
| Summarise Notes on a Topic | `search_kb` + AI summarisation | `true` |

Admin builds the agent tasks first, then builds and exports the user tasks. Joanna runs the user tasks without ever seeing the agent layer.

---

## 13. Data Model

### 13.1 SQLite Tables

**`tasks`**

| Column | Type | Description |
|---|---|---|
| id | TEXT (UUID) | Unique identifier |
| name | TEXT | Display name (e.g. "Find a Hotel") |
| version | INTEGER | Increments on every save. Used for export/import conflict resolution. |
| display_category | TEXT | Sidebar grouping (e.g. "Travel", "Work") |
| access_category | TEXT | Access control: `user`, `admin`, or `agent` |
| ai_required | INTEGER | Boolean — 1 if AI provider call required, 0 if local execution |
| status | TEXT | `draft` or `active` |
| plain_summary | TEXT | User-facing plain English description |
| intent | TEXT | AI-facing description of task purpose |
| constraints | TEXT (JSON) | Rules and restrictions |
| form_schema | TEXT (JSON) | Field definitions for form rendering |
| execution_instructions | TEXT (JSON) | How the task should execute |
| tools_required | TEXT (JSON) | List of tool names needed |
| retain_last_session | INTEGER | Boolean — 1 or 0 |
| last_inputs | TEXT (JSON) | Field values from last run |
| last_results | TEXT | Results from last run |
| definition_history | TEXT (JSON) | Conversation that built this task |
| created_at | TEXT | ISO timestamp |
| last_used_at | TEXT | ISO timestamp |
| last_modified_at | TEXT | ISO timestamp |

---

**`local_tools`**

| Column | Type | Description |
|---|---|---|
| id | TEXT (UUID) | Unique identifier |
| name | TEXT | Snake_case tool identifier — unique |
| version | INTEGER | Increments on each approved change |
| description | TEXT | Plain English — what the AI reads to decide if this tool fits |
| parameters | TEXT (JSON) | Input schema: name, type, description for each parameter |
| returns | TEXT | Plain English description of what the tool returns |
| code | TEXT | Executable Node.js JS code |
| status | TEXT | `pending_review`, `approved`, `disabled` |
| ai_generated | INTEGER | Boolean — 1 if AI-generated, 0 if hand-written |
| created_at | TEXT | ISO timestamp |
| approved_at | TEXT | ISO timestamp (null until approved) |

---

**`documents`**

| Column | Type | Description |
|---|---|---|
| id | TEXT (UUID) | Unique identifier |
| name | TEXT | Human-readable name |
| doc_type | TEXT | `safety_rule`, `system_prompt`, `script`, `template` |
| context | TEXT | `admin`, `user`, `system` |
| content | TEXT | The document content |
| format | TEXT | `text`, `markdown`, `prompt`, `json` |
| is_editable | INTEGER | Boolean |
| version | INTEGER | Increments on each save |
| created_at | TEXT | ISO timestamp |
| updated_at | TEXT | ISO timestamp |

**Starting documents (minimum set):**

| Name | doc_type | context | Notes |
|---|---|---|---|
| User Safety Rules | `safety_rule` | `user` | Applied by Safety Agent in user mode |
| Admin Safety Rules | `safety_rule` | `admin` | Applied by Safety Agent in admin mode |
| User Wizard Design Script | `script` | `user` | Advisory guidelines for Wizard in user mode — includes local-vs-AI assessment, safety checks, scope checks |
| Admin Wizard Design Script | `script` | `admin` | Advisory guidelines for Wizard in admin mode — looser rules reflecting admin capabilities |
| User Wizard Review Script | `script` | `user` | Hard-gate review at save time for user-mode tasks |
| Admin Wizard Review Script | `script` | `admin` | Hard-gate review at save time for admin-mode tasks |

---

**`event_log`**

| Column | Type | Description |
|---|---|---|
| id | TEXT (UUID) | Unique identifier |
| timestamp | TEXT | ISO datetime (indexed) |
| event_type | TEXT | e.g. `ai_interaction`, `error` |
| severity | TEXT | `info`, `warning`, `error` |
| payload | TEXT (JSON) | Full event detail — structure varies by event type |

**Starting event types:**

| Event Type | Trigger | Payload includes |
|---|---|---|
| `ai_interaction` | Every AI API call | interaction_type, mode, messages, response, model, provider, tokens, duration, tools called |
| `error` | Any caught application failure | context, message, stack (internal only) |

---

**`app_config`**

| Column | Type | Description |
|---|---|---|
| key | TEXT | Config key |
| value | TEXT | Config value |
| updated_at | TEXT | ISO timestamp |

**Starting config keys:**

| Key | Description |
|---|---|
| `ai_provider` | Active AI provider (e.g. `anthropic`, `openai`) |
| `ai_model` | Active model identifier (e.g. `claude-sonnet-4-6`) |
| `app_mode` | Current mode: `user` or `admin` |
| `obsidian_vault_path` | Absolute path to the Obsidian vault folder |

---

### 13.2 Form Schema Format (JSON)

Field types supported:

```json
{
  "fields": [
    {
      "id": "location",
      "type": "text",
      "label": "Location",
      "description": "The city or address you'll be working near",
      "required": true
    },
    {
      "id": "budget",
      "type": "number",
      "label": "Max budget per night (£)",
      "required": true
    },
    {
      "id": "distance",
      "type": "select",
      "label": "Max distance from location",
      "options": ["1 mile", "2 miles", "5 miles", "10 miles"],
      "required": false,
      "default": "5 miles"
    },
    {
      "id": "import_file",
      "type": "file",
      "label": "Import file",
      "description": "Select the .json export file to import",
      "required": true
    }
  ],
  "results_display": "text_area"
}
```

Supported field types: `text`, `number`, `select`, `date`, `file`.

---

## 14. AI Interaction Model

There are four distinct AI interaction types, each with a separate system prompt and context package. All interactions are logged as `ai_interaction` events.

### 14.1 Task Definition (Wizard)

```
System prompt:
  "You are helping a user define a reusable task.
   Consult the Wizard Design Script at key decision points.
   Determine whether the task can run locally (ai_required = false)
   or needs AI. Check the local tool library for available tools.
   Check the agent task library for available agent tasks.
   Identify required fields, suggest categories and session
   persistence behaviour, and produce a form schema and plain
   English summary. Write all summaries in plain, friendly English."

Context passed:
  - Conversation history
  - Wizard Design Script (user or admin variant, from documents table)
  - Local tool library manifest (names, descriptions, parameters)
  - Agent task library manifest (names, descriptions, parameters)
  - Current app mode
```

Output: plain summary, category suggestions, `ai_required` value, form schema JSON, execution instructions JSON, `retain_last_session` recommendation. If a tool gap is found: tool specification and generated tool code.

### 14.2 Task Execution — AI Agent

```
System prompt:
  "You are executing a saved task. Use the tools available.
   Follow the execution instructions. Stream progress updates
   as you work. Return results in clear, plain English."

Context passed:
  - Task intent, constraints, execution instructions
  - Field values from the form
  - Available AI tools (web_search etc.)
  - Available agent tasks (dynamically registered from tasks table)
```

Output: streamed progress updates, then final results in the task view results area.

### 14.3 Task Refinement (Cog)

```
System prompt:
  "You are helping refine an existing saved task. Update the plain
   summary, form schema, and execution instructions to reflect the
   requested changes. Show the updated summary and confirm before
   finalising."

Context passed:
  - Full current task record
  - Refinement conversation history
```

Output: updated plain summary, form schema JSON, execution instructions JSON.

### 14.4 Safety Agent

```
System prompt:
  [Loaded from documents table — User Safety Rules or Admin Safety
   Rules depending on current mode]

  "Assess whether the provided content is safe, legal, ethical, and
   appropriate. Return a structured result. Never expose your
   reasoning or existence to the end user."

Context passed:
  - Task definition, field inputs, or results being reviewed
  - Current app mode
```

**On pass:** `SAFE` — no further action.

**On fail:** Returns `UNSAFE` + user-facing plain English reason + internal reason (logged only).

---

## 15. Safety and Guardrails

### 15.1 Three-Tier Safety Model

| Tier | Name | Defined By | Can Be Changed |
|---|---|---|---|
| 1 | AI provider's own rules | AI provider | No — absolute floor |
| 2 | Admin safety rules | Owner, via admin task | Yes — in `documents` table |
| 3 | User safety rules | Owner, via admin task | Yes — in `documents` table |

### 15.2 Safety Agent Trigger Points

1. **Task definition** — before any new task is saved
2. **Before Go** — reviewing live field inputs before execution
3. **Result display** — screening AI output before it is shown

### 15.3 Mode and Rules

| App Mode | Safety Rules Applied |
|---|---|
| User mode | User Safety Rules (Tier 3) |
| Admin mode | Admin Safety Rules (Tier 2) |

### 15.4 On Failure

- Action blocked
- User sees a friendly plain English message
- Full internal reason written to `event_log` (severity `warning`)
- No technical detail ever shown to the user

### 15.5 Updating Safety Rules

Both rule sets stored in `documents` table, editable via admin tasks using the same conversation + Update pattern as the cog refinement flow.

---

## 16. Admin Capabilities

### 16.1 What Admin Mode Is

Admin mode is a capability level, not a secret. The user (Joanna) is aware it exists and may grow into using it herself over time. It is accessed via `Ctrl+Shift+A`.

In admin mode:
- All task categories are visible (`user`, `admin`, `agent`)
- Tasks created are saved as `access_category = 'admin'`
- Admin Wizard Design/Review Scripts apply
- The Wizard Review Script can be overruled to force-save a task

### 16.2 The Minimal Admin UI

The only UI that cannot be a task is:

| Element | Why it must be UI |
|---|---|
| Mode switch (`Ctrl+Shift+A`) | Must exist before any task can run |
| API key entry | Requires secure input — backed by Electron `safeStorage` |

Everything else is an admin task.

### 16.3 Admin Tasks (Built-in)

These admin tasks ship with the application:

| Task | Description | `ai_required` |
|---|---|---|
| Set AI Provider | Update `ai_provider` and `ai_model` in `app_config` | `false` |
| Set Obsidian Vault Path | Update `obsidian_vault_path` in `app_config` | `false` |
| Review Pending Tools | Query `local_tools WHERE status = 'pending_review'`, review and approve | `false` |
| Edit Safety Rules | Conversational update of User or Admin Safety Rules document | `true` |
| Edit Wizard Design Script | Conversational update of User or Admin Wizard Design Script | `true` |
| Edit Wizard Review Script | Conversational update of User or Admin Wizard Review Script | `true` |
| View Event Log | Query and display `event_log` entries with filters | `false` |
| App Diagnostics | Display version, task count, tool count, database size | `false` |
| Export Library | Export selected entities to a JSON file | `false` |
| Import Library | Import entities from a JSON file with conflict resolution | `false` |

---

## 17. Export and Import

### 17.1 Purpose

The export/import system allows tasks, tools, and documents to be moved between instances of JLH Desktop. Both are admin tasks (`access_category = 'admin'`, `ai_required = false`).

The primary driver is the local tool and agent task libraries — built and approved on one machine, distributed to others. Admin also builds and curates user-facing tasks on their machine and distributes them to the user's machine via export/import.

### 17.2 Two-Machine Workflow

| Machine | Role |
|---|---|
| **Admin machine** | Authoring environment. Builds agent tasks, local tools, user tasks. Tests and approves. Exports. |
| **User's machine** | Consumption environment. Imports. Runs. |

### 17.3 Exportable Entities

| Entity | Exported | Notes |
|---|---|---|
| Tasks | Yes | All access categories |
| Local tools | Yes | `approved` status only |
| Documents | Yes | All documents |
| `app_config` | No | Machine-specific — never exported |

Admin selects which entities to include via checkboxes in the export task form. Export produces a single JSON file via a native save dialog.

### 17.4 Export File Format

```json
{
  "export_version": "1",
  "exported_at": "2026-02-21T14:30:00Z",
  "source_app_version": "1.0.0",
  "entities": {
    "tasks": [ ],
    "local_tools": [ ],
    "documents": [ ]
  }
}
```

Each entity includes its `id` (GUID) and `version` integer.

### 17.5 Import Logic

| Situation | Action |
|---|---|
| GUID not found locally | Import — create new |
| GUID found, same version | Skip — already up to date |
| GUID found, imported version newer | Update local with imported version |
| GUID found, local version newer | Admin chooses: **skip** or **accept** |

### 17.6 Import Flow

The import task uses an agent task to handle conflict resolution conversationally:

1. Admin selects import file via file picker in the task form
2. Go is pressed — agent analyses file against local DB and returns a plain English preview in the results area:

```
Import preview — 16 entities found

  ✦ 5 new tasks           → will be imported
  ✦ 3 new local tools     → will be imported
  ✓ 5 already up to date  → no action

  ⚠ 1 conflict (your version is newer):
      Local Tool: "Scan Folder"  your v4 vs imported v3

  Type  proceed  to import (conflict skipped by default)
  or  accept scan_folder  to take the imported version
  or  cancel  to abort.
```

3. Admin types a response in the follow-up input
4. Agent interprets the response and commits or aborts

Conflicts default to **skip** (safe). Admin can explicitly accept individual conflicts by name.

### 17.7 Import Routing

Entities are routed automatically on import by their `access_category` field — no manual selection needed:

- `user` tasks → user library (visible to all)
- `admin` tasks → admin library (admin mode only)
- `agent` tasks → agent library (not shown in UI)

### 17.8 Dependency Resolution

If a task being imported requires a local tool that is not present locally and not in the import file:

> *"Task 'Scan Changed Documents' requires the tool `scan_folder` which is not in this import. The task will be imported as Draft until the tool is available."*

If the required tool is in the import file it is imported automatically alongside the task.

### 17.9 Versioning

All exportable entities carry a GUID (`id`) and integer `version`. These two fields together uniquely identify any specific revision across all instances:

- `tasks` — `id` + `version`
- `local_tools` — `id` + `version`
- `documents` — `id` + `version`

---

## 18. Future Considerations (Out of Scope for V1)

| Feature | Notes |
|---|---|
| Cloud sync | Out of scope. SQLite local only for V1. |
| Result export | Copy to clipboard, save as PDF, email. Add per task type as needed. |
| Additional tool types | New task categories added by registering new AI tools. |
| Multi-user support | Not required for V1. Single primary user on a single machine. |
| AI's own application-level safety rules | Identified as a future concept — requirements to be defined. |
| Local tool historical versioning | V1 retains current version only. |
| Arbitrary script generation | V1 uses a registered tool library. Generating and executing arbitrary user-defined scripts is a future consideration. |
| Task-defined response UI | During task creation, the Wizard agrees a set of response controls (buttons: proceed/cancel/yes/no) that render in the results area instead of a plain text follow-up input. Simple button types first, more complex controls later. |

---

## 19. Non-Functional Requirements

- **Performance:** App must feel responsive. AI API calls show a clear loading indicator with streamed progress. Long-running tasks must not block the UI.
- **Event logging:** Event log writes must be **non-blocking** — queued and written in the background. Never on the critical path of any user action.
- **Reliability:** SQLite writes must be transactional — no partial saves.
- **Security:** API keys stored via `safeStorage` only. The AI never receives more context than needed. AI-generated tool code requires admin approval before execution. DB write tools available to admin/agent tasks only.
- **Usability:** Every user-facing message must be plain English. No technical error messages shown to the user. All errors caught and presented as friendly, actionable guidance.
- **Maintainability:** Task types and tools registered in central configuration. Adding a new task type requires no changes to core logic. AI provider swapped via admin task without code changes.

---

## 20. Minimum PC Specification

| Component | Minimum | Recommended |
|---|---|---|
| **Operating System** | Windows 10 version 1809 (October 2018 Update) | Windows 10 21H2 or later / Windows 11 |
| **Architecture** | x64 (64-bit) only | x64 |
| **RAM** | 4 GB | 8 GB |
| **Storage** | 600 MB free (app ~300 MB + database growth) | 1 GB free |
| **Display** | 1280 × 720 | 1920 × 1080 |
| **Internet** | Required for AI tasks | Stable broadband |
| **CPU** | Any x64 (2 cores, ~2 GHz) | 4 cores |
| **GPU** | None required | Any |

### Notes
- **No Node.js required** on the target machine. The packaged `.exe` is fully self-contained.
- **No Office/Word required.** JLH Desktop is entirely independent of the JLH Word add-in.
- **Obsidian** must be separately installed if the user wishes to browse the knowledge base. JLH Desktop does not require Obsidian to be open to write notes.
- **Non-AI tasks run offline.** Internet is only required for tasks where `ai_required = true`.
- **Windows 10 build** — must be 17763 or higher (`Settings → System → About → OS Build`).

---

## 21. Platform Compatibility

### Development Machine
Windows 11 laptop. All development and testing happens here first.

### Target Machine
Windows 10 desktop (older hardware). The final application must run correctly on this machine.

### Known Considerations

| Area | Detail |
|---|---|
| **Electron** | Supports Windows 10 (1809) and later. Confirm Win10 build version before first run. |
| **WebView2** | Not a dependency — Electron bundles its own Chromium. WebView2 issues affecting the JLH Word add-in do not apply here. |
| **SQLite native bindings** | `better-sqlite3` compiles against the Node.js version bundled with Electron. Handled automatically by the build process. |
| **Windows Defender SmartScreen** | An unsigned `.exe` triggers a "Windows protected your PC" warning on first run. Click "More info → Run anyway". Expected for a personal unsigned app. |
| **Performance** | Electron embeds a full Chromium instance. Minimum 4GB RAM. Cold start may be slower on the Win10 machine. |
| **Network / Firewall** | Outbound HTTPS to the AI provider API and search API must not be blocked. Verify on the target network. |
| **Visual C++ Redistributables** | May be required by native modules. Bundle in the installer or verify pre-installed. |

### Build & Distribution
Final `.exe` produced using **electron-builder** or **electron-forge**. Self-contained Windows installer. No Node.js required on the target machine.

---

## 22. Companion Test Harness

Before building the main Electron application, a lightweight **CLI test harness** is developed and run on both machines.

### Purpose
- Verify all external dependencies work on both machines
- Prove the core loop before committing to the full build
- Identify environment-specific failures early

### Structure

```
jlh-desktop-harness/
├── src/
│   ├── tests/
│   │   ├── test-ai-api.ts            — Basic AI provider API call
│   │   ├── test-ai-json.ts           — AI produces valid JSON schema
│   │   ├── test-ai-tools.ts          — AI tool use (web_search invocation)
│   │   ├── test-search-api.ts        — Tavily Search API call
│   │   ├── test-sqlite.ts            — SQLite create / read / write
│   │   └── test-core-loop.ts         — Full mini loop end to end
│   └── run-all.ts                    — Runs all tests, reports pass/fail
├── package.json
└── tsconfig.json
```

### Tests

| Test | What It Proves |
|---|---|
| `test-ai-api` | Network connectivity, API key valid, basic response received |
| `test-ai-json` | AI reliably returns parseable, valid JSON schema |
| `test-ai-tools` | Tool use works — AI correctly invokes a registered tool |
| `test-search-api` | Search API key valid, results returned |
| `test-sqlite` | Native bindings compile correctly, CRUD operations succeed |
| `test-core-loop` | Full sequence: conversation → JSON schema → field values → tool call → result |

### Pass Criteria
All six tests must pass on **both machines** before Electron development begins.

### Output
Each test prints PASS / FAIL with a one-line reason. Detail available via `--verbose`.

---

## 23. Build Order

Development proceeds in phases. Each phase has a clear exit criterion — the next phase does not begin until it is met.

### Phase 0 — Companion Test Harness

Build and run the CLI test harness on both machines.

**Exit criterion:** All six harness tests pass on both machines.

---

### Phase 1 — Project Skeleton

- Electron window launches
- Basic sidebar (hardcoded, no data)
- SQLite connected — all tables created (`tasks`, `local_tools`, `documents`, `event_log`, `app_config`)
- Starting documents seeded (six Wizard/Safety documents)
- Built-in local tools seeded in `local_tools`
- AI provider connected — basic call confirmed

**Exit criterion:** App launches on both machines. SQLite created with all tables and seed data. AI provider responds.

---

### Phase 2 — Core Mechanic

Hardcoded — no dynamic task definition yet.

- AI receives a fixed prompt, returns a valid form schema JSON
- App renders a real form from the JSON
- "Go" submits field values to AI with a fixed execution instruction
- AI calls web search tool, streams progress, returns results
- Results displayed

**Exit criterion:** A hardcoded hotel search form renders. Filling in location and budget and pressing Go returns real hotel suggestions.

---

### Phase 3 — Task Definition Flow (The Wizard)

- "+ New Task" opens conversation view
- Wizard consults correct Wizard Design Script by mode
- Wizard steers toward local execution when appropriate
- Wizard checks local tool and agent task libraries
- Joanna clicks "Review" → correct Wizard Review Script runs as hard gate
- Pass: task saved as active. Fail: save as Draft. Admin: can overrule.
- Task appears in sidebar under correct category

**Exit criterion:** A real task defined through conversation, saved as active, visible in sidebar.

---

### Phase 4 — Task Execution and Results

- AI Agent execution: Tavily integrated, streamed progress, results displayed
- Non-AI Agent execution: local tools invoked, result returned
- Agent task invocation: agent tasks registered dynamically as AI SDK tools
- Input validation before Go
- Follow-up input active after results
- Session persistence working

**Exit criterion:** Hotel search (AI Agent), file scan (Non-AI Agent), and a task using an agent task all work end to end on both machines.

---

### Phase 5 — Task Refinement (Cog)

- Cog opens refinement dialog
- AI updates summary draft in real time
- Update saves revised task to SQLite
- Task view re-renders immediately

**Exit criterion:** Task modified via cog. Changes persist across restarts.

---

### Phase 6 — Local Tool and Agent Task Libraries

- Tool gap detection working in Wizard
- Wizard generates tool spec and code on gap
- Admin task: Review Pending Tools — view, edit, approve
- Approved tools immediately available
- Draft task retry flow working
- Agent tasks built via Wizard in admin mode
- Agent tasks dynamically registered as tools before AI interactions
- Depth limit enforced

**Exit criterion:** Gap detected, tool approved, Draft task retried successfully. Agent task invoked by an AI agent during execution.

---

### Phase 7 — Safety Agent and Admin Mode

- Safety Agent at all three trigger points
- Correct safety rules applied by mode
- Admin mode via `Ctrl+Shift+A`
- Minimal admin UI: mode indicator + API key entry
- All built-in admin tasks functional
- Knowledge base tasks working (add, search, read)
- Two-layer KB pattern: agent tasks + user wrapper tasks

**Exit criterion:** Unsafe task blocked with friendly message. Admin mode switches correctly. All built-in admin tasks run. KB tasks add and retrieve notes from Obsidian vault.

---

### Phase 8 — Export and Import

- Export Library admin task: entity selection, native save dialog, JSON output
- Import Library admin task: file picker, agent-driven preview and conflict resolution
- Import routing by `access_category` automatic
- Dependency resolution: missing tool tasks imported as Draft
- Version conflict handling: proceed/accept/cancel via follow-up input

**Exit criterion:** Task and tool exported from one instance, imported into a clean instance. Version conflict resolved correctly. Missing-tool dependency flagged and task saved as Draft.

---

### Phase 9 — Polish and Non-Technical UX

- Friendly plain English error messages throughout
- Loading indicators and streamed progress on all AI calls
- Event log writes confirmed non-blocking
- Draft badges visible on cards and task views
- Task summaries and field descriptions reviewed for clarity and tone
- Windows Defender SmartScreen behaviour confirmed
- Full end-to-end test on Win10 target machine

**Exit criterion:** The app can be used without assistance. All error states produce friendly, actionable messages. Acceptable performance on Win10.
