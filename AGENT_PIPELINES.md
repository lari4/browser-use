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

## 3. Extract Action Pipeline

The content extraction pipeline that retrieves relevant information from webpages.

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        EXTRACT ACTION CALLED                          │
│                     extract(query: str, ...)                          │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 1: GET PAGE CONTENT                        │
    ├──────────────────────────────────────────────────┤
    │  • Capture full DOM state                        │
    │  • Convert HTML → Markdown (markdownify)         │
    │  • Get page statistics:                          │
    │    - Total characters                            │
    │    - Number of elements                          │
    │    - Content type                                │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 2: FILTER CONTENT                          │
    ├──────────────────────────────────────────────────┤
    │  • Apply content filters:                        │
    │    - Remove ads (common ad selectors)            │
    │    - Remove tracking scripts                     │
    │    - Remove navigation menus                     │
    │    - Remove footers/headers                      │
    │  • Truncate if needed:                           │
    │    - Max 100K characters by default              │
    │    - Support start_from_char for continuation    │
    │  • Calculate filter statistics                   │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 3: BUILD EXTRACTION PROMPT                 │
    ├──────────────────────────────────────────────────┤
    │  System Prompt:                                  │
    │  ├─ "You are an expert at extracting data..."   │
    │  ├─ Input description                            │
    │  ├─ Instructions (no hallucination, list all)    │
    │  └─ Output format (concise, direct)              │
    │                                                   │
    │  User Prompt:                                    │
    │  ├─ <query>{user_query}</query>                  │
    │  ├─ <content_stats>{statistics}</content_stats>  │
    │  └─ <webpage_content>{filtered_md}</...>         │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 4: CALL LLM FOR EXTRACTION                 │
    ├──────────────────────────────────────────────────┤
    │  • Send system + user messages to LLM            │
    │  • LLM extracts relevant information             │
    │  • Returns extracted data as text                │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 5: RETURN RESULT                           │
    ├──────────────────────────────────────────────────┤
    │  Return ActionResult:                            │
    │  ├─ extracted_content: str (LLM response)        │
    │  ├─ is_done: False                               │
    │  ├─ include_in_memory: True (show in read_state) │
    │  └─ metadata: statistics                         │
    └──────────────────────────────────────────────────┘
```

### Prompts Used

**Extraction System Prompt** (inline in `tools/service.py`)

**Data Flow:**
```
Current Page DOM → Markdown → Filtered Content → LLM → Extracted Data → ActionResult
                                                                              ↓
                                                        Visible in <read_state> in next step
```

### Configuration Options

| Parameter | Default | Description |
|-----------|---------|-------------|
| `query` | Required | What information to extract |
| `max_chars` | 100000 | Max content length to send to LLM |
| `start_from_char` | 0 | For paginated extraction |

---

## 4. Judge Evaluation Pipeline

The post-execution evaluation pipeline that assesses agent performance.

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         EVALUATION TRIGGERED                          │
│           (After agent.run() completes or during testing)            │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 1: COLLECT EXECUTION DATA                  │
    ├──────────────────────────────────────────────────┤
    │  • Original task description                     │
    │  • Final result returned to user                 │
    │  • Agent trajectory (all steps):                 │
    │    - Step N: evaluation, memory, next_goal       │
    │    - Actions taken                               │
    │    - Action results                              │
    │  • Screenshots (last N images, default 10)       │
    │  • Ground truth (if provided)                    │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 2: FORMAT TRAJECTORY                       │
    ├──────────────────────────────────────────────────┤
    │  • Format agent steps as text:                   │
    │    "Step 1:                                      │
    │     Evaluation: [...]                            │
    │     Memory: [...]                                │
    │     Next Goal: [...]                             │
    │     Actions: [...]                               │
    │     Results: [...]"                              │
    │  • Truncate if needed (max 40K chars each)       │
    │  • Encode screenshots to base64                  │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 3: BUILD JUDGE PROMPT                      │
    ├──────────────────────────────────────────────────┤
    │  System Message:                                 │
    │  ├─ Evaluation framework                         │
    │  ├─ Ground truth section (if provided)           │
    │  ├─ Primary criteria (5 levels)                  │
    │  ├─ Verdict guidelines (true/false)              │
    │  ├─ Failure conditions                           │
    │  ├─ Impossible task detection                    │
    │  ├─ CAPTCHA detection                            │
    │  └─ Response format (JSON schema)                │
    │                                                   │
    │  User Message:                                   │
    │  ├─ <task>{original_task}</task>                 │
    │  ├─ <ground_truth>{...}</ground_truth> (if any)  │
    │  ├─ <agent_trajectory>{steps}</agent_trajectory> │
    │  ├─ <final_result>{output}</final_result>        │
    │  └─ [Screenshot 1, Screenshot 2, ...]            │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 4: CALL LLM FOR EVALUATION                 │
    ├──────────────────────────────────────────────────┤
    │  • Send messages to LLM                          │
    │  • LLM analyzes entire trajectory + screenshots  │
    │  • Returns structured JSON evaluation            │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 5: PARSE EVALUATION                        │
    ├──────────────────────────────────────────────────┤
    │  Parse JSON response:                            │
    │  {                                               │
    │    "reasoning": "Detailed analysis...",          │
    │    "verdict": true or false,                     │
    │    "failure_reason": "Why it failed..." | "",    │
    │    "impossible_task": true or false,             │
    │    "reached_captcha": true or false              │
    │  }                                               │
    │  Return JudgementResult                          │
    └──────────────────────────────────────────────────┘
```

