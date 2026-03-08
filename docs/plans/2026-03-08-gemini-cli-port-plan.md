# Gemini CLI Port of Superpowers Implementation Plan

> **For Gemini:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Adapt the `obra/superpowers` repository into a native Gemini CLI extension that leverages `dispatch-harness` for execution.

**Architecture:** Create standard Gemini extension scaffolding (`gemini-extension.json`), implement a `SessionStart` hook for invisible initialization, and rewrite the dynamic subagent prompts to utilize the user's local `dispatch run` command for task execution.

**Tech Stack:** JSON, Bash, Markdown (Gemini CLI Extensions, dispatch-harness)

---

### Task 1: Scaffold Extension Identity

**Files:**
- Create: `gemini-extension.json`
- Modify: `package.json` (Optional, to add basic info if needed, but sticking to extension manifest)

**Step 1: Write the manifest test/validation**
*(Note: As this is a config file, "testing" is ensuring the format is correct for Gemini CLI. We will write the file and then ensure it parses.)*

**Step 2: Write minimal implementation**
Create `gemini-extension.json` with the following content:
```json
{
  "name": "superpowers",
  "version": "4.3.1",
  "description": "Core skills library for Gemini CLI: TDD, debugging, collaboration patterns, and proven techniques",
  "contextFileName": "GEMINI.md",
  "hooks": ".gemini/hooks.json"
}
```

**Step 3: Run test to verify it passes**
Run: `cat gemini-extension.json | jq .`
Expected: Valid JSON output.

**Step 4: Commit**
```bash
git add gemini-extension.json
git commit -m "feat: add Gemini CLI extension manifest"
```

---

### Task 2: Implement Session Initialization Hook

**Files:**
- Create: `.gemini/hooks.json`
- Create: `hooks/gemini-session-start.sh`

**Step 1: Write the minimal implementation for hooks config**
Create `.gemini/hooks.json`:
```json
{
  "SessionStart": [
    {
      "matcher": "*",
      "hooks": [
        {
          "name": "superpowers-intro",
          "type": "command",
          "command": "bash "${extensionPath}/hooks/gemini-session-start.sh""
        }
      ]
    }
  ]
}
```

**Step 2: Write the hook execution script**
Create `hooks/gemini-session-start.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]:-$0}")" && pwd)"
PLUGIN_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

using_superpowers_content=$(cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md" 2>&1 || echo "Error reading using-superpowers skill")

# Escape for JSON
escape_for_json() {
    local s="$1"
    s="${s//\/\}"
    s="${s//"/"}"
    s="${s//$'
'/
}"
    s="${s//$''/}"
    s="${s//$'	'/	}"
    printf '%s' "$s"
}

using_superpowers_escaped=$(escape_for_json "$using_superpowers_content")
session_context="<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'activate_skill' tool:**

${using_superpowers_escaped}
</EXTREMELY_IMPORTANT>"

cat <<EOF
{
  "hookSpecificOutput": {
    "additionalContext": "${session_context}"
  }
}
EOF
exit 0
```

**Step 3: Make script executable and test output**
Run: `chmod +x hooks/gemini-session-start.sh && ./hooks/gemini-session-start.sh | jq .`
Expected: Valid JSON with `hookSpecificOutput.additionalContext` containing the escaped markdown.

**Step 4: Commit**
```bash
git add .gemini/hooks.json hooks/gemini-session-start.sh
git commit -m "feat: implement SessionStart hook for Gemini context injection"
```

---

### Task 3: Adapt Execution Skills for dispatch-harness

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`
- Modify: `skills/dispatching-parallel-agents/SKILL.md`
- Modify: `skills/executing-plans/SKILL.md`
- Modify: `skills/writing-plans/SKILL.md`

**Step 1: Update `subagent-driven-development/SKILL.md`**
Replace references to Claude's `/subagent` with `dispatch run`.
Search for text instructing to "dispatch a subagent" and replace with:
```markdown
For each task, use the `run_shell_command` tool to execute `dispatch-harness` via:
`dispatch run --worker gemini --task-id <task_number> --workdir . "<prompt>"`

The `<prompt>` should contain the exact task instructions, constraints, and the files to modify. Wait for the command to complete and review the output.
```

**Step 2: Update `dispatching-parallel-agents/SKILL.md`**
Modify parallel execution to use background shell commands with `dispatch run`.
```markdown
To execute tasks in parallel, use the `run_shell_command` tool with `is_background: true` for each:
`dispatch run --worker gemini --task-id <id> --workdir . "<prompt>"`
```

**Step 3: Update `executing-plans/SKILL.md`**
Update instructions to reference the `run_shell_command` instead of subagents where applicable, ensuring the primary agent coordinates the execution loop.

**Step 4: Update `writing-plans/SKILL.md`**
Update the "Execution Handoff" section at the bottom.
Replace references to Claude Subagents with Gemini + dispatch-harness.
```markdown
**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Main agent loops through tasks, executing `dispatch run --worker gemini ...` for each.
```

**Step 5: Commit**
```bash
git add skills/subagent-driven-development/SKILL.md skills/dispatching-parallel-agents/SKILL.md skills/executing-plans/SKILL.md skills/writing-plans/SKILL.md
git commit -m "refactor: adapt execution skills to use dispatch-harness instead of native subagents"
```

---

### Task 4: Update Documentation

**Files:**
- Modify: `README.md`

**Step 1: Write modifications to README.md**
Add a Gemini CLI section under Installation.

```markdown
### Gemini CLI

This port utilizes your local `dispatch-harness` to execute complex plans. Ensure `dispatch` is in your PATH.

1. Clone this repository locally or link directly:
```bash
gemini extensions install https://github.com/obra/superpowers.git#gemini-cli-port
```
```

**Step 2: Commit**
```bash
git add README.md
git commit -m "docs: add Gemini CLI installation instructions"
```
