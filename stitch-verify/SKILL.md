---
name: stitch-verify
version: 0.1.0
description: |
  Verify that a web app screen matches its Google Stitch design reference.
  Framework-agnostic: works with any web stack (Next.js, Vite, Nuxt, static HTML, etc.).
  Fetches Stitch HTML source via MCP, serves it locally, renders both in headless
  browser, compares computed CSS values, and fixes mismatches in a /loop until
  converged. Use when asked to "verify stitch", "does it match stitch",
  "stitch check", or "compare with stitch".
  Proactively invoke after implementing a screen that has a Stitch reference
  in the plan file or CLAUDE.md. (ystack)
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - mcp__stitch__list_projects
  - mcp__stitch__list_screens
  - mcp__stitch__get_screen
---

# /stitch-verify — Design Convergence Loop

Verify that the implementation matches the Stitch design reference exactly.
Framework-agnostic: compares computed CSS values in a headless browser — works
with any web stack that runs a dev server. The Stitch HTML is the oracle.

## Phase 0: Resolve Stitch Screen

### Input parsing

Accept any of these input forms:
- Screen label: `/stitch-verify "Login"`
- Screen label + project: `/stitch-verify "Login" --project "Elegant PWA Redesign"`
- Screen ID: `/stitch-verify 94cebef1552b466f9d62ffad81edb9fe`
- No args: auto-detect from plan file or CLAUDE.md `## Design Reference` section

### Find the screen

1. If no project specified, use `mcp__stitch__list_projects` and pick the first owned project.
   If multiple projects, use AskUserQuestion to let the user pick.

2. Use `mcp__stitch__get_screen` with the resolved project + screen IDs.

3. If input is a label (not an ID), search `screenInstances` from the project for
   a matching `label` field (case-insensitive partial match). If multiple matches,
   use AskUserQuestion.

### Download source files

```bash
mkdir -p /tmp/stitch-verify
```

1. Download `htmlCode.downloadUrl` → `/tmp/stitch-verify/source.html`
2. Download `screenshot.downloadUrl` → `/tmp/stitch-verify/reference.png`

```bash
curl -sL "<htmlCode.downloadUrl>" -o /tmp/stitch-verify/source.html
curl -sL "<screenshot.downloadUrl>" -o /tmp/stitch-verify/reference.png
```

3. Read `/tmp/stitch-verify/source.html` to extract:
   - **External asset URLs** (from `style="background-image: url('...')"` and `src="..."`)
   - **Stitch Tailwind config** (from `<script id="tailwind-config">`)
   - **Custom CSS** (from `<style>` blocks)

4. Download each external asset to the project's `public/` directory:
   ```bash
   curl -sL "<asset_url>" -o <project>/apps/web/public/<semantic-name>.png
   ```
   Update the component to reference local paths instead of external URLs.

### Detect target file

Look for the target component file. Check in order:
1. User-specified `--target` argument
2. Plan file `## Design Reference` → `Target:` field
3. Infer from the screen label and project structure (e.g., search for files matching the screen name)
4. If ambiguous, use AskUserQuestion

### Write state file

```bash
cat > /tmp/stitch-verify/state.json << 'STATEEOF'
{
  "phase": "verify",
  "tick": 0,
  "max_ticks": 5,
  "converged": false,
  "target_file": "<resolved path>",
  "mismatches": [],
  "viewport": "<390x844 for MOBILE, 1280x720 for DESKTOP>"
}
STATEEOF
```

Set viewport based on `deviceType` from the Stitch screen metadata.

## Phase 1: Start Servers

### Stitch ground truth server

```bash
cd /tmp/stitch-verify && python3 -m http.server 8888 &
STITCH_PID=$!
echo $STITCH_PID > /tmp/stitch-verify/stitch-server.pid
```

### Dev server

Check if already running:
```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:3000 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:3030 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:8080 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:5173 2>/dev/null || echo "NO_SERVER"
```

If NO_SERVER, start it:
```bash
pnpm -C apps/web dev --port 3030 &
DEV_PID=$!
echo $DEV_PID > /tmp/stitch-verify/dev-server.pid
sleep 5
```

Record the dev server port in state.json.

### Setup browse

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/ystack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/ystack/browse/dist/browse"
[ -z "$B" ] && B=~/.claude/skills/ystack/browse/dist/browse
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "BROWSE: $B"
else
  echo "NEEDS_SETUP: run cd ~/.claude/skills/ystack/browse && ./setup"
fi
```

## Phase 2: Verify Loop

This is the core convergence loop. Enter `/loop` (self-paced).

### Each tick:

#### Step 1: Render Stitch ground truth

```bash
$B goto http://localhost:8888/source.html
$B viewport <VIEWPORT>
```

Wait for content to load:
```bash
$B wait --load
```

#### Step 2: Collect expected CSS values

Define the check elements by reading the Stitch HTML structure.
For each significant element in `source.html`, collect computed CSS:

```bash
# Identify key selectors from the Stitch HTML
# These are the elements that define the visual design
SELECTORS=(
  "body"
  "main"
  "h1"
  "button"
  "p"
  "[class*=logo]"
  "[class*=brand]"
  "[style*=background-image]"
)

