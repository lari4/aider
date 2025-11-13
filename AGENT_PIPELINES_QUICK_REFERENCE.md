# Aider Pipelines - Quick Reference Guide

## File Locations & Key Methods

### Base Coder - Main Orchestration
**File:** `/home/user/aider/aider/coders/base_coder.py` (2,450+ lines)

**Key Classes:**
- `Coder` - Main orchestration class

**Key Methods:**
```python
# Initialization
def __init__(self, main_model, io, repo=None, fnames=None, ...)
    → Initializes coder with model, IO handler, files
    
@classmethod
def create(cls, main_model=None, edit_format=None, io=None, from_coder=None, ...)
    → Factory method to create appropriate Coder subclass based on edit_format
    
def clone(self, **kwargs)
    → Creates new coder with same context via from_coder
    
# Main Loop
def run(self, with_message=None, preproc=True)
    → Main REPL loop: gets input, calls run_one in loop
    
def run_one(self, user_message, preproc)
    → Processes single user message:
    1. preproc_user_input() - handle commands, file mentions, URLs
    2. send_message() - send to LLM and process response
    3. Handle reflections (up to max_reflections=3)
    
def get_input(self)
    → Gets next user input from io.get_input()
    
def preproc_user_input(self, inp)
    → Preprocessing: run commands, detect files/URLs
    
# Input Handling
def check_for_file_mentions(self, content)
    → Parses response for file mentions, auto-adds files
    
def check_for_urls(self, inp: str) -> List[str]
    → Detects URLs, offers to add to chat
    
# Message Assembly
def send_message(self, inp)
    → Orchestrates: format → check tokens → send → process response
    ├─ format_messages() - build complete message list
    ├─ warm_cache() - keep cached tokens warm (if enabled)
    ├─ send() - call LLM
    ├─ add_assistant_reply_to_cur_messages() - save response
    ├─ apply_updates() - apply edits from response
    ├─ auto_commit() - commit changes
    ├─ lint_edited() - run linter if auto_lint
    └─ run_shell_commands() - execute suggested commands
    
def format_messages(self)
    → Calls format_chat_chunks() + adds cache headers if enabled
    
def format_chat_chunks(self) → ChatChunks
    → Builds message structure: system + examples + repo + readonly + 
      chat_files + done + cur + reminder
    
def fmt_system_prompt(self, prompt)
    → Formats system prompt with: fence, platform, language, etc.
    
def choose_fence(self)
    → Selects code block delimiters (triple backticks preferred)
    
# File Content Assembly
def get_chat_files_messages(self)
    → Returns list of file contents in chat with path labels
    
def get_readonly_files_messages(self)
    → Returns read-only files for reference
    
def get_repo_messages(self)
    → Returns repo map content (if enabled)
    
def get_files_content(self, fnames=None)
    → Reads file contents
    
# LLM Communication
def send(self, messages, model=None, functions=None)
    → Sends messages to LLM, handles streaming
    ├─ model.send_completion() - calls litellm
    ├─ show_send_output() or show_send_output_stream()
    ├─ calculate_and_show_tokens_and_cost()
    └─ log_llm_history()
    
def show_send_output(self, completion)
    → Processes non-streaming response
    
def show_send_output_stream(self, completion)
    → Processes streaming response, yields chunks
    
# Token Management
def check_tokens(self, messages) → bool
    → Checks if messages fit in model's context window
    
def warm_cache(self, chunks)
    → Spawns thread to keep cached tokens warm (if enabled)
    
def calculate_and_show_tokens_and_cost(self, messages, completion)
    → Computes and displays token usage and cost
    
# Chat History
def summarize_start(self)
    → Starts background thread to summarize done_messages
    
def summarize_end(self)
    → Finalizes summarization if needed
    
def move_back_cur_messages(self, message)
    → Moves cur_messages to done, starts fresh
    
# Response Processing
def add_assistant_reply_to_cur_messages(self)
    → Appends {"role": "assistant", "content": response} to cur_messages
    
def apply_updates(self)
    → Gets edits from response, validates, applies, handles errors
    ├─ get_edits() - parse response
    ├─ apply_edits_dry_run() - validate
    ├─ prepare_to_edit() - check git status
    └─ apply_edits() - write files
    
def get_edits(self, mode="update") → []
    → Stub method - overridden by subclasses to parse edits
    
def apply_edits(self, edits)
    → Stub method - overridden by subclasses to apply edits
    
def apply_edits_dry_run(self, edits) → edits
    → Validates edits without writing
    
def prepare_to_edit(self, edits)
    → Checks git status, prepares for editing
    
# Git & Commits
def auto_commit(self, edited, context=None)
    → Commits edited files with auto-generated message
    
def dirty_commit(self)
    → Commits all dirty files
    
# Shell Commands
def run_shell_commands(self) → str
    → Offers user confirmation, executes suggested shell commands
    
def handle_shell_commands(self, commands_str, group)
    → Processes one shell command block
    
# Linting & Testing
def lint_edited(self, fnames)
    → Runs linter on edited files
    
# File Management
def add_rel_fname(self, rel_fname)
    → Adds file to chat
    
def drop_rel_fname(self, fname)
    → Removes file from chat
    
def get_inchat_relative_files(self) → list
    → Returns list of files in chat (relative paths)
    
def abs_root_path(self, path)
    → Converts relative path to absolute
    
def get_rel_fname(self, fname) → str
    → Converts absolute path to relative
    
# Utilities
def get_announcements(self) → list
    → Returns startup information messages
    
def show_announcements(self)
    → Prints startup announcements
    
def keyboard_interrupt(self)
    → Handles ^C interrupt
    
def get_user_language(self) → str
    → Detects user's language from locale
    
def get_platform_info(self) → str
    → Returns OS/platform info for shell commands
```

