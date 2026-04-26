# MVP/Future Separation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Separate MVP requirements from future-feature details across the documentation.

**Architecture:** Keep MVP documents focused on current implementation scope and move detailed future functionality into `docs/06_future_features.md`. Preserve short cross-references from MVP docs to the future-feature document.

**Tech Stack:** Markdown documentation in `docs/`; verification via `rg` and `git diff`.

---

### Task 1: Create Future Feature Document

**Files:**
- Create: `docs/06_future_features.md`
- Modify: `docs/01_requirements.md`

- [ ] **Step 1: Create `docs/06_future_features.md`**

Add sections for 2-player mode, life mode, explanation timer, voting timer, custom word lists, idle support, transcription/Katakana detection, avatar shortcuts, and in-app voice chat.

- [ ] **Step 2: Replace `docs/01_requirements.md` future section**

Keep `01_requirements.md` as the MVP source of truth. Replace the long future-feature section with a short pointer to `docs/06_future_features.md`.

- [ ] **Step 3: Run future-feature scan**

Run: `rg -n "### 11\\.|####|ライフ制|説明時間・時間切れ|投票時間|WebRTC|文字起こし|省力化" docs/01_requirements.md`

Expected: no long future-feature subsections remain in `01_requirements.md`; short references may remain.

### Task 2: Tighten Supporting Docs

**Files:**
- Modify: `docs/02_architecture.md`
- Modify: `docs/03_database.md`
- Modify: `docs/04_api.md`
- Modify: `docs/05_sitemap.md`

- [ ] **Step 1: Update architecture reference**

Point future WebRTC details to `06_future_features.md` while keeping MVP audio policy in `02_architecture.md`.

- [ ] **Step 2: Move future data/API detail references**

In `03_database.md` and `04_api.md`, keep MVP entities and commands. Convert future expansion notes into short references to `06_future_features.md`.

- [ ] **Step 3: Keep sitemap MVP-focused**

Make `/help` and optional overlays clearly MVP-optional, and leave future screens out unless they are direct MVP support.

### Task 3: Verify Links and Diff

**Files:**
- Review all changed Markdown files.

- [ ] **Step 1: Check future doc is linked**

Run: `rg -n "06_future_features" docs`

Expected: MVP documents link to the new future-feature document where future details were removed.

- [ ] **Step 2: Check diff**

Run: `git diff -- docs`

Expected: future details are moved, not silently lost; MVP requirements remain clear.

- [ ] **Step 3: Commit documentation changes**

Run:

```bash
git add docs
git commit -m "docs: separate MVP and future features"
```
