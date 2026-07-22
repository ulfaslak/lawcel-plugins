# Lawcel plugins for Claude Code

[Lawcel](https://lawcel.com) keeps a company's legal documents — privacy policy, terms of
service, DPA, cookie policy — accurate as its product changes. It maintains a **legal
profile** of the organization (the factual claims those documents are generated from) and
flags product changes that contradict them.

This repository is the Claude Code marketplace for Lawcel. It currently ships one plugin.

## Install

```
/plugin marketplace add ulfaslak/lawcel-plugins
/plugin install lawcel
```

Then run any Lawcel tool. Claude Code opens a browser, you sign in to Lawcel and click
**Authorize**, and the connection is live. There is no API key to copy, generate, or paste
anywhere — the plugin config contains no credential, and neither does this repository.

To connect an agent other than Claude Code, see
[Connections → MCP](https://app.lawcel.com/connections/mcp) in Lawcel.

## What the plugin adds

**The hosted Lawcel MCP server** (`https://app.lawcel.com/mcp`, Streamable HTTP, OAuth 2.1):

| Tool | What it does |
|---|---|
| `list_open_profile_questions` | Open legal-profile questions that are answerable from a codebase |
| `submit_profile_answer` | Propose an answer, with required `{ path, line, comment }` evidence |
| `submit_profile_needs_human_review` | Flag a question the code does not settle |
| `list_cases` | The 50 most recently updated compliance cases |
| `list_documents` | Current legal documents, full text |

Everything is read-only except proposing profile answers. There is no tool that creates a
case or edits a document.

**A skill** (`lawcel:legal-profile`) that makes the agent use them. Installing tools is not
the same as an agent reaching for them: without the skill, five tool descriptions sit in
context and wait to be asked about. With it, an agent working in your repository notices that
Lawcel has open questions it can answer from the code in front of it, offers to work through
them, and holds itself to file-and-line evidence for every claim.

The skill covers what good evidence looks like (a call site, not a `package.json` line), when
to flag a question as needing a human instead of answering it, and how to report back.

## What you get out of it

The questions Lawcel needs answered are things like *which third parties receive user data*,
*how long a deleted account is retained*, *which channels you use to contact users*. The
answers are in your repository and nowhere else. An agent that is already reading that
repository can answer them in a few minutes, with citations, instead of a person
reconstructing them from memory.

Every agent answer lands as **pending review**. Nothing is applied to the legal profile and
nothing reaches a published document until someone at your company accepts it in the Lawcel
dashboard, with the evidence references shown alongside. A wrong answer would otherwise
propagate into documents real users read, so the review step is not optional and the plugin
does not offer a way around it.

## Requirements

- A [Lawcel](https://app.lawcel.com) workspace on a **paid plan**. Agent access is not
  included on Free — the browser authorization step will tell you so.
- Claude Code with plugin support.

## Other agents

Plugins are a Claude Code format. Codex, Cursor and other MCP clients connect to the same
server with a two-line config and get the same tools plus the server's built-in usage
instructions; the copy-pasteable snippets are at
[Connections → MCP](https://app.lawcel.com/connections/mcp).

## Support

Issues and pull requests are welcome here. For anything account- or billing-related, contact
support@lawcel.com.

## License

MIT. See [LICENSE](LICENSE).
