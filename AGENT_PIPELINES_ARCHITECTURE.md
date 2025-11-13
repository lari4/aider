# Aider Agent Pipelines: Comprehensive Architecture Guide

## 1. MAIN REQUEST FLOW - OVERVIEW

```
┌────────────────────────────────────────────────────────────────────┐
│                    USER INTERACTION LAYER                          │
│                     (main.py :: main())                            │
└────────────────────────────────────────────────────────────────────┘
                                  ↓
┌────────────────────────────────────────────────────────────────────┐
│                    CODER INITIALIZATION                            │
│  Coder.create() selects edit_format, creates appropriate Coder    │
│           (base_coder.py :: Coder.create())                        │
└────────────────────────────────────────────────────────────────────┘
                                  ↓
┌────────────────────────────────────────────────────────────────────┐
│                    MAIN LOOP                                       │
│               (base_coder.py :: run())                             │
│  • Gets user input via io.get_input()                             │
│  • Calls run_one() for each message                               │
│  • Handles KeyboardInterrupt                                      │
└────────────────────────────────────────────────────────────────────┘
                                  ↓
┌────────────────────────────────────────────────────────────────────┐
│                    MESSAGE PROCESSING                              │
│           (base_coder.py :: run_one())                            │
│  1. Preprocess user input (commands, file mentions, URLs)         │
│  2. Call send_message() with user input                           │
│  3. Handle reflections (up to max_reflections=3)                  │
└────────────────────────────────────────────────────────────────────┘
                                  ↓
┌────────────────────────────────────────────────────────────────────┐
│                    MESSAGE ASSEMBLY & SENDING                      │
│           (base_coder.py :: send_message())                        │
│  1. Add user message to cur_messages                              │
│  2. Call format_messages() to build complete message list         │
│  3. Check tokens, warm cache                                      │
│  4. Call send() to send to LLM                                    │
│  5. Handle response streaming/output                              │
│  6. Process response content                                      │
│  7. Apply changes via apply_updates()                             │
│  8. Auto-commit, lint, test                                       │
└────────────────────────────────────────────────────────────────────┘
                                  ↓
┌────────────────────────────────────────────────────────────────────┐
│                    RESPONSE PROCESSING                             │
│  • reply_completed() [subclass-specific logic]                    │
│  • apply_updates() → get_edits() → apply_edits()                  │
│  • auto_commit() [generates commit via repo.commit()]             │
│  • lint_edited() [runs linter if auto_lint=True]                  │
│  • run_shell_commands() [executes suggested shell commands]       │
│  • auto_test() [runs tests if auto_test=True]                    │
└────────────────────────────────────────────────────────────────────┘
```

## 2. MESSAGE FORMATTING PIPELINE

```
send_message(user_input)
    ↓
add to cur_messages = [{"role": "user", "content": user_input}]
    ↓
format_messages()
    ├→ format_chat_chunks()
    │   ├→ Choose fence (code block delimiters)
    │   ├→ Build system prompt via fmt_system_prompt()
    │   ├→ Create ChatChunks with:
    │   │   ├─ system: [system prompt or user->assistant OK pair]
    │   │   ├─ examples: [example messages if applicable]
    │   │   ├─ repo: [repo map content if enabled]
    │   │   ├─ readonly_files: [read-only file contents]
    │   │   ├─ chat_files: [editable files in chat]
    │   │   ├─ done: [previous conversation history]
    │   │   ├─ cur: [current user message]
    │   │   └─ reminder: [final system reminder if room in context]
    │   └→ Handle token counting & context window management
    └→ add_cache_control_headers() [if cache enabled]
        └→ all_messages() returns full message list
    ↓
ChatChunks message structure:
    ┌─────────────────────────────────────┐
    │ System Prompt (or user->assistant)  │
    ├─────────────────────────────────────┤
    │ Example Messages (if applicable)    │
    ├─────────────────────────────────────┤
    │ Read-Only Files Context             │
    ├─────────────────────────────────────┤
    │ Repository Map (if enabled)         │
    ├─────────────────────────────────────┤
    │ Chat History (done_messages)        │
    ├─────────────────────────────────────┤
    │ Chat Files Content                  │
    ├─────────────────────────────────────┤
    │ Current User Message                │
    ├─────────────────────────────────────┤
    │ System Reminder (if fits)           │
    └─────────────────────────────────────┘
```

