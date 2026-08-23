---
name: docs-from-context
description: Turn any input context - a spec, link, GitHub issue or PR, code sample, transcript, email thread, folder, or pasted notes - into new or updated Microsoft Teams developer documentation in msteams-docs, then open a draft pull request so a human reviews and approves before anything ships.
version: 1.0.0
---

# Docs from Context — Teams Developer Documentation Skill

Ingest whatever context you are given, decide whether it means **new documentation** or an **update to existing documentation**, write it to msteams-docs conventions, and raise a **draft pull request** for human review.

## Quick summary

```
/docs-from-context <context> [scope-hint]
```

The skill reads the context, grounds every claim in it, finds the right article (or creates one), updates TOC and related navigation, validates the result, and opens a draft PR. **The draft PR is the human-in-the-loop gate.** The skill never merges, never pushes to `main`, and never marks the PR ready for review.

---

## Operating contract

### Human-in-the-loop

- The run is **fully automated up to the draft PR**. Do not stop mid-run to ask questions.
- Every judgment call the skill had to make, every gap in the context, and every unverified claim goes into the **Needs human decision** section of the PR body. Do not silently guess.
- The PR is opened as a **draft**. Never mark it ready for review, never merge, never request reviewers automatically unless the user asked for it.
- Never push directly to `main` or any protected branch.

### Grounding contract (most important rule)

- **Every technical statement must trace back to the supplied context or to an existing, verifiable source in the repo.**
- If the context does not say it, do not write it. Use a `<!-- NEEDS-INPUT: ... -->` marker instead and list it in the PR body.
- Never invent API names, parameter names, types, defaults, limits, error codes, supported clients, or release dates.
- Never state availability ("generally available", "public preview", "rolling out in March") unless the context states it explicitly.

### Safety contract

- Documentation files only. No product code, no build config, no workflow files.
- Never delete an article. If an article is superseded, replace its content and keep the file, or move it and add a redirect entry — never leave a dead published URL.
- Never delete an image or asset that is still referenced anywhere in the repo.
- Preserve all existing frontmatter fields. Only `ms.date` and genuinely content-driven fields change.
- Make surgical edits to existing articles. Do not reflow, reorder, or reformat untouched sections — it makes review impossible.

### Style contract

- Microsoft Learn style: second person ("you"), present tense, active voice, short scannable sentences.
- Sentence case headings. No gerund-only headings where an imperative is clearer.
- **Neighboring articles win.** When generic guidance and the local folder's established pattern disagree, follow the folder.

---

## Invocation

```
/docs-from-context <context> [scope-hint]
```

**`context`** *(required)* — one or more of, in any combination:

| Input type | How it is handled |
|---|---|
| Pasted text / notes / requirements | Read as the primary spec |
| Local file path (`.md`, `.docx`, `.json`, source code) | Read fully; for code, extract public surface and usage |
| Folder path | Enumerate recursively; skip binaries, `node_modules`, build output |
| URL (spec, blog, Learn page, SDK reference) | Fetch; follow at most one level of directly relevant in-page links |
| GitHub issue or PR URL / number | Read title, body, labels, linked items, and the full diff for PRs |
| Meeting transcript or email thread | Extract decisions and requirements; treat opinions as non-authoritative |
| Screenshots / diagrams | Use as described assets; never infer API behavior from a picture |

**`scope-hint`** *(optional)* — a folder or file path that narrows the target, for example `msteams-platform/bots/` or `msteams-platform/tabs/what-are-tabs.md`.

Examples:

```
/docs-from-context https://github.com/OfficeDev/.../issues/482
/docs-from-context C:\specs\streaming-ux.docx msteams-platform/bots/
/docs-from-context C:\repos\sdk-samples\emoji-reactions\ 
/docs-from-context "PM notes: the manifest now supports a new `scopes` value ..."
```

---

## Workflow

### Phase 0 — Ingest and normalize the context

1. Resolve every supplied input to raw text. For mixed inputs, read all of them before deciding anything.
2. Build a **context record**:
   - Feature or change name
   - One-paragraph capability summary
   - Audience (bot dev, tab dev, agent dev, admin, ISV)
   - APIs, parameters, return values, manifest properties, permissions/scopes
   - Code snippets, tagged by language
   - Prerequisites, limitations, known issues
   - Supported surfaces (Teams, Outlook, Microsoft 365 Copilot) and platforms (desktop, web, mobile, GCC/GCCH/DoD)
   - Provided images and their intended captions
   - Explicit **unknowns** — anything a doc normally needs that the context does not supply
3. Print:
   ```
   📥 Context: <n> source(s) — <short list>
   📋 Feature: <name>
   ❓ Unknowns: <count>
   ```

> [!IMPORTANT]
> If the context is too thin to write anything factual (no API surface, no behavior described, no scope), stop before writing and report what is missing. Do not open an empty or speculative PR.

---

### Phase 1 — Classify: new vs. update

