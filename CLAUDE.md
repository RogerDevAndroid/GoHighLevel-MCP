# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GoHighLevel MCP Server - An MCP (Model Context Protocol) server that provides 269+ tools for integrating GoHighLevel CRM with Claude Desktop and other MCP-compatible clients. This enables AI-driven automation of contacts, conversations, calendars, opportunities, invoices, payments, and more.

## Build and Development Commands

```bash
# Install dependencies
npm install

# Build TypeScript to dist/
npm run build

# Development with hot reload (HTTP server)
npm run dev

# Production HTTP server (for web/ChatGPT integration)
npm start
# or
npm run start:http

# CLI MCP server (for Claude Desktop via stdio)
npm run start:stdio

# Type checking without emit
npm run lint
```

## Testing Commands

```bash
# Run all tests
npm test

# Run a single test file
npm test -- tests/tools/contact-tools.test.ts

# Run tests matching a pattern
npm test -- --testNamePattern="create_contact"

# Watch mode
npm run test:watch

# Coverage report
npm run test:coverage
```

Tests are in `tests/` directory. Test files follow `*.test.ts` naming convention. Coverage threshold is 70%.

## Environment Configuration

Copy `.env.example` to `.env` and configure with your own credentials:
- `GHL_API_KEY` - GoHighLevel Private Integrations API key (not regular API key)
- `GHL_LOCATION_ID` - GoHighLevel location/sub-account ID
- `GHL_BASE_URL` - API base URL (default: `https://services.leadconnectorhq.com`)

Optional:
- `PORT` - HTTP server port (default: 8000)
- `CORS_ORIGINS` - CORS allowed origins (default: `*`)

## Architecture

### Two Server Entry Points

1. **`src/server.ts`** - CLI MCP server using stdio transport for Claude Desktop integration
2. **`src/http-server.ts`** - HTTP/SSE MCP server using Express for web deployment (Vercel, Railway, etc.)

Both servers share the same tool implementations and GHL API client.

### Core Components

- **`src/clients/ghl-api-client.ts`** - Central API client wrapping Axios for all GoHighLevel API calls. Handles authentication, request/response formatting, and error handling.

- **`src/types/ghl-types.ts`** - TypeScript interfaces for all GHL API requests/responses. Based on OpenAPI specs v2021-07-28 (Contacts) and v2021-04-15 (Conversations).

- **`src/tools/`** - MCP tool implementations organized by domain:
  - `contact-tools.ts` - Contact CRUD, tags, tasks, notes, workflows, campaigns (31 tools)
  - `conversation-tools.ts` - SMS, email, messaging, recordings (20 tools)
  - `opportunity-tools.ts` - Sales pipelines and opportunities (10 tools)
  - `calendar-tools.ts` - Calendars, appointments, scheduling (14 tools)
  - `invoices-tools.ts` - Invoicing, estimates, billing (39 tools)
  - `payments-tools.ts` - Orders, transactions, subscriptions, coupons (20 tools)
  - Plus: blog, email, location, social-media, media, object (custom objects), association, workflow, survey, store, products tools

### Tool Pattern

Each tool class follows the same pattern:
1. Constructor receives `GHLApiClient` instance
2. `getToolDefinitions()` returns MCP Tool[] with name, description, inputSchema
3. `executeTool(name, args)` routes to specific handler methods
4. Handler methods call corresponding GHLApiClient methods

### Server Request Flow

1. MCP client sends `list_tools` request → server aggregates all tool definitions
2. MCP client sends `call_tool` request → server routes to appropriate tool class via `is*Tool()` helper methods
3. Tool class executes via GHLApiClient → returns JSON result

## Key Implementation Details

- Tool routing uses hardcoded name arrays in `is*Tool()` methods in both server files. When adding new tools, update these arrays.
- GHLApiClient methods return typed responses using interfaces from `ghl-types.ts`
- HTTP server exposes `/health`, `/tools`, `/capabilities`, and `/sse` endpoints
- SSE transport used for HTTP MCP protocol (ChatGPT integration)
- stdio transport used for Claude Desktop integration

## Testing the Server

```bash
# Test HTTP server health
curl http://localhost:8000/health

# List available tools
curl http://localhost:8000/tools

# Test SSE endpoint (for MCP clients)
curl -H "Accept: text/event-stream" http://localhost:8000/sse
```

## Adding New Tools

1. Create a new tool class in `src/tools/` following the existing pattern
2. Add corresponding API methods to `GHLApiClient` in `src/clients/ghl-api-client.ts`
3. Add TypeScript interfaces to `src/types/ghl-types.ts`
4. Import and instantiate the tool class in both `src/server.ts` and `src/http-server.ts`
5. Add an `is*Tool()` method with the tool names array for routing
6. Update the `CallToolRequestSchema` handler to route to the new tool class