### Prompts Used

**Judge System Prompt** (inline in `agent/judge.py`)
- Comprehensive evaluation framework
- 5-level criteria hierarchy
- Ground truth validation (highest priority)
- Examples of success/failure scenarios
- Structured JSON output format

### Ground Truth Priority

When ground truth is provided, the evaluation follows this priority:
```
1. Ground Truth Validation (HIGHEST PRIORITY)
   ↓ If satisfied
2. Task Satisfaction
   ↓ If satisfied
3. Output Quality
   ↓ If satisfied
4. Tool Effectiveness
   ↓ If satisfied
5. Browser Handling

Final Verdict: true/false
```

---

## 5. Task Completion Validation Pipeline

Quick validation check to determine if CodeAgent should continue working.

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    VALIDATION CHECK TRIGGERED                         │
│         (After each CodeAgent execution when max_validations > 0)    │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 1: BUILD VALIDATION PROMPT                 │
    ├──────────────────────────────────────────────────┤
    │  User Message (no system prompt):                │
    │  ├─ "You are a task completion validator..."    │
    │  ├─ **Original Task:** {user_task}               │
    │  ├─ **Agent's Output:** {current_output}         │
    │  ├─ **Your Task:** Determine if complete         │
    │  ├─ Consider:                                    │
    │  │   1. Has agent delivered what was requested?  │
    │  │   2. Is there actual data (if extraction)?    │
    │  │   3. Is task truly impossible?                │
    │  │   4. Can agent make meaningful progress?      │
    │  └─ **Response Format:**                         │
    │      Reasoning: [analysis]                       │
    │      Verdict: [YES or NO]                        │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 2: CALL LLM                                │
    ├──────────────────────────────────────────────────┤
    │  • Send user message (no system, no history)     │
    │  • LLM analyzes task vs output                   │
    │  • Returns text with Reasoning + Verdict         │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 3: PARSE VERDICT                           │
    ├──────────────────────────────────────────────────┤
    │  • Extract reasoning from response               │
    │  • Look for "Verdict: YES" or "Verdict: NO"      │
    │  • Return (is_complete: bool, reasoning: str)    │
    │                                                   │
    │  YES = Task complete OR impossible               │
    │  NO = Should continue working                    │
    └──────────────────────────────────────────────────┘
```

### Prompts Used

**Validation Prompt** (inline in `code_use/namespace.py`)
- Simple task completion check
- No system prompt (lightweight)
- YES/NO format

### Decision Flow

```
Agent Output → Validation → YES → Stop execution, return result
                         → NO → Continue to next cell
```

---

## 6. Element Finding Pipeline

Find DOM elements by natural language description (Actor API).

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                   get_element_by_prompt(prompt) CALLED                │
└──────────────────────────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 1: GET CURRENT PAGE STATE                  │
    ├──────────────────────────────────────────────────┤
    │  • Get serialized DOM with indexed elements      │
    │  • Format as LLM representation:                 │
    │    [123]<button>Submit</button>                  │
    │    [124]<input type="text">Name</input>          │
    │    [125]<a href="...">Next</a>                   │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 2: BUILD ELEMENT FINDING PROMPT            │
    ├──────────────────────────────────────────────────┤
    │  System Message:                                 │
    │  ├─ "You are an AI created to find elements..."  │
    │  ├─ <browser_state> format explanation           │
    │  ├─ Task: find element matching prompt           │
    │  ├─ Return None if no match                      │
    │  └─ Reason before returning index                │
    │                                                   │
    │  User Message:                                   │
    │  ├─ <browser_state>{indexed_elements}</...>      │
    │  └─ <prompt>{user_description}</prompt>          │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 3: CALL LLM                                │
    ├──────────────────────────────────────────────────┤
    │  • LLM analyzes elements vs description          │
    │  • Reasons about matches                         │
    │  • Returns element index or None                 │
    └──────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────┐
    │  STEP 4: PARSE & RETURN                          │
    ├──────────────────────────────────────────────────┤
    │  • Extract index from response                   │
    │  • Validate index exists in DOM                  │
    │  • Return DOMInteractedElement or None           │
    └──────────────────────────────────────────────────┘
```

### Prompts Used

