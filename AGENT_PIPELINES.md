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

