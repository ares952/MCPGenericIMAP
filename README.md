# IMAP MCP Server

A production-minded, strictly read-only IMAP server for the Model Context Protocol
(MCP). It exposes Streamable HTTP at `/mcp` and supports exactly these tools:

- `health_check`
- `list_mailboxes`
- `search_messages`
- `get_message`
- `get_attachment`

Messages are identified by mailbox plus IMAP UID. Mailboxes are always selected with
`readonly=True`; the server has no send, flag, move, delete, or raw-command tool.

## Remote SSH development

Clone the repository on the Docker host and open that directory with VS Code Remote -
SSH. Python 3.11 or newer is required for development. Docker and the Compose plugin
are required for container verification.

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e '.[dev]'
pytest
ruff check .
mypy
```

Automated tests use fakes and generated MIME messages; they do not access a mailbox.

## Configuration and secrets

Copy `.env.example` to the ignored `.env` file and set the host, port, username,
mailbox allowlist, and safety limits. TLS certificate verification cannot be disabled.
Supported TLS modes are `implicit_tls` and `starttls`.

Compose expects the password in an ignored file:

```bash
mkdir -p secrets
chmod 700 secrets
printf '%s' 'replace-with-the-real-password' > secrets/imap_password
sudo chown 10001:10001 secrets/imap_password
sudo chmod 600 secrets/imap_password
cp .env.example .env
```

The container runs as the non-root user and group `10001:10001`. File-backed Compose
secrets are bind-mounted with their host ownership and permissions, so a secret owned
by `root:root` with mode `0600` cannot be read by the application. The commands above
make the password readable only by the container's application identity.

To keep `root` as the owner instead, grant read access to the application group:

```bash
sudo chown root:10001 secrets/imap_password
sudo chmod 640 secrets/imap_password
```

Do not use mode `0644` and do not run the container as root merely to access the
secret. After changing ownership or permissions, recreate the service and verify its
health:

```bash
docker compose up -d --force-recreate mcp-server
docker compose ps
docker compose logs --tail=30 mcp-server
```

Do not add the password directly to `.env` when using Compose: the service injects
`IMAP_PASSWORD_FILE=/run/secrets/imap_password`. For a non-Compose launch, set exactly
one of `IMAP_PASSWORD` or `IMAP_PASSWORD_FILE` in the protected process environment.

The limits for results, body bytes, attachment bytes, query length, and socket timeout
are server configuration. Tool callers can request a smaller search page but cannot
raise these limits. Attachments are restricted to PDF, JPEG, PNG, CSV, and plain text;
accepted attachment bytes are returned as `content_base64`.

## Build and startup

```bash
docker compose build
docker compose up -d mcp-server
docker compose ps
docker compose logs --tail=100 mcp-server
```

The published endpoint is `http://127.0.0.1:8700/mcp` by default. It is not reachable
from another host. The image runs as UID 10001, uses a read-only filesystem, and keeps
no persistent mailbox data.

The optional development profile mounts only `src/` read-only and uses port 8701:

```bash
docker compose --profile dev up mcp-server-dev
```

## Verification

Run the complete checks inside the image:

```bash
docker build --target test -t imap-mcp:test .
docker run --rm imap-mcp:test
```

Run the opt-in smoke test only against a disposable account configured through the
same IMAP environment variables:

```bash
IMAP_INTEGRATION_TEST=1 pytest -q tests/test_real_imap_opt_in.py
```

That smoke test lists mailboxes only. Broader real-account validation should verify
all tools using non-sensitive fixture mail while independently checking that no flags
or mailbox state change.

To inspect the running Streamable HTTP endpoint, use MCP Inspector from a machine that
can reach the localhost-bound port, for example on the remote host:

```bash
npx @modelcontextprotocol/inspector http://127.0.0.1:8700/mcp
```

Confirm discovery of exactly the five tools listed above and that every tool has the
read-only annotation.

## Troubleshooting

- Configuration errors name the invalid variable but never print secret values.
- A TLS error normally means the hostname, port, trust store, or server certificate is
  wrong. Certificate verification is intentionally mandatory.
- An authentication error is deliberately generic. Verify the username and secret
  file permissions without printing the password.
- A missing mailbox may mean its exact IMAP name differs from the configured allowlist.
- Missing UID errors can occur after mailbox retention or deletion by another client;
  sequence numbers are never used as identifiers.
- Oversized bodies and attachments are rejected rather than truncated. Unsupported
  attachment types are rejected by default.
- The Docker health check proves that the MCP TCP listener accepts connections; it does
  not authenticate to IMAP.

The Secure MCP Tunnel, ChatGPT Work configuration, and scheduled analysis are deferred
until this local milestone has passed container, Inspector, and disposable-account
verification.
