---
name: document-feature
description: End-to-end AI-automated documentation for the msteams-docs repository. Ingests context from a link, file, or folder, classifies the task as a new feature or a documentation update, then writes/updates the article, TOC placement, headings, images, links, code snippets, zone pivots, and See also sections, and opens a pull request for human review.
version: 2.0.0
---

# Document Feature — Teams Platform Documentation Skill

GitHub Copilot skill that performs **end-to-end, AI-enabled documentation automation** for the [MicrosoftDocs/msteams-docs](https://github.com/MicrosoftDocs/msteams-docs) repository and opens a pull request for review.

## Quick Summary

Run `/document-feature <source>` where `<source>` is a **webpage URL, a file path, or a folder path**. The skill extracts context from that source, decides whether the work is a **new feature** doc or a **documentation update**, finds the right place in the TOC, restructures headings and lists, audits images/GIFs, repairs inbound and anchor links across the whole repo, validates code snippets against the Teams SDK docs, reworks zone pivots, refreshes the **See also** section, adds notes/examples/tables, and creates a PR.

## IMPORTANT: Fully automated with PR for review

**Do NOT ask the user questions mid-run. Do NOT pause for confirmation.** Execute every phase in order and create a pull request at the end. If a step cannot be completed, log a warning, flag it in the PR body, and continue.

**Safety contract:**

- Never delete existing documentation unless explicitly replacing it with updated content.
- Preserve existing frontmatter fields; only update `ms.date` and content-affecting fields.
- Do not modify files outside documentation scope (no product code changes).
- All generated content must be factual and grounded in the extracted source context or the official Teams SDK documentation — never invent API behavior, parameters, or capabilities.
- Never delete an image, GIF, or asset that is still referenced anywhere in the repo.
- Never break an existing published URL; when a file must move, add a redirect entry instead of deleting.

**Style contract:**

- Follow Microsoft Learn documentation style: clear, concise, developer-focused.
- Use second person ("you"). Present tense. Active voice. Short, scannable sentences.
- Match the tone, structure, and formatting of neighboring articles in the same folder — the repo's own style always wins over generic guidance.

---

## Invocation

```
/document-feature <source> [scope-hint]
```

- **`source`** *(required)* — the context source. One of:
  - **Link** — a webpage URL (spec, blog post, Learn article, SDK reference, ADO/GitHub item).
  - **File** — a local path to a spec, design doc, code sample, changelog, or existing `.md` article.
  - **Folder** — a directory of specs, samples, or docs to be read recursively.
- **`scope-hint`** *(optional)* — a target path or area hint (for example, `msteams-platform/bots/`).
- Examples:
  - `/document-feature https://learn.microsoft.com/... bot emoji reactions API`
  - `/document-feature C:\specs\streaming-ux-spec.docx`
  - `/document-feature C:\specs\agents-sdk\ msteams-platform/agents-in-teams/`

---

## Workflow

### Phase 0: Ingest the Source and Extract Context

1. **Resolve the source type** — link, file, or folder.
   - **Link:** fetch the page. Follow one level of directly relevant in-page links (spec appendices, SDK reference pages). Capture headings, API names, parameters, code snippets, screenshots, and version/release information.
   - **File:** read the whole file. For code files, extract public APIs, types, and usage patterns. For `.md`, extract structure and claims.
   - **Folder:** enumerate recursively, skip binaries and build output, and read every relevant spec, README, sample, and manifest.
2. **Build a context record** containing: feature name, capability summary, target audience, APIs/parameters/returns, code snippets by language, prerequisites, limitations, supported platforms/clients (Teams, Outlook, Microsoft 365 Copilot), and any images provided.
3. **Classify the task type:**
   - `new-feature` — the capability has no existing article in the repo.
   - `doc-update` — an article already covers this area and needs revision.
   - Decide by searching the repo for the feature name, API names, and close synonyms. If a strong match exists (title, headings, or API names overlap), choose `doc-update` and record the target file(s). Otherwise choose `new-feature`.
4. Print `📥 Source: <type> — <location>` and `🎯 Task type: <new-feature|doc-update>`.

---

### Phase 1: Locate the Article and Its TOC Placement

1. **Map the feature to a doc area** under `msteams-platform/`:
   - `agents-in-teams/` — agents built with the Teams AI/Agents SDK
   - `bots/` — bot framework features
   - `tabs/` — tab apps
   - `messaging-extensions/` — message extensions
   - `apps-in-teams-meetings/` — meeting, call, and live-share apps
   - `task-modules-and-cards/` — dialogs and Adaptive Cards
   - `webhooks-and-connectors/` — webhooks and connectors
   - `m365-apps/` — Microsoft 365 cross-platform apps
   - `messaging-extensions/`, `concepts/` — cross-cutting concepts (auth, design, build, deploy, publish)
   - `graph-api/` — Graph API integrations
   - `toolkit/` — Microsoft 365 Agents Toolkit
   - `resources/` — references, schemas, and samples
2. **Read `msteams-platform/TOC.yml`** and determine the exact insertion point:
   - Find the parent node whose siblings share the same conceptual level and audience.
   - Prefer placing the article next to the closest sibling topic (same feature family, same app type).
   - Respect the existing ordering convention of that node (build → integrate → test → publish, or basic → advanced).
   - For `doc-update`, verify the existing TOC entry is still in the right place; move it if the article's scope changed.
3. Record the proposed TOC entry (`name`, `href`, `displayName`) and the parent path in the tree.
4. Print `📂 Target area: <folder>` and `🧭 TOC placement: <parent > child path>`.

---

### Phase 2: Discover Repo Conventions

1. **Read adjacent articles** in the target folder to learn:
   - Frontmatter shape (`title`, `description`, `ms.topic`, `ms.localizationpriority`, `ms.date`, `author`, `ms.owner`, `zone_pivot_groups`).
   - Heading depth and naming patterns; sentence-case headings.
   - Use of includes, zone pivots, and tabbed content.
   - Image referencing patterns (`~/assets/images/<area>/...`) and `:::image:::` usage.
   - Cross-reference style (relative `.md` paths with anchors, Learn absolute URLs for non-repo content).
   - Callout syntax (`> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!CAUTION]`).
   - "See also" and "Next step" section conventions at the bottom of articles.
2. **Check `msteams-platform/includes/`** for reusable includes that already cover parts of the content — reuse instead of duplicating.
3. **Check `zone-pivot-groups.yml`** (or the repo's pivot definition file) for existing pivot group IDs and their pivot values.
4. **Apply current branding:**
   - "Microsoft 365 Agents Toolkit" (not "Teams Toolkit")
   - "App manifest" (not "Teams app manifest")
   - "Microsoft 365 Agents Playground" (not "Test Tool")
   - "Microsoft 365 Copilot" (not "Microsoft Copilot" for the Copilot app surface)

Print `✅ Conventions discovered`.

---

### Phase 3: Write or Update the Article

#### For a new feature (`new-feature`)

1. **Create the article** at `msteams-platform/<area>/<kebab-case-title>.md` with frontmatter matching neighboring files:

```yaml
---
title: <Descriptive, search-friendly title>
description: <Learn how/about ... — 1-2 sentences, max 160 chars, includes key search terms>
ms.topic: conceptual | how-to | reference
ms.localizationpriority: medium | high
ms.date: <MM/DD/YYYY — today's date>
ms.owner: <owner alias if the folder uses it>
zone_pivot_groups: <group-id, only if pivots are used>
---
```

2. **Write the content** using this structure, adapted to the folder's conventions:
   - `# Title`
   - **Introduction** — 2-3 sentences: what the feature is, who it's for, why it matters.
   - **Prerequisites** — bulleted list, when applicable.
   - **Concept sections** — how it works, with diagrams or tables where they aid comprehension.
   - **Implementation steps** — numbered, each step with the code required to complete it.
   - **Code samples** — fenced blocks with language identifiers; multi-language via tabs (`## [C#](#tab/csharp)` / `## [JavaScript](#tab/javascript)` / `---`).
   - **Examples and tables** — parameter tables, property tables, supported-surface matrices, and at least one realistic end-to-end example.
   - **Limitations and known issues.**
   - **Next step** — the single logical follow-on article.
   - **See also** — related references (see Phase 8).

#### For a documentation update (`doc-update`)

1. **Read the existing article completely** before editing.
2. **Make surgical edits** — revise only sections affected by the new context; do not rewrite unaffected prose.
3. **Reconcile conflicts** — where the source contradicts the article, update to the source and flag the change in the PR body.
4. **Remove or mark deprecated content** — move retired behavior into a clearly labeled note or delete it if the source confirms removal; never silently drop supported behavior.
5. **Update `ms.date`** to today's date.

Print `✏️ Documentation written: <file-path>`.

---

### Phase 4: Restructure Headings, Subheadings, and Lists

1. **Enforce a single `#` H1**; no heading level skips (`##` → `####` is invalid).
2. **Rewrite headings** to be sentence case, task-oriented, and scannable ("Send a streaming response", not "Streaming Responses Overview").
3. **Reorder sections** into the reading order a developer follows: concept → prerequisites → setup → implementation → examples → limitations → next step → see also.
4. **Promote or demote sections** so that related content nests correctly under its parent concept.
5. **Normalize lists:**
   - Convert prose paragraphs that enumerate items into bulleted lists.
   - Convert bulleted sequences that must be followed in order into numbered lists.
   - Keep list items parallel in grammar and consistent in end punctuation.
   - Convert two-dimensional bulleted content (item + attributes) into a table.
6. **Record every heading that was renamed, added, removed, or re-leveled** — this list drives the anchor-link repair in Phase 6.

Print `🧱 Structure normalized: <N> headings changed`.

---

### Phase 5: Audit Images, Screenshots, and GIFs

1. **Inventory every asset referenced** by the article (`:::image:::`, Markdown image syntax, and includes).
2. For each asset, decide:
   - **Keep** — still accurate for the documented behavior and current UI.
   - **Update** — the UI, labels, product name, or flow shown no longer matches the content; mark it `NEEDS-NEW-SCREENSHOT` and flag it in the PR body with the exact reason.
   - **Add** — a step or concept that the source describes visually has no supporting visual; add a placeholder reference with descriptive alt text and flag it.
   - **Remove** — the asset documents removed behavior; remove the reference (and the file only if nothing else in the repo references it).
3. **Validate every asset path resolves** to a file under `msteams-platform/assets/images/<area>/`.
4. **Validate alt text** exists, is descriptive, and is not a duplicate of the caption.
5. **GIFs** — confirm the animation still matches the described flow; if the flow changed, flag for re-recording.
6. If the source provided new images, place them under `msteams-platform/assets/images/<area>/` using the folder's naming convention and reference them with `:::image type="content" source="~/assets/images/<area>/<file>" alt-text="<description>":::`.

Print `🖼️ Images: <kept> kept, <updated> flagged for update, <added> added, <removed> removed`.

---

### Phase 6: Repair Inbound Links and Anchors Across the Repo

1. **Find all inbound references** — search the entire repo for links to the current article (file name, relative paths from other folders, and `TOC.yml` `href` entries).
2. **Validate each inbound link** resolves to the article's current path; fix any that are broken or that point at a renamed/moved file.
3. **Validate anchor links** — for every inbound link containing `#anchor`, check the anchor against the article's **current** heading slugs (lowercase, spaces → hyphens, punctuation stripped).
   - Cross-reference the Phase 4 rename log: every renamed or re-leveled heading invalidates its old anchor.
   - Update each failing inbound anchor to the new slug, or to the nearest surviving section when the heading was removed.
4. **Validate outbound links** in the article — every relative link, anchor, include path, and Learn URL must resolve. Fix broken ones; flag any that require content that doesn't exist yet.
5. **Re-check after edits** — run the link and anchor validation a second time after all fixes so that repairs made in step 3 didn't introduce new failures.
6. **Update `TOC.yml` and `.openpublishing.redirection.json`** if the article path changed.

Print:

```
🔗 Links: <N> inbound checked, <A> fixed, <B> anchors repaired, <C> outbound fixed, <D> flagged
```

---

### Phase 7: Validate Code Snippets Against the Teams SDK

1. **Extract every code block** in the article with its language tag.
2. For each snippet, verify against the official Teams SDK / Bot Framework / Adaptive Cards reference documentation and the source context:
   - Namespaces, imports, and package names are current.
   - Class, method, property, and parameter names exist and are spelled correctly.
   - Method signatures, argument order, and return types match the SDK.
   - The snippet reflects the current recommended pattern, not a deprecated one.
3. **Classify each snippet:**
   - `valid` — leave as is.
   - `outdated` — rewrite to the current SDK API and note the change in the PR body.
   - `irrelevant` — the snippet no longer supports the surrounding content; replace it with one that does, or remove it.
   - `unverifiable` — cannot be confirmed against SDK docs; keep it, mark it in the PR body for SME review.
4. **Ensure completeness** — snippets are copy-paste ready with the context needed to run, not pseudo-code fragments.
5. **Ensure every fence has a language identifier** and that tabbed multi-language sets are complete and consistently ordered.

Print `💻 Code: <valid> valid, <fixed> updated, <removed> removed, <flagged> flagged`.

---

### Phase 8: Zone Pivots, Notes, See Also, and Enrichment

1. **Zone pivots:**
   - Determine whether the content genuinely differs by pivot dimension (SDK/language, app type, or platform surface).
   - **Add** pivots when the source describes divergent instructions per surface or SDK; declare `zone_pivot_groups` in frontmatter and wrap sections with `::: zone pivot="<value>"` / `::: zone-end`.
   - **Update** existing pivots when a new surface or SDK is introduced by the source.
   - **Remove** pivots when the instructions have converged and the branches are now identical.
   - Verify every pivot value used exists in the repo's zone pivot group definition, and that every declared pivot has content in the article.
2. **Add `> [!IMPORTANT]` notes** for behavior that causes failures if missed: preview/GA status, licensing or admin-consent requirements, breaking changes, platform limitations, and required manifest versions. Use `[!NOTE]` for supplementary detail, `[!TIP]` for optimizations, and `[!CAUTION]` for data-loss or security risk.
3. **Add "For more information" pointers** inline — wherever the article summarizes a concept covered in depth elsewhere in the repo, add `For more information, see [<article title>](<relative-path>).` Link only to pages that already exist in the repo.
4. **Add or refresh the See also section** at the bottom:
   - Include prerequisite articles, sibling articles in the same feature family, the deeper reference pages for concepts touched, and the logical next topic.
   - Use the article's real title as the link text.
   - Remove stale entries that no longer resolve or are no longer relevant.
   - Keep the section formatted exactly like neighboring articles in the folder.
5. **Add examples and tables** — parameter/property tables, supported-client matrices, error-code tables, and at least one realistic scenario walkthrough where the content warrants it.

Print `📎 Pivots, notes, and See also updated`.

---

### Phase 9: Update Navigation and What's New

1. **Insert the TOC entry** at the placement decided in Phase 1, with `name`, `href`, and `displayName` keywords.
2. **Update `whats-new.md`** (for `new-feature` only) under the correct date section, matching the existing format:

```
   * ***<Month Day, Year>***: [<Feature description>](<relative-path>) is now available.
```

3. **Add reciprocal cross-references** — where a related existing article should point at the new content, add it to that article's See also section.

Print `🧭 Navigation updated`.

---

### Phase 10: Final Validation

1. **Frontmatter** — required fields present, `ms.date` current, `zone_pivot_groups` valid when pivots are used.
2. **Structure** — one H1, no heading level skips, sections in the correct reading order.
3. **Links** — all inbound, outbound, anchor, include, and TOC links resolve.
4. **Images** — all paths resolve, all have alt text, flagged items listed.
5. **Code** — all fences have languages, tab sets complete, SDK-verified.
6. **Style** — no first person, sentence-case headings, consistent list punctuation, active voice.
7. **Branding** — current product names throughout.
8. **Pivots** — every declared pivot has content; every pivot value is defined.

Print:

```
✅ Validation
  - Frontmatter: OK
  - Structure: OK
  - Links: <N> verified, <M> flagged
  - Images: <N> verified, <M> flagged
  - Code: <N> verified, <M> flagged
  - Style / Branding / Pivots: OK
```

---

### Phase 11: Create the Pull Request

1. **Create a feature branch** from `main`.
2. **Commit all changes** with a descriptive message:

```
  docs: <add|update> <feature-name> documentation

   - <Added|Updated> <article>.md
   - Restructured headings and lists
   - Repaired <N> inbound/anchor links
   - Validated code snippets against Teams SDK
   - Updated TOC.yml and whats-new.md
```

3. **Open the pull request** using the PR body template below.

Print:

```
🚀 Pull request created: #<number>
   Title: <title>
   Files: <N> added, <M> modified
   Review items: <count>
```

---

## Content Quality Rules

1. **Be precise** — exact API, parameter, and type names from the source or SDK docs.
2. **Show, don't just tell** — a runnable code example for every API or configuration described.
3. **Complete examples** — copy-paste ready, never pseudo-code.
4. **Document error cases** — common errors and how to handle them.
5. **Respect scope** — document current behavior only; no speculation about future releases.
6. **Ground every claim** — if it isn't in the source or SDK docs, don't state it; flag it instead.
7. **Match the repo** — when generic style guidance conflicts with the surrounding folder's conventions, follow the repo.
8. **SEO** — titles, descriptions, headings, and `displayName` use terms developers actually search.

---

## Edge Cases and Error Handling

| Situation | Behavior |
| --- | --- |
| Source link is unreachable or gated | Continue with any partial context; flag missing context prominently in the PR body |
| Folder source contains unrelated files | Ignore binaries/build output; document which files were used as context |
| Task type is ambiguous (new vs. update) | Prefer `doc-update` when any existing article overlaps; state the decision and the alternative in the PR body |
| Feature area cannot be determined | Place in the closest matching folder and ask the reviewer in the PR body to confirm |
| TOC insertion point unclear | Place at the end of the most relevant node and flag for reviewer |
| Existing article conflicts with the source | Update to the source and show both versions in the PR body |
| Heading rename breaks external (non-repo) anchors | Keep the old heading text as a bookmark or add a redirect note; flag it |
| Broken inbound link points to a file that doesn't exist | Repair to the nearest correct target if unambiguous; otherwise flag |
| Screenshot/GIF appears stale | Keep it in place, add `NEEDS-NEW-SCREENSHOT` to the PR body with the reason — never ship a blank image reference |
| Code snippet can't be verified against SDK docs | Keep it, mark it `unverified`, and request SME confirmation in the PR body |
| Zone pivot value not defined in the repo | Do not invent one; use non-pivoted content and flag the request in the PR body |
| No code samples in the source | Write the structure with `<!-- TODO: Add code sample -->` and flag in the PR |
| Feature uses deprecated branding | Auto-correct to current branding |

---

## Output Files Summary

| File | Purpose |
| --- | --- |
| `msteams-platform/<area>/<article>.md` | New or updated documentation article |
| `msteams-platform/TOC.yml` | Navigation entry added, moved, or corrected |
| `msteams-platform/whats-new.md` | What's new entry (new features only) |
| `msteams-platform/assets/images/<area>/*` | New or replaced images, screenshots, and GIFs |
| `msteams-platform/includes/<name>.md` | Shared include files (if created or updated) |
| `msteams-platform/**/*.md` | Inbound link and anchor repairs, reciprocal See also entries |
| `.openpublishing.redirection.json` | Redirects (only if an article moved or was renamed) |

---

## PR Body Template

```markdown
## Summary

<1-2 sentence description of what this PR adds or changes.>

**Context source:** <link | file | folder>
**Task type:** <new-feature | doc-update>

## Changes

| File | Action | Description |
|------|--------|-------------|
| `<path>` | Added/Modified | <brief description> |

## Automated audit results

| Check | Result |
|-------|--------|
| TOC placement | <parent > child path> |
| Headings/lists restructured | <N> changes |
| Images/GIFs | <kept> kept, <flagged> need updates |
| Inbound links | <N> checked, <A> fixed |
| Anchor links | <B> repaired |
| Code snippets | <valid> valid, <fixed> updated, <flagged> unverified |
| Zone pivots | <added|updated|removed|none> |
| See also | <added|refreshed> |

## Review checklist

- [ ] Content accuracy — technical claims match the source spec and SDK docs
- [ ] Code samples — snippets compile and run against the current SDK
- [ ] Links — inbound, outbound, and anchor links all resolve
- [ ] Images — screenshots and GIFs reflect current UI
- [ ] Zone pivots — every pivot renders with correct content
- [ ] Branding — current product names used throughout
- [ ] TOC placement — navigation entry is in the correct location
- [ ] See also — links are relevant and complete

## Items flagged for review

<List every warning, placeholder, stale screenshot, unverified snippet, and ambiguity that needs human judgment.>
```

---

*The skill is complete when the pull request is created. Do not ask follow-up questions.*
