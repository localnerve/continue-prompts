# Continue.dev Tool Routing — Quick Reference

Two files, two different jobs.

## `.continue/rules/tool-routing-reminder.md`
**Type:** Rule (`alwaysApply: true`)
**Loads:** Automatically, every turn, on `.ts/.tsx/.js/.jsx/.html/.css/.md` files
**What it does:** A ~4-line tripwire. Doesn't contain any routing logic itself — just tells
the model "if this needs live state/docs/spec-checking, use a tool, and the full rules live
at `/route-to-tools`." Cheap enough to run on every message without adding real latency or
context noise.

## `.continue/prompts/route-to-tools.prompt.md`
**Type:** Prompt (`invokable: true`)
**Loads:** Only when you explicitly call it with `/route-to-tools`
**What it does:** The actual decision tree — which of your 6 tool sources (Playwright's
`browser_*` tools, read-only GitHub tools, Brave, Context7, specification.website) to use
for a given request, in what order, with real tool names and the GitHub read-only
mutation-blocklist baked in.

**How they work together:** the rule is always watching in the background and can nudge the
model toward invoking the prompt; the prompt itself only actually loads its full ruleset when
called.

---

## Example: invoking it

Continue.dev slash commands are typed in the chat/agent input box. Type `/`, pick the prompt
from the list (or type its name), then add your actual request after it:

```
/route-to-tools Check if the login page renders correctly and show me any console errors
```

This sends the full `route-to-tools.prompt.md` body as context alongside your message, so the
model applies the routing rules to "check if the login page renders" — landing on rule 1
(visual/runtime verification) and calling `browser_navigate` → `browser_snapshot` →
`browser_console_messages`.

You can invoke it for any of the categories it covers, e.g.:

```
/route-to-tools Are there any open PRs I haven't reviewed yet?
```
```
/route-to-tools What's the current API for useEffect cleanup in React 19?
```
```
/route-to-tools Before I build the signup page, what am I missing from the security category?
```

If you don't invoke `/route-to-tools` explicitly, the always-on rule is still there as a
lighter-weight nudge — but for anything non-trivial, calling the prompt directly gives the
model the full rule set rather than just the reminder.