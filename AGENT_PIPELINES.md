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

## 2. Message Assembly Pipeline

This pipeline shows how messages are organized and assembled before being sent to the LLM.

### Message Organization Flow

```
format_messages()
    │
    ├─ format_chat_chunks()
    │   │
    │   ├── Step 1: Choose Code Fence
    │   │    └─ choose_fence() → [opening, closing]
    │   │       • Usually: ["```", "```"]
    │   │       • If code contains ```, use: ["````", "````"]
    │   │
    │   ├── Step 2: Format System Prompt
    │   │    └─ fmt_system_prompt(gpt_prompts.main_system)
    │   │       • See Section 3 for details
    │   │
    │   ├── Step 3: Create ChatChunks Object
    │   │    └─ ChatChunks() with sections:
    │   │       ├─ system: System prompt or user→assistant pair
    │   │       ├─ examples: Example conversations
    │   │       ├─ repo: Repository map (code structure)
    │   │       ├─ readonly_files: Read-only reference files
    │   │       ├─ chat_files: Editable files in chat
    │   │       ├─ done: Previous conversation history
    │   │       ├─ cur: Current user message
    │   │       └─ reminder: System reminder (added last if space)
    │   │
    │   └── Step 4: Token Management
    │        └─ all_messages() assembles final list
    │           • Check token count
    │           • Drop old messages if over limit
    │           • Ensure context window compliance
    │
    └─ add_cache_control_headers() [if caching enabled]
        └─ Mark sections for prompt caching
```

### ChatChunks Structure

The `ChatChunks` class organizes messages into logical sections that are assembled in a specific order:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. SYSTEM PROMPT                                            │
│    From: fmt_system_prompt(gpt_prompts.main_system)       │
│    Prompts: See PROMPTS_DOCUMENTATION.md                   │
│    • EditBlock: "Use SEARCH/REPLACE blocks..."            │
│    • WholeFile: "Output entire file contents..."           │
│    • Patch: "Use V4A diff format..."                       │
│    • Ask: "Act as expert code analyst..."                  │
│    • Architect: "Provide direction to editor..."           │
│    • Context: "Identify files that need modification..."   │
│    • Help: "Expert on Aider usage..."                      │
├─────────────────────────────────────────────────────────────┤
│ 2. EXAMPLE MESSAGES (if applicable)                        │
│    From: gpt_prompts.example_messages                      │
│    • Shows user→assistant conversation examples           │
│    • Demonstrates expected edit format                     │
│    • EditBlock has 2 examples                              │
│    • WholeFile has 1 example                               │
│    • Patch has 2 examples                                  │
├─────────────────────────────────────────────────────────────┤
│ 3. READ-ONLY FILES (reference only)                        │
│    From: self.abs_read_only_fnames                         │
│    Prefix: gpt_prompts.read_only_files_prefix              │
│    • Files for context only                                │
│    • AI instructed not to edit these                       │
├─────────────────────────────────────────────────────────────┤
│ 4. REPOSITORY MAP (if enabled)                             │
│    From: RepoMap.get_repo_map()                            │
│    Prefix: gpt_prompts.repo_content_prefix                 │
│    • Tree-sitter extracted symbols                         │
│    • Classes, functions, methods from repo                 │
│    • Helps AI understand codebase structure                │
├─────────────────────────────────────────────────────────────┤
│ 5. CHAT HISTORY (done_messages)                            │
│    From: self.done_messages                                │
│    • Previous user→assistant exchanges                     │
│    • Summarized if token limit exceeded                    │
│    • See Section 9 for summarization pipeline              │
├─────────────────────────────────────────────────────────────┤
│ 6. CHAT FILES (editable files)                             │
│    From: self.abs_fnames                                   │
│    Prefix: gpt_prompts.files_content_prefix                │
│    • Full contents of files in chat                        │
│    • AI can propose changes to these                       │
│    • Marked as "true contents" to trust over history       │
├─────────────────────────────────────────────────────────────┤
│ 7. CURRENT USER MESSAGE                                    │
│    From: self.cur_messages                                 │
│    • The active request from user                          │
├─────────────────────────────────────────────────────────────┤
│ 8. SYSTEM REMINDER (if space permits)                      │
│    From: fmt_system_prompt(gpt_prompts.system_reminder)   │
│    • Final instructions and rules                          │
│    • Format requirements                                   │
│    • Behavioral reminders                                  │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Through Prompts

```
User Message: "Add a login function to auth.py"
    │
    ↓
Files in Chat: auth.py, database.py
    │
    ↓
Message Assembly:
    │
    ├─ System Prompt (EditBlockPrompts.main_system)
    │   └─ "Act as an expert software developer..."
    │      "Use *SEARCH/REPLACE* blocks..."
    │      "{final_reminders}" → includes lazy_prompt, language
    │
    ├─ Example Messages (EditBlockPrompts.example_messages)
    │   └─ Shows 2 examples of SEARCH/REPLACE format
    │
    ├─ Repository Map
    │   └─ "class AuthManager in auth_manager.py"
    │      "def hash_password() in utils.py"
    │
    ├─ Chat Files
    │   ├─ Prefix: "I have *added these files to the chat*..."
    │   ├─ auth.py: [full contents]
    │   └─ database.py: [full contents]
    │
    ├─ Current Message
    │   └─ "Add a login function to auth.py"
    │
    └─ System Reminder (EditBlockPrompts.system_reminder)
        └─ "# *SEARCH/REPLACE block* Rules:"
           "Every *SEARCH/REPLACE block* must use this format..."
    │
    ↓