## 3. LLM INTERACTION FLOW

```
send(messages, model, functions)
    ├→ Log messages to history via io.log_llm_history("TO LLM")
    ├→ model.send_completion(messages, functions, stream, temperature)
    │   └→ [Called to litellm.completion() via models.py]
    ├→ IF streaming:
    │   └→ show_send_output_stream(completion)
    │       ├→ Accumulate streaming chunks
    │       ├→ Handle reasoning content tags
    │       ├→ Update mdstream for live output
    │       └→ Yield text as it arrives
    ├→ ELSE:
    │   └→ show_send_output(completion)
    │       ├→ Extract content/reasoning/tool_calls
    │       └→ Display complete response
    ├→ calculate_and_show_tokens_and_cost()
    └→ Log response via io.log_llm_history("LLM RESPONSE")

Response Content Extraction:
    completion.choices[0].message
        ├─ .content → LLM text response (for edits)
        ├─ .reasoning_content/.reasoning → Extended thinking
        ├─ .tool_calls[0].function.arguments → Function call (if using tools)
        └─ .finish_reason → "stop", "length", "tool_calls"

Reasoning Content Handling:
    • If model supports reasoning (claude-opus, etc.)
    • Wrapped in <reasoning_tag_name> ... </reasoning_tag_name>
    • Stripped before displaying full response
    • Shown separately if desired
```

## 4. SYSTEM PROMPT ASSEMBLY

```
fmt_system_prompt(prompt_template)
    │
    ├─ fmt_system_prompt(gpt_prompts.main_system)
    │   ├─ Format with:
    │   │   ├─ fence: [opening_fence, closing_fence]
    │   │   ├─ quad_backtick_reminder: [warning if using ````]
    │   │   ├─ final_reminders: [lazy/overeager/language prompts]
    │   │   ├─ platform: [OS info: macOS, Linux, Windows]
    │   │   ├─ shell_cmd_prompt: [shell command instructions]
    │   │   ├─ rename_with_shell: [file rename instructions]
    │   │   ├─ shell_cmd_reminder: [final shell reminder]
    │   │   ├─ go_ahead_tip: [tips about accepting edits]
    │   │   └─ language: [detected user language]
    │
    ├─ If model has system_prompt_prefix:
    │   └─ Prepend it to main_system
    │
    ├─ If examples_as_sys_msg:
    │   └─ Append example messages as "## ROLE: content"
    │
    ├─ fmt_system_prompt(gpt_prompts.system_reminder)
    │   └─ Append mode-specific final instructions
    │
    └─ Return fully formatted system prompt

Source Files:
    • base_prompts.py :: CoderPrompts (base class)
    • editblock_prompts.py :: EditBlockPrompts
    • patch_prompts.py :: PatchPrompts
    • wholefile_prompts.py :: WholeFilePrompts
    • udiff_prompts.py :: UnifiedDiffPrompts
    • ask_prompts.py :: AskPrompts
    • architect_prompts.py :: ArchitectPrompts
    • context_prompts.py :: ContextPrompts
    • help_prompts.py :: HelpPrompts
```

## 5. DIFFERENT CODER TYPES & EDIT FORMATS

### 5.1 EditBlock Coder (edit_format = "diff")
```
File: editblock_coder.py
Prompt: editblock_prompts.py

Edit Format: SEARCH/REPLACE blocks
────────────────────────────────
    filename.py
    ```python
    <<<<<<< SEARCH
    old_code_to_find
    =======
    new_code_to_replace_with
    >>>>>>> REPLACE
    ```

Flow:
    1. LLM generates SEARCH/REPLACE blocks
    2. get_edits() parses response to find blocks
    3. find_original_update_blocks() extracts (path, original, updated) tuples
    4. Shell commands extracted if found (no path = shell command)
    5. apply_edits() applies changes:
       ├─ do_replace() performs line-based search/replace
       ├─ Tries perfect match, whitespace-flexible match
       └─ Reports failed blocks for user correction

Key Methods:
    • get_edits() → find_original_update_blocks()
    • apply_edits(edits, dry_run=False)
    • do_replace(full_path, content, original, updated, fence)
    • find_original_update_blocks(content, fence, fnames)
```

