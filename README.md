# Enhanced MCP Server

A Model Context Protocol (MCP) server and command-line client built with [FastMCP](https://github.com/jlowin/fastmcp) and Anthropic's Claude. The server exposes **tools**, **resources**, and **prompts** for working with local files. The client gives you a CLI menu and a Claude chatbot that can call those tools.

## Features

| Capability | Name | What it does |
|---|---|---|
| Tool | `write_file` | Creates a file (and parent directories) with progress reporting |
| Tool | `delete_file` | Deletes a file, refusing directories and missing paths |
| Resource template | `file:///{file_name}` | Reads a file's contents |
| Resource | `dir://.` | Lists files and folders with size and timestamps |
| Prompt | `code_review` | Builds a code review prompt from a file |
| Prompt | `documentation_generator` | Asks you for a source file and doc name (elicitation), then builds a documentation prompt |

The server also uses MCP Context for:

- **Logging**: info, warning, and error messages sent to the client
- **Progress reporting**: chunked writes report percentage complete
- **User elicitation**: structured input requested from the user mid-workflow

## Architecture

```
┌──────────────┐   STDIO (MCP)   ┌──────────────┐
│  client.py   │ ◄─────────────► │  server.py   │
│  CLI + Claude│                 │  FastMCP     │
└──────┬───────┘                 └──────────────┘
       │
       ▼
  Anthropic API
```

The client launches `server.py` as a subprocess and talks to it over standard input/output. When you chat with the agent, Claude receives the server's tools and runs an agentic loop until it has a final answer.

## Requirements

- Python 3.10+
- An [Anthropic API key](https://console.anthropic.com/)

## Installation

```bash
git clone https://github.com/<your-username>/enhanced-mcp-server.git
cd enhanced-mcp-server

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Create a `.env` file in the project root:

```env
ANTHROPIC_API_KEY=your-api-key-here
```

## Usage

Start the client and point it at the server:

```bash
python client.py server.py
```

You'll see the menu:

```
Select from the Menu
1. Generate Documentation
2. Review Code
3. Read File
4. Read Current Directory
5. Converse with Agent
q. Quit
```

| Option | What happens |
|---|---|
| 1 | The server elicits a source file and doc name, then Claude writes the doc using `write_file` |
| 2 | You enter a file path and Claude reviews the code |
| 3 | Reads a file through the `file:///` resource |
| 4 | Lists the current directory through the `dir://.` resource |
| 5 | Free-form chat where Claude can call the file tools |

Example prompts for option 5:

```
Create a file hello.py that prints "Hello, MCP!"
Delete hello.py
```

> Chat, review, and documentation options make Anthropic API calls, so watch your usage.

## Project Structure

```
enhanced-mcp-server/
├── server.py          # FastMCP server: tools, resources, prompts
├── client.py          # MCP client: CLI menu, handlers, Claude loop
├── requirements.txt   # Dependencies
└── .env               # API key (not committed)
```

## Configuration

| Setting | Location | Default |
|---|---|---|
| Claude model | `MODEL_ID` in `client.py` | `claude-sonnet-4-5-20250929` |
| Working root | `BASE_DIR` in `server.py` | Current working directory |
| Max response tokens | `max_tokens` in `client.py` | `4096` |

## Security Notes

All paths pass through `get_path()`, which resolves them and rejects anything outside `BASE_DIR`. This is a learning project, so review and harden it before exposing it to untrusted input. Things to consider:

- Add an allowlist or confirmation step before `delete_file` runs
- Cap file sizes for reads and writes
- Run the server in a sandboxed directory or container

## Troubleshooting

- **`Server script must be a .py, .js, or .ts file`**: pass the server path, e.g. `python client.py server.py`.
- **Module not found**: activate the virtual environment and reinstall with `pip install -r requirements.txt`.
- **Server changes not showing up**: restart the client. The server is a subprocess and doesn't hot reload.
- **Authentication errors**: check that `ANTHROPIC_API_KEY` is set in `.env`.

## Ideas for Extending

- Add tools for file search, code formatting, or database access
- Expose new resources such as system metrics or an external API
- Build multi-step prompts for debugging or data analysis
- Add tests for path handling and tool error cases

