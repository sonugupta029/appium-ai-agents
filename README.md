# Appium AI Agents for Kiro CLI

Two production-ready AI agents for mobile test automation, built for the [Kiro CLI](https://kiro.dev).

---

## Agents

### 🔍 appium-debug-helper
Diagnoses Appium test failures by analyzing logs, HTML reports, and screenshots. Uses a **gated two-phase workflow** — the agent always presents findings and waits for your confirmation before touching any file.

**What it handles:**
- Changed element IDs (XML layout → Jetpack Compose migrations)
- Stale locators and `StaleElementReferenceException`
- `NoSuchElementException` — wrong locator or screen not loaded
- Timing / `TimeoutException` — permission dialogs or slow page loads
- Test data issues — stale accounts, missing platform keys in JSON
- App flow changes — new screens or dialogs inserted in the navigation path
- Platform-specific failures (Android works, iOS fails, or vice versa)
- Appium session errors — device disconnected or server misconfigured

---

### 🏗️ mobile-automation-engineer
Generates Kotlin/Appium Page Object Models (POMs), TestNG test scripts, and reusable flows by reading your existing code patterns. Always shows generated code for review before writing.

**What it generates:**
- Screen classes (POMs) with dual `@AndroidFindBy` / `@iOSXCUITFindBy` locators
- TestNG test scripts following your repo's conventions
- Reusable action flows
- Test data entries in `TestData.json` and `ValidationData.json`

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Kotlin |
| Automation | Appium Java Client |
| Test Runner | TestNG |
| Build | Gradle (Kotlin DSL) |
| Reporting | ExtentReports |
| Logging | Log4j2 |
| Platforms | Android + iOS |

---

## How to Use

### Prerequisites
- [Kiro CLI](https://kiro.dev) installed
- A Kotlin/Appium mobile test automation project

### Setup

1. Clone this repo:
   ```bash
   git clone https://github.com/YOUR_USERNAME/appium-ai-agents.git
   ```

2. Copy the `.kiro` folder into your mobile automation project root:
   ```bash
   cp -r appium-ai-agents/.kiro /path/to/your/project/
   ```

3. Open your project with Kiro CLI:
   ```bash
   cd /path/to/your/project
   kiro chat
   ```

4. Switch to an agent:
   ```
   /agent appium-debug-helper
   ```
   or
   ```
   /agent mobile-automation-engineer
   ```

### Adapt to Your Project

Update the `resources` paths in each agent JSON to point to your actual source directories. Update `Util.appPackage` in the POM template to match your app's package name.

---

## Project Structure

```
.kiro/
├── agents/
│   ├── appium-debug-helper.json          ← agent config, tool access controls
│   └── mobile-automation-engineer.json   ← agent config, tool access controls
│
├── agent-prompts/
│   ├── appium-debug-helper/
│   │   └── appium-debug-helper-prompt.md ← system prompt
│   └── mobile-automation-engineer/
│       └── mobile-automation-engineer.md ← system prompt
│
└── skills/
    └── senior-test-engineer/
        └── SKILL.md                      ← reusable skill loaded at runtime
```

---

## Key Design Decisions

### Gated two-phase workflow (debug agent)
The debug agent **always stops after Phase 1** and presents its analysis before making any changes. This prevents silent fixes — you always see what it found and why before it touches a file.

### `tools` vs `allowedTools` split
Every agent JSON declares two tool lists:
- `tools` — everything the agent can see
- `allowedTools` — what runs without asking for user approval

`write` and `shell` are in `tools` but NOT in `allowedTools`. This means file edits and shell commands always require explicit confirmation.

### Path and command allowlisting
`toolsSettings` restricts what the agent can read, write, and execute:
- Write access: only `src/**` directories — never `.git`, settings, or secrets
- Shell: only read-only git commands and gradle test commands — no commits, pushes, or destructive ops

### Skills as composable context
Skills (`SKILL.md` files) are loaded into agent context at runtime via `skill://` resource references. This keeps reusable knowledge (debugging patterns, review checklists) separate from the agent prompt and shareable across agents without duplication.

---

## Architecture

See [docs/architecture.md](docs/architecture.md) for a detailed breakdown of how the agents are structured and how they interact with the codebase.

---

## License

MIT
