---
description: "Approve the current PRD and begin parallel story execution. Analyzes dependencies, creates execution batches, launches background Task agents for each story, and monitors progress. Usage: /plan-cascade:approve [--flow <quick|standard|full>] [--tdd <off|on|auto>] [--confirm] [--no-confirm] [--agent <name>] [--impl-agent <name>] [--retry-agent <name>] [--no-verify] [--verify-agent <name>] [--no-review] [--no-fallback] [--auto-run]"
---

# Hybrid Ralph - Approve PRD and Execute

You are approving the PRD and starting parallel execution of user stories.

## Execution Flow Parameters

This command accepts flow control parameters that affect quality gates and execution:

### Parameter Priority

Parameters can be specified in three places. When the same parameter is defined in multiple sources, the following priority order applies:

1. **Command-line flags** (highest priority)
   - Example: `/plan-cascade:approve --flow full`
   - Overrides all other sources

2. **PRD configuration** (prd.json)
   - `flow_config.level`, `tdd_config.mode`, `execution_config.require_batch_confirm`
   - Used when no command-line flag is provided

3. **Default values** (lowest priority)
   - Used when no command-line flag or PRD config is present

**Example Priority Resolution:**
```bash
# Scenario 1: Command-line overrides PRD
prd.json contains: flow_config.level = "standard"
You run: /plan-cascade:approve --flow full
Result: FLOW = "full" (command-line wins)

# Scenario 2: PRD config used when no command-line flag
prd.json contains: tdd_config.mode = "on"
You run: /plan-cascade:approve
Result: TDD = "on" (from PRD)

# Scenario 3: Default used when neither specified
prd.json has no tdd_config
You run: /plan-cascade:approve
Result: TDD = "auto" (default)
```

**Debugging Tip:** The command displays resolved parameter values at Step 2.1, showing which source each parameter came from.

### `--flow <quick|standard|full>`

Override the execution flow depth. This controls quality gate strictness.

| Flow | Gate Mode | AI Verification | Code Review | Test Enforcement |
|------|-----------|-----------------|-------------|------------------|
| `quick` | soft (warnings) | **disabled** | no | no |
| `standard` | soft (warnings) | enabled | no | no |
| `full` | **hard (blocking)** | enabled | **required** | **required** |

**FULL Flow Enforcement:**
- Quality gates BLOCK execution on failure (not just warn)
- Code review is REQUIRED after each story
- Test file changes are REQUIRED alongside code changes
- Batch confirmation is shown before each batch

### `--tdd <off|on|auto>`

Control Test-Driven Development mode for story execution.

| Mode | Description |
|------|-------------|
| `off` | TDD disabled, no compliance checks |
| `on` | TDD enabled with prompts and compliance gate |
| `auto` | Automatically enable TDD for high-risk stories (default) |

### `--confirm`

Require explicit user confirmation before starting each batch execution.

### `--no-confirm`

Explicitly disable batch confirmation, even if FULL flow would normally require it.

This is useful for:
- CI/CD pipelines where interactive confirmation is not possible
- Automated testing environments
- When you want strict quality gates but uninterrupted execution

**Note**: `--no-confirm` only affects batch-level confirmation. It does NOT disable quality gates (verification, code review, TDD compliance) - those still run and can block on failures in FULL flow.

**Precedence**: `--no-confirm` overrides `--confirm`, PRD config, and FULL flow's default confirmation requirement.

## Execution Modes

The approve command supports three execution modes:

### Mode 1: Auto Mode (Default in traditional flow)
- Automatically progresses through batches
- Pauses only on errors requiring manual intervention
- Uses Claude-based task agents for story execution

### Mode 2: Manual Mode
- Requires user approval before each batch
- Full control and review capability
- Best for high-risk or learning scenarios

### Mode 3: Full Auto Mode (Recommended)
- **Python-based execution with automatic retry**
- Quality gates run automatically with auto-retry on failures
- Up to 3 retry attempts per story with exponential backoff
- Failure context injected into retry prompts
- Best for CI/CD and unattended execution

**Full Auto Mode Features:**
| Feature | Description |
|---------|-------------|
| Auto Retry | Failed stories automatically retry up to 3 times |
| Failure Context | Error details and suggested fixes added to retry prompts |
| Exponential Backoff | 5s → 10s → 20s → ... delays between retries |
| Agent Switching | Optional: switch to `--retry-agent` on failures |
| State Persistence | Progress saved to `.iteration-state.json` for recovery |

To use Full Auto mode, select option `[3]` when prompted for execution mode, or run:
```bash
uv run python scripts/auto-execute.py --prd prd.json --flow full --tdd on
```

## Path Storage Modes

This command works with both new and legacy path storage modes:

### New Mode (Default)
- PRD file: In worktree directory or `~/.plan-cascade/<project-id>/prd.json`
- State files: `~/.plan-cascade/<project-id>/.state/`
- Agent outputs: In worktree or project root `.agent-outputs/`

### Legacy Mode
- PRD file: In worktree or project root `prd.json`
- State files: In project root
- Agent outputs: In project root `.agent-outputs/`

User-visible files (`progress.txt`, `findings.md`) always remain in the working directory.

## Multi-Agent Collaboration

This command supports multiple AI agents for story execution. The system automatically selects the best agent based on:

1. Command-line parameters (highest priority)
2. Story-level `agent` field in PRD
3. Story type inference (bugfix→codex, refactor→aider)
4. Phase defaults from agents.json
5. Fallback to claude-code (always available)

### Supported Agents

| Agent | Type | Best For |
|-------|------|----------|
| `claude-code` | task-tool | General purpose (default, always available) |
| `codex` | cli | Bug fixes, quick implementations |
| `aider` | cli | Refactoring, code improvements |
| `amp-code` | cli | Alternative implementations |
| `cursor-cli` | cli | IDE-integrated tasks |

### Command Parameters

```
--agent <name>       Global agent override (all stories use this agent)
--impl-agent <name>  Agent for implementation phase
--retry-agent <name> Agent for retry phase (after failures)
--no-fallback        Disable automatic fallback to claude-code
--auto-run           Start execution immediately after approval
--no-verify          Disable AI verification gate (enabled by default)
--verify-agent <name> Agent for verification (default: claude-code task-tool)
--no-review          Disable AI code review (enabled by default)
--review-agent <name> Agent for code review (default: claude-code task-tool)
```

## Tool Usage Policy (CRITICAL)

**To avoid command confirmation prompts during automatic execution:**

1. **Use Read tool for file reading** - NEVER use `cat` via Bash
   - ✅ `Read("prd.json")`, `Read("progress.txt")`
   - ❌ `Bash("cat prd.json")`

2. **Use Grep tool for content search** - NEVER use `grep` via Bash
   - ✅ `Grep("[COMPLETE]", path="progress.txt")`
   - ❌ `Bash("grep -c '[COMPLETE]' progress.txt")`

3. **Only use Bash for actual system commands:**
   - Git operations
   - File writing: `echo "..." >> progress.txt`
   - Running tests or build commands

4. **For monitoring loops:** Use Read tool to poll `progress.txt`, then parse the content in your response to count markers

## Step 1: Detect Operating System and Shell

Detect the current operating system to use appropriate commands:

```bash
# Detect OS
OS_TYPE="$(uname -s 2>/dev/null || echo Windows)"
case "$OS_TYPE" in
    Linux*|Darwin*|MINGW*|MSYS*)
        SHELL_TYPE="bash"
        echo "✓ Detected Unix-like environment (bash)"
        ;;
    *)
        # Check if PowerShell is available on Windows
        if command -v pwsh >/dev/null 2>&1 || command -v powershell >/dev/null 2>&1; then
            SHELL_TYPE="powershell"
            echo "✓ Detected Windows environment (PowerShell)"
        else
            SHELL_TYPE="bash"
            echo "✓ Using bash (default)"
        fi
        ;;
esac
```

## Step 2: Parse Parameters

Parse all parameters including flow control and agent settings:

