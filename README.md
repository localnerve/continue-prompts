# Tool Routing Directives — Quick Reference

This is a two-file pattern for steering an AI coding assistant's tool use. It's not specific
to any one harness — the same split works in Continue, Cursor, Cline, Windsurf, or any setup
that distinguishes between "always-loaded system context" and "on-demand invokable context."
File names/locations below are the Continue.dev convention; adapt paths to whatever your
harness expects.

## The always-on tripwire (a "rule")
**Continue.dev location:** `.continue/rules/tool-routing-reminder.md`
**VS Code Copilot equivalent:** `.github/copilot-instructions.md` or `.github/instructions/*.instructions.md` (with an `applyTo` glob)
**Loads:** Automatically, every turn (scoped to relevant file types)
**What it does:** A ~4-line nudge, cheap enough to sit in context permanently without adding
real latency or noise. It doesn't contain routing logic — it just tells the model "if this
needs live state/docs/spec-checking, use a tool, and the full rules are available on demand."
Keeping this tiny matters: bloating always-on context is the main way you slow down or
degrade a local/small model on every single message, whether or not that message even needs
a tool.

## The full routing logic (a "prompt" / on-demand context)
**Continue.dev location:** `.continue/prompts/route-to-tools.prompt.md`
**VS Code Copilot equivalent:** `.github/prompts/route-to-tools.prompt.md`
**Loads:** Only when explicitly invoked
**What it does:** The actual decision tree — which tool source to use for a given request, in
what order, with real tool names and any hard constraints (e.g. a read-only token's
mutation-blocklist). This is where the detail lives, because it only costs context on the
turns where it's actually relevant.

**How they work together:** the tripwire is always watching in the background and can point
the model toward the full ruleset; the full ruleset itself only loads when called, keeping
per-turn cost low the rest of the time.

---

## Invoking the full ruleset

The exact syntax depends on your harness:

| Harness | Typical invocation |
|---|---|
| Continue.dev | `/route-to-tools <your request>` |
| GitHub Copilot (VS Code) | `/route-to-tools <your request>` — prompt file lives at `.github/prompts/route-to-tools.prompt.md`; the tripwire equivalent goes in `.github/copilot-instructions.md` or `.github/instructions/*.instructions.md` (always-on, no invocation needed) |
| Cursor | `@route-to-tools <your request>` (as a saved/pinned rule or doc reference) |
| Cline / Windsurf | reference the file path directly, or use their equivalent slash-command/rule syntax |
| Generic / manual | paste or `@`-reference the file's contents alongside your request |

Example (Continue.dev / VS Code Copilot Chat syntax — both use the same `/name` pattern):

```
/route-to-tools Check if the login page renders correctly and show me any console errors
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