### Chat Chunks - Message Organization
**File:** `/home/user/aider/aider/coders/chat_chunks.py`

```python
@dataclass
class ChatChunks:
    system: List[dict]          # System prompt
    examples: List[dict]        # Example messages
    done: List[dict]           # Previous conversation
    repo: List[dict]           # Repo map content
    readonly_files: List[dict] # Read-only files
    chat_files: List[dict]     # Editable files
    cur: List[dict]            # Current message
    reminder: List[dict]       # Final reminder
    
    def all_messages(self) → List[dict]
        → Concatenates all chunks in order
    
    def add_cache_control_headers(self)
        → Adds Claude prompt cache headers
    
    def add_cache_control(self, messages)
        → Marks message chunk as cacheable
    
    def cacheable_messages(self) → List[dict]
        → Returns messages up to cache boundary
```

## Coder Implementations

### EditBlock Coder (SEARCH/REPLACE blocks)
**Files:** `editblock_coder.py`, `editblock_prompts.py`

```python
class EditBlockCoder(Coder):
    edit_format = "diff"
    gpt_prompts = EditBlockPrompts()
    
    def get_edits(self) → List[tuple]
        → Finds SEARCH/REPLACE blocks in response
        ├─ find_original_update_blocks()
        └─ Returns [(path, original, updated), ...]
    
    def apply_edits(self, edits, dry_run=False)
        ├─ do_replace() - performs search/replace
        ├─ On failure, returns detailed error
        └─ Supports "Did you mean" suggestions
    
    def apply_edits_dry_run(self, edits) → edits
        → Validates without writing
```

### Patch Coder (V4A diff format)
**Files:** `patch_coder.py`, `patch_prompts.py`

```python
class PatchCoder(Coder):
    edit_format = "patch"
    gpt_prompts = PatchPrompts()
    
    def get_edits(self) → List[tuple]
        ├─ find_diffs() - finds diff blocks
        └─ Returns [(path, hunk), ...]
    
    def apply_edits(self, edits)
        ├─ normalize_hunk()
        ├─ apply_hunk() with fuzz matching
        └─ Handles SearchTextNotUnique errors
```

### WholeFile Coder (Full file contents)
**Files:** `wholefile_coder.py`, `wholefile_prompts.py`

```python
class WholeFileCoder(Coder):
    edit_format = "whole"
    gpt_prompts = WholeFilePrompts()
    
    def get_edits(self, mode="update"|"diff") → List[tuple]
        ├─ Parses code blocks
        ├─ Extracts filename from previous line
        └─ Returns [(fname, fname_source, new_lines), ...]
    
    def apply_edits(self, edits)
        └─ io.write_text() full file content
    
    def render_incremental_response(self, final) → str
        → Shows live diffs as user types
    
    def do_live_diff(self, full_path, new_lines, final) → list
        → Generates diff for incremental display
```

