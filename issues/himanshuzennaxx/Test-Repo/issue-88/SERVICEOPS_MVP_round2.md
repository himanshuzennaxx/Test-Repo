# ServiceOps MVP — Round 2 (attachments, pinned tabs, markdown editing)

**Date:** 2026-08-04
**Build reviewed:** SiiA-ServiceOps-develop (latest)

Three features. Task A is the biggest (attachments + a label flow so the AI pipeline picks files up safely). Tasks B and C improve the ticket view.

---

## Task A (HIGH) — Attachments on Ticket, Issue comments, and PR comments

### Goal
Staff can attach multiple files when (1) creating a ticket, (2) commenting on an Issue, (3) commenting on a PR. Files are delivered to the AI pipeline through a dedicated branch, and the `awaiting-files` label ensures the AI never starts working before the files have finished uploading.

### Current state
No upload capability exists anywhere (no multipart / FormData / multer in client or server). This is net-new.

### A1. Upload UI (multi-file)
- On the **ticket-create form** and in **`CommentComposer.tsx`** (both Issue and PR comment modes), add a multi-file picker (button + drag-and-drop).
- Show selected files as a removable list before submit.
- Limits: **25 MB per file** (GitHub's limit). Allow `.docx, .xlsx, .pptx, .pdf, .md, .txt, .zip`, and images.

### A2. Where files go — the `ServiceOps` branch
Files are NOT committed to `develop` or any code branch. They go to a dedicated branch named exactly **`ServiceOps`** (single word, capital S and O).

- If the `ServiceOps` branch doesn't exist on the repo yet, **create it** (an orphan/empty branch — it only holds files, no code history).
- Paths:
  ```
  issues/{org}/{repo}/issue-{N}/<filename>               ← ticket + Issue-comment files
  issues/{org}/{repo}/issue-{N}/pr-{prNumber}/<filename> ← PR-comment files
  ```
  Example:
  ```
  issues/siia-group-dev/siia-cicd-bare-source/issue-3/MVP.docx
  issues/siia-group-dev/siia-cicd-bare-source/issue-3/pr-4/spec.docx
  ```
- Use the **real issue number** in the path. You already have it — `createIssue` returns `res.data.number`.

### A3. Backend upload handling
- Accept multipart upload (add `multer` or equivalent).
- Save the files, then **git push them to the `ServiceOps` branch** at the paths above.

### A4. The `awaiting-files` label flow  ← the critical part

`awaiting-files` is a gate. While it's on an issue, the AI pipeline does nothing on that ticket. It guarantees the AI never reads half-uploaded requirements.

**Sequence when files are attached:**
1. **The moment an upload starts** (ticket create, or a comment with files), add the **`awaiting-files`** label to the issue.
2. Write a `SERVICEOPS:FILES` block listing the full paths (with the **real** issue number):
   - ticket / Issue-comment files → block goes in the **issue body**
   - PR-comment files → block goes in the **PR comment**
   ```
   <!-- SERVICEOPS:FILES -->
   /{org}/{repo}/issue-3/MVP.docx
   /{org}/{repo}/issue-3/pr-4/spec.docx
   <!-- END SERVICEOPS:FILES -->
   ```
   The block content is written once and does not change — the only thing that changes is the label (added on upload, removed on success).
3. git push the files to the `ServiceOps` branch.
4. **When the push of ALL files succeeds → remove the `awaiting-files` label.**
5. **If the push fails → keep the label**, retry, remove only once push succeeds. A stuck upload therefore stays visibly `awaiting-files` so a human can see it.

**For PR comments:** adding `awaiting-files` must **NOT change the PR's existing status labels** (`in-development`, `qc-passed`, etc.). It's a temporary pause laid on top — removing it lets the PR continue from whatever status it was in.

**For tickets with no attachments:** do nothing special — no `awaiting-files`, no FILES block. A ticket without a FILES block is a normal text-only ticket and the pipeline processes it as it does today.

### A5. How the pipeline reads this (so you know why the label matters)
The AI pipeline (scan.sh) decides what to do in this order:
1. **`awaiting-files` label present?** → skip this ticket entirely (wait). Nothing else is checked.
2. Label gone → **is there a `SERVICEOPS:FILES` block?**
   - **No block** → text-only ticket → process normally.
   - **Block present** → go to step 3.
3. Check the listed files actually exist in the pipeline's copy of the `ServiceOps` branch.
   - **All present** → read them, proceed.
   - **Any missing** → add a `fail-to-load-file` label and wait for a human (surfaced, not silently stuck).

So: the label is the gate; the FILES block is optional (its absence just means "no attachments"); actual file existence is the final check.

### A6. Acceptance
- Attaching files to a ticket/issue/PR (a) creates the `ServiceOps` branch if missing, (b) pushes files to the correct `issue-N/` or `pr-N/` path, (c) writes the `SERVICEOPS:FILES` block with real issue numbers, (d) adds `awaiting-files` during the push and removes it **only** on success.
- Existing PR status labels are untouched.
- A push failure leaves `awaiting-files` on the issue (not silently dropped).
- A ticket with no attachments has no FILES block and no `awaiting-files` label.

---

## Task B (MEDIUM) — Pin the Issue / PR tabs (stop the scrolling)

### Current state
Tabs already exist in `ActivityThread.tsx` (`FeedTab = "issue" | "pr"`, tab buttons around line ~253). The problem: the tab bar sits inline in the feed and **scrolls out of view**, so switching means scrolling back up.

### What to do
- Make the tab bar **sticky at the top** of the activity panel: `position: sticky; top: 0;` with a solid (non-transparent) background and a bottom border, so it reads as a fixed header while the feed scrolls underneath.
- Keep the counts you already compute (`issueCount`, `prCount`) visible in the pinned bar (e.g. "Issue 5 / PR 2").

### Acceptance
The Issue/PR tabs remain visible at the top while scrolling; switching never requires scrolling.

---

## Task C (MEDIUM) — Proper Markdown: GitHub-style view + edit

### C1 — View: full Markdown rendering
**Current state:** `markdownLite.tsx` is minimal (only bold / code / lists). It can't handle tables, headings, links, blockquotes, or nested lists — which the AI's RFC/PM output uses heavily.

**What to do:**
- Replace `markdownLite` with **`react-markdown` + `remark-gfm`** (GitHub-Flavored Markdown: tables, task lists, strikethrough, autolinks).
- Keep stripping the `<!-- SERVICEOPS:... -->` markers before rendering, as you already do.
- **Do NOT enable raw-HTML rendering** on comment content (XSS risk). react-markdown is safe by default — don't add `rehype-raw` for untrusted content.

### C2 — Edit: GitHub-style Markdown editor
**Current state:** `CommentComposer.tsx` only posts *new* comments. There's no way to **edit** an existing Issue/PR comment.

**Use a Markdown editor, NOT an HTML/WYSIWYG editor.** This matters for us:
- GitHub's own comment editor is a **Markdown `<textarea>`** — you type Markdown, with a toolbar that inserts Markdown syntax, plus a **Write / Preview** toggle. It is not a rich-text/HTML editor.
- Our comments contain HTML-comment markers like `<!-- SERVICEOPS:META -->`, `<!-- SERVICEOPS:COMMENT -->`, `<!-- SERVICEOPS:FILES -->`. A WYSIWYG/HTML editor would escape or destroy these markers. A Markdown textarea preserves them exactly. **So a rich-text editor is not acceptable here.**

**What to do:**
- Add an **Edit** action on Issue/PR comments (subject to author/permission).
- The editor is a **Markdown textarea** with:
  - a small toolbar (bold, italic, code, link, list, heading) that inserts Markdown syntax,
  - a **Write / Preview** toggle — Preview uses the same `react-markdown` renderer from C1.
- Richer option: **`@uiw/react-codemirror` + `@codemirror/lang-markdown`** (the CodeMirror family GitHub is built on) with `react-markdown` for preview. A plain `<textarea>` + toolbar + preview is also fine and simpler.
- On save, PATCH the comment through the GitHub API, keeping all `<!-- SERVICEOPS:... -->` markers intact.

### Acceptance
- Comments render full GFM Markdown (tables, headings, lists, links).
- Staff can edit an Issue/PR comment in a Markdown editor with Write/Preview.
- Embedded `<!-- SERVICEOPS:... -->` markers survive an edit round-trip unchanged.

---

## Summary

| Task | What | Priority | Backend? |
|---|---|---|---|
| A | Multi-file attachments (ticket / issue / PR) + `awaiting-files` flow | **HIGH** | Yes (upload + git push + labels) |
| B | Pin the Issue/PR tabs (sticky) | MEDIUM | No |
| C1 | Full Markdown rendering (react-markdown + remark-gfm) | MEDIUM | No |
| C2 | Edit comments with a GitHub-style Markdown editor | MEDIUM | Yes (PATCH comment) |

**Suggested order:** A first (biggest, unblocks the AI pipeline). Then C1 (easy, big readability win). Then B (quick CSS). Then C2.

**Principles across all tasks:**
- **Markdown, never HTML/WYSIWYG** — protects the `<!-- SERVICEOPS:... -->` markers.
- **`awaiting-files` is the gate** — add when an upload starts, remove only when git push succeeds, never touch a PR's existing status labels.
- **Files go to the `ServiceOps` branch only** — never to `develop` or any code branch.
- **A ticket with no FILES block is a normal text-only ticket** — not an error.