### 5.2 Patch Coder (edit_format = "patch")
```
File: patch_coder.py
Prompt: patch_prompts.py

Edit Format: V4A Diff format with markers
──────────────────────────────────────────
    *** Begin Patch
    *** Update File: path/to/file.py
    @@
    -old line
    +new line
    *** End Patch

Flow:
    1. LLM generates patch format
    2. get_edits() calls find_diffs()
    3. Parses unified diff hunks
    4. apply_edits() applies hunks:
       ├─ normalize_hunk() prepares hunk
       ├─ hunk_to_before_after() extracts context
       ├─ apply_hunk() applies context matching with fuzz levels
       └─ Handles multiline, flexible whitespace

Key Methods:
    • get_edits() → find_diffs()
    • apply_edits(edits)
    • do_replace(fname, content, hunk)
    • apply_hunk(content, hunk)
```

### 5.3 WholeFile Coder (edit_format = "whole")
```
File: wholefile_coder.py
Prompt: wholefile_prompts.py

Edit Format: Full file contents in code blocks
───────────────────────────────────────────────
    filename.py
    ```python
    def hello():
        print("hello")
    ```

Flow:
    1. LLM generates full file content
    2. get_edits() parses code blocks:
       ├─ Looks for fence[0] and fence[1]
       ├─ Extracts filename from previous line
       ├─ Handles special cases (**, backticks, #, etc.)
       └─ Returns (fname, fname_source, new_lines)
    3. render_incremental_response() shows live diffs
    4. apply_edits() writes new content:
       └─ io.write_text(full_path, new_content)

Key Methods:
    • get_edits(mode="update" or "diff")
    • apply_edits(edits)
    • do_live_diff(full_path, new_lines, final)
```

### 5.4 UnifiedDiff Coder (edit_format = "udiff")
```
File: udiff_coder.py
Prompt: udiff_prompts.py

Edit Format: Standard unified diff format
──────────────────────────────────────────
    --- path/to/file.py
    +++ path/to/file.py
    @@ -10,7 +10,8 @@
     context_line
    -removed_line
    +added_line
     context_line

Flow:
    1. LLM generates standard unified diff
    2. get_edits() finds diffs in response
    3. apply_edits():
       ├─ normalize_hunk()
       ├─ hunk_to_before_after()
       ├─ apply_hunk() with flexible matching
       └─ Handles SearchTextNotUnique errors

Key Methods:
    • get_edits() → find_diffs()
    • apply_edits(edits)
    • do_replace(fname, content, hunk)
    • apply_hunk(content, hunk)
```

### 5.5 Ask Coder (edit_format = "ask")
```
File: ask_coder.py
Prompt: ask_prompts.py

Purpose: Ask questions about code (read-only)

Flow:
    1. get_edits() returns []
    2. apply_edits() does nothing
    3. LLM response shown as analysis only
    
Key Methods:
    • get_edits() → []
    • apply_edits(edits) → pass
```

### 5.6 Architect Coder (edit_format = "architect")
```
File: architect_coder.py (extends AskCoder)
Prompt: architect_prompts.py

Purpose: AI proposes changes, then switches to editor coder to implement

Flow:
    1. AskCoder collects plan from main_model
    2. reply_completed():
       ├─ If auto_accept_architect or user confirms
       ├─ Create editor_coder with editor_model
       ├─ Switch edit_format to editor_edit_format
       └─ Call editor_coder.run(with_message=content)
    3. Changes are applied by editor_coder
    4. Results combined

Key Attributes:
    • main_model.editor_model
    • main_model.editor_edit_format
    • auto_accept_architect (default True)

Key Methods:
    • reply_completed() - triggers editor coder
```

