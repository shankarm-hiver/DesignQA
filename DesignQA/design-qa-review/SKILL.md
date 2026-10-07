---
name: "design-qa-review"
description: "Compare a Figma design against a staging build: measure live CSS, run six mandatory checks, annotate screenshots, and file ClickUp tickets. Use when given a Figma URL and a staging URL."
---

# Design QA Review

Compare Figma designs against staging builds. Navigate staging yourself, measure both sides precisely, annotate what is wrong on screenshots, and file ClickUp tickets, so designers don't have to hunt for problems by hand.

You drive the browser. Never ask the designer to upload screenshots, export cookies, or run commands.

## Engine: Chrome MCP

**Use Chrome MCP (Claude in Chrome, the `mcp__claude-in-chrome__*` tools) for all staging work: navigating, clicking and hovering, reading CSS, injecting annotations, and taking screenshots.**

- It uses the designer's signed-in Chrome session, so there's no auth setup and nothing expires.
- Core tools: `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `resize_window`, `computer` (hover, click, screenshot), `javascript_tool` (computed CSS, overlays), `browser_batch` (sequencing), and `find` and `upload_image` (ClickUp attachments). Load them all in one ToolSearch call at the start.
- Call `tabs_context_mcp` first, then open a new tab for the review. Don't reuse the designer's existing tabs.
- Don't use the built-in app browser, Playwright, headless crawlers, Railway or cookie injection.
- If Chrome MCP isn't connected or stops responding, tell the designer and ask how to proceed. Don't switch engines on your own.

## Requirements

- **Chrome MCP**: the Claude in Chrome extension, connected and signed in to staging.
- **Figma MCP**: read only. Never edit the Figma file.
- **ClickUp MCP**: for filing tickets. Ask which workspace and list to file into if the designer hasn't said.

## Triggers

- **Start:** a Figma URL plus a staging URL, or "Review [Figma URL] against [staging URL]"
- **Tickets:** "create all tickets", "critical and major only" or "skip minor ones"

If invoked with no URLs, reply:

> **Design QA ready.**
>
> Paste a Figma URL and a staging URL. I'll capture the Figma frames, walk every user flow and state on staging in your Chrome myself, and check content, components, colour, text, spacing and sizing. Everything is measured from the live DOM, not eyeballed. You'll get annotated screenshots and ClickUp tickets.
>
> No uploads, cookies or API keys. Just the two URLs.

## Workflow

1. List the Figma frames, then map the user flows and states in scope
2. Capture the Figma screens and tokens
3. Walk each flow on staging in Chrome and capture every state
4. Measure and run all six mandatory checks
5. Annotate the issues on screenshots
6. Report the issues, with UX suggestions kept separate
7. File ClickUp tickets once the designer confirms

### 1. Map user flows and states

Before capturing anything, build a coverage map and show it to the designer to confirm.

**User flows.** Identify every flow the design covers, for example "Export accounts: toolbar → export menu → preparing → success toast → notification". Read them from Figma prototype connections, section and frame order, and frame names. For each flow, list:

- Its entry point (where the user starts and what they click)
- Each step and the screen or state it produces
- Its exit points: success, cancel, back, close, and error

**States.** For every screen and component in those flows, list which of these states apply:

- Default, hover, focus, active/pressed, selected, disabled
- Loading/preparing, empty, error, success/confirmation
- Open/closed (modals, dropdowns, popovers, tooltips)
- Content extremes: long text and truncation, many items, a single item
- Role or plan variants (admin vs agent, entitled vs upsell) where the design has them

Show the map as a table: flow → step → state → Figma frame → how to reach it on staging. Mark any state that has **no Figma frame** but exists on staging (or the other way round). Raise those as open questions; don't invent the expected design.

### 2. Capture Figma

- Run `get_metadata` to list sections and frames. If the designer names a feature, filter the list and show a fuzzy-match confidence table before going further.
- Work out whether the frames are **separate screens** or **one screen in different states** (a hover tooltip, an open modal, an empty state), and say which is which.
- Run `get_screenshot` on each frame with `enableBase64Response: true` and `maxDimension: 1440`. Show each one inline and name it. Screenshot node IDs must be frame-level.
- Run `get_variable_defs` on the component node to get colour and type tokens. **Tokens are the source of truth.** Never infer type values from a text node's bounding-box height.
- Extraction tips:
  - When `fontSize`/`fontName` returns `figma.mixed`, use `getStyledTextSegments`.
  - Resolve token names with `figma.variables.getVariableByIdAsync(boundVariables.color.id)`.
  - Skip hidden layers by walking the parent chain and checking `visible === false`. Otherwise they inflate the output.
  - Frame-relative position = `node.absoluteBoundingBox.x - frame.absoluteBoundingBox.x`.
  - Figma screenshot URLs expire within minutes, so attach them to ClickUp straight away.

### 3. Walk flows and capture states in Chrome

- Ask which interaction reveals any state you can't find yourself: a modal behind an icon, a hover, a dropdown, or a scroll position.
- Use `resize_window` to set the window to **1440px**, then confirm with `window.innerWidth` via `javascript_tool`. Chrome can drift between actions, so check again before annotating.
- Use `navigate` to open the staging URL. The session carries over, so expect no auth redirect.
- **Walk each flow end to end** in the order mapped in step 1. Use `computer` to click and hover, and capture every state with `computer` screenshots. Check the transitions too: the right next screen, the right focus, and that close, cancel and back return the user where the design says.
- **Before any action that saves, sends, deletes or changes data**, ask the designer first. Otherwise, click and hover only as much as you need to reach a state.
- Hover styles: in one `browser_batch`, do hover → short wait → `getComputedStyle`, so the read happens while `:hover` is still active.
- Transient states (toasts, loading): hold them by pausing the CSS animation or intercepting the XHR. Undo both before you finish.
- If a state or flow step can't be reached, say so and continue. List it under "Not assessed" with the reason.

### 4. Measure: the six mandatory checks

**Never eyeball spacing, type or colour from images.** Read computed CSS from the live DOM with `javascript_tool` (`getComputedStyle`, `getBoundingClientRect`) and compare it with Figma geometry and tokens. Viewport width never blocks this: a fixed-width component has the same internal geometry in CSS px at any window size.

Run all six checks on **every state** in the coverage map:

1. **Content**: copy, labels, tooltips, missing or extra elements
2. **Component**: the right design-system component and variant
3. **Colour**: text, surfaces, borders, icons, states
4. **Text**: font family, size, line-height, weight
5. **Spacing**: margin, padding, gap
6. **Height & width**

A review isn't complete until all six checks are run on every state. Anything that can't be checked goes under a **"Not assessed"** heading in both the chat summary and the ticket. Leaving something out without saying so is the failure to avoid.

**Look for the systemic cause.** Several deltas with the same value are usually one token fix, not many tickets.

Do **not** flag:

- Dynamic content (names, dates, counts, saved values). That's data, not design.
- Anything you're unsure about
- Differences that could be intentional feature changes
- A width delta on an element whose text also differs. The text drives the width, so defer it and say why.

### 5. Annotate in Chrome

Annotated screenshots are **mandatory**, one per category with issues. Make them on the live staging page in Chrome, not by editing images afterwards. **Annotate only what is wrong**: no ticks or callouts for elements that match.

- Colour code: **red** = content · **blue** = spacing and size · **purple** = text and colour · **orange** = UX suggestion
- Use `javascript_tool` to inject absolutely positioned overlay divs into the live DOM, placed from `getBoundingClientRect()`. Positions must be exact, never placed by hand. A solid box means the element is the wrong size, a shaded band means the gap is wrong, and a dashed outline marks the whole component's size.
- Give each issue a numbered badge, and add a legend that lists each ref as `found → expected`. Badge numbers must match the ticket table.
- Give every overlay one class (e.g. `__qa_ov`) and inject helper functions once on `window.__qa`, then reuse them.
- Put the JS injection and the `computer` screenshot in **one `browser_batch` call**. Otherwise the DOM resets between them.
- Before capturing, move the mouse to a neutral spot and wait, to dismiss stuck tooltips.
- Take the annotated screenshot, remove the overlays with `document.querySelectorAll('.__qa_ov').forEach(e => e.remove())`, then take a plain screenshot.
- Never delete overlays by testing for empty `textContent`. That removes every box.
- Remove all overlays when you're done.

### 6. Report

**Report only what is wrong.** Don't list elements that match the design.

Group issues by flow, then by screen and state, then by severity (🔴 Critical / 🟡 Major / 🔵 Minor).

Explain each issue clearly enough that a developer can fix it without opening Figma:

- **Where:** flow → screen → state → element, plus how to reach it
- **What's wrong:** one plain sentence, e.g. "The Cancel button is narrower than the design, so its label sits off-centre."
- **Found → Expected:** exact measured values or copy, with the token name when there is one
- **Why it matters:** the effect on the user or on consistency
- **Fix:** the specific change, e.g. "use an 8px gap instead of 6px", or the token to use

Then, in a **separate "UX suggestions" section**, list improvements that go beyond the design: places where staging matches Figma but the experience could be better. Examples are unclear copy, a missing confirmation or error state, an awkward flow step, accessibility (contrast, focus visibility, target size), or inconsistency with other parts of the product. For each one, give where it is, what the problem is, the suggestion, and why. Label them clearly as suggestions for the designer to decide on, not bugs. Never mix them into the issues table.

Close with:

> Content ✅ · Component ✅ · Colour ✅ · Text ✅ · Spacing ✅ · Height/Width ✅
> X flows · Y states reviewed · Z issues (A critical, B major, C minor) · D UX suggestions

(A ✅ here means the check was run, not that nothing was found.)

### 7. ClickUp tickets

Ask first: *"Should I create ClickUp tickets? I can file all of them, or just the critical/major ones."*

**Parent task:** `[Screen/Flow] — Design QA — [date]`. Include the flow and state coverage map, every annotated image, the full issues table, "Excluded as data", "Not assessed", and any open questions that need a design call.

**Subtasks** (create them with the parent ID): one per content issue, plus one per category for spacing and one for text/colour. Each one needs where, what's wrong, found → expected, why it matters, fix, the Figma node, and the staging route with the interaction needed to reach the state. Priority: urgent for critical, high for major, normal for minor. Tag `design-qa` if the tag exists in the space.

**UX suggestions:** file them as one separate subtask, `UX suggestions — for design review`, at low priority, so they don't land in a developer's queue as bugs.

**Attachments:**

- **Figma images:** `clickup_attach_task_file` with `file_url` (ClickUp fetches them server-side). ClickUp attachment URLs are permanent and can be reused across tasks.
- **Staging images:** in Chrome, open the task page, `find` its file input, then `upload_image` onto it. Chrome saves screenshots to the designer's machine, not the sandbox, so this is the only route. Open tasks as `https://app.clickup.com/t/<workspace_id>/<task_id>`; the short form redirects to the list view. Uploading injects the image at the top of the description, so rewrite the description afterwards to put it in the right place. Confirm with `clickup_get_task` including attachments.

Finish by confirming with links to the tasks.

## Rules

- **Chrome MCP first.** All navigation, screenshots and annotation go through it. If it's unavailable, stop and ask; don't swap engines.
- **Measure, don't estimate.** If a value wasn't read from computed CSS or a Figma token, don't state it. Be specific: "Cancel button is 70px, should be 94px", not "button looks small."
- **Only what's wrong.** Don't spend output on things that match.
- **No false positives.** If you're not certain, don't flag it.
- **Issues and UX suggestions stay separate.** An issue is a deviation from Figma; a suggestion is an improvement beyond it.
- **Don't switch approaches mid-run.** Fix the current one.
- **Complete output only.** Don't hand over partial content for someone else to assemble.
- **If something was missed,** recheck it right away, then update the subtask, re-annotate and re-upload the corrected image.