```bash
# Parse arguments
GLOBAL_AGENT=""
IMPL_AGENT=""
RETRY_AGENT=""
VERIFY_AGENT=""
REVIEW_AGENT=""
NO_FALLBACK=false
AUTO_RUN=false

# Flow control parameters
FLOW_LEVEL=""           # --flow <quick|standard|full>
TDD_MODE=""             # --tdd <off|on|auto>
CONFIRM_MODE=false      # --confirm
NO_CONFIRM_MODE=false   # --no-confirm (overrides --confirm and FULL flow default)
CONFIRM_EXPLICIT=false  # set when --confirm is provided
NO_CONFIRM_EXPLICIT=false  # set when --no-confirm is provided

# Quality gate defaults (may be overridden by flow level)
ENABLE_VERIFY=true      # Default: enabled
ENABLE_REVIEW=true      # Default: enabled
GATE_MODE="soft"        # Default: soft (warnings only)
REQUIRE_REVIEW=false    # Default: not required
ENFORCE_TEST_CHANGES=false  # Default: not enforced

NEXT_IS_FLOW=false
NEXT_IS_TDD=false
NEXT_IS_AGENT=false
NEXT_IS_IMPL_AGENT=false
NEXT_IS_RETRY_AGENT=false
NEXT_IS_VERIFY_AGENT=false
NEXT_IS_REVIEW_AGENT=false

for arg in $ARGUMENTS; do
    case "$arg" in
        # Flow control flags
        --flow=*) FLOW_LEVEL="${arg#*=}" ;;
        --flow) NEXT_IS_FLOW=true ;;
        --tdd=*) TDD_MODE="${arg#*=}" ;;
        --tdd) NEXT_IS_TDD=true ;;
        --confirm) CONFIRM_MODE=true; CONFIRM_EXPLICIT=true ;;
        --no-confirm) NO_CONFIRM_MODE=true; NO_CONFIRM_EXPLICIT=true ;;
        # Agent flags
        --agent=*) GLOBAL_AGENT="${arg#*=}" ;;
        --agent) NEXT_IS_AGENT=true ;;
        --impl-agent=*) IMPL_AGENT="${arg#*=}" ;;
        --impl-agent) NEXT_IS_IMPL_AGENT=true ;;
        --retry-agent=*) RETRY_AGENT="${arg#*=}" ;;
        --retry-agent) NEXT_IS_RETRY_AGENT=true ;;
        --verify-agent=*) VERIFY_AGENT="${arg#*=}" ;;
        --verify-agent) NEXT_IS_VERIFY_AGENT=true ;;
        --review-agent=*) REVIEW_AGENT="${arg#*=}" ;;
        --review-agent) NEXT_IS_REVIEW_AGENT=true ;;
        # Other flags
        --no-fallback) NO_FALLBACK=true ;;
        --auto-run) AUTO_RUN=true ;;
        --no-verify) ENABLE_VERIFY=false ;;
        --no-review) ENABLE_REVIEW=false ;;
        *)
            # Handle space-separated flag values
            if [ "$NEXT_IS_FLOW" = true ]; then
                FLOW_LEVEL="$arg"
                NEXT_IS_FLOW=false
            elif [ "$NEXT_IS_TDD" = true ]; then
                TDD_MODE="$arg"
                NEXT_IS_TDD=false
            elif [ "$NEXT_IS_AGENT" = true ]; then
                GLOBAL_AGENT="$arg"
                NEXT_IS_AGENT=false
            elif [ "$NEXT_IS_IMPL_AGENT" = true ]; then
                IMPL_AGENT="$arg"
                NEXT_IS_IMPL_AGENT=false
            elif [ "$NEXT_IS_RETRY_AGENT" = true ]; then
                RETRY_AGENT="$arg"
                NEXT_IS_RETRY_AGENT=false
            elif [ "$NEXT_IS_VERIFY_AGENT" = true ]; then
                VERIFY_AGENT="$arg"
                NEXT_IS_VERIFY_AGENT=false
            elif [ "$NEXT_IS_REVIEW_AGENT" = true ]; then
                REVIEW_AGENT="$arg"
                NEXT_IS_REVIEW_AGENT=false
            fi
            ;;
    esac
done
```

### 2.1: Apply Flow Configuration

**CRITICAL**: Apply flow level settings to quality gates.

```
# Apply flow level configuration
If FLOW_LEVEL == "quick":
    GATE_MODE = "soft"
    ENABLE_VERIFY = false       # Disable AI verification
    ENABLE_REVIEW = false       # Disable code review
    REQUIRE_REVIEW = false
    ENFORCE_TEST_CHANGES = false
    echo "Flow Level: QUICK - Minimal gating, fast execution"

Elif FLOW_LEVEL == "full":
    GATE_MODE = "hard"          # Blocking gates
    ENABLE_VERIFY = true        # Enable AI verification
    ENABLE_REVIEW = true        # Enable code review
    REQUIRE_REVIEW = true       # REQUIRED, not optional
    ENFORCE_TEST_CHANGES = true # Require test changes with code
    # --no-confirm overrides FULL flow's default confirmation
    If NO_CONFIRM_MODE is true:
        CONFIRM_MODE = false    # Explicitly disabled
        echo "Flow Level: FULL - Strict gating (batch confirm DISABLED by --no-confirm)"
    Elif CONFIRM_EXPLICIT is false:
        CONFIRM_MODE = true     # Default to confirm for full flow
        echo "Flow Level: FULL - Strict gating, all quality checks required"
    Else:
        echo "Flow Level: FULL - Strict gating, all quality checks required"

Elif FLOW_LEVEL == "standard" or FLOW_LEVEL is empty:
    GATE_MODE = "soft"
    ENABLE_VERIFY = true        # Enable AI verification
    ENABLE_REVIEW = true        # Enable code review (optional)
    REQUIRE_REVIEW = false      # Not required
    ENFORCE_TEST_CHANGES = false
    echo "Flow Level: STANDARD - Balanced gating"

# Command-line flags can still override flow settings
# --no-verify overrides even full flow
# --no-review overrides even full flow
```

Display configuration with parameter sources:
```
============================================================
EXECUTION CONFIGURATION (with sources)
============================================================
Flow Level: ${FLOW_LEVEL:-"standard"}
  Source: ${FLOW_LEVEL_SOURCE:-"default"}
  Gate Mode: ${GATE_MODE}

TDD Mode: ${TDD_MODE:-"auto"}
  Source: ${TDD_MODE_SOURCE:-"default"}

Batch Confirm: ${NO_CONFIRM_MODE ? "DISABLED (--no-confirm)" : (CONFIRM_MODE ? "enabled" : "disabled")}
  Source: ${CONFIRM_SOURCE:-"default"}

Quality Gates:
  AI Verification: ${ENABLE_VERIFY ? "enabled" : "disabled"}
    ${NO_VERIFY_EXPLICIT ? "(disabled by --no-verify)" : ""}
  Code Review: ${ENABLE_REVIEW ? "enabled" : "disabled"}${REQUIRE_REVIEW ? " (REQUIRED)" : ""}
    ${NO_REVIEW_EXPLICIT ? "(disabled by --no-review)" : ""}
  Test Enforcement: ${ENFORCE_TEST_CHANGES ? "enabled" : "disabled"}

Agent Configuration:
  Global Override: ${GLOBAL_AGENT:-"none (use priority chain)"}
  Implementation: ${IMPL_AGENT:-"per-story resolution"}
  Retry: ${RETRY_AGENT:-"per-story resolution"}
  Verify: ${VERIFY_AGENT:-"claude-code (default)"}
  Review: ${REVIEW_AGENT:-"claude-code (default)"}
  Fallback: ${NO_FALLBACK ? "disabled" : "enabled"}

Parameter Sources Legend:
  [CLI]     - Command-line flag (highest priority)
  [PRD]     - prd.json configuration
  [DEFAULT] - Built-in default value
============================================================
```

**Parameter Source Tracking (for debugging):**

Track the source of each parameter as it's resolved:

