# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Kanban board for a fictitious bank's internal IT PMO (demo/training tool). The whole app is `index.html`: markup, one `<style>` block, one `<script>` block. There is no package.json, build step, bundler, test suite, linter or git repo.

## Running and checking

- Run: open `index.html` directly in a browser (double-click). No server is needed.
- Syntax-check the script without a browser: extract the `<script>` contents to a temp file and run `node --check` on it.
- Constraint audit: grep `index.html` for `localStorage|sessionStorage|indexedDB|document\.cookie|!important|https?://`. The only URL hit should be `FORMSUBMIT_ENDPOINT`.

## Hard constraints (from the original spec)

- Vanilla HTML/CSS/JS only: no frameworks, CDN scripts, web fonts, image files or build tooling. Icons are inline SVG or Unicode glyphs.
- No persistence of any kind: no localStorage, sessionStorage, IndexedDB or cookies. A refresh resets the board to seed data, and the UI says so.
- No `!important` in CSS. Palette and spacing come from the CSS custom properties in `:root`.
- Neutral "IT PMO" text wordmark and a blue palette only. Do not add real bank logos or trademarks. The `UOB-ITPM-####` task ID prefix (`ID_PREFIX`) and the `[UOB IT PMO]` email subject (in `notifyNewTask`) are required by the spec.
- Email goes only through the FormSubmit AJAX endpoint. `FORMSUBMIT_ENDPOINT` is the only place the address appears.

## Architecture

The script is numbered, commented sections. The core pattern is a single `state` object (`tasks`, `filters`, `ui`) that is the source of truth, and everything is rendered from it.

- **Mutations go through `addTask`, `moveTask` and `deleteTask`.** Each changes `state` and then calls `renderBoard()`.
- **`renderBoard()` is the only place card DOM is produced.** It rebuilds each column's `<ul data-list>` via `renderCard()` (an HTML string), updates the count badges, and calls `renderSummary()`. The four `.column` sections are static in the HTML; only their lists are re-rendered, so drag listeners on the columns survive.
- **Escape every user-supplied string with `escapeHtml()`** before it goes into a template string. `renderCard` is the main injection point.
- **Event delegation on `#board`** (`click`, `keydown`, and the drag events). Never attach listeners to individual cards, because they are destroyed on every render.
- **Transient UI state lives in `state.ui`**, not the DOM. `deleteId` is the card showing the inline "Delete? Yes/No", `moveId` is the card with its "Move ▸" menu open, and `sending` is the form's in-flight flag. `state.ui.focus` holds a CSS selector that `renderBoard()` focuses after rendering. Set it before calling `renderBoard()` whenever an action replaces the element the user was on, or keyboard focus is lost.
- **Drag and drop** uses the native HTML5 API. `draggedId` is a module variable, and the hovered column gets the `.drop-target` class. The "Move ▸" menu is the keyboard fallback, and both paths end in `moveTask`.
- **Add Task flow** (`handleSubmit`): validate, then `addTask` immediately (optimistic), reset the form, close the dialog, toast, and then `await notifyNewTask()` inside try/catch. A failure only shows a warning toast and never touches board state. The form uses `novalidate` with JS validation (`validateForm` and `showFormErrors`). Length limits are enforced in JS, with no `maxlength` attribute, so the inline errors can actually fire.
- **Overdue** is computed, not stored: `isOverdue()` compares the ISO due date to `todayISO()` (local time) and ignores Done tasks. Dates are handled as `YYYY-MM-DD` strings, and `formatDate` parses the parts manually to avoid timezone shifts.
- **Seed data** (`seedTasks`) uses due dates relative to today, so the demo always has some overdue and some upcoming cards. The ID counter (`idCounter`) is shared with `addTask`, so seeds take 0001–0008.
- **Toast region** is `aria-live="polite"`. `showToast` uses `textContent`, never `innerHTML`.
