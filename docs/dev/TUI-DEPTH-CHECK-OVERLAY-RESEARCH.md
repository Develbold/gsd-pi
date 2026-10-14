# TUI "Depth Check" Overlay Overlaps the Required Reading — Research

**Branch:** `docs/research-tui-depth-check-overlay` (from `main` @ `48c7d41c`)
**Date:** 2026-10-14
**Question:** The "Depth Check" message rendered by the TUI overlaps the content the reader needs to read. Intentional or bug?
**Verdict:** **Intentional mechanism, un-mitigated side effect — deliberate design debt (a `#2333` follow-up), not an accidental bug.**

---

## 1. Where the "Depth Check" message comes from

The `ask_user_questions` interview dialog. The GSD discuss prompts instruct the model to open it:

- `src/resources/extensions/gsd/prompts/discuss.md:170` — *"use `ask_user_questions` with header **"Depth Check"**, question **"Did I capture the depth right?"**, options "Yes, you got it (Recommended)" and "Not quite — let me clarify". The question ID must contain `depth_verification` … so the write-gate can detect it."*
- `src/resources/extensions/gsd/prompts/guided-discuss-project.md:92` — same wording with id `depth_verification_project_confirm`; `consent-question.ts:378-385` maps the discuss unit types (`discuss-project`, `discuss-requirements`, `discuss-milestone`) to their gate ids.
- Flow (`docs/dev/new-milestone-discuss-flow.md:91-92`): `DepthSummary --> DepthGate: depth_verification question` — **the model first sends the depth summary as a chat message, then opens the dialog.** The reader must read that summary (in the transcript) to answer.

## 2. Why it overlaps (mechanism, verified to source)

| Step | Evidence |
|------|----------|
| The dialog renders as a floating overlay | `src/resources/extensions/ask-user-questions.ts:339,377` — `showInterviewRound(params.questions, { …, overlay: true } …)`, annotated: *"Overlay (#2333): replacing the editor mid-stream makes the bottom-anchored viewport jump."* |
| No geometry is customized | `src/resources/extensions/shared/interview-ui.ts:947` — passes **only** `{ overlay: true }`, no `overlayOptions` |
| pi-tui defaults apply | `packages/pi-tui/src/tui.ts` (`resolveOverlayLayout`): width = `min(80, availWidth)`, **anchor = "center"**, **unbounded height** (no default `maxHeight`) |
| Rendered on top of content | `compositeOverlays` (`tui.ts`) replaces the live-region lines in place — the dialog covers the middle band of the screen, including recent transcript rows |
| Chat scroll is dead while it pends | `showOverlay` focuses the overlay (keyboard capture); `nonCapturing` defaults to `false` (`tui.ts:464-467, 468-530`) — the covered content cannot be scrolled into view |

## 3. History: the deliberate choice behind it

Issue [#2333](https://github.com/open-gsd/gsd-pi/issues/2333): the dialog used to **replace the editor** in the bottom-anchored layout. GSD's re-rendering chrome (spinner 80ms, dashboards 1s, status strip height flips) triggered destructive shrink/repaint cycles at ~1s cadence — the transcript bounced and became unreadable. The issue listed three fixes:

1. Pass `{ overlay: true }` at the ask-user-questions call sites — *"the overlay path never touches `editorContainer`"* → **this shipped** (with the `#2333` annotation in the code).
2. Height-stable GSD chrome (always reserve the `healthSummary` row) → **not shipped**.
3. In-place handling of small shrinks (≤2 lines) in `tui.ts` (pattern exists in the deleted-lines branch) → **not shipped**, though the issue said "any of 1-2 **plus 3 recommended**".

So the overlap is the accepted side effect of fix 1 — the fix was validated for "can you answer the question", not "can you read what the question asks about".

## 4. The gap, precisely

The depth check is the *worst* case for this trade-off: it is the only dialog whose required reading is content **the dialog is covering** (the depth summary just written by the model, sitting in the bottom-anchored recent transcript). With an unbounded-height centered overlay, taller question sets (3 questions, preview panels) reach further into that band, and scroll-back is unavailable because the overlay captures focus.

Related context in this workspace: the Ollama-only startup fix ([#2782](https://github.com/open-gsd/gsd-pi/issues/2782), PR [#2783](https://github.com/open-gsd/gsd-pi/pull/2783)) is adjacent domain knowledge (the same interview dialog is the gateway for GSD's consent gates), but the overlap issue is independent of it.

## 5. Options for addressing (none applied)

1. **Surgical (recommended):** in `interview-ui.ts:947`, pass explicit `overlayOptions` — e.g. `{ anchor: "top-left", maxHeight: "60%" }` (or `row: "0%"`, `margin: 1`) so dialogs float at the **top** and the bottom-anchored recent transcript (the depth summary) stays visible. Zero behavior change for the dialog itself; cannot regress #2333 (overlays never touch the bottom chrome either way).
2. **The deferred #2333 fixes:** height-stable GSD chrome + in-place shrink handling in `tui.ts` → allows reverting to the editor-replacement layout entirely. Bigger TUI-core change.
3. **Keybinding seam:** let chat-scroll keys pass through to the chat controller while a capturing overlay is up. Larger change, touches focus management.

Suggested next step if pursued: upstream issue framed as a follow-up to #2333 with a tiny PR for option 1.

## Environment

- open-gsd/gsd-pi `main` @ `48c7d41c` ("fix(vscode): guard canonical native Copilot project-context reads (#2699)")
- Related upstream: #2333 (the overlay switch), #2123/#2124 (same shrink mechanism, other triggers), #2077-family startup model work (#2782/#2783 — adjacent, independent)