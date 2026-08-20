---
name: Route to Tools
description: Deterministic tool-selection rules for web dev tasks — call before answering or editing
invokable: true
---

# Tool Routing Directives

You have several MCP tool sources available: many `browser_*` tools (Playwright), many read-only
GitHub tools, `brave_web_search`, two Context7 tools, and `specification`. Before answering, check
this list top to bottom and pick the FIRST rule that matches. If none match, answer from
context/code alone — do not call a tool "just in case."

Available tools:
- `browser_*` (MCP, docker host, Playwright) — 20 discrete tools for browser automation and
  runtime inspection. Most relevant for your workflow (debugging + assisting with test
  writing/verification):
  - Navigate/setup: `browser_navigate`, `browser_navigate_back`, `browser_tabs`, `browser_resize`,
    `browser_close`, `browser_install` (only if a "browser not installed" error occurs)
  - Inspect: `browser_snapshot` (accessibility snapshot — use this, not screenshot, when the next
    step is to act on the page), `browser_take_screenshot` (visual-only, cannot be acted on),
    `browser_console_messages`, `browser_network_requests`
  - Interact: `browser_click`, `browser_type`, `browser_fill_form`, `browser_hover`, `browser_drag`,
    `browser_select_option`, `browser_press_key`, `browser_file_upload`, `browser_handle_dialog`
  - Script/wait: `browser_evaluate`, `browser_wait_for`
- GitHub tools (MCP, docker host) — **read-only token; mutation tools will fail auth and must
  never be called.** Only use:
  - Files/commits: `get_file_contents`, `list_commits`, `get_commit`, `list_branches`,
    `list_tags`, `get_tag`, `list_releases`, `get_latest_release`, `get_release_by_tag`
  - Issues: `issue_read`, `list_issues`, `search_issues`
  - PRs: `pull_request_read`, `list_pull_requests`, `search_pull_requests`
  - Discovery: `search_code`, `search_repositories`, `search_users`
  - Context: `get_me` (only when needed to build another call, e.g. resolving "my" repos)
  - NEVER call: `add_issue_comment`, `assign_copilot_to_issue`, `create_branch`,
    `create_or_update_file`, `create_pull_request`, `create_repository`, `delete_file`,
    `fork_repository`, `issue_write`, `merge_pull_request`, `push_files`,
    `request_copilot_review`, `sub_issue_write`, `update_pull_request`. If the user asks for
    one of these actions, say the connected token is read-only rather than attempting the call.
- `brave_web_search` (MCP, docker host) — general external web search. (Brave's MCP also exposes
  image/news search tools — not relevant for coding tasks, ignore those.)
- `resolve-library-id` + `query-docs` (MCP, docker host, Context7) — two-step: call
  `resolve-library-id` first to turn a package/framework name into a Context7-compatible library ID,
  then `query-docs` with that ID for current, version-pinned documentation. Never call `query-docs`
  with a guessed ID — always resolve first.