Sent to LLM → Response contains SEARCH/REPLACE blocks
```

### Key Files and Classes

| File | Class/Method | Purpose |
|------|--------------|---------|
| `aider/chat_chunks.py` | `ChatChunks` | Organizes message sections |
| `aider/coders/base_coder.py` | `format_messages()` | Main assembly method |
| `aider/coders/base_coder.py` | `format_chat_chunks()` | Creates ChatChunks |
| `aider/coders/base_coder.py` | `choose_fence()` | Selects code fence markers |
| `aider/repomap.py` | `RepoMap.get_repo_map()` | Extracts code symbols |

---

## 3. System Prompt Construction

This section shows how system prompts are built with template variables.

### Prompt Building Pipeline

```
fmt_system_prompt(template)
    │
    ├─ Collect Template Variables:
    │   │
    │   ├─ fence: [opening_fence, closing_fence]
    │   │   └─ From: choose_fence()
    │   │   └─ Example: ["```", "```"] or ["````", "````"]
    │   │
    │   ├─ quad_backtick_reminder
    │   │   └─ If fence is ````, warns to use quadruple backticks
    │   │
    │   ├─ final_reminders
    │   │   ├─ lazy_prompt (if model.lazy == True)
    │   │   │   └─ "You NEVER leave comments without implementing code!"
    │   │   ├─ overeager_prompt (if model.overeager == True)
    │   │   │   └─ "Do what they ask, but no more!"
    │   │   └─ language instruction
    │   │       └─ "Reply in {language}.\n"
    │   │
    │   ├─ platform
    │   │   └─ From: get_platform_info()
    │   │   └─ "Platform: macOS, Python: 3.11, Test: pytest"
    │   │
    │   ├─ shell_cmd_prompt
    │   │   └─ From: shell.py
    │   │   └─ "*Concisely* suggest any shell commands..."
    │   │
    │   ├─ shell_cmd_reminder
    │   │   └─ "Examples of when to suggest shell commands..."
    │   │
    │   ├─ rename_with_shell
    │   │   └─ "To rename files use shell commands..."
    │   │
    │   ├─ go_ahead_tip
    │   │   └─ "If user says 'ok' or 'go ahead'..."
    │   │
    │   └─ language
    │       └─ Detected user language (e.g., "English", "Russian")
    │
    ├─ Format Template String:
    │   └─ template.format(**variables)
    │       • Replaces {fence[0]}, {platform}, {final_reminders}, etc.
    │
    ├─ Handle Model-Specific Additions:
    │   ├─ If model.system_prompt_prefix exists:
    │   │   └─ Prepend to main_system
    │   └─ If model.examples_as_sys_msg:
    │       └─ Append example messages to system prompt
    │
    └─ Return Complete Prompt
```

### Variable Resolution Example

For EditBlock mode with a request in Russian:

```
Input Template (EditBlockPrompts.main_system):
    "Act as an expert software developer.
     {final_reminders}
     Use *SEARCH/REPLACE* blocks..."

Variable Collection:
    fence = ["```", "```"]
    final_reminders = "You are diligent and tireless!\n" +
                      "You NEVER leave comments without implementing code!\n" +
                      "Reply in Russian.\n"
    platform = "Platform: Linux, Python: 3.11, Test: pytest"
    shell_cmd_prompt = "4. *Concisely* suggest any shell commands..."

Formatted Output:
    "Act as an expert software developer.
     You are diligent and tireless!
     You NEVER leave comments without implementing code!
     Reply in Russian.
     Use *SEARCH/REPLACE* blocks..."
```

### Prompt Composition for Different Coders

| Coder Type | Main System Prompt | System Reminder | Example Messages |
|------------|-------------------|-----------------|------------------|
| EditBlock | "Use SEARCH/REPLACE blocks" | SEARCH/REPLACE format rules | 2 examples |
| WholeFile | "Output entire file contents" | File listing format rules | 1 example |
| Patch | "Use V4A diff format" | V4A diff format rules | 2 examples |
| UnifiedDiff | "Write unified diff" | Unified diff rules | 1 example |
| Ask | "Act as code analyst" | Brief responses | None |
| Architect | "Provide implementation direction" | None | None |
| Context | "Identify files to modify" | "NEVER RETURN CODE!" | None |
| Help | "Expert on Aider" | None | None |

### Key Files

| File | Content |
|------|---------|
| `aider/coders/base_coder.py` | `fmt_system_prompt()` method |
| `aider/coders/base_prompts.py` | `CoderPrompts` base class |
| `aider/coders/editblock_prompts.py` | EditBlock prompts |
| `aider/coders/wholefile_prompts.py` | WholeFile prompts |
| `aider/coders/patch_prompts.py` | Patch prompts |
| `aider/coders/udiff_prompts.py` | UnifiedDiff prompts |
| `aider/coders/ask_prompts.py` | Ask prompts |
| `aider/coders/architect_prompts.py` | Architect prompts |
| `aider/coders/context_prompts.py` | Context prompts |
| `aider/coders/help_prompts.py` | Help prompts |
| `aider/coders/shell.py` | Shell command prompts |

---

