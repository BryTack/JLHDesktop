# JLH Desktop — Working Specification

**Version:** 0.0.4
**Date:** 2026-02-21
**Status:** Draft

---

## 1. Project Overview

**JLH Desktop** is a standalone Windows desktop application (.exe) that acts as a personal task library for a non-technical user (Joanna). It allows her to define reusable tasks once, in plain English, through a guided AI conversation. Defined tasks persist permanently and can be returned to days, weeks, or months later — fully self-explanatory and ready to use.

### Core Philosophy

> *Define a task once. Come back months later. Understand it immediately. Use it. Get results.*

- The AI does the thinking — Claude handles housekeeping, suggestions, categorisation, and layout decisions
- Joanna makes decisions, not configurations
- Every task is self-documenting — no external knowledge required to use it
- Non-technical user experience throughout — complexity is hidden or flagged for specialist help

---

## 2. Research Findings

Prior to technical design, research was conducted into existing GitHub projects, commercial products, open source libraries, and known production patterns for Electron + AI applications. Key findings are summarised here.

### 2.1 Originality Assessment

The concept is **genuinely novel** as a combined product. The distinguishing three-stage loop does not exist as a cohesive, non-technical, locally-running desktop application anywhere in the market:

1. **Define** — a guided AI conversation extracts what a task does and what inputs it needs, then persists this as a typed schema with a human-readable summary
2. **Parameterise** — the saved schema is rendered as a clean, purpose-built form — no chat required, no re-explaining
3. **Execute** — the AI executes the task using the filled-in values and real tools

### 2.2 Closest Existing Products

| Product | What It Shares | What It Lacks |
|---|---|---|
| Raycast AI PromptLab | Saved AI commands with variable placeholders | macOS only. No guided definition conversation. No tool execution. Prompts are static templates, not typed schemas. |
| Zapier Copilot | Natural language workflow creation, saves reusable flows | Cloud/SaaS only. Requires app integrations (Slack, Gmail etc.). Not a personal local library. Cannot run free-form tasks. |
| AnythingLLM | Desktop app, local storage, agent builder | Agent builder is node-graph-based, not conversational. No form generation from schema. Skews technical in configuration. |
| PyGPT | Desktop, presets, plugin system, supports Claude | Presets are prompt templates, not parameterised schemas. No form generation. Task definition and execution are not separated concepts. |
| Lindy AI | Conversational agent definition, recurring tasks | Cloud-only. No local storage. No form generation. Runs on schedules/triggers, not on-demand form execution. |
| Flowise / LangFlow | Visual workflow builder, local, open source | Requires technical drag-and-drop node editing. Not accessible to a non-technical user. |

### 2.3 Confirmed Gaps This Project Fills

- **Conversationally-defined, schema-first task persistence** — no existing tool cleanly separates "define a task with AI" from "run a task with AI"
- **AI-generated form UI as the daily interaction surface** — all AI products lead with chat; the form is the novel UX pattern here
- **Local-first, Windows desktop, non-technical user** — no Windows desktop product targets this combination
- **Personal library metaphor** — tasks are owned, named, reusable assets that accumulate over time; no current AI product frames itself this way

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
All SQLite and LLM API calls must run in Electron's main process. The renderer communicates through a typed, named `contextBridge` (preload script). Exposing raw `ipcRenderer` to the renderer is a common security mistake in amateur Electron projects and must be avoided.

**ASAR archive exclusion**
Electron packages app code into a read-only ASAR archive. The SQLite database must live in `app.getPath('userData')` — outside the archive — so it can be written to at runtime. First launch runs migration SQL to create the empty schema.

**JSON schema generation reliability**
Prompt-only JSON generation from Claude fails to produce valid output approximately 5% of the time. Using Claude Structured Outputs with a Zod schema reduces this to effectively zero. This is the single most important reliability decision in the application.

**Drizzle-kit in packaged apps**
The `drizzle-kit` migration CLI must not run inside the packaged application. The correct approach: generate migrations during development, bundle the resulting SQL files into the app, and execute them programmatically at startup using `better-sqlite3`'s `exec()` method.

**Streaming on task execution**
When "Go" is pressed, tasks may take 10–60 seconds. Streaming intermediate progress back to the UI ("Searching the web for…", "Reviewing results…") is expected UX for AI applications in 2025. A static loading spinner with no feedback is perceived as broken.

