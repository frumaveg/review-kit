---
name: change-presentation
description: Build a self-contained HTML slide deck that presents a code change and judges whether it is built correctly, in code-review terms — a review verdict (Approve / Request changes) with its scope, a component diagram of the files involved, a map of where each decision lives, a structure assessment by named criteria with severity and evidence, a comparison to the project's existing pattern, the alternatives considered, a sequence diagram of the flow, tests and risks. Written in the language the user converses in, published as a private Claude Artifact (with a local-file fallback) and handed over as a link. Use when the user runs /change-presentation [branch|PR|commit range|files] [team] [local], or says "present the changes", "make a slide deck of this change", "presentation of the changes", "slides about the structure of this code", "תכיני מצגת של השינויים", "מצגת סיכום שינויים", "תעשי מצגת על השינוי", "מצגת על המבנה של הקוד".
argument-hint: [branch | PR number | commit range | files] [team] [local] (default: current branch against its base, learning mode, Artifact only)
---

# Change presentation

The user reads this deck to answer one question: **is this change built correctly?**
They do not read the diff. The deck therefore has two jobs, in this order:

1. **Judge the structure.** Say whether each piece of logic sits in the right place,
   and prove it from the code. A deck that only describes what changed is a failure.
2. **Teach enough to follow the judgment.** The user is treated as a beginner who does
   not know the code. Every slide makes one idea clear, with a concrete example.

The deck is also shown to other people, so it must read as a professional review:
every claim carries evidence, the tone is factual, and nothing is padded.

## 0. Language

The deck is written in **the language the user converses in** — the language of the
message that asked for the deck, or of the conversation as a whole. Code, paths,
identifiers, class names, route names and the professional vocabulary (criterion names,
verdict words) stay in English in every language.

Decide two template values from the language:

| Language | `{{LANG}}` | `{{DIR}}` |
|---|---|---|
| English, French, Spanish, German, … | the BCP-47 code (`en`, `fr`, …) | `ltr` |
| Hebrew, Arabic, Persian, Urdu, … | `he`, `ar`, … | `rtl` |

The template handles both directions by itself (logical CSS properties, mirrored
arrows and keyboard keys). Everything in this file that says "in the deck language"
means: translate the text; keep the HTML structure and class names exactly as given.

## 1. Resolve what "the change" is

Argument (optional) decides the scope:

| Argument | Scope |
|---|---|
| none | current branch vs. merge-base with the main branch, **plus** uncommitted and untracked changes |
| PR number | `gh pr diff <n>` and `gh pr view <n>` |
| commit range (`a..b`) or single commit | `git diff a..b` / `git show <sha>` |
| file paths | only those files (working tree vs. HEAD, or vs. merge-base if committed) |

Two words may be added to any scope: `team` selects the audience (see "Audience"),
`local` also keeps a copy of the deck on disk (see "Save and publish").

Find the main branch with `git symbolic-ref refs/remotes/origin/HEAD` (fallback `master`/`main`).

**Exclude generated noise** from the deck entirely: build output (`public/*.js`,
`public/css/*`, `public/vendor/**`, `mix-manifest.json`, `dist/`, `build/`), lock files,
minified files, binary files. Mention in the closing slide that they were omitted.

**Parallel agents may share the working tree.** Before writing, print the list of files
you plan to include, as clickable markdown links, so the user can see if another agent's
files slipped in. Do not stop for approval — just show it and continue; drop files that
clearly belong to an unrelated change only if the user says so afterwards.

### Audience

| Mode | When | Effect |
|---|---|---|
| learning (default) | no `team` argument | Full plain-language explanations, a terms slide, every jargon word defined. |
| `team` | the argument contains `team`, or the user says the deck is for other people | Skip the terms slide and the basic definitions. Keep the verdict, diagrams, responsibility map, assessment, comparison and alternatives in full. |