### 5.7 Context Coder (edit_format = "context")
```
File: context_coder.py
Prompt: context_prompts.py

Purpose: Identify which files need to be edited

Flow:
    1. Uses larger repo_map (map_mul_no_files multiplier)
    2. LLM suggests which files need edits
    3. reply_completed():
       ├─ Extracts mentioned filenames
       ├─ Compares with current files in chat
       ├─ Updates abs_fnames if needed
       └─ Triggers reflection if mismatch
    4. Iterates until correct files identified

Key Methods:
    • reply_completed() - updates file list
    • check_for_file_mentions() - disabled
    • get_file_mentions(content, ignore_current=True)
```

### 5.8 Help Coder (edit_format = "help")
```
File: help_coder.py
Prompt: help_prompts.py

Purpose: Provide interactive help about aider

Flow:
    1. get_edits() returns []
    2. apply_edits() does nothing
    3. Shows help information
```

## 6. RESPONSE PARSING & APPLICATION

```
send_message() after LLM response:
    ├─ self.partial_response_content → contains text
    ├─ self.partial_response_function_call → tool call
    │
    ├→ add_assistant_reply_to_cur_messages()
    │   └─ Appends {"role": "assistant", "content": response} to cur_messages
    │
    ├→ check_for_file_mentions(content)
    │   └─ Parse user references to files, add if needed
    │
    ├→ reply_completed()  [subclass-specific]
    │   └─ Can trigger additional logic (architect, context modes)
    │   └─ Can set reflected_message for reflection loop
    │
    ├→ apply_updates()  [if not interrupted]
    │   ├─ get_edits() → parse response for edits
    │   ├─ apply_edits_dry_run(edits) → validate changes
    │   ├─ prepare_to_edit(edits) → check git, gather metadata
    │   └─ apply_edits(edits) → write files
    │       └─ On ValueError: num_malformed_responses++, set reflected_message
    │
    ├→ auto_commit(edited) [if edited files and auto_commits]
    │   ├─ repo.commit(fnames=edited, context=...)
    │   ├─ Generates commit message via get_commit_message()
    │   ├─ Handles author/committer attribution
    │   ├─ Returns (commit_hash, commit_message)
    │   └─ Adds to cur_messages: saved_message
    │
    ├→ lint_edited(edited) [if auto_lint and edited]
    │   ├─ Runs linter on edited files
    │   └─ Triggers reflection if lint errors and user confirms
    │
    ├→ run_shell_commands() [if suggest_shell_commands]
    │   └─ Asks user to confirm, then executes suggested shell commands
    │
    └→ auto_test() [if auto_test]
        └─ Runs test_cmd on edited files
```

## 7. COMMIT GENERATION PIPELINE

```
auto_commit(edited, context=None):
    ├─ Check: if not repo or not auto_commits or dry_run → return
    ├─ repo.commit(fnames=edited, context=context, aider_edits=True, coder=self)
    │   │
    │   ├─ Check if files are dirty
    │   ├─ get_diffs(fnames) → Git diffs of changes
    │   ├─ get_commit_message(diffs, context, user_language)
    │   │   ├─ Build content from diffs and context
    │   │   ├─ Use commit prompt (default or custom)
    │   │   ├─ Call model.simple_send_with_retries() to generate message
    │   │   └─ Clean up quotes if present
    │   │
    │   ├─ Determine attribution (based on args flags):
    │   │   ├─ attribute_author
    │   │   ├─ attribute_committer
    │   │   ├─ attribute_co_authored_by
    │   │   └─ attribute_commit_message_author/committer
    │   │
    │   ├─ Set git environment variables for author/committer names
    │   ├─ repo.git.commit(message, files)
    │   ├─ Get commit hash
    │   └─ Return (commit_hash, commit_message)
    │
    └─ Move cur_messages: add saved_message with commit info

Commit Message Generation:
    ┌─────────────────────────────────────┐
    │ System Prompt (prompts.commit_system)│
    │ "Generate a single concise commit   │
    │  message that describes the diff."  │
    └─────────────────────────────────────┘
    ┌─────────────────────────────────────┐
    │ User Message:                       │
    │ [Chat context + Git diffs]          │
    │ "# Diffs:\n...<diff output>..."     │
    └─────────────────────────────────────┘
    ↓
    Model generates commit message
```

