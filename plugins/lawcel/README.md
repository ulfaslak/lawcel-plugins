# Lawcel

Connects Claude Code to [Lawcel](https://lawcel.com), so an agent working in your codebase can
answer the open legal-profile questions behind your privacy policy, terms of service, DPA and
cookie policy.

## Install

```
/plugin marketplace add ulfaslak/lawcel-plugins
/plugin install lawcel
```

The first tool call opens a browser: sign in to Lawcel, click **Authorize**, done. No API key.

## Contents

- **`.mcp.json`** — the hosted Lawcel MCP server (`https://app.lawcel.com/mcp`, Streamable
  HTTP). Credential-free: authorization is OAuth 2.1, discovered by the client from the
  server's `401` challenge.
- **`skills/legal-profile/SKILL.md`** — when to check for open profile questions, what
  counts as evidence for an answer, when to flag a question for a human instead, and when to
  reach for `list_cases` / `list_documents`.

## Requires

A Lawcel workspace on a paid plan. Agent access is not included on Free.

Full documentation: [the repository README](https://github.com/ulfaslak/lawcel-plugins).