**Element Finding Prompt** (inline in `actor/page.py`)
- Browser state format documentation
- Element hierarchy explanation
- Reasoning requirement

### Use Cases

```python
# Instead of:
element = page.get_element_by_selector('button.submit-btn')

# Use natural language:
element = await page.get_element_by_prompt('the submit button at the bottom')
```

---

## Summary: Pipeline Comparison

| Pipeline | Trigger | Prompts | Input | Output | Use Case |
|----------|---------|---------|-------|--------|----------|
| **Standard Agent** | `agent.run()` | System prompt (4 variants) + message construction | Task, browser state, history | AgentOutput (actions) | Main browser automation |
| **Code-Use Agent** | `code_agent.run()` | Code-use system prompt | Task, browser state, namespace | Python code cells | Complex data extraction |
| **Extract Action** | `extract(query)` | Extraction system prompt | Query, page content | Extracted information | Page data extraction |
| **Judge Evaluation** | `construct_judge_messages()` | Judge system prompt | Task, trajectory, screenshots | JudgementResult | Performance evaluation |
| **Task Validation** | CodeAgent validation | Validation prompt | Task, current output | YES/NO verdict | Completion check |
| **Element Finding** | `get_element_by_prompt()` | Element finding prompt | Description, DOM state | Element index | Natural language element selection |

## Prompt Selection Decision Tree

```
┌─────────────────────────────────────────┐
│      Which agent are you using?        │
└─────────────────────────────────────────┘
         ↓                    ↓
    Standard Agent      Code-Use Agent
         ↓                    ↓
    Flash mode?         code_use/system_prompt.md
    ↓YES    ↓NO             + Optional validation prompt
    │       │
    │       Use thinking?
    │       ↓YES    ↓NO
    │       │       │
    Anthropic?  system_prompt.md  system_prompt_no_thinking.md
    ↓YES  ↓NO
    │     │
flash_anthropic.md  system_prompt_flash.md
```

---

## Advanced: Message Construction Details

### Standard Agent Message Format (Each Step)

```python
messages = [
    # Message 1: System Prompt
    SystemMessage(content=selected_system_prompt_text),
    
    # Message 2: Complete Context
    UserMessage(content=[
        # Text Part 1: Agent History
        ContentPartTextParam(text="""
<agent_history>
<step_1>
Evaluation: {prev_evaluation}
Memory: {prev_memory}
Next Goal: {prev_goal}
Actions: {prev_actions}
Results: {prev_results}
</step_1>
...
</agent_history>
        """),
        
        # Text Part 2: Agent State
        ContentPartTextParam(text="""
<agent_state>
<user_request>{original_task}</user_request>
<file_system>{file_summary}</file_system>
<todo_contents>{todo_md_content}</todo_contents>
<step_info>{current_step}/{max_steps}</step_info>
</agent_state>
        """),
        
        # Text Part 3: Browser State
        ContentPartTextParam(text="""
<browser_state>
Current URL: {url}
Open Tabs: {tabs}
Interactive Elements:
{indexed_elements}
</browser_state>
        """),
        
        # Image Part: Screenshot (if use_vision=True)
        ContentPartImageParam(image_url={
            'url': 'data:image/png;base64,...',
            'media_type': 'image/png'
        }),
        
        # Text Part 4: Read State (if extract/read_file was called)
        ContentPartTextParam(text="""
<read_state>
{one_time_data_from_last_extract_or_read}
</read_state>
        """)
    ])
]
```

### Code-Use Agent Message Format (Each Cell)

```python
messages = [
    # Message 1: System Prompt
    SystemMessage(content=code_use_system_prompt_text),
    
    # Message 2-N: Previous cells (history)
    UserMessage(content="Previous cell code"),
    AssistantMessage(content="Previous cell output"),
    
    # Message N+1: Current context
    UserMessage(content=f"""
User Task: {original_task}

Current Browser State:
{dom_with_indices}

Loading Status: {pending_requests}

Previous Cell Output:
{stdout_from_last_cell}

Write the next code cell to continue the task.
    """)
]
```

---

## Event Bus Integration

All pipelines integrate with the event bus for observability:

```
Agent Events:
├─ CreateAgentSessionEvent (agent initialization)
├─ CreateAgentTaskEvent (task start)
├─ CreateAgentStepEvent (each step)
├─ UpdateAgentTaskEvent (task completion)
└─ CreateAgentOutputFileEvent (file generation)

Browser Events:
├─ BrowserStateUpdateEvent (DOM changes)
├─ ScreenshotCapturedEvent (screenshot taken)
├─ DownloadEvent (file downloaded)
└─ NavigationEvent (page navigation)

Tool Events:
├─ ActionExecutedEvent (action completion)
├─ ActionErrorEvent (action failure)
└─ ExtractCompleteEvent (extraction done)
```

---

*For implementation details, refer to the source files mentioned in each pipeline section.*

