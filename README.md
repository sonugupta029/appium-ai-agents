# Appium AI Agents for Kiro CLI

A reference implementation of two AI agents for mobile test automation, built for the [Kiro CLI](https://kiro.dev).

---

## Agents

### 🔍 appium-debug-helper
Diagnoses Appium test failures by analyzing logs, HTML reports, and screenshots. Uses a **gated two-phase workflow** — the agent always presents findings with cited evidence and waits for your confirmation before touching any file.

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
   git clone https://github.com/sonugupta029/appium-ai-agents.git
   ```

2. Copy the `.kiro` folder into your mobile automation project root:
   ```bash
   cp -r appium-ai-agents/.kiro /path/to/your/project/
   ```

3. Open your project with Kiro CLI:
   ```bash
   cd /path/to/your/project
   kiro-cli chat
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

Update the `resources` paths in each agent JSON to point to your actual source directories. Update `Util.appPackage` in the POM template to match your app's package name. Set your Jira project key and bug link type in `skills/senior-test-engineer/SKILL.md`.

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
        └── SKILL.md                      ← Jira ticket creation skill (automation agent only)
```

---

## Key Design Decisions

### Gated two-phase workflow (debug agent)
The debug agent always stops after Phase 1 and presents its analysis — with cited evidence — before proposing any change. You see what it found, where it found it, and why before anything is written.

### `tools` vs `allowedTools` split
Every agent JSON declares two tool lists:
- `tools` — everything the agent is aware of
- `allowedTools` — what runs without asking for user approval

Both agents auto-approve only: `read`, `glob`, `grep`, `code`, `todo`, `introspect`. Everything else — `write`, `shell`, and all MCP tools — requires explicit confirmation. The debug agent has no MCP tools at all (`includeMcpJson: false`).

### `deniedPaths` instead of `allowedPaths`
Write access is controlled via a denylist rather than an allowlist. `allowedPaths` can auto-approve matching writes in some tool implementations even when `write` is not in `allowedTools`. A denylist with `.kiro/**` blocked means the agent cannot overwrite its own config via the write tool. Note: shell commands approved by the user are not subject to `write.deniedPaths` — the denylist only applies to the write tool.

### Anchored shell commands
Shell `allowedCommands` use anchored regex patterns (`^...$`) rather than loose wildcards. `cat`, `head`, `tail`, and `find` are absent — file reading uses the dedicated `read` and `glob` tools instead, which respect `read.deniedPaths`.

### Skills loaded selectively
The `senior-test-engineer` skill (Jira ticket creation) is loaded only into the automation agent. The debug agent has no Jira workflow and does not load it.

---

## Architecture

See [docs/architecture.md](docs/architecture.md) for a detailed breakdown of the agent configs, tool access controls, and runtime data flow.

---

## License

Apache 2.0