**Execution reliability**
Production AI agent hallucination rates are reported at 17–33% even in commercial tools. Mitigation: keep individual tasks narrow in scope (a task that does three things is three tasks), show intermediate results where possible, and store execution logs per run for debugging.

---

## 3. Technology Stack

The stack was initially drafted and then validated and refined against research into existing projects, open source libraries, and known Electron production patterns. The table below reflects the confirmed choices.

| Layer | Technology | Reason |
|---|---|---|
| **Build tooling** | electron-vite + TypeScript | Official 2025 Electron build tool. Vite-based — fast hot reload, smaller bundles than Webpack. Scaffolds Electron + React + TypeScript cleanly. |
| **Desktop shell** | Electron | Bundles its own Chromium — no WebView2 dependency (avoids the JLH Word add-in blank screen issue entirely). Produces a self-contained Windows `.exe`. |
| **UI framework** | React | Component-based, works naturally with dynamically rendered JSON schemas. |
| **UI components** | Shadcn/UI + TailwindCSS | Ships source code into the project (not a black-box dependency). Fully controllable styling. Consistent, clean, non-technical appearance. |
| **Form rendering** | react-jsonschema-form (RJSF) | The standard library for rendering validated forms from JSON Schema in React. Claude generates the schema; RJSF renders the form. Supports conditional fields, custom widgets, validation. |
| **Database driver** | better-sqlite3 | Fastest synchronous SQLite driver for Node.js. Runs in Electron main process. |
| **ORM / migrations** | Drizzle ORM | Lightweight TypeScript-first ORM. Schema defined in code, type-safe queries. Migration SQL files bundled into the app and run at startup — drizzle-kit itself only runs on the dev machine. |
| **LLM orchestration** | Vercel AI SDK | TypeScript-first. `generateObject` with Zod schemas constrains Claude's output to valid JSON — critical for reliable form schema generation. `streamText` with tool definitions handles task execution with real-time progress. |
| **Structured outputs** | Claude API + Zod | Claude Structured Outputs (beta) combined with Zod schema validation guarantees the AI returns valid, complete JSON schemas. Eliminates the ~5% malformed-output failure rate of prompt-only JSON generation. |
| **AI model** | Claude (Anthropic) | Conversation, task definition, form schema generation, task execution, safety review. |
| **Web search tool** | Tavily (`@tavily/core`) | Purpose-built search API for AI agents. Returns structured, LLM-ready results. 1,000 free calls/month. Official TypeScript/Node.js SDK. |
| **API key storage** | Electron `safeStorage` | OS-level encryption (Windows DPAPI) for storing Claude and Tavily API keys. Keys are never in source code or config files. Available since Electron 15. Fully supported on Windows 10. |
| **Database location** | `app.getPath('userData')` | OS-correct persistent storage location. Database file lives outside the ASAR archive (which is read-only) and persists across app updates. |

---

## 3. Application Structure

### 3.1 Navigation

The app uses a **persistent left sidebar** for navigation. The main area renders the selected view.

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

### 3.2 Views

| View | Description |
|---|---|
| **Home** | Shown on launch. Displays task library as cards. Entry point for new task creation. |
| **Task View** | Full view of a selected task — summary, fields, Go button, results area |
| **Admin Page** | Hidden. Accessed via `Ctrl+Shift+A`. Not visible to Joanna. |

---

## 4. Home View

Displayed on application launch. Contains:

- A grid/list of **task cards**, each showing:
  - Task name
  - One-line summary
  - Category
  - Last used date
- A **"+ New Task"** button that begins the task definition conversation

As the library grows, cards are grouped by category matching the sidebar. Clicking any card opens the Task View for that task.

---

## 5. Task View

### 5.1 Always Visible

Every task view permanently displays:

- **Task name** (top of view)
- **Plain English summary** — what this task does, written for Joanna
- **Field descriptions** — each input field has a plain English label and explanation of what to enter
- **Any restrictions or notes** — e.g. "Results are from public sources — always verify before booking"
- **Last used date**

### 5.2 Input Form

Dynamically rendered from the task's stored JSON schema. Field types include:

- Text input
- Number input
- Dropdown / select
- Date picker
- (Extensible — new field types added as needed)

### 5.3 Buttons

| Button | Action |
|---|---|
| **Go** | Validates inputs, runs the task via Claude + tools, displays results |
| **New** | Clears last session inputs and results (only shown if `retain_last_session = true`) |
| **⚙ (Cog icon)** | Opens the task refinement dialog |