**Both modes use the professional vocabulary**, in English, exactly as a team lead
would say it: the criterion names from step 3, the verdict and severity words, and the
diagram names. Learning mode adds the plain meaning in the deck language beside each
term the first time it appears ("Single source of truth: every rule is defined in one
place"); `team` mode uses the term alone.

## 2. Understand before you write

Read the full diff **and** the surrounding code: callers, routes, models, tests, and the
project's instruction files (`CLAUDE.md`, `AGENTS.md`, `CONTEXT.md`, ADRs). Work out:

- The one-sentence purpose: what the user of the product can now do, or what bug is gone.
- Before vs. after, illustrated with one real example (real ids, names, numbers from
  the code, tests or seeders — e.g. "order 4512 of customer 88 with two refunded lines").
- The real call graph: which file calls which, through which method.
- What the tests prove.

## 3. Review the structure

This step is mandatory and comes before any slide is written. Answer every criterion
below from the code. Each answer is either **ok** or a finding with a severity, and
carries evidence: file, line, and the few lines of code that show it.

| # | Criterion (the name shown in the deck) | Question | How to check |
|---|---|---|---|
| 1 | **Single source of truth (DRY)** | Is each business rule defined once? | Grep for the same condition, constant or query in other files. The same rule in two places is a finding. |
| 2 | **Separation of concerns** | Does each file do only its layer's job? | Calculations or queries inside a controller or a UI component, HTTP concerns inside a service, business rules inside a migration. |
| 3 | **Consistency with conventions** | Is it built the way similar things already are? | Find one existing feature of the same kind and compare file by file. Name it. |
| 4 | **Reuse** | Was something new written where a helper, service, component or library already exists? | Search for the existing one before judging. |
| 5 | **Project rules** | Does it break a rule written in the project's instruction files? | Read them and check each rule that touches the changed area. |
| 6 | **Dependency direction / Coupling** | Do calls go one way (UI → controller → service → model), and does each file depend on as few others as it can? | A service calling a controller, a model reaching for the request, a circular import. |
| 7 | **Maintainability** | What has to be edited in two places, or breaks silently, the next time someone works here? | Think of the most likely follow-up request and trace it. |
| 8 | **Test coverage** | Do the tests cover each rule at the level where it is defined? | A rule in a service tested only through a browser test, or not at all. |

Severity of a finding:

| Severity | Meaning | Style class |
|---|---|---|
| **Minor** | Works and is safe; worth improving. | `warn` |
| **Major** | A structural problem that will cost later; fix before merge. | `bad` |
| **Blocker** | Wrong behavior, data risk or a broken project rule; must not merge. | `bad` |

Rules for honesty:

- **No invented findings.** "ok, nothing found" is a valid answer and most criteria
  will get it. A finding without a file and line is not a finding.
- **No softened findings.** If the rule is duplicated, say it is duplicated.
- A criterion that does not apply to this change (e.g. no UI) is dropped, not answered.

Then derive:

- **The verdict**, in code-review terms: **Approve** (no findings, class `ok`),
  **Approve with comments** (only Minor, class `warn`) or **Request changes** (at least
  one Major or Blocker, class `bad`).
- **The review scope**, stated as fact: the commit sha (or PR number and head sha) the
  review was done on, what was read, and what was *not* done — typically "tests were
  not run", "not checked against production data", "not tried in the browser". If the
  deck is built in the same session that wrote the code, say so ("self-review, written
  in the session that made the change").
- **Alternatives:** one to three other ways the change could have been built, each with
  a real advantage and a real cost, and why the chosen one is better — or, if an
  alternative is better, say so; that becomes a finding.
- **Recommended fixes**, ordered by importance, each tied to a finding.

## 4. Build the deck

Start from [template.html](template.html) in this skill directory. Copy it and replace
every placeholder:

| Placeholder | Value |
|---|---|
| `{{LANG}}`, `{{DIR}}` | from step 0 |
| `{{TITLE}}` | the deck name, two to four words (`Refund weight items`), no colon and no explanation |
| `{{FOOTER}}` | `project · branch or PR · YYYY-MM-DD` |
| `{{NAV_NEXT}}`, `{{NAV_PREV}}` | "Next" / "Previous" in the deck language (button labels for screen readers) |
| `{{SLIDES}}` | the slides |

Do not change the engine or the styles unless a slide genuinely needs a small addition.
The file must stay **fully self-contained**: no external scripts, fonts, CDNs or images,
so it opens offline and inside the Artifact sandbox.

Each slide is `<section class="slide">…</section>` with one `h2`. Slide text is in the
deck language; code, paths and identifiers stay as-is in English. Paths go in
`<span class="path">` and code in `pre.code` so they always render left-to-right. The
engine adds the footer and slide number to every slide and fills the contents list by
itself.

### Slide order

1. **Title** — `.kicker` (project · branch), `h1` change name, `.lead` one sentence of
   what and why, `.meta` chips: date, number of files, `+added / -removed` lines.
2. **Review verdict** — `.verdict.ok|warn|bad` with the verdict in one line
   (Approve / Approve with comments / Request changes) and one sentence of why. Below
   it, the three strongest reasons as a list, each starting with
   `<span class="mark ok|warn|bad">` holding "OK" or the severity word. Last, a
   `p.scope` with the review scope from step 3. A reader who sees only this slide
   knows the state of the change and how far the review went.
3. **Contents** — `<h2>Contents</h2><ol class="toc" id="toc"></ol>` (heading in the deck
   language); the engine fills it. Only in decks of 9 slides or more.
4. **The problem and the solution** — `.grid-2` with `.box.before` / `.box.after`,
   each showing the same concrete example. Add a `.callout` with the key idea.
5. **Terms** (learning mode only, and only if needed) — short table: term → one plain
   sentence. Every jargon word used later appears here or is defined inline with
   `<span class="term">` + `<span class="term-def">`.
6. **Component diagram** — an inline SVG of the files and the calls between them (see
   "Component diagram"), with the full `.legend` below it.
7. **Responsibilities** — `table.resp`, one row per decision the change makes: the
   decision in plain words ("when a line counts as refunded") → where it lives (`.path`
   with file and method) → a `.mark` → why this is or is not the right place. A decision
   that lives in more than one place gets one row listing all of them.
8. **Structure assessment** — `.checks`, one `.check.ok|warn|bad` card per criterion
   from step 3: the criterion name as `h3` followed by a `.mark` with "OK" or the
   severity, the answer in one or two sentences, and `.evidence` with the file and
   line. A finding shows the offending lines in a `pre.code` (5–12 lines, HTML-escaped)
   and one sentence on what the fix would be. Findings come first, most severe first.
   Split over two slides if there are more than four cards with code.
9. **Consistency with the existing pattern** — `.grid-2` with `.box.ref` ("this is how
   X is already built", naming the existing feature) and `.box.after` ("this is how it
   was built here"), the same aspects listed in the same order on both sides, then one
   line: matches, or differs and whether the difference is justified. Drop the slide if
   no comparable feature exists.
10. **Alternatives considered** — `.alts` with one `.alt` per option and `.alt.chosen`
    for the one in the code; each has `ul.pros` and `ul.cons` (the trade-offs). End
    with a `.callout` explaining the choice. Drop the slide if there is no real
    alternative.
11. **Sequence diagram** — an inline SVG of what happens from the user's click (or the
    command / cron trigger) until the data is saved and shown (see "Sequence diagram"),
    with its `.legend`. Learning mode adds two or three plain sentences under it that
    walk through the same example used on the problem slide.
12. **Tests** — table: test name → what it proves in plain words → which decision from
    the responsibility map it covers. Name the decisions no test covers. If there are no
    tests, say so explicitly.
13. **Risks and what to check** — bullets; each risk with a concrete scenario and how to
    verify it manually. Runtime risks only (migrations, data changes, permissions,
    performance, behavior changes for existing records); structural issues already
    appear on slide 8.
14. **Summary and recommended fixes** — the verdict again in one line, the recommended
    fixes as a numbered list ordered by severity, each starting with its `.mark` (or
    "No fixes required" in the deck language), and a short note listing omitted
    generated files.

There are no per-file "what changed" slides and no diff slides. Code appears only as
evidence for a finding or to show a pattern.

Small changes (up to about 5 files): merge slides 2, 7 and 8 into one or two slides,
skip the contents slide, and keep the deck under 9 slides. Never pad.

### Component diagram

An inline `<svg>` inside `<div class="diagram">`, drawn from the real call graph. The
diagram itself is always laid out left-to-right (`.diagram` forces `direction: ltr`);
only the `.d-role` labels are in the deck language.

```html
<div class="diagram"><svg viewBox="0 0 1100 520" role="img" aria-label="component diagram">
  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path class="d-arr" d="M0 0 10 5 0 10z"/></marker>
    <marker id="arr-bad" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path class="d-arr-bad" d="M0 0 10 5 0 10z"/></marker>
  </defs>
  <text class="d-layer" x="10" y="48">Controller</text>
  <rect class="d-box mod" x="150" y="20" width="230" height="56" rx="10"/>
  <text class="d-name" x="265" y="44">OrdersController</text>
  <text class="d-role" x="265" y="63">marks a refunded line</text>
  <path class="d-edge" d="M265 76 L265 150"/>
  <text class="d-call" x="265" y="117">isFullyRefunded()</text>
</svg></div>
```

- One row per layer, top-down in the direction of the calls; `.d-layer` label at the
  left of each row. Boxes are 230×56 with at least 30px between them; grow the
  `viewBox` height by about 130 per row.
- One `.d-box` per file: `.new`, `.mod`, `.del`, or no modifier for an unchanged file
  that is needed to understand the flow. `.d-name` is the class or file name,
  `.d-role` is 3–6 words on its role in this change.
- One `.d-edge` per real call, with a `.d-call` label naming the method. Do not draw an
  edge you did not see in the code.
- A call that goes against the layer direction, or a duplicated rule, is drawn as
  `.d-edge.bad` and must also appear as a finding on the assessment slide.
- Check the coordinates: no box overlaps another, no label sits on a box, every edge
  starts and ends on a box border.
- More than about 12 files: draw only the files that carry a decision, and list the
  rest under the diagram in one line.
- The legend explains every colour and every arrow that appears, and nothing that does
  not (texts in the deck language):

```html
<div class="legend">
  <span><span class="badge new">new</span></span>
  <span><span class="badge mod">changed</span></span>
  <span><span class="badge del">deleted</span></span>
  <span>no colour: unchanged, shown for context</span>
  <span><i class="key-line"></i> calls</span>
  <span><i class="key-line bad"></i> dependency in the wrong direction</span>
</div>
```

### Sequence diagram

The standard UML form: one column per participant, time running top to bottom, one
horizontal arrow per call. It reuses the markers and classes of the component diagram;
define the two `<marker>` elements again inside this `<svg>`.

```html
<div class="diagram"><svg viewBox="0 0 1100 420" role="img" aria-label="sequence diagram">
  <defs><!-- the two markers, as in the component diagram --></defs>
  <rect class="d-box" x="20" y="10" width="170" height="44" rx="8"/>
  <text class="d-name" x="105" y="37">Browser</text>
  <path class="d-life" d="M105 54 L105 400"/>
  <rect class="d-box new" x="240" y="10" width="170" height="44" rx="8"/>
  <text class="d-name" x="325" y="37">Middleware</text>
  <path class="d-life" d="M325 54 L325 400"/>
  <path class="d-edge" d="M105 100 L325 100"/>
  <text class="d-call" x="215" y="92">POST refund_weight_item</text>
  <path class="d-edge ret" d="M325 150 L105 150"/>
  <text class="d-call" x="215" y="142">422 already refunded</text>
</svg></div>
```

- Participants left to right in the order they are first reached: the actor (browser,
  cron, queue) first, the database last. Four to seven participants; group files that
  play no separate part in this flow. Boxes are 170×44, one every 220px, coloured
  `.new` / `.mod` like in the component diagram.
- One `.d-life` under each participant, from its box to the bottom of the diagram.
- One `.d-edge` per call, 50px below the previous one, with a `.d-call` label 8px above
  it: the route, method or query. A response that matters (a rejection, the returned
  value the next step depends on) is a `.d-edge.ret` going back; do not draw a return
  for every call.
- Follow the concrete example from the problem slide. When the change adds a branch
  (rejected vs. accepted), draw the path the change introduces and mention the other in
  one line under the diagram.
- Legend: `<i class="key-line"></i> call` and `<i class="key-line ret"></i> response`
  (in the deck language), plus the colour badges that appear.

## 5. Save and publish

1. **Write the file** to a temporary location first (the session's scratchpad directory
   when one exists, otherwise a temp directory). File name: `<YYYY-MM-DD>-<slug>.html`,
   where slug is the branch name (or `pr-<n>` / short sha), lowercased, `/` replaced
   by `-`.
2. **Verify the file**: it exists, size is reasonable, and it contains no `{{`
   placeholders (`grep -c '{{' <file>` must print 0).
3. **Publish it as a Claude Artifact** — this is the default home of the deck. If an
   `artifact-design` skill is available, load it first (the template already follows its
   contract: colour tokens on `:root`, dark mode, phone width, explicit body
   background). Then call the Artifact tool with the file path, `icon: "slides"` and a
   one-sentence `description` (the verdict and the change, in the deck language). The
   artifact is private by default; the user decides whether to share it. Do not commit
   the file.
4. **Fallback — no Artifact tool** (not available in this session, or the publish is
   refused for a reason you cannot fix): save the deck locally instead, in
   `<presentations_dir>/<project>/`, and say in the hand-over that the Artifact was not
   available. `presentations_dir` is `~/Documents/presentations` unless the project
   has `.claude/review-kit/project.md` with a line `presentations_dir: <path>`. Project
   name = basename of the git root. If a file with the same name exists, overwrite it
   only if it is from the same scope; otherwise add `-2`, `-3`.
5. **`local` argument**: do both — publish the Artifact **and** save the local copy as
   in step 4, and hand over both.

## 6. Hand over the link

End the reply with the Artifact URL on its own line (or, for a local file, the absolute
`file://` URL inside a code block, so it can be pasted into a browser's address bar).

Above the link, give a short summary in the conversation language: the verdict and its
main reason, the findings that need a fix (if any), how many slides the deck has, and
navigation (arrow keys, or the buttons at the bottom; `#5` in the URL jumps to slide 5;
Ctrl+P saves a PDF with one slide per page).