## 8. CHAT HISTORY & CONTEXT MANAGEMENT

```
Message Accumulation:
    ├─ cur_messages: [{"role": "user"/"assistant", "content": ...}]
    │               Current conversation in this session
    │
    ├─ done_messages: [prev_user, prev_assistant, ...]
    │               Conversation history from previous interactions
    │               Restored on startup if restore_chat_history
    │
    └─ summarized_done_messages: [...]
        Compressed version of done_messages if too large

Chat History Management:
    1. Persistent Storage:
       ├─ io.chat_history_file (markdown format)
       └─ utils.split_chat_history_markdown() parses it
    
    2. Restoration on Startup:
       ├─ restore_chat_history flag
       └─ Calls summarize_start() in background thread
    
    3. Summarization (history.py :: ChatSummary):
       ├─ too_big(messages) checks if exceeds max_tokens
       ├─ summarize_real(messages, depth) recursively summarizes
       ├─ summarize_all(messages) compresses to summary
       └─ Uses weak_model for efficiency
    
    4. Context Window Management:
       ├─ format_chat_chunks() manages token budgets
       ├─ Prioritizes recent messages
       ├─ Drops old messages if needed
       └─ Indicates context exhaustion to user

Key Classes:
    • ChatSummary(models, max_tokens)
    • ChatChunks (dataclass with message categories)
```

## 9. KEY FILE LOCATIONS & CLASS HIERARCHY

```
Base Infrastructure:
    /home/user/aider/aider/coders/base_coder.py
        └─ class Coder(Base orchestration class)
            ├─ __init__(main_model, io, repo, ...)
            ├─ create() [factory method]
            ├─ run() [main loop]
            ├─ run_one(user_message)
            ├─ send_message(inp)
            ├─ send(messages)
            ├─ format_messages()
            ├─ format_chat_chunks()
            ├─ fmt_system_prompt(prompt)
            ├─ apply_updates()
            ├─ get_edits() [stub - overridden by subclasses]
            └─ apply_edits() [stub - overridden by subclasses]

Coder Implementations (edit_format, gpt_prompts):

Editing Coders:
    ├─ editblock_coder.py :: EditBlockCoder ("diff")
    │   └─ gpt_prompts = EditBlockPrompts()
    ├─ patch_coder.py :: PatchCoder ("patch")
    │   └─ gpt_prompts = PatchPrompts()
    ├─ wholefile_coder.py :: WholeFileCoder ("whole")
    │   └─ gpt_prompts = WholeFilePrompts()
    ├─ udiff_coder.py :: UnifiedDiffCoder ("udiff")
    │   └─ gpt_prompts = UnifiedDiffPrompts()
    └─ udiff_simple.py :: UnifiedDiffSimpleCoder ("udiff_simple")
        └─ gpt_prompts = UnifiedDiffSimplePrompts()

Special Mode Coders:
    ├─ ask_coder.py :: AskCoder ("ask")
    │   └─ gpt_prompts = AskPrompts()
    ├─ architect_coder.py :: ArchitectCoder ("architect")
    │   └─ extends AskCoder
    │   └─ gpt_prompts = ArchitectPrompts()
    ├─ context_coder.py :: ContextCoder ("context")
    │   └─ gpt_prompts = ContextPrompts()
    └─ help_coder.py :: HelpCoder ("help")
        └─ gpt_prompts = HelpPrompts()

Prompt Files (all in coders/):
    ├─ base_prompts.py :: CoderPrompts (base class)
    ├─ editblock_prompts.py :: EditBlockPrompts
    ├─ patch_prompts.py :: PatchPrompts
    ├─ wholefile_prompts.py :: WholeFilePrompts
    ├─ udiff_prompts.py :: UnifiedDiffPrompts
    ├─ ask_prompts.py :: AskPrompts
    ├─ architect_prompts.py :: ArchitectPrompts
    ├─ context_prompts.py :: ContextPrompts
    └─ help_prompts.py :: HelpPrompts

Supporting Classes:
    ├─ chat_chunks.py :: ChatChunks (message organization)
    ├─ search_replace.py :: (parsing and applying edits)
    └─ shell.py :: (shell command generation)

Repository & History:
    ├─ /home/user/aider/aider/repo.py
    │   ├─ class GitRepo(Repository management)
    │   └─ def commit() [commit generation]
    └─ /home/user/aider/aider/history.py
        └─ class ChatSummary(Chat compression)

Repo Analysis:
    ├─ /home/user/aider/aider/repomap.py
    │   └─ class RepoMap(Code map generation)
    └─ /home/user/aider/aider/repo.py
        └─ get_repo_map() [in base_coder.py]

Main Orchestration:
    └─ /home/user/aider/aider/main.py
        └─ main() [CLI entry point]
```

