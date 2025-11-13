# Aider Agent Pipelines Documentation

This document describes all agent workflows and pipelines in Aider, showing how user requests are processed, which prompts are invoked, and how data flows through the system.

## Table of Contents

1. [Main Request Processing Flow](#1-main-request-processing-flow)
2. [Message Assembly Pipeline](#2-message-assembly-pipeline)
3. [System Prompt Construction](#3-system-prompt-construction)
4. [EditBlock (SEARCH/REPLACE) Pipeline](#4-editblock-searchreplace-pipeline)
5. [WholeFile Pipeline](#5-wholefile-pipeline)
6. [Diff/Patch Pipelines](#6-diffpatch-pipelines)
7. [Specialized Coder Pipelines](#7-specialized-coder-pipelines)
8. [Commit Generation Pipeline](#8-commit-generation-pipeline)
9. [Chat History Management](#9-chat-history-management)

---

## 1. Main Request Processing Flow

This is the core pipeline that processes every user request in Aider.

### High-Level Flow

```
┌──────────────────────────────────────────────────────────────┐
│                      USER INPUT                              │
│  (User types a request or command)                           │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│               CODER INITIALIZATION                           │
│  File: aider/coders/base_coder.py :: Coder.create()        │
│  • Selects appropriate coder based on edit_format          │
│  • Creates EditBlockCoder, WholeFileCoder, PatchCoder, etc. │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│                  MAIN LOOP                                   │
│  File: aider/coders/base_coder.py :: run()                 │
│  • Gets user input via io.get_input()                       │
│  • Calls run_one() for each message                         │
│  • Handles interrupts and commands                          │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│            MESSAGE PREPROCESSING                             │
│  File: aider/coders/base_coder.py :: run_one()             │
│  • Process commands (/add, /drop, /ask, etc.)              │
│  • Detect file mentions in user input                       │
│  • Handle URLs and screenshots                              │
│  • Add files to chat if needed                              │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│             SEND MESSAGE TO LLM                              │
│  File: aider/coders/base_coder.py :: send_message()        │
│  • Add user message to cur_messages                         │
│  • Call format_messages() to assemble full context         │
│  • Check token count and warm cache                         │
│  • Call send() to communicate with LLM                      │
│  • Stream or receive complete response                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│           PROCESS LLM RESPONSE                               │
│  File: aider/coders/base_coder.py :: send_message()        │
│  • Add assistant reply to cur_messages                      │
│  • Check for file mentions in response                      │
│  • Call reply_completed() (coder-specific logic)           │
│  • Parse edits via get_edits()                              │
│  • Apply changes via apply_edits()                          │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│            POST-PROCESSING                                   │
│  File: aider/coders/base_coder.py                          │
│  • auto_commit() - Generate commit message and commit       │
│  • lint_edited() - Run linter if enabled                    │
│  • run_shell_commands() - Execute suggested commands        │
│  • auto_test() - Run tests if enabled                       │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ↓
┌──────────────────────────────────────────────────────────────┐
│            REFLECTION (if needed)                            │
│  • If errors occur or response malformed                    │
│  • max_reflections = 3                                      │
│  • Re-run with error context                                │
│  • Loop back to send_message()                              │
└──────────────────────────────────────────────────────────────┘
                     │
                     ↓
                 READY FOR NEXT INPUT
```

### Prompt Flow in Main Pipeline

```
Step 1: System Prompt Selection
    ├─ Based on edit_format, select prompt class:
    │   ├─ "diff" → EditBlockPrompts
    │   ├─ "whole" → WholeFilePrompts
    │   ├─ "patch" → PatchPrompts
    │   ├─ "udiff" → UnifiedDiffPrompts
    │   ├─ "ask" → AskPrompts
    │   ├─ "architect" → ArchitectPrompts
    │   ├─ "context" → ContextPrompts
    │   └─ "help" → HelpPrompts
    │
Step 2: System Prompt Assembly (see Section 3)
    ├─ main_system prompt
    ├─ Formatted with variables (fence, platform, etc.)
    ├─ example_messages (if applicable)
    └─ system_reminder
    │
Step 3: Context Assembly (see Section 2)
    ├─ System prompt
    ├─ Example messages
    ├─ Read-only files
    ├─ Repository map
    ├─ Chat history (summarized if needed)
    ├─ Editable files content
    ├─ Current user message
    └─ System reminder (if space permits)
    │
Step 4: Send to LLM
    └─ Response contains code changes or analysis
    │
Step 5: Parse Response
    ├─ Coder-specific parsing (get_edits)
    └─ Extract file changes
    │
Step 6: Apply Changes
    ├─ Write to files
    └─ Track changed files
```

### Key Files and Methods

| File | Method | Purpose |
|------|--------|---------|
| `aider/coders/base_coder.py` | `run()` | Main event loop |
| `aider/coders/base_coder.py` | `run_one()` | Process single message |
| `aider/coders/base_coder.py` | `send_message()` | Send to LLM and process response |
| `aider/coders/base_coder.py` | `format_messages()` | Assemble message context |
| `aider/coders/base_coder.py` | `send()` | Communicate with LLM API |
| `aider/coders/base_coder.py` | `apply_updates()` | Apply code changes |
| `aider/coders/base_coder.py` | `auto_commit()` | Generate and create commit |
| `aider/chat_chunks.py` | `ChatChunks` | Organize message chunks |

---