### UnifiedDiff Coder (Standard unified diff)
**Files:** `udiff_coder.py`, `udiff_prompts.py`

```python
class UnifiedDiffCoder(Coder):
    edit_format = "udiff"
    gpt_prompts = UnifiedDiffPrompts()
    
    def get_edits(self) → List[tuple]
        ├─ find_diffs() - standard diff format
        └─ Returns [(path, hunk), ...]
    
    def apply_edits(self, edits)
        ├─ apply_hunk() with flexible matching
        └─ SearchTextNotUnique error handling
```

### Ask Coder (Read-only analysis)
**Files:** `ask_coder.py`, `ask_prompts.py`

```python
class AskCoder(Coder):
    edit_format = "ask"
    gpt_prompts = AskPrompts()
    
    # Inherits from Coder but get_edits() returns []
```

### Architect Coder (Two-stage planning)
**Files:** `architect_coder.py`, `architect_prompts.py`

```python
class ArchitectCoder(AskCoder):
    edit_format = "architect"
    gpt_prompts = ArchitectPrompts()
    auto_accept_architect = False
    
    def reply_completed(self) → bool
        ├─ If user confirms or auto_accept_architect
        ├─ Create editor_coder with editor_model
        ├─ editor_coder.run(with_message=content)
        └─ Switch implementation to editor coder
```

### Context Coder (File selection)
**Files:** `context_coder.py`, `context_prompts.py`

```python
class ContextCoder(Coder):
    edit_format = "context"
    gpt_prompts = ContextPrompts()
    
    def __init__(self, ...):
        → Increases repo_map size (8x multiplier)
    
    def reply_completed(self) → bool
        ├─ Extracts mentioned files
        ├─ Compares with current chat files
        ├─ Updates abs_fnames if needed
        └─ Returns True to stop (or reflect to fix)
    
    def check_for_file_mentions(self, content)
        → Overridden to do nothing
```

### Help Coder (Interactive help)
**Files:** `help_coder.py`, `help_prompts.py`

```python
class HelpCoder(Coder):
    edit_format = "help"
    gpt_prompts = HelpPrompts()
    
    def get_edits(self, mode="update") → []
    def apply_edits(self, edits) → None
        # Read-only, no edits
```

## Repository Management

**File:** `/home/user/aider/aider/repo.py`

```python
class GitRepo:
    def commit(self, fnames=None, context=None, message=None, 
               aider_edits=False, coder=None) → (str, str) | None
        → Commits edited files with auto-generated or provided message
        ├─ get_diffs(fnames) - Git diffs
        ├─ get_commit_message(diffs, context) - AI-generated message
        ├─ Handle attribution flags (author/committer/co-authored-by)
        ├─ Set git environment variables
        └─ Execute git commit
    
    def get_commit_message(self, diffs, context, user_language=None) → str
        ├─ Build system prompt from prompts.commit_system
        ├─ Call model.simple_send_with_retries()
        ├─ Clean up quotes
        └─ Return commit message
    
    def get_diffs(self, fnames=None) → str
        ├─ git diff HEAD (if commits exist)
        └─ git diff --cached (if new branch)
    
    def get_tracked_files(self) → set
        → Files in git tree
    
    def is_dirty(self) → bool
        → Any uncommitted changes
    
    def path_in_repo(self, path) → bool
        → Is path in this repo
    
    def git_ignored_file(self, path) → bool
        → Does gitignore match path
    
    def ignored_file(self, path) → bool
        → Does .aiderignore match path
```

## Chat History & Summarization

**File:** `/home/user/aider/aider/history.py`

```python
class ChatSummary:
    def __init__(self, models, max_tokens=1024)
    
    def too_big(self, messages) → bool
        → Does message list exceed max_tokens
    
    def summarize(self, messages, depth=0) → List[dict]
        → Recursively compresses messages that exceed max_tokens
    
    def summarize_real(self, messages, depth=0) → List[dict]
        ├─ Check if within token limit
        ├─ Split into head/tail at token boundary
        ├─ Summarize head with summarize_all()
        └─ Recursively compress if still too large
    
    def summarize_all(self, messages) → List[dict]
        ├─ Extract all user/assistant content
        ├─ Call model.simple_send_with_retries() with summarize prompt
        └─ Return single-message summary
    
    def tokenize(self, messages) → List[(tokens, msg)]
        → Count tokens for each message
```