## 10. PROMPT VARIABLES & FORMATTING

All prompts are formatted with these variables via `fmt_system_prompt()`:

```python
prompt.format(
    fence=(opening, closing),                 # Triple backticks
    quad_backtick_reminder=warning_text,      # If using ````
    final_reminders=lazy_overeager_language,  # Model behavior reminders
    platform=platform_text,                   # macOS, Linux, Windows, etc.
    shell_cmd_prompt=shell_instructions,      # If suggest_shell_commands
    rename_with_shell=rename_instructions,    # File renaming via shell
    shell_cmd_reminder=final_shell_reminder,  # Closing shell instructions
    go_ahead_tip=tip_text,                    # Accepting edits guidance
    language=detected_user_language,          # User's detected language
)
```

## 11. SPECIALIZED PIPELINES

### 11.1 Architect Pipeline
```
User → ArchitectCoder → Plan generated
                        ↓
                  User confirms or auto-accepts
                        ↓
                  Switch to EditorCoder
                  (editor_model, editor_edit_format)
                        ↓
                  Editor makes code changes
                        ↓
                  Commit and complete
```

### 11.2 Context Selection Pipeline
```
User question → ContextCoder
                ├─ Large repo_map (8x multiplier)
                ├─ Suggests files to edit
                ├─ Extracts mentions via get_file_mentions()
                ├─ Reflects if mismatch with current files
                └─ Adds suggested files to chat
                ↓
            Files confirmed → Ready for editor coder
```

### 11.3 Reflection Loop
```
send_message()
    ↓
Apply edits → Check for errors
    ├─ Malformed response → reflected_message = error
    ├─ Lint errors (if auto_lint) → reflected_message = lint_errors
    ├─ Test errors (if auto_test) → reflected_message = test_errors
    └─ File mentions missed → reflected_message = file suggestions
    ↓
If reflected_message and num_reflections < max_reflections:
    └─ run_one(reflected_message) [retry with context]
```

## 12. TOKEN & COST TRACKING

```
calculate_and_show_tokens_and_cost(messages, completion):
    ├─ input_tokens = model.token_count(messages)
    ├─ output_tokens = completion.usage.completion_tokens
    ├─ cost = compute_costs_from_tokens(input_tokens, output_tokens)
    ├─ Update:
    │   ├─ message_tokens_sent/received
    │   ├─ total_tokens_sent/received
    │   └─ total_cost
    └─ show_usage_report()

Tracking During Session:
    • message_tokens_sent/received: Per-message token count
    • total_tokens_sent/received: Session accumulated
    • message_cost: Cost of current message
    • total_cost: Accumulated session cost
    • usage_report: Displayed to user
```

## 13. CACHING & OPTIMIZATION