### 5.4 Results Area

Displayed below the form after "Go" is pressed. Format is defined per task during setup (text area, list, table, etc.). A follow-up question input remains active after results are shown — Joanna can ask Claude further questions in context.

### 5.5 Session Persistence

Each task has a `retain_last_session` flag (boolean, set during task definition by Claude). When true:
- Last inputs and results are shown when the task is reopened
- "New" button is visible to clear and start fresh

When false:
- Task always opens clean

Claude suggests the appropriate default during task creation and states it plainly in the task summary. Joanna can override via the cog.

---

## 6. Task Definition — The AI Conversation

### 6.1 Starting a New Task

Triggered by "+ New Task". A conversation view opens. Claude asks:

> *"How can I help?"*

Joanna describes her task in plain English. Claude then:

1. Asks clarifying questions to fully understand the task
2. Identifies what fields/data are needed
3. Decides field types (text, dropdown, number, etc.)
4. Suggests a task name and category
5. Proposes a plain English summary
6. Proposes session persistence behaviour

This is an iterative back-and-forth until both parties are satisfied.

### 6.2 Complexity Threshold

During the definition conversation, Claude assesses whether the task is within scope for a non-technical user. If the task would require custom integrations, private system access, or specialist configuration, Claude responds:

> *"This task would need some technical setup to work properly — you might want to ask someone with IT experience to help configure it. Would you like to note this idea for later?"*

No technical detail is given. The message is friendly and non-alarming.

### 6.3 Confirming and Saving

When Joanna is happy, she clicks **"Do it"**. Claude:
- Passes the task through the hidden safety reviewer
- If safe: saves the task to SQLite, generates the form, adds to sidebar under the suggested category
- If blocked: shows a polite plain English message, does not save

### 6.4 Category Management

Claude suggests a category during task creation. Joanna confirms or suggests an alternative. New categories are created automatically. Tasks can be moved via the cog at any time.

---

## 7. Task Refinement — The Cog

Accessed via the ⚙ icon on any task view. Opens a dialog containing:

- The current plain English summary of the task
- A conversation interface
- An **"Update"** button

Joanna converses with Claude to request changes:
- Logic changes: *"The hotel must be within 5 miles of the location"*
- UI changes: *"I want a dropdown for distance instead of a text box"*
- Both: *"Actually I want restaurants, not hotels"*

Claude updates the task summary in the dialog as the conversation progresses. When Joanna agrees with the updated summary, she clicks Update. The task definition, form schema, and execution instructions are all updated in SQLite. The task view re-renders immediately.

---

## 8. Task Types

The system supports multiple task types, each mapping to one or more Claude tools. New types are added by registering new tools.

| Category | Examples | Tools Used |
|---|---|---|
| Information retrieval | Hotel search, price comparison, fact lookup | `web_search` |
| Content creation | Draft a letter, write an email, create a report | `create_document` |
| Data processing | Calculate expenses, convert units, summarise pasted text | `calculate`, `summarise` |
| File operations | Read a local CSV, reformat data, extract information | `read_file`, `process_data` |
| Planning assistance | Trip itinerary, packing list, schedule suggestions | `web_search`, `calculate` |
| Communication drafting | Compose an email (Joanna sends manually) | `create_document` |

---

## 9. Data Model

### 9.1 SQLite Tables

**`tasks`**

| Column | Type | Description |
|---|---|---|
| id | TEXT (UUID) | Unique identifier |
| name | TEXT | Display name (e.g. "Find a Hotel") |
| category | TEXT | Sidebar category |
| plain_summary | TEXT | Joanna-facing plain English description |
| intent | TEXT | AI-facing description of task purpose |
| constraints | TEXT (JSON) | Rules and restrictions |
| form_schema | TEXT (JSON) | Field definitions for form rendering |
| execution_instructions | TEXT (JSON) | How Claude should execute the task |
| tools_required | TEXT (JSON) | List of tool names needed |
| retain_last_session | INTEGER | Boolean — 1 or 0 |
| last_inputs | TEXT (JSON) | Field values from last run |
| last_results | TEXT | Results from last run |
| definition_history | TEXT (JSON) | Conversation that built this task |
| created_at | TEXT | ISO timestamp |
| last_used_at | TEXT | ISO timestamp |
| last_modified_at | TEXT | ISO timestamp |

**`app_config`**