```bash
# Track flow level source
If FLOW_LEVEL set from command-line:
    FLOW_LEVEL_SOURCE="CLI (--flow ${FLOW_LEVEL})"
Elif FLOW_LEVEL set from prd.json:
    FLOW_LEVEL_SOURCE="PRD (flow_config.level)"
Else:
    FLOW_LEVEL_SOURCE="DEFAULT"

# Track TDD mode source
If TDD_MODE set from command-line:
    TDD_MODE_SOURCE="CLI (--tdd ${TDD_MODE})"
Elif TDD_MODE set from prd.json:
    TDD_MODE_SOURCE="PRD (tdd_config.mode)"
Else:
    TDD_MODE_SOURCE="DEFAULT"

# Track confirm mode source
If NO_CONFIRM_EXPLICIT:
    CONFIRM_SOURCE="CLI (--no-confirm)"
Elif CONFIRM_EXPLICIT:
    CONFIRM_SOURCE="CLI (--confirm)"
Elif CONFIRM_MODE set from prd.json:
    CONFIRM_SOURCE="PRD (execution_config.require_batch_confirm)"
Elif FLOW_LEVEL == "full":
    CONFIRM_SOURCE="FULL flow default"
Else:
    CONFIRM_SOURCE="DEFAULT (disabled)"

# Track gate override flags
NO_VERIFY_EXPLICIT = (--no-verify flag provided)
NO_REVIEW_EXPLICIT = (--no-review flag provided)
```

## Step 2.5: Load Agent Configuration

Read `agents.json` if it exists to get agent definitions and phase defaults:

```
If agents.json exists:
    Load agent configuration:
    - agents: Map of agent_name → {type, command, args, ...}
    - phase_defaults: {implementation: {...}, retry: {...}, ...}
    - story_type_defaults: {bugfix: "codex", refactor: "aider", ...}
Else:
    Use default: claude-code only
```

## Step 3: Ensure Auto-Approval Configuration

Ensure command auto-approval settings are configured (merges with existing settings):

```bash
# Run the settings merge script from project root
uv run python "${CLAUDE_PLUGIN_ROOT}/scripts/ensure-settings.py" 2>/dev/null || uv run python scripts/ensure-settings.py 2>/dev/null || uv run python ../scripts/ensure-settings.py 2>/dev/null || echo "Warning: Could not update settings, continuing..."
```

This script intelligently merges required auto-approval patterns with any existing `.claude/settings.local.json`, preserving user customizations.

## Step 3: Verify PRD Exists

Check if `prd.json` exists:

```bash
if [ ! -f "prd.json" ]; then
    echo "ERROR: No PRD found. Please generate one first with:"
    echo "  /plan-cascade:hybrid-auto <description>"
    echo "  /plan-cascade:hybrid-manual <path>"
    exit 1
fi
```

## Step 4: Read and Validate PRD

Read `prd.json` and validate:
- Has `metadata`, `goal`, `objectives`, `stories`
- Each story has `id`, `title`, `description`, `priority`, `dependencies`, `acceptance_criteria`
- All dependency references exist

If validation fails, show errors and suggest `/plan-cascade:edit`.

## Step 4.0.5: Definition of Ready (DoR) Gate

**CRITICAL**: Validate PRD meets readiness criteria before execution.

```bash
echo "Running DoR gate..."
FLOW_LEVEL="${FLOW_LEVEL:-standard}" \
CLAUDE_PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT}" \
uv run python << 'PYTHON_EOF'
import sys
import json
import os
from pathlib import Path

# Setup path - try multiple locations
plugin_root = os.environ.get("CLAUDE_PLUGIN_ROOT") or ""
plugin_root_candidates = [Path(plugin_root) if plugin_root else None, Path.cwd(), Path.cwd().parent]

for candidate in plugin_root_candidates:
    if candidate and (candidate / "src").exists():
        sys.path.insert(0, str(candidate / "src"))
        break

try:
    from plan_cascade.core.readiness_gate import ReadinessGate, GateMode
except ImportError:
    print("[DoR] ReadinessGate not available, skipping (install plan-cascade)")
    sys.exit(0)

# Load PRD
prd_path = Path("prd.json")
if not prd_path.exists():
    print("[DoR] No prd.json found")
    sys.exit(1)

with open(prd_path) as f:
    prd = json.load(f)

# Create gate based on flow level
flow = os.environ.get("FLOW_LEVEL") or "standard"
mode = GateMode.HARD if flow == "full" else GateMode.SOFT
gate = ReadinessGate(mode=mode)

# Run checks
result = gate.check_prd(prd)
print(result.get_summary())

if not result.passed:
    print("\n[DoR_FAILED] PRD readiness check failed")
    if mode == GateMode.HARD:
        sys.exit(1)
    else:
        print("[DoR_WARNING] Continuing in soft mode...")
elif result.warnings:
    print("\n[DoR_WARNING] PRD has warnings (soft mode, continuing)")
else:
    print("\n[DoR_PASSED] PRD meets readiness criteria")
PYTHON_EOF

DOR_EXIT=$?
if [ $DOR_EXIT -ne 0 ] && [ "${GATE_MODE}" == "hard" ]; then
    echo "DoR Gate blocked execution. Fix PRD with: /plan-cascade:edit"
    exit 1
fi
```

## Step 4.1: Apply PRD Flow and Quality Gate Configuration

After reading prd.json, check for flow/tdd/gate settings. **Priority order: command-line > PRD config > defaults**.

```
prd_content = Read("prd.json")
prd = parse_json(prd_content)

# 1. Check PRD-level flow configuration (from hybrid-auto/worktree)
# Only apply if command-line --flow was NOT specified
If FLOW_LEVEL is empty AND prd has "flow_config" field:
    prd_flow = prd.flow_config.level
    FLOW_LEVEL = prd_flow
    echo "Note: Using flow level from PRD: ${prd_flow}"

    # Apply flow settings (same logic as Step 2.1)
    If prd_flow == "quick":
        GATE_MODE = "soft"
        ENABLE_VERIFY = false
        ENABLE_REVIEW = false
        REQUIRE_REVIEW = false
        ENFORCE_TEST_CHANGES = false
    Elif prd_flow == "full":
        GATE_MODE = "hard"
        ENABLE_VERIFY = true
        ENABLE_REVIEW = true
        REQUIRE_REVIEW = true
        ENFORCE_TEST_CHANGES = true
        If NO_CONFIRM_MODE is false AND CONFIRM_EXPLICIT is false:
            CONFIRM_MODE = true
    Elif prd_flow == "standard":
        GATE_MODE = "soft"
        ENABLE_VERIFY = true
        ENABLE_REVIEW = true
        REQUIRE_REVIEW = false
        ENFORCE_TEST_CHANGES = false

# 2. Check PRD-level TDD configuration
# Only apply if command-line --tdd was NOT specified
If TDD_MODE is empty AND prd has "tdd_config" field:
    TDD_MODE = prd.tdd_config.mode  # "off", "on", or "auto"
    echo "Note: Using TDD mode from PRD: ${TDD_MODE}"

# 3. Check PRD-level execution config (for confirm mode)
# --no-confirm from command line takes absolute precedence
If NO_CONFIRM_MODE is true:
    CONFIRM_MODE = false
    echo "Note: Batch confirmation DISABLED by --no-confirm flag"
# If user explicitly requested --confirm, respect it (do not override from PRD)
Elif CONFIRM_EXPLICIT is true:
    CONFIRM_MODE = true
    echo "Note: Batch confirmation enabled by --confirm flag"
# Check if PRD has no_confirm_override (from hybrid-auto --no-confirm)
Elif prd has "execution_config" AND prd.execution_config.no_confirm_override == true:
    CONFIRM_MODE = false
    NO_CONFIRM_MODE = true  # Mark as explicitly disabled
    echo "Note: Batch confirmation DISABLED by PRD config (--no-confirm)"
# Otherwise check if PRD enables confirm
Elif CONFIRM_MODE is false AND prd has "execution_config" field:
    If prd.execution_config.require_batch_confirm == true:
        CONFIRM_MODE = true
        echo "Note: Batch confirmation enabled by PRD config"

# 4. Check PRD-level verification gate configuration
# PRD config can DISABLE gates (but cannot override command-line)
If prd has "verification_gate" field:
    If prd.verification_gate.enabled == false:
        ENABLE_VERIFY = false
        echo "Note: AI Verification disabled by PRD config"

# 5. Check PRD-level code review configuration
If prd has "code_review" field:
    If prd.code_review.enabled == false:
        ENABLE_REVIEW = false
        echo "Note: AI Code Review disabled by PRD config"

# Command-line flags take final precedence (already parsed in Step 2)
# --no-verify overrides PRD config to disable
# --no-review overrides PRD config to disable
```

