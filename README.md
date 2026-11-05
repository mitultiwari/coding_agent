# Coding Agent

A minimal coding agent built from scratch in a single Python file. It shows the core loop behind tools like Claude Code, Codex and Cursor: an LLM, a few tools, and a loop that feeds tool results back to the model.

## How it works

1. **Tools**: three plain Python functions:
   - `read_file(filename)`: read a file's contents
   - `list_files(path)`: list a directory
   - `edit_file(path, old_str, new_str)`: replace text in a file, or create the file when `old_str` is empty
2. **System prompt**: each tool's name, signature and docstring go into the prompt. The model is told to request a tool with a single line:
   ```
   tool: read_file({"filename": "coding_agent.py"})
   ```
3. **Agent loop**:
   - Send the conversation to the model.
   - Parse any `tool: ...` lines in the reply.
   - Run those tools and append the output as `tool_result(...)`, then call the model again.
   - When the reply contains no tool calls, print it and wait for the next user message.

The tools are requested in plain text and parsed by the script. OpenAI's native function-calling API is not used. The `conversation` list is the agent's only memory.

## Setup

Requires **Python 3.12+**, because of the f-string syntax in `create_full_system_prompt`.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv
```

Create a `.env` file with your OpenAI API key:

```
OPENAI_API_KEY=sk-...
```

The model is set in `execute_llm_call`. Change it to any model your account can access.

## Usage

```bash
python coding_agent.py
```

Then chat at the `You:` prompt, for example:

```
You: list the files here and summarize coding_agent.py
You: create hello.py that prints "hello world"
```

Relative paths are resolved from the directory where you start the agent. Press `Ctrl+C` or `Ctrl+D` to exit.

## Caveats

- `edit_file` writes to disk with no confirmation step, so run the agent in a directory you don't mind it changing.
- Malformed tool lines are silently ignored, and an unknown tool name will crash the loop.
- The full conversation is re-sent on every call, so long sessions get slower and cost more.