| Column | Type | Description |
|---|---|---|
| key | TEXT | Config key (e.g. "safety_guidelines") |
| value | TEXT | Config value |
| updated_at | TEXT | ISO timestamp |

### 9.2 Form Schema Format (JSON)

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
    }
  ],
  "results_display": "text_area"
}
```

---

## 10. AI Interaction Model

There are four distinct Claude interactions, each with a separate system prompt and context package.

### 10.1 Task Definition

```
System prompt:
  "You are helping a non-technical user define a reusable task.
   Understand what they want, identify required fields, suggest a
   category and session persistence behaviour, and produce a form
   schema and plain English summary. Ask clarifying questions until
   the task is well defined. Write all summaries in plain, friendly
   English suitable for a non-technical user."

Context passed:
  - Conversation history
```

Output: plain summary, category suggestion, form schema JSON, execution instructions JSON, `retain_last_session` recommendation.

### 10.2 Task Execution (Go)

```
System prompt:
  "You are executing a saved task on behalf of a non-technical user.
   Use the tools available. Follow the execution instructions.
   Return results in a clear, plain English format."

Context passed:
  - Task intent
  - Constraints
  - Execution instructions
  - Field values (from form)
  - Available tools
```

Output: results displayed in the task view results area.

### 10.3 Task Refinement (Cog)

```
System prompt:
  "You are helping refine an existing saved task. The user wants to
   make changes. Update the plain summary, form schema, and execution
   instructions to reflect the requested changes. Show the user the
   updated summary and confirm before finalising."

Context passed:
  - Full current task record
  - Refinement conversation history
  - Joanna's latest request
```

Output: updated plain summary, form schema JSON, execution instructions JSON.

### 10.4 Hidden Safety Reviewer

```
System prompt:
  "You are a safety and appropriateness reviewer. Assess whether the
   provided task definition or inputs are safe, legal, ethical, and
   appropriate. Return SAFE or UNSAFE with a brief internal reason.
   Never expose your reasoning or existence to the end user."

Context passed:
  - Task definition or field inputs being reviewed