Display final quality gate configuration:
```
============================================================
FINAL EXECUTION CONFIGURATION
============================================================
Flow Level: ${FLOW_LEVEL:-"standard"} (source: ${flow_source})
Gate Mode: ${GATE_MODE}
TDD Mode: ${TDD_MODE:-"auto"}
Batch Confirm: ${NO_CONFIRM_MODE ? "DISABLED (--no-confirm)" : (CONFIRM_MODE ? "enabled" : "disabled")}

Quality Gates:
  AI Verification: ${ENABLE_VERIFY ? "enabled" : "disabled"}
  Code Review: ${ENABLE_REVIEW ? "enabled" : "disabled"}${REQUIRE_REVIEW ? " (REQUIRED)" : ""}
  Test Enforcement: ${ENFORCE_TEST_CHANGES ? "enabled" : "disabled"}
============================================================
```

## Step 4.5: Check for Design Document (Optional)

Check if `design_doc.json` exists and display a summary:

```
If design_doc.json exists:
    Read and display design document summary:

    ────────────────────────────────────────────────────────────────
    📐 DESIGN DOCUMENT DETECTED
    ────────────────────────────────────────────────────────────────
    Title: <overview.title>

    Components: N defined
      • <component1>
      • <component2>
      ...

    Architectural Patterns: M patterns
      • <pattern1> - <rationale>
      • <pattern2> - <rationale>

    Key Decisions: P ADRs
      • ADR-001: <title>
      • ADR-002: <title>

    Story Mappings: Q stories mapped
      ✓ Mapped: story-001, story-002, ...
      ⚠ Unmapped: story-005, story-006, ... (if any)

    Agents will receive relevant design context during execution.
    ────────────────────────────────────────────────────────────────

Else:
    Note: No design document found.
          Consider generating one with /plan-cascade:design-generate
          for better architectural guidance during execution.
```

This summary helps reviewers understand the architectural context that will guide story execution.

## Step 4.6: Detect External Framework Skills

Check for applicable external framework skills based on the project type:

```bash
# Detect and display loaded skills
if command -v uv &> /dev/null; then
    uv run python -c "
import sys
sys.path.insert(0, '${CLAUDE_PLUGIN_ROOT}/src')
from plan_cascade.core.external_skill_loader import ExternalSkillLoader
from pathlib import Path

loader = ExternalSkillLoader(Path('.'))
skills = loader.detect_applicable_skills(verbose=True)
if skills:
    loader.display_skills_summary('implementation')
else:
    print('[ExternalSkillLoader] No matching framework skills detected')
    print('                      (Skills are auto-loaded based on package.json/Cargo.toml)')
" 2>/dev/null || echo "Note: External skills detection skipped"
fi
```

This will display:
```
┌──────────────────────────────────────────────────────────┐
│  EXTERNAL FRAMEWORK SKILLS LOADED                        │
├──────────────────────────────────────────────────────────┤
│  ✓ React Best Practices (source: vercel)                │
│  ✓ Web Design Guidelines (source: vercel)               │
├──────────────────────────────────────────────────────────┤
│  Phase: implementation | Total: 2 skill(s)              │
└──────────────────────────────────────────────────────────┘
```

If no skills are detected, this is normal for projects without matching frameworks.

## Step 5: Calculate Execution Batches

Analyze story dependencies and create parallel execution batches:

- **Batch 1**: Stories with no dependencies (can run in parallel)
- **Batch 2**: Stories that depend only on Batch 1 stories
- **Batch 3+**: Continue until all stories are batched

Display the execution plan:
```
=== Execution Plan ===

Total Stories: X
Total Batches: Y

Batch 1 (can run in parallel):
  - story-001: Title
  - story-002: Title

Batch 2:
  - story-003: Title (depends on: story-001)

...
```

## Step 6: Choose Execution Mode

Ask the user to choose between execution modes:

```bash
echo ""
echo "=========================================="
echo "Select Execution Mode"
echo "=========================================="
echo ""
echo "  [1] Auto Mode     - Automatically progress through batches"
echo "                       Pause only on errors"
echo ""
echo "  [2] Manual Mode   - Require approval before each batch"
echo "                       Full control and review"
echo ""
echo "  [3] Full Auto     - Python-based execution with auto-retry"
echo "       (Recommended)  Quality gates + automatic retry on failures"
echo "                       Best for unattended/CI execution"
echo ""
echo "=========================================="
read -p "Enter choice [1/2/3] (default: 3): " MODE_CHOICE
MODE_CHOICE="${MODE_CHOICE:-3}"

if [ "$MODE_CHOICE" = "2" ]; then
    EXECUTION_MODE="manual"
    echo ""
    echo "✓ Manual mode selected"
    echo "  You will be prompted before each batch starts"
elif [ "$MODE_CHOICE" = "3" ]; then
    EXECUTION_MODE="full_auto"
    echo ""
    echo "✓ Full Auto mode selected"
    echo "  Python-based execution with automatic retry"
    echo "  Quality gates will auto-retry failures up to 3 times"
else
    EXECUTION_MODE="auto"
    echo ""
    echo "✓ Auto mode selected"
    echo "  Batches will progress automatically (pause on errors)"
fi

# Save mode to config for reference
echo "execution_mode: $EXECUTION_MODE" >> progress.txt
```

### Step 6.1: Full Auto Mode Execution (if MODE_CHOICE = 3)

If `EXECUTION_MODE == "full_auto"`, use the Python-based execution script for automatic retry:

```bash
# Build command with parameters
AUTO_EXEC_CMD="uv run python \"${CLAUDE_PLUGIN_ROOT}/scripts/auto-execute.py\""
AUTO_EXEC_CMD="$AUTO_EXEC_CMD --prd prd.json"
AUTO_EXEC_CMD="$AUTO_EXEC_CMD --flow ${FLOW_LEVEL:-standard}"
AUTO_EXEC_CMD="$AUTO_EXEC_CMD --tdd ${TDD_MODE:-auto}"
AUTO_EXEC_CMD="$AUTO_EXEC_CMD --gate-mode ${GATE_MODE:-soft}"

# Propagate agent routing parameters (hard-routing path)
if [ -n "$GLOBAL_AGENT" ]; then
    AUTO_EXEC_CMD="$AUTO_EXEC_CMD --agent ${GLOBAL_AGENT}"
fi
if [ -n "$IMPL_AGENT" ]; then
    AUTO_EXEC_CMD="$AUTO_EXEC_CMD --impl-agent ${IMPL_AGENT}"
fi
if [ -n "$RETRY_AGENT" ]; then
    AUTO_EXEC_CMD="$AUTO_EXEC_CMD --retry-agent ${RETRY_AGENT}"
fi
if [ "$NO_FALLBACK" = "true" ]; then
    AUTO_EXEC_CMD="$AUTO_EXEC_CMD --no-fallback"
fi

# Add retry configuration
if [ "$NO_RETRY" = "true" ]; then
    AUTO_EXEC_CMD="$AUTO_EXEC_CMD --no-retry"
else
    AUTO_EXEC_CMD="$AUTO_EXEC_CMD --max-retries 3"
fi

# Add parallel flag if desired
# AUTO_EXEC_CMD="$AUTO_EXEC_CMD --parallel"

echo ""
echo "Executing: $AUTO_EXEC_CMD"
echo ""

# Run the auto-execute script
eval $AUTO_EXEC_CMD
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
    echo ""
    echo "✓ All stories completed successfully"
    # Skip to Step 10 (Final Status)
else
    echo ""
    echo "⚠️ Execution completed with failures (exit code: $EXIT_CODE)"
    echo "Check progress.txt and .agent-outputs/ for details"
fi

# Exit - full_auto mode handles everything
exit $EXIT_CODE
```

**Features of Full Auto Mode:**
- **Automatic Retry**: Failed stories are automatically retried up to 3 times
- **Failure Context Injection**: Retry prompts include error details and suggested fixes
- **Exponential Backoff**: 5s → 10s → 20s delays between retries
- **Agent Switching**: Can switch to different agent on retry (optional)
- **Quality Gates**: Verification, code review, TDD compliance all run automatically
- **State Persistence**: Progress saved to `.iteration-state.json` for recovery

