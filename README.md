# Tool Routing Directives — Quick Reference

This is a two-file pattern for steering an AI coding assistant's tool use. It's not specific
to any one harness — the same split works in Continue.dev, GitHub Copilot, Cline, Kilo Code,
or any setup that distinguishes between "always-loaded system context" and "on-demand
invokable context."

## The always-on tripwire (a "rule")
**Loads:** Automatically, every turn (scoped to relevant file types)
**What it does:** A ~4-line nudge, cheap enough to sit in context permanently without adding
real latency or noise. It doesn't contain routing logic — it just tells the model "if this
needs live state/docs/spec-checking, use a tool, and the full rules are available on demand."
Keeping this tiny matters: bloating always-on context is the main way you slow down or
degrade a local/small model on every single message, whether or not that message even needs
a tool.

## The full routing logic (a "prompt" / on-demand context)
**Loads:** Only when explicitly invoked
**What it does:** The actual decision tree — which tool source to use for a given request, in
what order, with real tool names and any hard constraints (e.g. a read-only token's
mutation-blocklist). This is where the detail lives, because it only costs context on the
turns where it's actually relevant.

**How they work together:** the tripwire is always watching in the background and can point
the model toward the full ruleset; the full ruleset itself only loads when called, keeping
per-turn cost low the rest of the time.

---

## Harness configuration locations

File paths and terminology differ by harness. This is where each piece lives:

| Harness | Always-on rule location | On-demand prompt/workflow location |
|---|---|---|
| Continue.dev | `.continue/rules/*.md` (`alwaysApply: true`) | `.continue/prompts/*.prompt.md` |
| GitHub Copilot (VS Code) | `.github/copilot-instructions.md` or `.github/instructions/*.instructions.md` (with an `applyTo` glob) | `.github/prompts/*.prompt.md` |
| Cline | `.clinerules/*.md` or a single `.clinerules` file (always loaded) | `.clinerules/workflows/*.md` ("workflows" — inject on-demand only, don't sit in system context) |
| Kilo Code | rules block in `kilo.jsonc` (project root or `.kilo/kilo.jsonc`), or mode-specific rules in `.kilo/rules-{mode}/` | `.kilo/commands/*.md` ("workflows"/slash commands) |

Note: Kilo Code's config format has changed across major versions (legacy `.kilocoderules` /
`.kilocode/workflows/` vs. the current `kilo.jsonc` + `.kilo/commands/` layout) — check your
installed version's docs if the above doesn't match what you see.

---

## Tools available and their focus

- **Playwright (`browser_*`, 20 tools)** — browser automation and runtime inspection: navigate/
  interact with a live page, capture accessibility snapshots or screenshots, read console
  messages and network requests. Used for visual/runtime verification and test debugging.
- **GitHub (read-only tools only)** — live repo state: file contents, commits, branches, tags,
  releases, issues, and pull requests, plus code/repo/user search. Token is read-only, so
  mutation tools (comment, merge, create, update, delete) are explicitly excluded from use.
- **Brave Web Search (`brave_web_search`)** — general external web search for anything current
  or outside the other tools' scope (library comparisons, CVEs, changelogs, general research).
- **Context7 (`resolve-library-id` + `query-docs`)** — current, version-pinned library/framework
  documentation. Two-step: resolve the package name to a Context7 ID, then query docs with it.
- **specification.website** — a 168-topic baseline of web platform standards (Foundations, SEO,
  Accessibility, Security, Well-Known URIs, Agent Readiness, Performance, Privacy, Resilience,
  Internationalisation) sourced from WHATWG/W3C/IETF/WCAG. Used during planning to check for
  gaps against a "correctly built site" baseline — not a coding-time linter.

---

## Invoking the full ruleset

The exact syntax depends on your harness:

| Harness | Typical invocation |
|---|---|
| Continue.dev | `/route-to-tools <your request>` |
| GitHub Copilot (VS Code) | `/route-to-tools <your request>` |
| Cline | `/route-to-tools.md <your request>` (workflow files are invoked with the `.md` extension) |
| Kilo Code | `/route-to-tools <your request>` (from `.kilo/commands/route-to-tools.md`) |

Example (Continue.dev / GitHub Copilot syntax — both use the same `/name` pattern):

```
/route-to-tools Check if the login page renders correctly and show me any console errors
```

Example (Cline syntax — workflows are invoked with the file extension included):

```
/route-to-tools.md Check if the login page renders correctly and show me any console errors
```

This sends the full routing-rules file as context alongside your message, so the model
applies the decision tree to "check if the login page renders" — landing on the
visual/runtime-verification rule and calling the appropriate browser tools in sequence.

A few more examples spanning different rule categories:

```
/route-to-tools Are there any open PRs I haven't reviewed yet?
```
```
/route-to-tools What's the current API for useEffect cleanup in React 19?
```
```
/route-to-tools Before I build the signup page, what am I missing from the security category?
```

If you don't invoke the full ruleset explicitly, the always-on tripwire is still there as a
lighter-weight nudge — but for anything non-trivial, invoking the full prompt directly gives
the model the complete rule set rather than just the reminder.