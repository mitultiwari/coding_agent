# Feature Enhancements

Proposed improvements to `coding_agent.py`, grouped by priority. Line numbers refer to the current version of the file.

## P0: Robustness

The agent currently crashes or stops early in common situations. Fix these first.

- [ ] **Catch tool errors.** A missing file makes `read_file` raise, and the whole agent exits. Wrap each tool call in `try/except` and return `{"error": "..."}` as the tool result so the model can recover.
- [ ] **Handle unknown tool names.** `TOOL_REGISTRY[name]` (line 184) raises `KeyError` when the model invents a tool. Return an error listing the valid tool names instead.
- [ ] **Report malformed tool calls.** The parser skips bad `tool:` lines without saying so (line 134). The reply then has no tool calls and is treated as the final answer. Send a parse error back to the model instead.
- [ ] **Reject prose after a tool call.** The model sometimes writes a tool call and then keeps talking on the same line, e.g. `tool: read_file({...})I don't have access to hello_world.py...`. The parser runs the tool and ignores the rest, but the user sees the model guess at a result it hasn't received. Tell the model in the system prompt to stop after a tool call, and ignore or cut off any text that follows it.
- [ ] **Replace the if/elif dispatch** (lines 186-191) with `TOOL_REGISTRY[name](**args)`, so a new tool needs only a function and a registry entry.
- [ ] **Limit tool rounds.** Cap the inner loop (e.g. 25 rounds) so the model can't call tools forever.
- [ ] **Handle API failures.** Retry with backoff on rate limits and timeouts. Catch `KeyboardInterrupt` during a model call so Ctrl+C cancels the current turn, not the program.

## P1: New tools

- [ ] **`run_command`.** Run a shell command with a timeout and capped output. This is the biggest gap: without it the agent can't run tests or check its own changes.
- [ ] **`search`.** Search file contents for a pattern (ripgrep if installed, Python fallback otherwise).
- [ ] **`glob`.** Find files by pattern across subdirectories.
- [ ] **Better `read_file`.** Number the lines, support `offset` and `limit`, and truncate very large files. Remove the debug `print(full_path)` (line 41).
- [ ] **Better `list_files`.** Skip `.git`, `.venv`, `__pycache__` and `node_modules`. Add an optional `recursive` flag.

## P1: Safer edits

- [ ] **Require a unique match.** `edit_file` replaces the first occurrence of `old_str` (line 89). Reject the edit when `old_str` matches more than once and ask the model for more context.
- [ ] **Don't silently overwrite.** An empty `old_str` overwrites an existing file. Refuse unless the file is new or `overwrite: true` is passed.
- [ ] **Confirm before acting.** Show a unified diff (`difflib`) before writing, and show the command before running it. Wait for y/n. Add a `--yes` flag to skip confirmation.
- [ ] **Keep paths in the project.** Reject paths that resolve outside the working directory.

## P2: Native function calling

- [ ] **Use the API's tool support.** Pass `tools=[...]` with JSON schemas to the Responses API instead of parsing `tool:` lines from text. This:
  - removes the parser and its single-line JSON limit;
  - allows several tool calls in one turn;
  - returns results as `function_call_output` items instead of fake `user` messages.
- [ ] **If the text protocol stays,** fix the system prompt. It says "exactly one line" (line 19), but the parser accepts several `tool:` lines.

## P2: Context management

- [ ] **Cap tool result size** before adding it to the conversation.
- [ ] **Compact old history.** Summarise or drop old tool results once the conversation passes a token budget.
- [ ] **Raise or detect the output limit.** `max_output_tokens=5000` can cut off a long `edit_file` call mid-JSON. Check whether the response was truncated and tell the model.
- [ ] **Add environment details to the system prompt:** working directory, OS and date.
- [ ] **Load project instructions** from an `AGENTS.md` file in the working directory when one exists.

## P3: Developer experience

- [ ] **Configurable model.** Read the model name from an env var (e.g. `AGENT_MODEL`) with a default, instead of hard-coding it (line 143).
- [ ] **Support older Python.** Change `f"\n{"="*15}\n"` (line 114) to use `'='` so the script runs on Python 3.8+.
- [x] **Stream responses** so long replies appear as they're generated.
- [ ] **Compact tool display.** Print `→ read_file(coding_agent.py)` instead of the raw `tool:` line. Because replies now stream, this means holding back any line that starts with `tool:` until it's complete, then printing the short form.
- [ ] **Stream errors and interrupts.** A dropped connection or Ctrl+C mid-stream currently ends the program. Catch both, keep the partial text, and return to the `You:` prompt.
- [ ] **Packaging.** Add `requirements.txt` or `pyproject.toml`.
- [ ] **Tests.** Unit tests for `extract_tool_invocations` and `edit_file`; neither needs the API.

## Done

- **Streaming replies** (`execute_llm_call`). The reply is requested with `stream=True` and each `response.output_text.delta` is printed as it arrives. The full text is still returned, so tool parsing is unchanged. Tested against the live API.

## Suggested order

1. P0 robustness: a small change that stops the crashes.
2. `run_command` and `search`: the largest jump in what the agent can do.
3. Safer edits, so it's reasonable to give the agent a shell.
4. Native function calling and context management.
5. Developer experience items as time allows.