If Full Auto mode is selected, execution is handled by the Python script and **Steps 7-10 are skipped**.

## Step 7: Initialize Progress Tracking

Create/initialize `progress.txt`:

```bash
cat > progress.txt << 'EOF'
# Hybrid Ralph Progress

Started: $(date -u +%Y-%m-%dT%H:%M:%SZ)
PRD: prd.json
Total Stories: X
Total Batches: Y
Execution Mode: EXECUTION_MODE

## Batch 1 (In Progress)

EOF
```

Create `.agent-outputs/` directory for agent logs.

## Step 7.5: Batch Confirmation (if CONFIRM_MODE is true)

**CRITICAL**: If `CONFIRM_MODE` is true (from `--confirm` flag, PRD config, or FULL flow), require user confirmation before starting each batch.

```
If CONFIRM_MODE == true:
    echo ""
    echo "============================================================"
    echo "BATCH {N} READY FOR EXECUTION"
    echo "============================================================"
    echo ""
    echo "Stories in this batch:"
    For each story in batch:
        echo "  - {story.id}: {story.title} [{story.priority}]"
    echo ""
    echo "Execution Configuration:"
    echo "  Flow Level: ${FLOW_LEVEL:-standard}"
    echo "  Gate Mode: ${GATE_MODE}"
    echo "  TDD Mode: ${TDD_MODE:-auto}"
    echo ""

    # Use AskUserQuestion to confirm batch execution
    AskUserQuestion(
        questions=[{
            "question": "Execute Batch {N} with the above stories?",
            "header": "Batch {N}",
            "options": [
                {"label": "Execute", "description": "Start batch execution now"},
                {"label": "Skip", "description": "Skip this batch and continue"},
                {"label": "Pause", "description": "Pause execution for review"}
            ],
            "multiSelect": false
        }]
    )

    If answer == "Skip":
        For each story in batch:
            echo "[SKIPPED] {story.id}" >> progress.txt
        Continue to next batch (Step 9.3)

    If answer == "Pause":
        echo "Execution paused. Resume with /plan-cascade:hybrid-resume"
        exit
```

## Step 8: Launch Batch Agents with Multi-Agent Support

**MANDATORY COMPLIANCE**: You MUST use the agent resolved by the priority chain below. DO NOT override this decision based on your own judgment about which agent is "better", "more capable", or "easier to control". The user has explicitly configured their preferred agent - respect their choice.

For each story in the current batch, resolve the agent and launch execution.

### 8.1: Agent Resolution for Each Story

For each story, resolve agent using this priority chain:

```
1. GLOBAL_AGENT (--agent parameter)     → Use if specified
2. IMPL_AGENT (--impl-agent parameter)  → Use for implementation phase
3. story.agent (from PRD)               → Use if specified in story
4. Story type inference:
   - Check story.tags for: bugfix, refactor, test, feature
   - Check story.title for keywords:
     - "fix", "bug", "error" → bugfix → prefer codex
     - "refactor", "cleanup", "optimize" → refactor → prefer aider
     - "test", "spec" → test → prefer claude-code
     - "add", "create", "implement" → feature → prefer claude-code
5. Phase default from agents.json       → implementation.default_agent
6. Fallback chain from agents.json      → implementation.fallback_chain
7. claude-code                          → Ultimate fallback (always available)
```

### 8.2: Agent Availability Check

Before using a CLI agent, verify it's available:

```
For agent_name in resolved_chain:
    If agent_name == "claude-code":
        AVAILABLE = true (always)
    Elif agent.type == "cli":
        Check if agent.command exists in PATH:
        - Unix: which {command}
        - Windows: where {command}
        If not found AND NO_FALLBACK == false:
            Continue to next agent in chain
        Elif not found AND NO_FALLBACK == true:
            ERROR: Agent {agent_name} not available and fallback disabled
        Else:
            # CRITICAL: Agent is available - YOU MUST USE IT
            # DO NOT switch to claude-code just because you think it's "better"
            echo "✓ {agent_name} is available and will be used as configured"
            AVAILABLE = true

    If AVAILABLE:
        RESOLVED_AGENT = agent_name
        Break
```

**CRITICAL RULES**:
1. If the resolved agent is available → **USE IT**
2. Only use fallback if the agent is **NOT FOUND** on the system
3. "Available but I prefer claude-code" is **NOT** a valid reason to switch
4. All CLI agents are fully capable of story execution

### 8.3: Launch Story Execution

For each story in the batch:

```
# Resolve agent for this story
RESOLVED_AGENT = resolve_agent(story, phase="implementation")

# Log agent selection
echo "[{story_id}] Using agent: {RESOLVED_AGENT}"
echo "[AGENT] {story_id} -> {RESOLVED_AGENT}" >> progress.txt

# Build prompt
PROMPT = """
You are executing story {story_id}: {title}

Description:
{description}

Acceptance Criteria:
- {criterion1}
- {criterion2}

Dependencies: {dependencies or "None"}

{If design_doc.json exists and has story_mappings[story_id]:}
## Technical Design Context

Components to use:
{list of relevant components from design_doc}

Patterns to follow:
{list of relevant patterns}

Relevant decisions:
{list of relevant ADRs}
{End if}

{If external skills were detected for the project:}
## Framework-Specific Best Practices

{Inject skill content from ExternalSkillLoader.get_skill_context("implementation")}

(Note: Follow these framework-specific guidelines when implementing)
{End if}

Your task:
1. Read relevant code and documentation
2. Implement the story according to acceptance criteria
3. Test your implementation
4. Update findings.md with discoveries (use <!-- @tags: {story_id} -->)
5. Mark complete by appending to progress.txt: [COMPLETE] {story_id}

Execute all necessary bash/powershell commands directly to complete the story.
Work methodically and document your progress.
"""

# Execute based on agent type
If RESOLVED_AGENT == "claude-code":
    # Use Task tool (built-in)
    Task(
        prompt=PROMPT,
        subagent_type="general-purpose",
        run_in_background=true
    )
    Store task_id for monitoring

Elif agents[RESOLVED_AGENT].type == "cli":
    # Use CLI agent via Bash
    agent_config = agents[RESOLVED_AGENT]
    command = agent_config.command  # e.g., "aider", "codex"
    args = agent_config.args        # e.g., ["--message", "{prompt}"]

    # Replace {prompt} placeholder with actual prompt
    # Replace {working_dir} with current directory
    full_command = f"{command} {' '.join(args)}"

    Bash(
        command=full_command,
        run_in_background=true,
        timeout=agent_config.timeout or 600000
    )
    Store task_id for monitoring
```

### 8.4: Display Agent Launch Summary

```
=== Batch {N} Launched ===

Stories and assigned agents:
  - story-001: claude-code (task-tool)
  - story-002: aider (cli) [refactor detected]
  - story-003: codex (cli) [bugfix detected]

{If any fallbacks occurred:}
⚠️ Agent fallbacks:
  - story-002: aider → claude-code (aider CLI not found)
{End if}

Waiting for completion...
```

## Step 9: Wait for Batch Completion (Using TaskOutput)

After launching batch agents, wait for completion:

**CRITICAL**: Use TaskOutput to wait instead of sleep+grep polling. This avoids Bash confirmation prompts.

### 9.1: Wait for Current Batch Agents

```
For each story_id, task_id in current_batch_tasks:
    echo "Waiting for {story_id}..."

    result = TaskOutput(
        task_id=task_id,
        block=true,
        timeout=600000  # 10 minutes per story
    )

    echo "✓ {story_id} agent completed"
```

### 9.2: Verify Batch Completion

After all TaskOutput calls return, verify using Read tool (NOT Bash grep):

```
# Use Read tool to get progress.txt content
progress_content = Read("progress.txt")

# Parse content yourself in your response
complete_count = count occurrences of "[COMPLETE]" in progress_content
error_count = count occurrences of "[ERROR]" in progress_content
failed_count = count occurrences of "[FAILED]" in progress_content

if error_count > 0 or failed_count > 0:
    echo "⚠️ ISSUES DETECTED IN BATCH"
    # Show which stories failed
    # Offer retry with different agent (see Step 9.2.5)

if complete_count >= expected_count:
    echo "✓ Batch complete!"
```