1. Search the repo for the feature name, API names, manifest properties, and close synonyms across `msteams-platform/**/*.md` and `TOC.yml`.
2. Decide:
   - **`update`** — an existing article already owns this topic (title, headings, or API names overlap strongly). Record every affected file, not just the best match.
   - **`new`** — no article owns it. Record the proposed path.
   - **`mixed`** — a new article is needed *and* existing articles must link to it or drop superseded content.
3. Prefer `update` when in doubt. A new article that duplicates an existing one is worse than an expanded existing one.
4. Print `🎯 Task type: <new|update|mixed>` and the file list.

---

### Phase 2 — Locate the target and its TOC placement

Map the feature to a doc area under `msteams-platform/`:

| Folder | Covers |
|---|---|
| `agents-in-teams/` | Agents built with the Teams AI / Agents SDK |
| `bots/` | Bot framework features |
| `tabs/` | Tab apps |
| `messaging-extensions/` | Message extensions |
| `apps-in-teams-meetings/` | Meeting, call, and Live Share apps |
| `task-modules-and-cards/` | Dialogs and Adaptive Cards |
| `webhooks-and-connectors/` | Webhooks and connectors |
| `m365-apps/` | Microsoft 365 cross-platform apps |
| `concepts/` | Cross-cutting: auth, design, build, deploy, publish, manifest |
| `graph-api/` | Graph API integrations |
| `toolkit/` | Microsoft 365 Agents Toolkit |
| `get-started/` | Onboarding and quickstarts |
| `resources/` | References, schemas, samples |

Then:

1. Read `msteams-platform/TOC.yml`. Find the parent node whose children share the same audience and conceptual level.
2. Respect the parent node's existing ordering convention (build → integrate → test → publish, or basic → advanced).
3. For `update`, confirm the existing TOC entry is still correctly placed; move it only if the article's scope genuinely changed, and say so in the PR body.
4. Print `📂 Area: <folder>` and `🧭 TOC: <parent> › <child>`.

If placement is genuinely ambiguous, pick the closest sibling, and flag the alternative in **Needs human decision**.

---

### Phase 3 — Discover repo conventions

Read 2–4 adjacent articles in the target folder and extract:

- Frontmatter shape actually in use: `title`, `description`, `ms.topic`, `ms.localizationpriority`, `ms.date`, `author`, `ms.author`, `ms.owner`, `zone_pivot_groups`.
- Heading depth, naming pattern, and how "Prerequisites" / "Next step" / "See also" are used.
- Include usage — check `msteams-platform/includes/` for content that already exists. **Reuse an include instead of duplicating prose.**
- Zone pivots — check `msteams-platform/zone-pivot-groups.yml` for existing group IDs before inventing one.
- Image pattern — `:::image type="content" source="~/assets/images/<area>/<file>.png" alt-text="...":::`.
- Link style — relative `.md` paths with anchors inside the repo; absolute Learn URLs for external content.
- Callouts — `> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!CAUTION]`.

**Branding check** — normalize to current names:

| Use | Not |
|---|---|
| Microsoft 365 Agents Toolkit | Teams Toolkit |
| Microsoft 365 Agents Playground | Teams App Test Tool |
| app manifest | Teams app manifest |
| Microsoft 365 Copilot | Microsoft Copilot (for the Copilot surface) |
| Microsoft Teams | Teams (on first mention in an article) |
| Microsoft Entra ID | Azure AD, AAD |

Print `✅ Conventions discovered`.

---

### Phase 4 — Write or update

#### New article

Frontmatter:

```yaml
---
title: <Sentence-case, searchable, ≤60 chars>
description: <Learn how to… — one or two sentences, ≤160 chars, includes the primary keyword>
ms.topic: conceptual | how-to | reference | overview
ms.localizationpriority: medium
ms.date: <MM/DD/YYYY — today>
---
```