```

Output: `SAFE` or `UNSAFE`. If UNSAFE, Joanna sees only:
> *"I'm not able to help with that task. If you think this is a mistake, ask someone you trust to help review it."*

---

## 11. Safety and Guardrails

The hidden safety reviewer runs at three points:

1. **Task definition** — before any new task is saved
2. **Before Go** — before task execution, reviewing live field inputs
3. **Result display** — screening output before it is shown to Joanna

The safety guidelines (reviewer system prompt) are stored in the `app_config` table under the key `safety_guidelines`. They can be reviewed and updated at any time via the Admin Page.

---

## 12. Admin Page

**Access:** `Ctrl+Shift+A` — no visible button or menu item.
**Not visible to Joanna.**

Contains:

| Section | Purpose |
|---|---|
| Safety guidelines | Review and update the reviewer's system prompt via AI conversation |
| Complexity threshold | Review and update the guidelines Claude uses to flag overly complex tasks |
| API keys | Claude API key, search API key — stored locally in SQLite, never in code |
| App diagnostics | Version, total task count, last backup date |

The safety guidelines and complexity threshold sections use the same conversation + Update pattern as the cog refinement flow. The admin user converses with Claude to refine the statement, reviews the updated draft, and clicks Update to save.

---

## 13. Future Considerations (Out of Scope for V1)

| Feature | Notes |
|---|---|
| Export / import tasks | Serialise a task record to JSON file. Import reverses this. No architectural changes needed — add when required. |
| Cloud sync | Out of scope. SQLite local only for V1. |
| Result export | Copy to clipboard, save as PDF, email. Add per task type as needed. |
| Additional tool types | New task categories added by registering new Claude tools. |
| Multi-user support | Not required. Single user (Joanna) on a single machine. |

---

## 14. Non-Functional Requirements

- **Performance:** App must feel responsive. Claude API calls should show a clear loading indicator. Long-running tasks should not block the UI.
- **Reliability:** SQLite writes must be transactional — no partial saves.
- **Security:** API keys stored locally only, never in source code or logs. Claude never receives more context than needed for each interaction.
- **Usability:** Every user-facing message must be plain English. No technical error messages shown to Joanna. All errors are caught and presented as friendly, actionable guidance.
- **Maintainability:** Task types and tools are registered in a central configuration. Adding a new task type requires no changes to core application logic.

---

## 15. Minimum PC Specification

These are the minimum requirements for running the final packaged application on any target machine (including the Win10 machine).

| Component | Minimum | Recommended |
|---|---|---|
| **Operating System** | Windows 10 version 1809 (October 2018 Update) | Windows 10 version 21H2 or later / Windows 11 |
| **Architecture** | x64 (64-bit) only — 32-bit not supported | x64 |
| **RAM** | 4 GB | 8 GB |
| **Storage** | 600 MB free (app ~300 MB + database growth) | 1 GB free |
| **Display** | 1280 × 720 minimum resolution | 1920 × 1080 |
| **Internet** | Required for task execution (Claude API + web search) | Stable broadband |
| **CPU** | Any x64 processor (2 cores, ~2 GHz) | 4 cores |
| **GPU** | None required — Electron can run in software rendering mode | Any |

### Notes
- **No Node.js required** on the target machine. The packaged `.exe` is fully self-contained.
- **No Office/Word required.** JLH Desktop is entirely independent of JLH (the Word add-in).
- **Internet is required at runtime** for any task that uses the Claude API or web search. The app can be opened and tasks browsed offline, but pressing "Go" will fail gracefully with a friendly message if no connection is available.
- **Windows 10 build version** — check via `Settings → System → About → OS Build`. Must be 17763 or higher (build number for version 1809).

---

## 16. Platform Compatibility

### Development Machine
Windows 11 laptop. All development and testing happens here first.

### Target Machine
Windows 10 desktop (older hardware). The final application must run correctly on this machine.

### Known Considerations

| Area | Detail |
|---|---|
| **Electron** | Supports Windows 10 (1809 / October 2018 update) and later. Confirm Win10 build version on target machine before first run. |
| **WebView2** | Not a dependency — Electron bundles its own Chromium. The WebView2 issues affecting JLH (the Word add-in) do not apply here. |
| **SQLite native bindings** | `better-sqlite3` compiles against the Node.js version bundled with Electron. Handled automatically by the build process — no manual steps on target machine. |
| **Windows Defender SmartScreen** | An unsigned `.exe` will trigger a "Windows protected your PC" warning on first run. Joanna must click "More info → Run anyway". Expected behaviour for a personal unsigned app. |
| **Performance** | Electron embeds a full Chromium instance. Target machine should have a minimum of 4GB RAM. Cold start may be slower than on development machine. |
| **Network / Firewall** | Outbound HTTPS calls to the Claude API and web search API must not be blocked. Verify on the target machine's specific network before deployment. |
| **Visual C++ Redistributables** | May be required by native modules. Bundle within the installer or verify pre-installed on target machine. |

### Build & Distribution
The final `.exe` is produced using **electron-builder** or **electron-forge**, generating a self-contained Windows installer. No Node.js installation required on the target machine. The installer handles all dependencies.

---

## 17. Companion Test Harness

Before building the main Electron application, a lightweight **CLI test harness** is developed and run on both machines. This de-risks the environment and proves the core technical mechanic before any UI work begins.

### Purpose

- Verify all external dependencies work on both Windows 11 (dev) and Windows 10 (target)
- Prove the core loop — conversation → JSON schema → tool execution — before committing to the full build
- Identify environment-specific failures early (network, API keys, native modules)

### Structure

A small standalone TypeScript project. Runs from the command line with Node.js. No Electron, no UI.

```
jlh-desktop-harness/
├── src/
│   ├── tests/
│   │   ├── test-claude-api.ts        — Basic Claude API call
│   │   ├── test-claude-json.ts       — Claude produces valid JSON schema
│   │   ├── test-claude-tools.ts      — Claude tool use (web_search invocation)
│   │   ├── test-search-api.ts        — Tavily / Brave Search API call
│   │   ├── test-sqlite.ts            — SQLite create / read / write
│   │   └── test-core-loop.ts         — Full mini loop end to end
│   └── run-all.ts                    — Runs all tests, reports pass/fail
├── package.json
└── tsconfig.json
```

### Tests

| Test | What It Proves |
|---|---|
| `test-claude-api` | Network connectivity, API key valid, basic response received |
| `test-claude-json` | Claude reliably returns parseable, valid JSON schema from a prompt |
| `test-claude-tools` | Tool use works — Claude correctly invokes a registered tool |
| `test-search-api` | Search API key valid, results returned |
| `test-sqlite` | Native bindings compile correctly, CRUD operations succeed |
| `test-core-loop` | Full sequence: conversation → JSON schema → field values → tool call → result |

### Pass Criteria

All six tests must pass on **both machines** before Electron development begins. Any failure on the Win10 machine is resolved before proceeding.

### Output

Each test prints a clear PASS / FAIL with a one-line reason. No technical stack traces visible at the summary level — detail available on request via a `--verbose` flag.

---

## 18. Build Order

Development proceeds in phases. Each phase has a clear exit criterion — the next phase does not begin until it is met.

### Phase 0 — Companion Test Harness
*(Pre-build environment validation)*

Build and run the CLI test harness on both the Win11 development machine and the Win10 target machine.

**Exit criterion:** All six harness tests pass on both machines.

---

### Phase 1 — Project Skeleton

Set up the Electron + TypeScript + React project. No application logic yet.

- Electron window launches
- Basic sidebar (hardcoded, no data)
- Main area placeholder
- SQLite connected, `tasks` and `app_config` tables created
- Claude API connected — basic call confirmed working within Electron context

**Exit criterion:** App launches on both machines. SQLite file created. Claude responds to a test call.

---

### Phase 2 — Core Mechanic (Highest Priority)

Prove the fundamental loop works before building anything around it. Hardcoded conversation — no dynamic task definition yet.

- Claude receives a fixed prompt and returns a valid form schema JSON
- App reads the JSON and renders a real form (correct field types, labels, descriptions)
- "Go" button submits field values to Claude with a fixed execution instruction
- Claude calls the web search tool, returns results
- Results displayed in the main area

**Exit criterion:** A hardcoded hotel search form renders correctly. Filling in location and budget and pressing Go returns real hotel suggestions. Proves the loop end to end.

---

### Phase 3 — Task Definition Flow

Make Phase 2 dynamic — driven by a real AI conversation.

- "+ New Task" opens a conversation view
- Claude asks "How can I help?" and conducts the full definition dialogue
- Claude produces form schema, plain summary, category suggestion, session persistence recommendation
- Joanna confirms → "Do it" saves task to SQLite
- Task appears in sidebar under the correct category

**Exit criterion:** A real task is defined through conversation, saved to SQLite, and visible in the sidebar with correct category.

---

### Phase 4 — Task Execution and Results

Wire up the full execution layer properly.

- Tavily / Brave Search API integrated as a registered Claude tool
- Input validation before Go (Claude checks fields, asks Joanna to clarify if needed)
- Results displayed cleanly below the form
- Follow-up question input active after results shown
- Session persistence (`retain_last_session`) working — last inputs and results restored on reopen, "New" button clears them

**Exit criterion:** Full hotel search task works end to end on both machines. Results display correctly. Follow-up questions work. Session is retained and clearable.

---

### Phase 5 — Task Refinement (Cog)

- Cog icon opens refinement dialog showing current plain English summary
- Joanna converses with Claude to request changes
- Claude updates the summary draft in real time during conversation
- "Update" saves revised task definition, form schema, and execution instructions to SQLite
- Task view re-renders immediately with updated form

**Exit criterion:** An existing task is successfully modified through the cog. Form updates correctly. Changes persist across app restarts.

---

### Phase 6 — Safety and Admin

- Hidden safety reviewer running at all three trigger points (task definition, before Go, result display)
- Admin page accessible via `Ctrl+Shift+A` — not visible to Joanna
- Safety guidelines editable via conversation on admin page, saved to `app_config`
- Complexity threshold editable via admin page
- API keys stored and managed via admin page (removed from any config files or code)

**Exit criterion:** A clearly unsafe task definition is blocked with a friendly message. Admin page successfully updates safety guidelines. Updated guidelines take effect immediately on next reviewer call.

---

### Phase 7 — Polish and Non-Technical UX

Final layer — the app is functionally complete but needs to feel right for a non-technical user.

- Friendly plain English error messages throughout — no technical language visible to Joanna
- Loading indicators on all Claude API calls
- Complexity threshold triggers correctly during task definition
- Last used dates displayed on task cards and task views
- Task summaries and field descriptions reviewed for clarity and tone
- Windows Defender SmartScreen behaviour confirmed and documented for Joanna
- Full end-to-end test on Win10 target machine

**Exit criterion:** Joanna can use the application without assistance. All error states produce friendly, actionable messages. App performs acceptably on the Win10 machine.