### 9.2.5: Retry Failed Stories with Different Agent

When a story fails, offer to retry with a different agent:

```
For each failed_story in failed_stories:
    # Determine retry agent
    If RETRY_AGENT specified:
        retry_agent = RETRY_AGENT
    Elif phase_defaults.retry.default_agent in agents.json:
        retry_agent = phase_defaults.retry.default_agent
    Else:
        retry_agent = "claude-code"

    # Check if retry agent is different from original
    original_agent = get_original_agent(failed_story)

    If retry_agent != original_agent AND is_agent_available(retry_agent):
        echo "Retrying {story_id} with {retry_agent} (was: {original_agent})"

        # Build retry prompt with error context
        RETRY_PROMPT = """
        You are RETRYING story {story_id}: {title}

        PREVIOUS ATTEMPT FAILED. Error details:
        {error_message from progress.txt}

        Please analyze what went wrong and try a different approach.
        {rest of original prompt}
        """

        # Launch retry agent
        Launch agent (retry_agent) with RETRY_PROMPT
        echo "[RETRY] {story_id} -> {retry_agent}" >> progress.txt
    Else:
        echo "Cannot retry {story_id}: no alternative agent available"
        # Pause for manual intervention
```

### 9.2.6: Parallel Quality Gates (Verify + Review + TDD)

**CRITICAL**: The three quality gates (AI Verification, AI Code Review, TDD Compliance) are **independent** — they all read the same git diff but produce separate judgments. Launch all enabled gates **in parallel** to reduce total gate time.

```
============================================================
QUALITY GATES — Parallel Execution
============================================================
  AI Verification: ${ENABLE_VERIFY ? "enabled" : "DISABLED"}
  Code Review:     ${ENABLE_REVIEW ? "enabled" : "DISABLED"}${REQUIRE_REVIEW ? " (REQUIRED)" : ""}
  TDD Compliance:  ${run_tdd ? "enabled" : "DISABLED"}
============================================================
```

**9.2.6.1: Determine Which Gates to Run**

```
# Collect gate flags
run_verify = ENABLE_VERIFY  # default true, disabled by --no-verify
run_review = ENABLE_REVIEW  # default true, disabled by --no-review

# TDD gate: enabled for "on" mode or auto-detected high-risk stories
run_tdd = false
If TDD_MODE == "on":
    run_tdd = true
Elif TDD_MODE == "auto":
    For each completed_story in batch:
        story_tags = completed_story.tags
        story_priority = completed_story.priority
        story_title = completed_story.title
        If tags contain "security|auth|database|payment|critical|sensitive":
            run_tdd = true
        Elif priority == "high":
            run_tdd = true
        Elif title matches "auth|login|password|credential|token|session|encrypt|secure|permission|role":
            run_tdd = true

enabled_gates = []
If run_verify: enabled_gates.append("verify")
If run_review: enabled_gates.append("review")
If run_tdd: enabled_gates.append("tdd")

If len(enabled_gates) == 0:
    echo "All quality gates disabled. Skipping to Step 9.3."
    Continue to Step 9.3

echo "Launching {len(enabled_gates)} quality gate(s) in parallel: {', '.join(enabled_gates)}"
```

**9.2.6.2: Parallel Launch — All Gates at Once**

For each completed story in the batch, launch all enabled gates simultaneously in a **single message with multiple tool calls**:

```
# Get git diff ONCE (shared by all gates)
git_diff = Bash("git diff HEAD~1 HEAD -- . 2>/dev/null || git diff HEAD -- .")

# Get changed file list ONCE (shared by TDD gate)
# Hoisted outside the per-story loop to avoid redundant git calls
changed_files = Bash("git diff --name-only HEAD~1 HEAD 2>/dev/null || git diff --cached --name-only 2>/dev/null || echo ''")

# Load design context ONCE (shared by review gate)
design_context = ""
If design_doc.json exists:
    Read design_doc.json
    Extract story_mappings for each story

# Initialize task tracking lists
verification_tasks = []   # List of (story_id, task_id) tuples
review_tasks = []         # List of (story_id, task_id) tuples
tdd_results = {}          # Dict of story_id → "passed" | "failed" (inline, no task_id)

For each completed_story in batch:
    story_id = completed_story.id

    # ── Gate 1: AI Verification ──
    If run_verify:
        VERIFY_PROMPT = """
        You are an implementation verification agent. Verify that story {story_id} is properly implemented.

        ## Story: {story_id} - {title}
        {description}

        ## Acceptance Criteria
        {acceptance_criteria as bullet list}

        ## Git Diff (Changes Made)
        ```diff
        {git_diff}
        ```

        ## Your Task
        Analyze the code changes and verify:
        1. Each acceptance criterion is implemented (not just stubbed)
        2. No skeleton code (pass, ..., NotImplementedError, TODO, FIXME in new code)
        3. The implementation is functional, not placeholder

        ## Skeleton Code Detection Rules
        Mark as FAILED if you find ANY of these in NEW code:
        - Functions with only `pass`, `...`, or `raise NotImplementedError`
        - TODO/FIXME comments in newly added code
        - Placeholder return values like `return None`, `return ""`, `return []` without logic
        - Empty function/method bodies
        - Comments like "# implement later" or "# stub"

        ## Output Format
        Write your result to the per-gate output file (NOT progress.txt):

        If ALL criteria met and NO skeleton code:
          Write "[VERIFIED] {story_id} - All acceptance criteria implemented" to .agent-outputs/{story_id}.verify.result

        If issues found:
          Write "[VERIFY_FAILED] {story_id} - <brief reason>" to .agent-outputs/{story_id}.verify.result

        Then write a detailed verification report to .agent-outputs/{story_id}.verify.md

        **IMPORTANT**: Write the single-line result marker to `.agent-outputs/{story_id}.verify.result`
        (a separate file from the detailed report). Do NOT append to progress.txt.
        """

        verify_task = Task(
            prompt=VERIFY_PROMPT,
            subagent_type="general-purpose",
            run_in_background=true,
            description="Verify {story_id}"
        )
        verification_tasks.append((story_id, verify_task))

    # ── Gate 2: AI Code Review ──
    If run_review:
        REVIEW_PROMPT = """
        You are an AI code reviewer. Review story {story_id} implementation for quality.

        ## Story: {story_id} - {title}
        {description}

        ## Git Diff
        ```diff
        {git_diff}
        ```

        ## Review Dimensions
        Score each dimension (total 100 points):

        1. Code Quality (25 pts) - Clean code, error handling, no code smells
        2. Naming & Clarity (20 pts) - Clear naming, self-documenting code
        3. Complexity (20 pts) - Appropriate complexity, no over-engineering
        4. Pattern Adherence (20 pts) - Follows project patterns
        5. Security (15 pts) - No vulnerabilities, proper input validation

        {If design_doc.json exists:}
        ## Architecture Context
        Components: {relevant_components}
        Patterns: {relevant_patterns}
        ADRs: {relevant_adrs}
        {End if}

        ## Severity Levels
        - critical: Must fix (security, data loss)
        - high: Should fix (bugs, poor patterns)
        - medium: Consider fixing (code smells)
        - low: Minor suggestions
        - info: Informational notes

        ## Output Format
        Write your result to the per-gate output file (NOT progress.txt):

        If score >= 70% AND no critical findings:
          Write "[REVIEW_PASSED] {story_id} - Score: X/100" to .agent-outputs/{story_id}.review.result

        If score < 70% OR has critical findings:
          Write "[REVIEW_ISSUES] {story_id} - Score: X/100 - <brief summary>" to .agent-outputs/{story_id}.review.result

        Then write detailed report to .agent-outputs/{story_id}.review.md with:
        - Dimension scores and notes
        - All findings with severity, file, line
        - Suggestions for improvement

        **IMPORTANT**: Write the single-line result marker to `.agent-outputs/{story_id}.review.result`
        (a separate file from the detailed report). Do NOT append to progress.txt.
        """

        review_task = Task(
            prompt=REVIEW_PROMPT,
            subagent_type="general-purpose",
            run_in_background=true,
            description="Review {story_id}"
        )
        review_tasks.append((story_id, review_task))

    # ── Gate 3: TDD Compliance (lightweight, inline — no subagent needed) ──
    # TDD runs synchronously here — no task_id needed for wait step.
    If run_tdd:
        code_files = filter changed_files for code extensions, excluding test patterns
        test_files = filter changed_files for test patterns

        If code_files exist AND test_files empty:
            tdd_results[story_id] = "failed"
        Else:
            tdd_results[story_id] = "passed"

# IMPORTANT: Launch verify_task and review_task in the SAME message
# (multiple tool calls in one response) so they run in parallel.
# TDD check already completed inline above (just a file-list comparison).
```

