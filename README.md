# IMAP MCP Server

This repository contains a production-minded, strictly read-only IMAP server for the
Model Context Protocol (MCP). It will expose Streamable HTTP after the IMAP access,
MIME parsing, and tool layers are implemented.

## Current status

The initial project skeleton and configuration model are in place. The server does not
yet connect to IMAP or expose MCP tools.

Configuration is read from environment variables. Copy `.env.example` for local
reference, but do not commit a real environment file. In production, set
`IMAP_PASSWORD_FILE` to a Docker secret mounted at `/run/secrets/imap_password`.
TLS certificate verification will be mandatory in both supported TLS modes.

## Local development

Python 3.11 or newer is required. Create an isolated environment and install the
development dependencies:

```bash
python -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e '.[dev]'
pytest
ruff check .
mypy
```

The next implementation stage adds a mocked IMAP client and MIME parser. Docker,
Compose, Streamable HTTP, and MCP Inspector instructions will be added with their
corresponding implementation.