Body structure (adapt to the folder's pattern):

1. `# Title`
2. **Intro** — 2–3 sentences: what it is, who it is for, why it matters.
3. **Prerequisites** — only if real ones exist in the context.
4. **How it works / concept sections** — one idea per heading.
5. **Steps** — numbered, imperative, one action per step, for how-to articles.
6. **Code samples** — fenced blocks with a language tag, copy-paste ready. Multi-language uses the repo's tab syntax:
   ```
   # [C#](#tab/csharp)
   ...
   # [JavaScript](#tab/javascript)
   ...
   ---
   ```
7. **Limitations / known issues** — from the context only.
8. **Next step** — single forward link, if the folder uses that pattern.
9. **See also** — related articles, bulleted.

#### Existing article (`update`)

1. Read the full file first.
2. Change only the affected sections. Keep surrounding text byte-identical.
3. If a statement in the article now contradicts the context, **replace it and record both versions** in the PR body under **Needs human decision** — do not quietly overwrite a technical claim.
4. Update `ms.date` to today.
5. Do not renumber, re-wrap, or re-order untouched content.

Print `✏️ Written: <path>` for each file.

---

### Phase 5 — Navigation, cross-links, and side effects

1. **`TOC.yml`** — add or move the entry:
   ```yaml
   - name: <Nav display name>
     href: <relative/path.md>
     displayName: <comma, separated, search, keywords>
   ```
2. **`whats-new.md`** — for genuinely new capabilities only, and only if the context supplies a real date. Follow the file's existing entry format exactly.
3. **Redirects** — if a file is renamed or moved, add an entry to the repo's redirect file (`.openpublishing.redirection.json` or equivalent). Never leave a broken published URL.
4. **Inbound links** — search the repo for links to any renamed/moved file and fix them.
5. **Bidirectional cross-references** — add the new article to the "See also" of the one or two most closely related existing articles.

Print `🔗 Navigation updated: <n> file(s)`.

---

### Phase 6 — Validate

Run these checks and report results. Do not skip a check just because it passes.

| Check | What to verify |
|---|---|
| Frontmatter | All required fields present, `ms.date` is today, `description` ≤160 chars |
| Links | Every relative link resolves to a real file; every anchor matches a real heading |
| Images | Every referenced asset exists on disk; every image has meaningful `alt-text` |
| Code | Fenced blocks have language tags; snippets match the APIs in the context |
| TOC | `href` resolves; entry is not duplicated |
| Style | No first person, sentence-case headings, consistent list punctuation |
| Branding | Current product names throughout (Phase 3 table) |
| Includes | Every `[!INCLUDE []]` target exists |
| Grounding | Every technical claim traces to the context; unsupported ones are marked `NEEDS-INPUT` |

If the repo has a docs linter or build (for example a `docfx` build or a markdown lint config), run it and report the result. Do not add new tooling.

Print:

```
✅ Validation
   Frontmatter: OK
   Links: <n> checked, <m> flagged
   Images: <n> checked, <m> missing
   Style / branding: OK
   Grounding: <n> claims, <m> need input
```

---

### Phase 7 — Open the draft pull request

1. Create a branch off the default branch: `docs/<kebab-feature-name>`.
2. Stage **only** the documentation files this run touched. Never `git add -A`.
3. Commit:
   ```
   docs: <what changed, imperative, ≤72 chars>

   - <file>: <what and why>
   - <file>: <what and why>

   Source context: <short description of the input>
   ```
4. Push the branch and open a **draft** PR against the default branch using the template below.
5. Print:
   ```
   🚀 Draft PR: #<n> — <url>
      Files: <a> added, <m> modified
      Needs human decision: <count>
   ```

---

## PR body template

```markdown
## Summary

<One or two sentences: what this PR documents and why.>

**Source context:** <link / file / folder / description of the input>
**Task type:** new | update | mixed

## Changes

| File | Action | What changed |
|------|--------|--------------|
| `<path>` | Added / Modified | <brief> |

## Needs human decision

<Every judgment call, gap, and conflict. One bullet each. Write "None" only if genuinely none.>

- [ ] **<Topic>** — <what is uncertain and what the skill assumed>

## Unverified claims

<Anything written that the source context did not fully confirm, with the `NEEDS-INPUT` markers pointing to it. Write "None" if fully grounded.>

## Reviewer checklist

- [ ] Technical accuracy — claims match the spec / implementation
- [ ] Code samples compile and run
- [ ] Links and anchors resolve
- [ ] Images are accurate and have useful alt text
- [ ] TOC placement is correct
- [ ] Branding uses current product names
- [ ] Nothing in scope was missed

## Validation

<Paste the Phase 6 validation output.>

---
🤖 Generated by the `docs-from-context` skill. **Draft PR — needs human review before it ships.**
```

---

## Edge cases

| Situation | Behavior |
|---|---|
| Context too thin to write anything factual | Stop before Phase 4. Report exactly what is missing. No PR. |
| Context contradicts an existing article | Write the new version, show both in **Needs human decision**. |
| No code samples in context | Write the structure with `<!-- NEEDS-INPUT: code sample for <scenario> -->` and flag it. |
| Images referenced but not supplied | Keep the `:::image:::` with descriptive alt-text and a `NEEDS-INPUT` marker; flag it. |
| Feature area ambiguous | Use the closest sibling; list the alternative in the PR body. |
| Context describes an unreleased feature | Document it, but never assert availability or dates. Flag for the PM. |
| Context is a PR diff with no prose | Derive the public surface from the diff; flag every behavioral inference. |
| Existing article is the wrong place for the change | Propose a split; do the minimal correct change and flag the restructure. |
| Change spans many articles (>10 files) | Do the full change, but call out the blast radius at the top of the PR body. |
| Branch or PR already exists for this feature | Add commits to the existing branch instead of opening a duplicate PR. |

---

## Definition of done

- [ ] Every claim in the output traces to the supplied context, or is marked `NEEDS-INPUT`.
- [ ] Navigation, redirects, and inbound links are consistent.
- [ ] Phase 6 validation ran and its output is in the PR body.
- [ ] A **draft** PR exists, with **Needs human decision** filled in honestly.
- [ ] Nothing was merged, and nothing was pushed to a protected branch.