**CRITICAL**: The verify and review Task calls MUST be in a **single message with multiple tool calls** to ensure true parallel execution. Do NOT launch one, wait for it, then launch the next.

**Example of correct parallel launch:**
```
# In a SINGLE response, make both tool calls:
Task(prompt=VERIFY_PROMPT, subagent_type="general-purpose", run_in_background=true)
Task(prompt=REVIEW_PROMPT, subagent_type="general-purpose", run_in_background=true)
# Both agents start immediately and run concurrently
```

**9.2.6.3: Wait for All Gates**

```
# Wait for all background gate agents to complete
# Build unified task list from the tracking lists populated in 9.2.6.2
all_task_ids = []
For story_id, task_id in verification_tasks:
    all_task_ids.append(task_id)
For story_id, task_id in review_tasks:
    all_task_ids.append(task_id)

For each task_id in all_task_ids:
    TaskOutput(task_id=task_id, block=true, timeout=180000)

echo "All quality gates completed."
```

**9.2.6.4: Collect Results from Per-Gate Files and Merge to progress.txt**

After all gates complete, read per-gate output files and merge results into `progress.txt` atomically.
This avoids the concurrent-write race condition that would occur if subagents appended to progress.txt directly.

```
# ── Collect verification results from per-gate files ──
verified_count = 0
verify_failed_count = 0
For story_id, _ in verification_tasks:
    result_file = ".agent-outputs/{story_id}.verify.result"
    result_content = Read(result_file)
    If "[VERIFIED]" in result_content:
        verified_count += 1
        echo result_content >> progress.txt
    Elif "[VERIFY_FAILED]" in result_content:
        verify_failed_count += 1
        echo result_content >> progress.txt
    Else:
        # Malformed or missing result — treat as failure
        verify_failed_count += 1
        echo "[VERIFY_FAILED] {story_id} - No valid result marker in output" >> progress.txt

# ── Collect review results from per-gate files ──
review_passed_count = 0
review_issues_count = 0
For story_id, _ in review_tasks:
    result_file = ".agent-outputs/{story_id}.review.result"
    result_content = Read(result_file)
    If "[REVIEW_PASSED]" in result_content:
        review_passed_count += 1
        echo result_content >> progress.txt
    Elif "[REVIEW_ISSUES]" in result_content:
        review_issues_count += 1
        echo result_content >> progress.txt
    Else:
        review_issues_count += 1
        echo "[REVIEW_ISSUES] {story_id} - No valid result marker in output" >> progress.txt

# ── Collect TDD results (already computed inline in 9.2.6.2) ──
tdd_passed_count = 0
tdd_failed_count = 0
For story_id, result in tdd_results:
    If result == "passed":
        tdd_passed_count += 1
        echo "[TDD_PASSED] {story_id}" >> progress.txt
    Else:
        tdd_failed_count += 1
        echo "[TDD_FAILED] {story_id} - Code changes without tests" >> progress.txt

echo ""
echo "============================================================"
echo "QUALITY GATE RESULTS"
echo "============================================================"
If run_verify:
    echo "  Verification: {verified_count} passed, {verify_failed_count} failed"
If run_review:
    echo "  Code Review:  {review_passed_count} passed, {review_issues_count} issues"
If run_tdd:
    echo "  TDD:          {tdd_passed_count} passed, {tdd_failed_count} failed"
echo "============================================================"

# Determine overall gate status
any_failures = (verify_failed_count > 0) or (review_issues_count > 0) or (tdd_failed_count > 0)
```

**9.2.6.5: Handle Gate Failures (Unified)**

If any gate reported failures, handle them together based on GATE_MODE:

```
If any_failures == false:
    echo "✓ All quality gates passed."
    Continue to Step 9.2.9 (DoD Gate)

# ── Failures detected ──
echo ""
echo "⚠️ QUALITY GATE FAILURES DETECTED"
echo ""

# List all failures
If verify_failed_count > 0:
    echo "Verification failures:"
    For each verify_failed story: echo "  - {story_id}: {reason}"

If review_issues_count > 0:
    echo "Code review issues:"
    For each review_issues story:
        review_report = Read(".agent-outputs/{story_id}.review.md")
        echo "  - {story_id}: Score X/100"

If tdd_failed_count > 0:
    echo "TDD compliance failures:"
    For each tdd_failed story: echo "  - {story_id}: Code changes without tests"

# ── Handle based on GATE_MODE ──
If GATE_MODE == "hard":
    echo ""
    echo "============================================================"
    echo "HARD GATE: EXECUTION BLOCKED"
    echo "============================================================"
    echo "Flow Level FULL requires all quality gates to pass."
    echo ""

    # Build failure summary for the question
    failure_summary = []
    If verify_failed_count > 0: failure_summary.append("Verification: {verify_failed_count} failed")
    If review_issues_count > 0: failure_summary.append("Code Review: {review_issues_count} issues")
    If tdd_failed_count > 0: failure_summary.append("TDD: {tdd_failed_count} non-compliant")

    AskUserQuestion(
        questions=[{
            "question": f"Quality gates failed: {'; '.join(failure_summary)}. How do you want to proceed?",
            "header": "Gate Blocked",
            "options": [
                {"label": "Retry", "description": "Re-implement failed stories with different agent"},
                {"label": "Add Tests", "description": "Write tests for TDD-failing stories (if applicable)"},
                {"label": "Pause", "description": "Pause for manual intervention"}
            ],
            "multiSelect": false
        }]
    )

    If answer == "Retry":
        # Go to Step 9.2.5 retry logic for verify/review failures
    Elif answer == "Add Tests":
        # Launch test writing agent for TDD-failing stories
    Else:
        echo "Execution paused. Fix issues and resume with /plan-cascade:hybrid-resume"
        exit 1

Else:  # GATE_MODE == "soft"
    echo ""
    echo "⚠️ SOFT GATE: Warnings only"
    echo ""

    AskUserQuestion(
        questions=[{
            "question": "Quality gate issues found. How do you want to proceed?",
            "header": "Gate Warnings",
            "options": [
                {"label": "Continue", "description": "Acknowledge warnings and continue"},
                {"label": "Retry", "description": "Re-implement failed stories"},
                {"label": "Pause", "description": "Pause for manual review"}
            ],
            "multiSelect": false
        }]
    )

    If answer == "Continue":
        If verify_failed_count > 0:
            For each failed_story: echo "[VERIFY_SKIPPED] {story_id}" >> progress.txt
        If review_issues_count > 0:
            For each issues_story: echo "[REVIEW_ACKNOWLEDGED] {story_id}" >> progress.txt
        If tdd_failed_count > 0:
            For each tdd_story: echo "[TDD_WARNING] {story_id} - Code changes without tests" >> progress.txt
        # Continue to DoD gate
    Elif answer == "Retry":
        # Go to Step 9.2.5 retry logic
    Else:
        echo "Execution paused. Resume with /plan-cascade:hybrid-resume"
        exit 1
```

**9.2.6.6: CLI Verification (Alternative)**

If using external LLM CLI for verification (configured in agents.json):

```json
{
  "verification_gate": {
    "type": "cli",
    "command": "claude",
    "args": ["-p", "{prompt}"],
    "timeout": 120
  }
}
```

```
# Execute CLI verification
Bash(
    command="claude -p '{VERIFY_PROMPT}' >> .agent-outputs/{story_id}.verify.md",
    timeout=120000
)

# Parse output and update progress.txt accordingly
```

**Notes**:
- AI verification: Disable with `/plan-cascade:approve --no-verify` or PRD config `"verification_gate": {"enabled": false}`
- AI code review: Disable with `/plan-cascade:approve --no-review` or PRD config `"code_review": {"enabled": false}"`
- TDD compliance: Controlled by `--tdd off|on|auto`
- **Performance**: Parallel gate execution reduces total gate time from `verify_time + review_time + tdd_time` to `max(verify_time, review_time, tdd_time)`