## Repository Map

**File:** `/home/user/aider/aider/repomap.py`

```python
class RepoMap:
    def __init__(self, max_map_tokens, root, main_model, io, 
                 repo_content_prefix, verbose, max_inp_tokens, 
                 map_mul_no_files=8.0, refresh="auto")
    
    def get_repo_map(self, fnames=None, force_refresh=False) → str
        ├─ get_ranked_tags_map() - get important code symbols
        ├─ render_tree() - format as tree
        └─ Return formatted code map
    
    def get_ranked_tags(self, fname, rel_fname) → List[tag]
        ├─ get_tags() - extract from file
        ├─ get_ranked_tags_map() - score by importance
        └─ Return top tags
    
    def get_tags(self, fname, rel_fname) → List[tag]
        ├─ Try cached tags_cache
        ├─ Fallback to get_tags_raw()
        └─ Call tree-sitter parser
    
    def get_tags_raw(self, fname, rel_fname) → List[tag]
        → Extract functions, classes, definitions from file
    
    def render_tree(self, abs_fname, rel_fname, lois) → str
        → Format tagged code sections as tree
    
    def to_tree(self, tags, chat_rel_fnames) → str
        → Convert tags to tree structure
```

## Prompts

**File:** `/home/user/aider/aider/coders/base_prompts.py`

```python
class CoderPrompts:
    main_system = ""
    example_messages = []
    system_reminder = ""
    
    # Commit messages
    files_content_gpt_edits = "I committed the changes with git hash..."
    files_content_gpt_no_edits = "I didn't see any properly formatted edits..."
    files_content_gpt_edits_no_repo = "I updated the files."
    
    # File descriptions
    files_content_prefix = "I have *added these files to the chat*..."
    files_content_assistant_reply = "Ok, any changes I propose..."
    files_no_full_files = "I am not sharing any files yet."
    files_no_full_files_with_repo_map = "Don't try and edit code without..."
    repo_content_prefix = "Here are summaries of some files..."
    read_only_files_prefix = "Here are some READ ONLY files..."
    
    # Model behavior
    lazy_prompt = "You are diligent and tireless!..."
    overeager_prompt = "Pay careful attention to scope..."
    
    # Shell commands
    shell_cmd_prompt = ""
    shell_cmd_reminder = ""
    rename_with_shell = ""
    go_ahead_tip = ""
```

## Main Entry Point

**File:** `/home/user/aider/aider/main.py` (1,274 lines)

```python
def main(argv=None, input=None, output=None, force_git_root=None, 
         return_coder=False) → Coder | None
    → Main CLI entry point
    ├─ Parse args
    ├─ Setup git repo
    ├─ Load models
    ├─ Create coder via Coder.create()
    ├─ Run main loop coder.run()
    └─ Return coder if return_coder=True
```

## Key Data Structures

### Message Format
```python
{
    "role": "user" | "assistant" | "system",
    "content": str | List[dict],  # str or list of content items
    "prefix": bool,  # [optional] For assistant prefill
}

# Content items (for images, etc.):
{
    "type": "text" | "image" | ...,
    "text": str,  # for text
    "image_url": {...},  # for images
    "cache_control": {"type": "ephemeral"},  # [optional] for caching
}
```

### Edit Tuple Format (varies by coder)
```
# EditBlockCoder:
(path, original_text, updated_text)

# PatchCoder:
(path, hunk_diff)

# WholeFileCoder:
(fname, fname_source, new_lines)
```

## Critical Constants

```python
# Max reflections (retries on error)
Coder.max_reflections = 3

# Context window exhaustion tokens
RETRY_TIMEOUT = 60  # seconds exponential backoff max

# Fence characters for code blocks
fence = ("```", "```")  # default, can be quadruple backticks etc.

# Model capabilities
main_model.streaming: bool  # Can stream responses
main_model.use_system_prompt: bool  # Has system role
main_model.examples_as_sys_msg: bool  # Embed examples in system
main_model.use_repo_map: bool  # Should use repo map
```

