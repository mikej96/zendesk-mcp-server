# Zendesk MCP Server

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A Model Context Protocol server for Zendesk.

This server provides a comprehensive integration with Zendesk. It offers:

- Tools for retrieving and managing Zendesk tickets and comments
- Specialized prompts for ticket analysis and response drafting
- Full access to the Zendesk Help Center articles as knowledge base

![demo](https://res.cloudinary.com/leecy-me/image/upload/v1736410626/open/zendesk_yunczu.gif)

## Setup

The published image at [`7sigmasystems/zendesk-mcp-server`](https://hub.docker.com/r/7sigmasystems/zendesk-mcp-server) on Docker Hub is the easiest way to run this server — no build step required. The image is rebuilt and pushed automatically on every push to `main` (see [`.github/workflows/docker-publish.yml`](.github/workflows/docker-publish.yml)).

1. Copy `.env.example` to `.env` and fill in `ZENDESK_SUBDOMAIN`, `ZENDESK_EMAIL`, and `ZENDESK_API_KEY`. Keep this file outside version control.

2. Register the server with Claude Code:

   ```bash
   claude mcp add zendesk -- /usr/bin/docker run --rm -i \
     --env-file /path/to/zendesk-mcp-server/.env \
     7sigmasystems/zendesk-mcp-server:latest
   ```

   Docker will pull the image automatically on first use. Run `which docker` if you need to confirm the binary path on your system.

   To configure by hand instead, add an entry to Claude Code's `settings.json` (or `~/.claude.json` / `.mcp.json`):

   ```json
   {
     "mcpServers": {
       "zendesk": {
         "command": "/usr/bin/docker",
         "args": [
           "run",
           "--rm",
           "-i",
           "--env-file",
           "/path/to/zendesk-mcp-server/.env",
           "7sigmasystems/zendesk-mcp-server:latest"
         ]
       }
     }
   }
   ```

3. Restart Claude Code, then run `/mcp` to verify the `zendesk` server shows as connected.

> **Do not launch the container yourself with `docker compose up` or a bare `docker run`.** This server speaks MCP over STDIN/STDOUT, so it is not a long-running daemon — the MCP client (Claude Code / Claude Desktop) spawns the container on demand so it owns the stdio pipes. A standalone `docker compose up` will print `zendesk mcp server started` and then sit idle with nothing connected.

The image installs dependencies from `requirements.lock`, drops privileges to a non-root user, and expects configuration exclusively via environment variables.

### Building from source

You only need to build locally if you're modifying the server. CI keeps `:latest` on Docker Hub in sync with the tip of `main`.

**Python / `uv`** — run directly from a checkout:

```bash
uv venv && uv pip install -e .   # or `uv build` for a wheel
```

Then point Claude Code at the local checkout:

```json
{
  "mcpServers": {
    "zendesk": {
      "command": "uv",
      "args": ["--directory", "/path/to/zendesk-mcp-server", "run", "zendesk"]
    }
  }
}
```

**Docker** — rebuild the image locally for testing:

```bash
docker compose build              # incremental
docker compose build --no-cache   # clean rebuild
```

This tags the result as `7sigmasystems/zendesk-mcp-server:latest`, **shadowing the published image on this machine** until you `docker pull` again. Your Claude Code config does not need to change — it will pick up whichever `:latest` Docker resolves locally.

## Resources

- zendesk://knowledge-base, get access to the whole help center articles.

## Prompts

### analyze-ticket

Analyze a Zendesk ticket and provide a detailed analysis of the ticket.

### draft-ticket-response

Draft a response to a Zendesk ticket.

## Tools

### get_tickets

Fetch the latest tickets with pagination support

- Input:
  - `page` (integer, optional): Page number (defaults to 1)
  - `per_page` (integer, optional): Number of tickets per page, max 100 (defaults to 25)
  - `sort_by` (string, optional): Field to sort by - created_at, updated_at, priority, or status (defaults to created_at)
  - `sort_order` (string, optional): Sort order - asc or desc (defaults to desc)

- Output: Returns a list of tickets with essential fields including id, subject, status, priority, description, timestamps, and assignee information, along with pagination metadata

### search_tickets

Search Zendesk tickets with free text and field filters

- Input:
  - `query` (string, optional): Free text search query or additional Zendesk search terms
  - `status` (string, optional): one of `new`, `open`, `pending`, `hold`, `solved`, `closed`
  - `priority` (string, optional): one of `low`, `normal`, `high`, `urgent`
  - `assignee` (integer, optional)
  - `requester` (integer, optional)
  - `commenter` (integer, optional)
  - `group` (integer, optional)
  - `organization` (integer, optional)
  - `tags` (array[string], optional)
  - `created_after` (string, optional): ISO8601 date or datetime
  - `created_before` (string, optional): ISO8601 date or datetime
  - `updated_after` (string, optional): ISO8601 date or datetime
  - `updated_before` (string, optional): ISO8601 date or datetime
  - `sort_by` (string, optional): one of `created_at`, `updated_at`, `priority`, `status`
  - `sort_order` (string, optional): `asc` or `desc`
  - `page` (integer, optional): Page number (defaults to 1)
  - `per_page` (integer, optional): Number of tickets per page, max 100 (defaults to 25)

- Output: Returns matching tickets with the applied Zendesk search query and pagination metadata

### get_ticket

Retrieve a Zendesk ticket by its ID

- Input:
  - `ticket_id` (integer): The ID of the ticket to retrieve

### get_ticket_comments

Retrieve all comments for a Zendesk ticket by its ID

- Input:
  - `ticket_id` (integer): The ID of the ticket to get comments for

### create_ticket_comment

Create a new comment on an existing Zendesk ticket

- Input:
  - `ticket_id` (integer): The ID of the ticket to comment on
  - `comment` (string): The comment text/content to add
  - `public` (boolean, optional): Whether the comment should be public (defaults to true)

### create_ticket

Create a new Zendesk ticket

- Input:
  - `subject` (string): Ticket subject
  - `description` (string): Ticket description
  - `requester_id` (integer, optional)
  - `assignee_id` (integer, optional)
  - `priority` (string, optional): one of `low`, `normal`, `high`, `urgent`
  - `type` (string, optional): one of `problem`, `incident`, `question`, `task`
  - `tags` (array[string], optional)
  - `custom_fields` (array[object], optional)

### update_ticket

Update fields on an existing Zendesk ticket (e.g., status, priority, assignee)

- Input:
  - `ticket_id` (integer): The ID of the ticket to update
  - `subject` (string, optional)
  - `status` (string, optional): one of `new`, `open`, `pending`, `on-hold`, `solved`, `closed`
  - `priority` (string, optional): one of `low`, `normal`, `high`, `urgent`
  - `type` (string, optional)
  - `assignee_id` (integer, optional)
  - `requester_id` (integer, optional)
  - `tags` (array[string], optional)
  - `custom_fields` (array[object], optional)
  - `due_at` (string, optional): ISO8601 datetime