### 9.2.9: Definition of Done (DoD) Gate

After all verification gates complete for a story, run DoD validation:

```bash
STORY_ID="${story_id}" \
FLOW_LEVEL="${FLOW_LEVEL}" \
CLAUDE_PLUGIN_ROOT="${CLAUDE_PLUGIN_ROOT}" \
uv run python << 'PYTHON_EOF'
import sys
import json
import os
from pathlib import Path

# Setup path
plugin_root = os.environ.get("CLAUDE_PLUGIN_ROOT") or ""
plugin_root_candidates = [Path(plugin_root) if plugin_root else None, Path.cwd(), Path.cwd().parent]

for candidate in plugin_root_candidates:
    if candidate and (candidate / "src").exists():
        sys.path.insert(0, str(candidate / "src"))
        break

try:
    from plan_cascade.core.done_gate import DoneGate, DoDLevel
except ImportError:
    print("[DoD] DoneGate not available, skipping")
    sys.exit(0)

# Optional: changed-files detection for FULL DoD checks
try:
    from plan_cascade.core.changed_files import ChangedFilesDetector
except ImportError:
    ChangedFilesDetector = None

# Story id (best-effort)
story_id = os.environ.get("STORY_ID") or "unknown"

# Determine DoD level from flow
flow = os.environ.get("FLOW_LEVEL") or "standard"
level = DoDLevel.FULL if flow == "full" else DoDLevel.STANDARD
gate = DoneGate(level=level)

# Gather gate outputs from progress.txt
progress_path = Path("progress.txt")
gate_outputs = {}
if progress_path.exists():
    content = progress_path.read_text(encoding="utf-8", errors="ignore")
    gate_outputs = {
        "verified": {"passed": f"[VERIFIED] {story_id}" in content or f"[VERIFY_SKIPPED] {story_id}" in content},
        "review_passed": {"passed": f"[REVIEW_PASSED] {story_id}" in content or f"[REVIEW_ACKNOWLEDGED] {story_id}" in content},
        "tdd_passed": {"passed": f"[TDD_PASSED] {story_id}" in content or f"[TDD_WARNING] {story_id}" in content},
    }

# Best-effort changed-files detection (for FULL DoD test-enforcement checks)
changed_files = []
try:
    if ChangedFilesDetector is not None:
        detector = ChangedFilesDetector(Path.cwd())
        changed_files = detector.get_changed_files(include_untracked=True)
except Exception:
    changed_files = []

# Check completion
result = gate.check(
    gate_outputs=gate_outputs,
    verification_result=None,
    review_result=None,
    changed_files=changed_files,
)

print(result.get_summary())
if not result.passed:
    print(f"\n[DoD_FAILED] {story_id}")
    sys.exit(1)
else:
    print(f"\n[DoD_PASSED] {story_id}")
PYTHON_EOF

if [ $? -ne 0 ]; then
    echo "[DoD_FAILED] ${story_id}" >> progress.txt
    if [ "${GATE_MODE}" == "hard" ]; then
        echo "DoD Gate blocked. Story does not meet Definition of Done."
        exit 1
    else
        echo "[DoD_WARNING] Story has DoD issues (soft mode, continuing)"
    fi
else
    echo "[DoD_PASSED] ${story_id}" >> progress.txt
fi
```

### 9.3: Progress to Next Batch

**AUTO MODE**: Automatically launch next batch

```
if more batches remain:
    echo "=== Auto-launching Batch {N+1} ==="

    # Launch agents for next batch stories
    # Store new task_ids
    # Go back to Step 9.1 to wait for them
```

**MANUAL MODE**: Ask before launching next batch

```
if more batches remain:
    echo "Batch {N+1} Ready"
    echo "Stories: {list}"

    # Use AskUserQuestion tool to ask for confirmation
    # If confirmed, launch next batch agents
    # Go back to Step 9.1
```

### Why TaskOutput Instead of Polling?

| Method | Confirmation Prompts | How it works |
|--------|---------------------|--------------|
| `sleep + grep` loop | YES - every iteration | Bash commands need confirmation |
| `TaskOutput(block=true)` | NO | Native wait for agent completion |

TaskOutput is the correct way to wait for background agents.

## Step 10: Show Final Status and Cleanup

```
=== All Batches Complete ===

Total Stories: X
Completed: X

All batches have been executed successfully.
```

### Step 10.1: Detect Execution Mode

Check if the current execution is in a worktree or in-place (non-worktree) mode:

```bash
if [ -f ".planning-config.json" ]; then
    EXECUTION_MODE="worktree"
else
    EXECUTION_MODE="in-place"
fi
```

### Step 10.2: Cleanup Based on Mode

**If `EXECUTION_MODE == "worktree"`**:

Worktree mode requires merge before cleanup. Show next steps and let the user run hybrid-complete:

```
Next steps:
  - /plan-cascade:hybrid-status - Verify completion
  - /plan-cascade:hybrid-complete - Finalize, merge, and cleanup
```

**If `EXECUTION_MODE == "in-place"`**:

**CRITICAL**: In non-worktree mode, planning files remain in the project root after execution. You MUST clean them up automatically. Do NOT skip this cleanup or defer it to the user.

```bash
echo "Cleaning up planning files..."

# Planning documents
rm -f prd.json findings.md progress.txt mega-findings.md
rm -f design_doc.json spec.json spec.md

# Status and state files
rm -f .agent-status.json .iteration-state.json .retry-state.json

# Context recovery files
rm -f .hybrid-execution-context.md .mega-execution-context.md

# Directories
rm -rf .agent-outputs
rm -rf .locks
rm -rf .state

echo "✓ Planning files cleaned up"

# Also clean up state files from user data directory (new mode)
USER_STATE_DIR=$(uv run python -c "from plan_cascade.state.path_resolver import PathResolver; from pathlib import Path; print(PathResolver(Path.cwd()).get_state_dir())" 2>/dev/null || echo "")
if [ -n "$USER_STATE_DIR" ] && [ -d "$USER_STATE_DIR" ]; then
    echo "Cleaning up state files from user data directory..."
    rm -f "$USER_STATE_DIR/.iteration-state.json" 2>/dev/null || true
    rm -f "$USER_STATE_DIR/.agent-status.json" 2>/dev/null || true
    rm -f "$USER_STATE_DIR/.retry-state.json" 2>/dev/null || true
    rm -f "$USER_STATE_DIR/spec-interview.json" 2>/dev/null || true
    # Remove .state directory if empty
    rmdir "$USER_STATE_DIR" 2>/dev/null || true
    echo "✓ State files cleaned"
fi
```

Then show:

```
=== Execution Complete ===

All stories executed and planning files cleaned up.
```

## Notes

### Execution Modes

**Auto Mode (default)**:
- Batches progress automatically when previous batch completes
- No manual intervention needed between batches
- Pauses only on errors or failures
- Best for: routine tasks, trusted PRDs, faster execution

**Manual Mode**:
- Requires user approval before launching each batch
- Batch completes → Review → Approve → Next batch starts
- Full control to review between batches
- Best for: critical tasks, complex PRDs, careful oversight

**Note**: In BOTH modes, agents execute bash/powershell commands directly without waiting for confirmation. The execution mode ONLY controls batch-to-batch progression.

### Shared Features

- Each agent runs in the background with its own task_id
- Agents write their findings to `findings.md` tagged with their story ID
- Progress is tracked in `progress.txt` with `[COMPLETE]`, `[ERROR]`, or `[FAILED]` markers
- Agent outputs are logged to `.agent-outputs/{story_id}.log`
- **Pause on errors**: Both modes pause if any agent reports `[ERROR]` or `[FAILED]`
- **Real-time monitoring**: Progress is polled every 10 seconds and displayed
- **Error markers**: Agents should use `[ERROR]` for recoverable issues, `[FAILED]` for blocking problems
- **Resume capability**: After fixing errors or interruption, run `/plan-cascade:hybrid-resume --auto` to intelligently resume
  - Auto-detects completed stories and skips them
  - Works with both old and new progress markers
  - Or run `/plan-cascade:approve` to restart current batch
