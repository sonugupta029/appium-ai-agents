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
  name           → identifier used to invoke the agent
  description    → shown in agent picker UI
  prompt         → file:// reference to the .md system prompt
  includeMcpJson → injects mcp.json tools (Jira, GitLab, Figma, etc.)
  tools[]        → ALL tools the agent is aware of
  allowedTools[] → tools that run without user approval
  toolsSettings  → per-tool access control (path/command rules)
  resources[]    → files pre-loaded into context at session start
  welcomeMessage → shown when agent is activated
}
```

### tools vs allowedTools

This is the core safety mechanism. Only tools in `allowedTools` run automatically. Everything else requires explicit user approval.

| Tool | appium-debug-helper | mobile-automation-engineer |
|---|---|---|
| read | ✅ auto | ✅ auto |
| glob | ✅ auto | ✅ auto |
| grep | ✅ auto | ✅ auto |
| code | ✅ auto | ✅ auto |
| todo | ✅ auto | ✅ auto |
| introspect | ✅ auto | ✅ auto |
| mcp / @atlassian | not present | ⛔ requires approval |
| @gitlab | not present | ⛔ requires approval |
| @Figma | not present | ⛔ requires approval |
| **write** | ⛔ requires approval | ⛔ requires approval |
| **shell** | ⛔ requires approval | ⛔ requires approval |

The debug agent has `includeMcpJson: false` and no MCP tools in its `tools` list at all — it has no Jira workflow.

---

## toolsSettings — Access Control

### Why deniedPaths instead of allowedPaths

An earlier version used `write.allowedPaths`. This has a risk: in some tool implementations, `allowedPaths` auto-approves matching writes even when `write` is not in `allowedTools`. Using `deniedPaths` only is safer — the denylist entries are absolute blocks regardless of what instructions the agent receives.

### read deniedPaths (both agents)

```json
"deniedPaths": [
  "**/.env", "**/.env.*",
  "**/*.pem", "**/*.key",
  "**/id_rsa*", "**/*credentials*",
  "~/.ssh/**", "~/.aws/**"
]
```

### write deniedPaths

**appium-debug-helper:**
```json
"deniedPaths": [
  ".git/**", ".kiro/**",
  "**/.env", "**/.env.*",
  "**/*.pem", "**/*.key",
  "**/project.properties",
  "**/TestData.json",
  "**/ValidationData.json"
]
```

**mobile-automation-engineer:**
```json
"deniedPaths": [
  ".git/**", ".kiro/**",
  "**/.env", "**/.env.*",
  "**/*.pem", "**/*.key",
  "**/build.gradle.kts", "**/build.gradle",
  "**/.gitlab-ci.yml", "**/gradle/**",
  "**/project.properties", "**/gradle.properties"
]
```

Both agents deny `.kiro/**`, which prevents the agent from overwriting its own config or prompt — **via the write tool**. Shell commands approved by the user are not subject to `write.deniedPaths`.

### shell allowedCommands

Shell commands use anchored regex patterns. `cat`, `head`, `tail`, and `find` are absent — file reading uses the dedicated `read` and `glob` tools, which respect `read.deniedPaths`. Loose wildcards like `find.*` can match destructive commands such as `find . -delete`; anchored patterns prevent this. Path arguments are required to start with `[a-zA-Z0-9._/]` to block arguments beginning with `-`.

**appium-debug-helper:**
```json
"allowedCommands": [
  "^git status$",
  "^git diff( --[a-z-]+)?( [a-zA-Z0-9._/][a-zA-Z0-9._/-]*)?$",
  "^git log( --[a-z-]+)?( [a-zA-Z0-9._/][a-zA-Z0-9._/-]*)?$",
  "^git show [a-zA-Z0-9._/][a-zA-Z0-9._/-]*$",
  "^git grep [a-zA-Z0-9][a-zA-Z0-9 _.-]* -- [a-zA-Z0-9._/][a-zA-Z0-9/_.*-]*$",
  "^ls( -[a-zA-Z]+)? [a-zA-Z0-9._/][a-zA-Z0-9/_.*-]*$"
]
```

**mobile-automation-engineer** adds:
```json
"^[.]/gradlew test --tests [a-zA-Z0-9._][a-zA-Z0-9._$#]+$",
"^[.]/gradlew clean$"
```

The `--tests` pattern requires the class name to start with a letter or digit (no `*`), which prevents `--tests *` from running the whole suite.

Both share the same denylist:
```json
"deniedCommands": [
  "git commit.*", "git push.*", "git reset.*", "git clean.*",
  "rm.*", "mv.*", "chmod.*", "curl.*", "wget.*"
]
```

---

## Resources

Resources are files pre-loaded into the agent's context at session start.

**appium-debug-helper:**
```json
"resources": [
  "file://README.md",
  "file://src/main/kotlin/com/appium/screens/**/*.kt",
  "file://src/main/kotlin/com/appium/reusables/**/*.kt"
]
```

No skills loaded — the debug agent has no Jira workflow and the senior-test-engineer skill is not relevant to it.

**mobile-automation-engineer:**
```json
"resources": [
  "file://README.md",
  "file://build.gradle.kts",
  "file://src/main/kotlin/com/appium/screens/**/*.kt",
  "file://src/main/kotlin/com/appium/reusables/**/*.kt",
  "file://src/test/kotlin/com/appium/tests/**/*.kt",
  "file://src/test/resources/ValidationData.json",
  "file://src/test/resources/testng.xml",
  "skill://.kiro/skills/**/SKILL.md"
]
```

`TestData.json` is deliberately excluded — it may contain test account credentials. The agent is told to ask the user for specific entries when needed.

---

## System Prompt Architecture

### appium-debug-helper prompt structure

```
Identity & objective
│
Project context          ← repo paths, framework, platform conventions
│
Compose locator strategy ← infer from page source evidence first;
│                           testTagsAsResourceId=false means tag is not
│                           exposed to UiAutomator at all; ask user when
│                           no page source is available
│
Common failure patterns  ← 12 failure types; patterns 5 & 6 (test data)
│                           instruct agent to ask user to paste the entry
│                           rather than reading the file directly
│
Diagnosis workflow
├── Phase 1: Analyze     ← read-only, ends with structured output + STOP
│   1. Read log/report
│   2. Inspect screen file locator
│   3. View failure screenshot
│   4. Check page source
│   5. Determine root cause
│   └── Present findings with evidence citations → STOP
│
└── Phase 2: Fix         ← gated behind explicit user approval
    1. Update locator only
    2. Grep for other usages
    3. Flag test-data issues (do not modify config files)
    4. Tell user which test to re-run manually
