# AI Prompt Library for Obsidian + DataCore JSX

A fully-featured React/DataCore component that turns any Obsidian vault into an AI-prompt knowledge base — complete with one-click copy, favorites, advanced search, tagging, categories, batch actions, and a slick card/table UI.  ￼

### Feature	Details
- One-click prompt copy:	Copy the prompt text directly to clipboard.
- Favorites tab	Drag-and-drop reorderable; horizontal two-row scroll with left/right arrows.
- Recently used	Auto-logs the last few copied prompts for lightning recall.
- Advanced search & filters	Debounced keyword search across title/description/prompt/tags, multi-tag filter, date range, favorites-only toggle, category filter, recent-search chips.
- Dynamic categories	Create, rename, color-code, and delete categories (persisted to localStorage + front-matter).
- Paginated table view	Sortable by title, description, tags, created date; choose rows per page.
- Responsive card grid	Adaptive two-row masonry layout with drag handles, tag pills, category badge, and copy/favourite buttons.
- Optimised filtering	Single-pass filter pipeline with performance timer; search remains snappy on large libraries.

⸻

# Folder Structure

<your-vault>/
└─ Notes/
   └─ Prompts/        <-- each prompt is a Markdown file with YAML front-matter
      ├─ My Prompt.md
      └─ …
PromptLibrary.md <---- Main component (can be embedded into seperate note using datacore - see installation)

 ** The component will auto-create Notes/Prompts/ on first save if it does not exist. **

⸻

🛠️ Installation
	1.	Add DataCore JSX to your vault (via the Datacore community plugin).
	2.	Drop AIPromptsManager.md (this component) into vault.
  3. If you would like to embed the component into a seperate note opposed to using the actual component file see below
            ```datacorejsx
            await dc.require("components/AIPromptsManager.jsx")
  4. Reload the note — the prompt manager UI should appear.

---

## Creating a Prompt

1. Click the floating **`＋`** FAB or the **“New Prompt”** button.  
2. Fill in:
   - **Title** (required)  
   - **Description** (optional)  
   - **Prompt text** (required – plain text or valid JSON)  
   - **Tags** (comma-separated)  
   - **Category** (pick or create)  
   - **Favourite** (checkbox)  
3. hit **Save**.  
A new Markdown file will be written to `Notes/Prompts/` with YAML front-matter like:

### Example

```yaml
---
title: My SQL fixer
description: Quick prompt for repairing broken SQL.
tags:
  - Prompts
  - SQL
prompt: "You are a senior DBA. Fix this query: {{input}}"
favorite: true
category: Databases
created: 2025-05-26T22:00:00.000Z
modified: 2025-05-26T22:00:00.000Z
---
```

# Features

### Search & Filters

- Search bar	Debounced; searches title / desc / prompt / tags (scope selectable).
- Filters panel	Toggle with Filters pill → filter by tags, favourites-only, category, date range.
- Category pills	Quick switch between “All” and individual categories.
- Recent searches	Click a chip to re-apply it.

### Tabs

- Favourites	Two-row scrollable cards; drag to rearrange.
- Recently Used	Auto-logged after every copy; most recent on the left.
- All Prompts (Table)	Spreadsheet-like view with pagination, sort, and bulk tools.

### Custom Configuration

Option	How
Rows per page	Dropdown bottom-left of the table.
Category colours	Manage → click colour dot → select → Save Color.
Storage keys	Favourites order, category colours, recent searches, recently used are stored via localStorage under prompt-manager-*.
Debounce delay	useDebounce(value, 200) — change 200 ms if you like.
Folder path	Edit promptNotesQuery (path("Notes/Prompts")) to point elsewhere.



### License

MIT — hack away and share improvements.