- `specification` (MCP, https://mcp.specification.website/mcp) — read-only, no-auth server over
  [The Website Specification](https://specification.website/): 168 platform-agnostic topics across
  10 categories (Foundations, SEO, Accessibility, Security, Well-Known URIs, Agent Readiness,
  Performance, Privacy, Resilience, Internationalisation) — what a "correctly built" website
  includes, sourced from WHATWG/W3C/IETF/WCAG. Not a code-lookup tool; it's a baseline-completeness
  reference, used mainly during planning, not during line-by-line coding.

## Rules (check in order)

1. **Visual/runtime verification of a web page or app**
   Trigger words: "check the page", "does it render", "screenshot", "click through",
   "is the button visible", "test the flow", "console errors", "what does it look like now"
   → Use the specific `browser_*` tool for the action — `browser_snapshot` if the result feeds
   into a further action, `browser_take_screenshot` only if the user wants to *see* the page
   visually, `browser_console_messages`/`browser_network_requests` for error/debug info. Take
   a snapshot before describing state; never assume DOM output from source code alone if a live
   check is possible.

2. **Anything about THIS repo's live state**
   Trigger words: "open PRs", "latest commit", "what changed in", "issue #", "who reviewed",
   "diff between branches", references to a specific PR/issue number
   → Use the matching read-only GitHub tool (see list above — `pull_request_read`/
   `list_pull_requests` for PRs, `issue_read`/`list_issues` for issues, `get_commit`/
   `list_commits` for history, `get_file_contents` for remote file state). Do not guess repo
   state from local files if the question is about remote/live state. If the request implies a
   write (comment, merge, create, update, close), tell the user the token is read-only instead
   of attempting it.

3. **Web app planning / "am I missing something" phase**
   Trigger words: "planning", "before I build", "what do I need for a launch-ready site",
   "what am I missing", "checklist", "is this baseline covered", "new project setup",
   "site audit", or the user naming one of the 10 categories (SEO, accessibility, security,
   well-known URIs, agent readiness, performance, privacy, resilience, i18n, foundations)
   → Use `specification`. Query by category or topic, not by code snippet. This is a
   periodic completeness check, not a per-line linter — use it when scoping a feature/page/
   release, not while writing an individual function. Cross-check the relevant category's
   topic list against what's planned/built and flag gaps (missing security headers, missing
   `/.well-known/` files, missing alt text conventions, missing hreflang, etc.).
   If the user names a specific standard or protocol question that isn't a "baseline
   completeness" question (e.g. "is this valid per the OpenAPI 3.1 spec"), that's outside
   this server's scope — fall through to `brave_web_search` instead.

4. **Library/framework API or usage question**
   Trigger words: "how do I use X in [library]", "current API for", "is this hook/method
   deprecated", "what's the signature for", references to a specific npm/pip package version
   → Call `resolve-library-id`, then `query-docs`, before answering. Do not answer framework/library
   API questions from memory alone — training data on library APIs goes stale fast and this
   is exactly what Context7 is for.

5. **General/current external information**
   Trigger words: "latest version of", "current best practice", "compare X vs Y library",
   "recent CVE", "changelog for", anything about events/releases/prices that could have
   changed after training, and anything not covered by rules 1-4
   → Use `brave_web_search`.

6. **Pure code writing/editing/local reasoning**
   Refactors, writing new components, explaining code already in context, local bug fixes
   with no runtime, repo-state, library-API, or baseline-completeness question attached
   → No tool call. Answer directly from the code in context.

## Conflict resolution

- If a request matches both rule 1 and rule 2 (e.g. "check if the PR's preview deploy
  renders correctly"): call `pull_request_read` to get the preview URL, then `browser_navigate`
  to that URL, then `browser_snapshot` to inspect it. This is a 3-call exception to the 2-tool
  cap below because each call depends on the previous one's output — allow it, but stop there.
- If a planning-phase question (rule 3) also implies a library-specific implementation
  detail (e.g. "does our Next.js setup cover the security headers baseline"): call
  `specification` first to establish what's required, then `resolve-library-id` + `query-docs`
  (Context7) if you need to confirm how the specific framework/library implements it.
- Never call more than 2 tools for a single turn unless the user explicitly asks for a
  multi-step investigation. Small local models lose the thread past that — resolve what
  you can from the first tool's result before deciding whether a second call is needed.
- If a tool call fails or times out, say so plainly and continue with best-effort
  reasoning. Do not silently retry more than once.

## Explicit non-triggers (don't call a tool for these)

- Style/formatting opinions, naming suggestions, explaining existing code
- Writing tests, boilerplate, or config from a known pattern
- Anything answerable with 100% confidence from files already open/in context
- Calling `specification` mid-implementation for every individual tag/attribute — that's
  planning-phase overhead, not a coding lint. Batch these checks at feature/page/release
  scoping time instead.
