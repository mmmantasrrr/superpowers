# Gemini CLI Port of Superpowers Implementation Design

**Goal:** Adapt the `obra/superpowers` repository to function natively within the Gemini CLI environment, preserving its autonomous workflow while replacing Claude's dynamic subagents with the local `dispatch-harness` system.

**Architecture:**
The extension will be structured as a standard, distributable Gemini CLI Extension. It will run alongside existing integrations (Claude, Codex) by utilizing Gemini-specific configuration files while sharing the core `skills/` directory.

---

### Component Specifications

#### 1. Extension Skeleton & Discovery
*   **Manifest:** A new `gemini-extension.json` file will be created in the repository root to define the extension package for Gemini.
*   **Skill Discovery:** The existing `skills/` directory will remain unmodified (mostly). Gemini CLI's native `activate_skill` tool will automatically discover these directories and read the `SKILL.md` frontmatter, providing identical autonomous discovery to the Claude version.

#### 2. Session Initialization Hook
*   **Purpose:** To silently inject the `using-superpowers` skill into the agent's context when a session starts, mimicking Claude's `SessionStart` hook.
*   **Configuration:** A new directory `.gemini/` will be created containing `hooks.json`. This file will map the `SessionStart` event to our custom script.
*   **Execution Script:** A new script `hooks/gemini-session-start.sh` will be written. It will read `skills/using-superpowers/SKILL.md` and output a JSON payload formatted specifically for Gemini CLI (`{"hookSpecificOutput": {"additionalContext": "..."}}` or similar depending on hook requirements, ensuring it outputs valid JSON to stdout).

#### 3. Execution Engine Integration (`dispatch-harness`)
*   **Purpose:** To replace Claude's native `/subagent` capabilities for complex planning and parallel execution with a robust local alternative.
*   **Modifications:** The prompt instructions within specific execution skills (e.g., `subagent-driven-development/SKILL.md`, `dispatching-parallel-agents/SKILL.md`) will be rewritten.
*   **New Workflow:** Instead of instructing the primary agent to "dispatch a subagent," the skills will instruct the primary agent to use its native `run_shell_command` tool to execute `dispatch run --worker gemini --task-id <id> --workdir . "<prompt>"`. This leverages the user's existing `dispatch-harness` to spawn headless background processes for individual tasks.

#### 4. Documentation
*   **README Updates:** The `README.md` will be updated to include a new "Gemini CLI" section detailing installation via `gemini extensions install` and outlining the prerequisite of having `dispatch-harness` installed for complex workflows.

---

### Implementation Plan (Next Steps)
1. Initialize extension files (`gemini-extension.json`, `.gemini/hooks.json`).
2. Write the `gemini-session-start.sh` script to handle context injection.
3. Modify the relevant `SKILL.md` files to route execution through `dispatch run`.
4. Update `README.md`.
5. Commit changes to the `gemini-cli-port` branch.