```
Prompt Caching (Claude models):
    1. add_cache_headers = True
    2. chunks.add_cache_control_headers()
       ├─ Mark examples chunk with cache_control
       ├─ Mark repo chunk with cache_control
       ├─ Mark chat_files chunk with cache_control
       └─ Format: {"type": "ephemeral"}
    
    3. Cache Warming Thread:
       ├─ Spawned if num_cache_warming_pings > 0
       ├─ Periodically sends minimal requests with cache headers
       ├─ Keeps cached tokens warm (5 min keepalive)
       └─ Tracks cache_hit_tokens for monitoring

Cache Headers Usage:
    • Attached to message content as list:
    ```python
    {"content": [{"type": "text", "text": "...", "cache_control": {"type": "ephemeral"}}]}
    ```
    • Applied to largest semantic chunks
    • Reduces token costs for repeated contexts
```

## 14. ERROR HANDLING & RECOVERY

```
Errors During send_message():

1. LiteLLM Exceptions:
   ├─ ContextWindowExceededError
   │   ├─ exhausted = True
   │   ├─ show_exhausted_error()
   │   └─ num_exhausted_context_windows++
   ├─ Rate limiting, API errors
   │   ├─ Retry with exponential backoff
   │   ├─ retry_delay *= 2 (up to RETRY_TIMEOUT)
   │   └─ Max retries controlled by exception metadata
   └─ KeyboardInterrupt
       └─ interrupted = True, add to cur_messages

2. Response Parsing Errors:
   ├─ FinishReasonLength (output truncated)
   │   ├─ If supports_assistant_prefill:
   │   │   └─ Append previous response, continue
   │   └─ Else:
   │       └─ exhausted = True, show error
   └─ Malformed edits
       ├─ num_malformed_responses++
       ├─ ValueError raised from get_edits()
       ├─ reflected_message = error explanation
       └─ User correction via reflection

3. File System Errors:
   ├─ Git errors → GitRepo exception handling
   ├─ File write errors → io.tool_error()
   └─ Permission errors → appropriate warnings

4. Reflection System:
   └─ On error, re-run with error message as context
       (up to max_reflections=3)
```

## 15. MESSAGES FLOW EXAMPLE

```
Example: User edits file with EditBlockCoder

1. User Input:
   "Add a greeting function to app.py"
   
   cur_messages = [{"role": "user", "content": "Add a greeting..."}]

2. Message Assembly:
   ChatChunks:
   ├─ system: [system prompt for SEARCH/REPLACE blocks]
   ├─ examples: [example SEARCH/REPLACE edits]
   ├─ repo: [repo map if enabled]
   ├─ readonly_files: [read-only reference files]
   ├─ chat_files: [{"role": "user", "content": "Here are the files..."}]
   │             [{"role": "assistant", "content": "Ok, I see the files"}]
   │             [actual file contents with paths]
   ├─ done: [previous conversation if any]
   ├─ cur: [{"role": "user", "content": "Add a greeting function..."}]
   └─ reminder: [final instruction about SEARCH/REPLACE]

3. LLM Response:
   "I'll add a greeting function to app.py.
   
   app.py
   ```python
   <<<<<<< SEARCH
   import sys
   =======
   import sys
   
   def greet(name):
       return f"Hello, {name}!"
   >>>>>>> REPLACE
   ```"

4. Response Processing:
   ├─ get_edits() parses SEARCH/REPLACE blocks
   ├─ Returns [(path, original, updated)]
   ├─ apply_edits_dry_run() validates
   ├─ apply_edits() calls do_replace()
   │   ├─ Finds "import sys" in file
   │   ├─ Replaces with "import sys\n\ndef greet(name)..."
   │   └─ io.write_text(full_path, new_content)
   ├─ add_assistant_reply_to_cur_messages()
   │   └─ cur_messages += [{"role": "assistant", "content": response}]
   ├─ apply_updates() returns {"app.py"}
   ├─ auto_commit({"app.py"})
   │   ├─ Gets diffs of app.py
   │   ├─ Generates "Add greeting function" message
   │   ├─ Commits as "aider: Add greeting function"
   │   └─ cur_messages += ["I committed the changes..."]
   └─ All edits applied and committed

5. Ready for next input
```

