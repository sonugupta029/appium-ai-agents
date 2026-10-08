# Architecture

A deep-dive into how the two agents are structured, how they enforce safety, and why the design choices were made.

---

## Directory Layout

```
.kiro/
├── agents/                    ← JSON config per agent
├── agent-prompts/             ← Markdown system prompts (one folder per agent)
└── skills/                    ← Reusable skill docs loaded at runtime
```

The agent JSON and the system prompt are intentionally separated. The JSON controls **what the agent can do** (tools, access controls, resources). The Markdown controls **how the agent thinks** (identity, workflow, rules). This makes each independently editable without touching the other.

---

## Agent Config Schema

Each agent JSON follows this structure:

```
{
  name          → identifier used to invoke the agent
  description   → shown in agent picker UI
  prompt        → file:// reference to the .md system prompt
  includeMcpJson→ injects mcp.json tools (Jira, GitLab, Figma, etc.)
  tools[]       → ALL tools the agent is aware of
  allowedTools[]→ tools that run without user approval
  toolsSettings → per-tool access control (path/command allowlists)
  resources[]   → files pre-loaded into context at session start
  welcomeMessage→ shown when agent is activated
}
```

### tools vs allowedTools

This is the core safety mechanism. Every agent sees the same full tool list but only a subset runs automatically:

| Tool | appium-debug-helper | mobile-automation-engineer |
|---|---|---|
| read | ✅ auto | ✅ auto |
| glob | ✅ auto | ✅ auto |
| grep | ✅ auto | ✅ auto |
| code | ✅ auto | ✅ auto |
| todo | ✅ auto | ✅ auto |
| mcp / @atlassian | ✅ auto | ✅ auto |
| @gitlab | — | ✅ auto |
| @Figma | — | ✅ auto |
| **write** | ⛔ requires approval | ⛔ requires approval |
| **shell** | ⛔ requires approval | ⛔ requires approval |

Read operations are fully autonomous. Write and shell always pause for confirmation.

---

## toolsSettings — Access Control

### write allowlists

**appium-debug-helper** — only fixes source files, never touches config or agent definitions:
```json
"allowedPaths": [
  "src/main/kotlin/**",
  "src/test/kotlin/**",
  "src/test/resources/**",
  "src/main/resources/**"
],
"deniedPaths": [
  ".git/**",
  ".kiro/agents/**",
  ".kiro/settings/**"
]
```

**mobile-automation-engineer** — writes only to screen, reusable, test, and resource files:
```json
"allowedPaths": [
  "**/screens/**/*.kt",
  "**/reusables/**/*.kt",
  "**/tests/**/*.kt",
  "**/resources/*.xml",
  "**/resources/*.json"
]
```

### shell allowlists

Both agents restrict shell to read-only git commands and (for the automation agent) gradle test commands. Destructive operations are explicitly blocked:

```json
"deniedCommands": [
  "git commit.*",
  "git push.*",
  "git reset --hard.*",
  "git clean -fd.*",
  "rm -rf.*"
]
```

---

## System Prompt Architecture

### appium-debug-helper prompt structure

```
Identity & objective
│
Project context          ← repo paths, framework, platform conventions
│
Common failure patterns  ← 12 enumerated failure types (changed ID, timing,
│                           Compose migration, session errors, etc.)
│
Diagnosis workflow
├── Phase 1: Analyze     ← read-only tools only
│   1. Read log/report
│   2. Inspect screen file locator
│   3. View failure screenshot
│   4. Check page source
│   5. Determine root cause
│   └── Present findings → STOP (do not proceed without confirmation)
│
└── Phase 2: Fix         ← gated behind explicit user approval
    1. Update locator
    2. Grep for other usages
    3. Handle timing/test-data edge cases
    4. Summarize changes
│
Rules                    ← hard constraints (approval-before-write,
│                           never guess IDs, read-before-write, etc.)
│
Output format templates  ← structured 🔍 / ✅ blocks
```

The **STOP instruction** after Phase 1 is intentional. Without it, an agent will naturally flow from analysis into fixes. This forces an explicit checkpoint that gives the engineer control over every change.

### mobile-automation-engineer prompt structure

```
Identity & scope
│
Tech stack + directory layout
│
POM pattern              ← canonical Kotlin template the agent must follow
│
Conventions (10 rules)   ← dual locators, nullable elements, Boolean returns,
│                           logging pattern, platform branching, etc.
│
Critical rules
├── Never invent locators — always ask for Appium Inspector output
├── Scroll direction — top-to-bottom flow, no back-and-forth
├── Check existing POMs before adding elements
└── Read current file state before every edit
│
Process (7 steps)        ← fetch Jira → read existing screens → ask for IDs
│                           → generate → show for review → write → update data
│
Locator strategy tables  ← iOS and Android strategy selection guide
```

---

## Runtime Data Flow

```
User input
    │
    ▼
Agent session starts:
  - System prompt loaded (agent-prompts/*.md)
  - Resources injected into context:
      README.md
      src/**/*.kt  (screen files, reusables)
      ValidationData.json
      skills/**/SKILL.md
  - MCP tools available (@atlassian, @gitlab, @Figma)
    │
    ▼
Agent operates:
  read / grep / glob / code  ──→  always available, no approval needed
  write / shell              ──→  blocked until user explicitly approves
  mcp / @atlassian           ──→  Jira ticket fetch, Confluence lookup
    │
    ▼
appium-debug-helper:
  Phase 1: reads logs → inspects locators → views screenshots
        → presents 🔍 Failure Analysis → STOPS
  Phase 2 (after approval): applies fix → summarizes ✅

mobile-automation-engineer:
  Reads existing screens → asks for element IDs
  → generates POM → presents for review → writes on approval
  → updates TestData.json / ValidationData.json
```

---

## Skills

Skills are Markdown files loaded into agent context via `skill://` resource references:

```json
"skill://.kiro/skills/**/SKILL.md"
```

They contain reusable procedural knowledge — debugging patterns, review checklists, platform-specific conventions — that the agent follows when a task matches. This keeps the system prompt lean while allowing deep specialization per task type.

The `senior-test-engineer` skill provides additional guidance on:
- Test design patterns
- When to add explicit waits vs global timeouts
- Assertion best practices for mobile UI testing

---

## Why Two Agents Instead of One

A single "mobile test automation" agent would need to handle both debugging and generation, leading to conflicting postures:

- **Debugging** requires a conservative, evidence-first, read-heavy posture with strong guardrails against guessing
- **Generation** requires a constructive, pattern-matching posture that reads existing code and produces new code

Splitting them into two agents means each has a focused identity, tighter tool access, and a clearer workflow. The debug agent never generates new screen classes; the automation agent never touches logs or screenshots.