PROPERTIES=(
  "background-color"
  "color"
  "font-family"
  "font-weight"
  "font-size"
  "letter-spacing"
  "text-transform"
  "opacity"
  "border-radius"
  "padding-top"
  "padding-bottom"
  "padding-left"
  "padding-right"
)
```

For each selector+property combination:
```bash
$B css "<selector>" "<property>"
```

Save all values to `/tmp/stitch-verify/expected-tick-{N}.json`.

#### Step 3: Render Next.js implementation

```bash
$B newtab http://localhost:<DEV_PORT>
$B viewport <VIEWPORT>
$B wait --load
```

#### Step 4: Collect actual CSS values

Same selectors and properties as Step 2:
```bash
$B css "<selector>" "<property>"
```

Save to `/tmp/stitch-verify/actual-tick-{N}.json`.

#### Step 5: Compare text content

```bash
$B tab 1
STITCH_TEXT=$($B text)

$B tab 2
NEXTJS_TEXT=$($B text)
```

Diff the text content. Flag any missing or different text.

#### Step 6: Check for console errors

```bash
$B tab 2
$B console --errors
```

Any JS errors count as a mismatch.

#### Step 7: Take comparison screenshots

```bash
$B tab 1
$B screenshot /tmp/stitch-verify/stitch-tick-{N}.png

$B tab 2
$B screenshot /tmp/stitch-verify/nextjs-tick-{N}.png
```

#### Step 8: Decision

Compare expected vs actual values. Build mismatch list:

```
MISMATCHES:
  h1 font-weight: expected 300, got 400
  button border-radius: expected 12px, got 16px
  [bg-layer] opacity: expected 0.2, got 0
```

**If zero mismatches AND no console errors AND text matches:**
→ CONVERGED. Go to Phase 3.

**If tick >= max_ticks (5):**
→ EXIT with remaining mismatches. Go to Phase 3 with `converged: false`.

**If mismatches found:**
→ FIX each mismatch (see Fix Protocol below)
→ ScheduleWakeup(60s, "re-verifying after fixing {N} CSS mismatches")
→ Continue loop

### Fix Protocol

For each mismatch `{selector, property, expected, got}`:

1. **Find the element in source.html**: Read `/tmp/stitch-verify/source.html`,
   find the element matching the selector, extract its exact classes and inline styles.

2. **Find the same element in the target component**: Read the target `.tsx` file,
   find the corresponding JSX element.

3. **Determine the correct fix**:
   - If it's a Tailwind class issue: identify which class controls the property,
     find the correct class from the Stitch HTML, map it through the project's
     Tailwind config (match by hex value if token names differ).
   - If it's an inline style issue: copy the style value directly.
   - If it's a missing element: the component is missing structure — copy from source.html.
   - If it's a stacking context issue (e.g., opacity: 0 when expected 0.2):
     check if a parent element's background is occluding the target.

4. **Apply the fix**: Edit the target component file.

5. **If a new Tailwind token is needed**: Add it to `tailwind.config.js` with the
   project's naming convention (e.g., `ody-` prefix for this project).

## Phase 3: Report

### If converged:

```
STATUS: DONE
Stitch screen "{screen_label}" verified against Next.js implementation.
Converged after {N} ticks. {total_checks} CSS properties checked.
All values match. No console errors. Text content matches.
```

Copy final screenshots to `docs/screenshots/`:
```bash
cp /tmp/stitch-verify/stitch-tick-{last}.png docs/screenshots/stitch-reference.png
cp /tmp/stitch-verify/nextjs-tick-{last}.png docs/screenshots/stitch-verified.png
git add docs/screenshots/
```

### If not converged (max ticks reached):

```
STATUS: DONE_WITH_CONCERNS
Stitch screen "{screen_label}" — {N} mismatches remain after {max_ticks} iterations.

Remaining mismatches:
  {selector} {property}: expected {expected}, got {got}
  ...

Screenshots attached for human review.
```

Use AskUserQuestion:
- A) Continue fixing (reset tick counter, 5 more iterations)
- B) Accept current state and proceed to /ship
- C) Stop — I'll fix manually

### Cleanup

```bash
# Kill the Stitch server if we started it
[ -f /tmp/stitch-verify/stitch-server.pid ] && kill $(cat /tmp/stitch-verify/stitch-server.pid) 2>/dev/null
```

Do NOT kill the dev server — the user may need it.

## Usage from /ship

When invoked from `/ship` Step 3.46, this skill runs inline with these modifications:
- Skip the preamble (already handled by /ship)
- Use the dev server already detected by /ship
- Report results back to /ship for inclusion in the PR body
- Gate logic: if not converged, /ship asks whether to proceed or fix first

## Token Mapping Reference

When Stitch uses different token names than the project, map by hex value:

```
Stitch class          Hex value    Project class
─────────────────────────────────────────────────
bg-surface            #131318      bg-ody-bg
bg-surface-container  #1f1f25      bg-ody-surface
bg-primary-container  #f5a623      bg-ody-accent
text-on-surface       #e4e1e9      text-ody-text
text-primary          #ffc880      text-ody-accent-light
font-headline         Space Grotesk font-display
font-body             Inter         font-sans
font-label            Space Grotesk font-display
```

Build/update this map on each run by reading both `tailwind.config.js` files.
If a hex value from Stitch has no match in the project config, add it.