│
Rules                    ← evidence-before-action (including "do not infer
│                           IDs from other screens"), approval-before-write,
│                           read-before-write, minimal changes, both platforms
│
Output format            ← 🔍 with Evidence block / ✅ with re-run instruction
```

### mobile-automation-engineer prompt structure

```
Identity & scope
│
Tech stack + directory layout
│
POM pattern              ← canonical Kotlin template; primaryElement declared;
│                           TODO_FILL_FROM_INSPECTOR placeholders (not blank strings)
│
Compose section          ← testTagsAsResourceId=true → id = "bare_tag";
│                           false/absent → tag not exposed, ask dev to enable it;
│                           infer from page source before asking user
│
Webview section          ← native nav bar screens do not need setWebviewContext();
│                           ask user which type before generating locators
│
Reusable flow template   ← illustrative; read nearest existing flow first
│
Test script template     ← illustrative; read nearest existing test first;
│                           BaseTestPage is in framework library, not src/main/kotlin
│
Conventions (10 rules)
│
Critical rules
├── Never invent locators — use TODO_FILL_FROM_INSPECTOR
├── Compose: ask about XML vs Compose and testTagsAsResourceId
├── Scroll direction — top-to-bottom flow
├── Check existing POMs before adding elements
└── Read current file state before every edit
│
Process (7 steps)        ← steps 6 and 7 both require explicit approval
│
Locator strategy tables
```

---

## Runtime Data Flow

```
User input
    │
    ▼
Agent session starts:
  - System prompt loaded (agent-prompts/*.md)

  appium-debug-helper resources:
    README.md
    src/main/kotlin/com/appium/screens/**/*.kt
    src/main/kotlin/com/appium/reusables/**/*.kt
    (no skills, no MCP tools)

  mobile-automation-engineer resources:
    README.md, build.gradle.kts
    src/main/kotlin/com/appium/screens/**/*.kt
    src/main/kotlin/com/appium/reusables/**/*.kt
    src/test/kotlin/com/appium/tests/**/*.kt
    src/test/resources/ValidationData.json
    src/test/resources/testng.xml
    skills/**/SKILL.md
    (MCP tools available but require approval)
    │
    ▼
Agent operates:
  read / grep / glob / code / introspect  ──→  auto, no approval
  write / shell                           ──→  blocked until user approves
  mcp / @atlassian / @gitlab / @Figma     ──→  automation agent only; blocked until approved
    │
    ▼
appium-debug-helper:
  Phase 1: reads logs → inspects locators → views screenshots
        → presents 🔍 Failure Analysis with Evidence → STOPS
  Phase 2 (after approval): proposes locator fix → user re-runs test manually

mobile-automation-engineer:
  Reads existing screens → asks for element IDs / Inspector output
  → asks about Compose/XML if relevant
  → generates POM with TODO_FILL_FROM_INSPECTOR placeholders
  → presents for review → writes on approval
  → updates ValidationData.json on separate approval
```

---

## Skills

The `senior-test-engineer` skill is loaded only into the `mobile-automation-engineer` agent. It is not loaded into the debug agent, which has no Jira workflow.

Skills are Markdown files referenced via `skill://`:

```json
"skill://.kiro/skills/**/SKILL.md"
```

The `senior-test-engineer` skill teaches the automation agent to:
- Infer all Jira fields from a casual description
- Search for duplicate tickets before creating
- Link bugs to the QA story being tested (inward) and find the dev story for assignee lookup
- Present a structured preview and never auto-create

---

## Why Two Agents Instead of One

- **Debugging** requires a conservative, evidence-first posture with strong guardrails against guessing. The debug agent never generates new screen classes.
- **Generation** requires a constructive, pattern-matching posture. The automation agent never touches logs or screenshots.

Splitting them gives each agent a focused identity, tighter tool access, and a clearer workflow. It also means the debug agent can have no MCP tools at all, which simplifies its trust surface.
