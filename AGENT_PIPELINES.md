# Browser-Use Agent Pipelines Documentation

This document provides a comprehensive overview of all agent execution pipelines in the browser-use library, including data flow, prompt usage, and system architecture.

## Table of Contents

1. [Standard Agent Pipeline](#1-standard-agent-pipeline)
2. [Code-Use Agent Pipeline](#2-code-use-agent-pipeline)
3. [Extract Action Pipeline](#3-extract-action-pipeline)
4. [Judge Evaluation Pipeline](#4-judge-evaluation-pipeline)
5. [Task Completion Validation Pipeline](#5-task-completion-validation-pipeline)
6. [Element Finding Pipeline](#6-element-finding-pipeline)

---

## 1. Standard Agent Pipeline

The main action-based agent that executes browser automation tasks through a decision-action loop.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AGENT INITIALIZATION                         │
├─────────────────────────────────────────────────────────────────────┤
│  1. Load System Prompt (based on use_thinking, flash_mode)          │
│  2. Initialize Browser Session (CDP or local launch)                │
│  3. Initialize Tools Registry (all available actions)               │
│  4. Initialize File System (todo.md, results tracking)              │
│  5. Initialize Message Manager (context management)                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         MAIN EXECUTION LOOP                          │
│                      (run() → step() × N times)                      │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
        ╔═══════════════════════════════════════════════╗
        ║              SINGLE STEP EXECUTION             ║
        ╚═══════════════════════════════════════════════╝
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  PHASE 1: PREPARE CONTEXT (_prepare_context)     │
    ├──────────────────────────────────────────────────┤
    │  • Get browser state (URL, DOM, tabs)            │
    │  • Take screenshot (always captured)             │
    │  • Check for downloads (PDFs, files)             │
    │  • Update action models (page-specific actions)  │
    │  • Create state messages:                        │
    │    - Agent history (previous steps)              │
    │    - Agent state (task, files, todos, step info) │
    │    - Browser state (indexed elements)            │
    │    - Browser vision (screenshot with boxes)      │
    │    - Read state (extract/read_file data)         │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  PHASE 2: GET NEXT ACTION (_get_next_action)     │
    ├──────────────────────────────────────────────────┤
    │  • Build input messages from context             │
    │  • Call LLM with system + user messages:         │
    │                                                   │
    │    INPUT:                                         │
    │    ├─ System Prompt (selected variant)           │
    │    ├─ Agent History (formatted steps)            │
    │    ├─ Current Browser State                      │
    │    ├─ Screenshot (if use_vision=True)            │
    │    └─ Available Actions                          │
    │                                                   │
    │    OUTPUT (AgentOutput):                         │
    │    ├─ thinking (if enabled)                      │
    │    ├─ evaluation_previous_goal                   │
    │    ├─ memory                                      │
    │    ├─ next_goal                                   │
    │    └─ action[] (list of actions to execute)      │
    │                                                   │
    │  • Parse model output into AgentOutput           │
    │  • Validate action parameters                    │
    │  • Store model output in state                   │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  PHASE 3: EXECUTE ACTIONS (_execute_actions)     │
    ├──────────────────────────────────────────────────┤
    │  • Execute actions sequentially (multi_act)      │
    │  • For each action:                              │
    │    - Validate parameters                         │
    │    - Execute via Tools registry                  │
    │    - Check for page changes (interrupts)         │
    │    - Store ActionResult                          │
    │  • Aggregate results                             │
    │  • Store in state.last_result                    │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  PHASE 4: POST-PROCESS (_post_process)           │
    ├──────────────────────────────────────────────────┤
    │  • Update history with step results              │
    │  • Check for task completion (done action)       │
    │  • Update file system state                      │
    │  • Trigger event bus notifications               │
    │  • Check for max steps reached                   │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  PHASE 5: FINALIZE (_finalize)                   │
    ├──────────────────────────────────────────────────┤
    │  • Save conversation/trajectory                  │
    │  • Update telemetry                              │
    │  • Increment step counter                        │
    │  • Check for errors/failures                     │
    └──────────────────────────────────────────────────┘
                              ↓
                  ┌─────────────────────┐
                  │   Continue loop?    │
                  └─────────────────────┘
                    ↓YES          ↓NO (done/max_steps)
             [Next Step]      [Task Complete]
```

### Prompts Used

**System Prompt Selection Logic:**
```python
if flash_mode:
    if is_anthropic:
        → system_prompt_flash_anthropic.md
    else:
        → system_prompt_flash.md
elif use_thinking:
    → system_prompt.md  # Main prompt with thinking block
else:
    → system_prompt_no_thinking.md
```

**Message Construction (Each Step):**
```
Message 1: SystemMessage(content=selected_system_prompt)
Message 2: UserMessage(content=[
    agent_history_text,  # Previous steps formatted
    agent_state_text,     # Task, files, todos
    browser_state_text,   # URL, tabs, indexed elements
    browser_vision_image, # Screenshot (if use_vision=True)
    read_state_text      # One-time data from extract/read_file
])
```

### Data Flow Between Steps

```
Step N-1 Output (AgentOutput)
├─ thinking: str
├─ evaluation_previous_goal: str
├─ memory: str
├─ next_goal: str
└─ action: list[ActionModel]
         ↓
[Execute Actions] → ActionResult
         ↓
Step N Input Context
├─ Agent History:
│  ├─ Step N-1: evaluation, memory, next_goal, action results
│  ├─ Step N-2: evaluation, memory, next_goal, action results
│  └─ ... (earlier steps)
├─ Browser State: current URL, tabs, indexed elements
├─ File System: todo.md, results files, downloaded files
└─ Read State: last extract() or read_file() output
         ↓
Step N LLM Call → Step N Output (AgentOutput)
         ↓
[Execute Actions] → ActionResult
         ↓
Step N+1 Input Context
```

### Key Configuration Options

| Parameter | Default | Effect on Pipeline |
|-----------|---------|-------------------|
| `use_thinking` | True | Adds `thinking` field to output, uses main system prompt |
| `flash_mode` | False | Uses ultra-condensed prompts, single `memory` field |
| `use_vision` | True | Includes screenshot in LLM input |
| `max_actions` | 10 | Max actions per step (injected into prompt) |
| `max_steps` | 100 | Total execution steps before forced termination |
| `include_screenshot_actions` | [] | Force screenshots for specific actions |

---

## 2. Code-Use Agent Pipeline

The alternative code-generation based agent that writes Python code in a Jupyter-like notebook environment.

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                      CODE AGENT INITIALIZATION                        │
├──────────────────────────────────────────────────────────────────────┤
│  1. Load Code-Use System Prompt (from system_prompt.md)              │
│  2. Initialize Browser Session                                       │
│  3. Initialize Namespace (execution environment)                     │
│  4. Pre-import libraries (json, asyncio, csv, re, datetime, etc.)    │
│  5. Inject tool functions (navigate, click, evaluate, etc.)          │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────┐
│                     CELL-BY-CELL EXECUTION LOOP                       │
│                    (Like Jupyter: write → execute → next)            │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
        ╔═══════════════════════════════════════════════╗
        ║           SINGLE CELL EXECUTION                ║
        ╚═══════════════════════════════════════════════╝
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 1: BUILD CONTEXT                           │
    ├──────────────────────────────────────────────────┤
    │  • Get current browser state:                    │
    │    - URL and DOM (compressed, indexed)           │
    │    - Loading status (pending network requests)   │
    │    - Element markers ([i_123] format)            │
    │    - Shadow DOM, iframes, scroll containers      │
    │  • Get execution environment status:             │
    │    - Persisted variables from previous cells     │
    │    - Error count (8 consecutive errors = stop)   │
    │    - Current working files                       │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 2: CALL LLM FOR CODE GENERATION            │
    ├──────────────────────────────────────────────────┤
    │  INPUT:                                           │
    │  ├─ System Prompt (code_use/system_prompt.md)    │
    │  ├─ User Task (original request)                 │
    │  ├─ Previous Cell Output (if any)                │
    │  ├─ Current Browser State (DOM snapshot)         │
    │  ├─ Available Tools (functions reference)        │
    │  └─ Execution History (previous cells)           │
    │                                                   │
    │  OUTPUT:                                          │
    │  ├─ [1-2 sentences]: reasoning about prev step   │
    │  ├─ [1-2 sentences]: plan for next step          │
    │  └─ Code Cell:                                    │
    │      ```python                                    │
    │      # Python code to execute                    │
    │      print(results)                               │
    │      ```                                          │
    │  OR Multi-block:                                  │
    │      ```js extract_products`                      │
    │      (function(){ ... })()                        │
    │      ```                                          │
    │      ```python                                    │
    │      products = await evaluate(extract_products) │
    │      ```                                          │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 3: PARSE & EXTRACT CODE BLOCKS             │
    ├──────────────────────────────────────────────────┤
    │  • Extract Python code block(s)                  │
    │  • Extract named non-Python blocks:              │
    │    - ```js name` → saved as string variable      │
    │    - ```markdown name` → saved as string         │
    │    - ```bash name` → saved as string             │
    │  • Validate syntax (basic checks)                │
    │  • Check for forbidden patterns (global keyword) │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 4: EXECUTE CODE IN NAMESPACE               │
    ├──────────────────────────────────────────────────┤
    │  • Inject named blocks as variables              │
    │  • Execute Python code in persistent namespace   │
    │  • Available functions:                          │
    │    - await navigate(url)                         │
    │    - await click(index=N)                        │
    │    - await input_text(index=N, text=str)         │
    │    - await evaluate(js_code, variables=dict)     │
    │    - await get_selector_from_index(index=N)      │
    │    - await scroll(down=bool, pages=float)        │
    │    - await done(text=str, success=bool, files=[])│
    │  • Capture stdout/stderr                         │
    │  • Track execution errors                        │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 5: COLLECT EXECUTION RESULTS               │
    ├──────────────────────────────────────────────────┤
    │  • Capture output:                               │
    │    - Print statements → stdout                   │
    │    - Return values → result                      │
    │    - Exceptions → error message                  │
    │  • Update browser state (after actions)          │
    │  • Track variable changes in namespace           │
    │  • Check for done() call (task completion)       │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 6: VALIDATE & CONTINUE                     │
    ├──────────────────────────────────────────────────┤
    │  • If done() was called:                         │
    │    → Optionally validate with LLM               │
    │    → Return final result                         │
    │  • If error occurred:                            │
    │    → Increment error counter                     │
    │    → If 8 consecutive errors → terminate         │
    │  • If max steps reached → terminate              │
    │  • Otherwise → continue to next cell             │
    └──────────────────────────────────────────────────┘
                              ↓
                  ┌─────────────────────┐
                  │   Continue loop?    │
                  └─────────────────────┘
                    ↓YES          ↓NO (done/errors/max_steps)
             [Next Cell]      [Task Complete]
```

### Prompts Used

**System Prompt:** `browser_use/code_use/system_prompt.md`
- Jupyter-like execution model instructions
- Tool documentation (navigate, click, evaluate, etc.)
- JavaScript patterns (IIFE, no comments, no backticks)
- Pagination strategies (URL first, then buttons)
- 5-phase execution flow (Exploration → Validation → Batch → Cleanup → Done)
- Complete working examples

**Optional Validation Prompt:** Task Completion Validation (see [Pipeline 5](#5-task-completion-validation-pipeline))

### Data Flow Between Cells

```
Cell N-1 Code
├─ Python code executed
└─ Output captured (prints, variables, errors)
         ↓
Cell N Context
├─ Previous Cell Output (stdout/stderr)
├─ Updated Browser State (after actions)
├─ Persisted Variables (namespace state)
└─ Error History (consecutive errors)
         ↓
Cell N LLM Call → Cell N Code Generated
         ↓
Cell N Execution → Cell N Output
         ↓
Cell N+1 Context
```

### Key Differences from Standard Agent

| Aspect | Standard Agent | Code-Use Agent |
|--------|---------------|----------------|
| **Output Format** | JSON actions | Python code |
| **Execution** | Action registry | Direct code execution |
| **State** | Agent history | Jupyter-like namespace |
| **Tool Access** | Indirect via actions | Direct function calls |
| **Flexibility** | Predefined actions only | Arbitrary Python logic |
| **Error Handling** | Action validation | Runtime exceptions |
| **Memory** | Structured fields | Variables in namespace |

---

