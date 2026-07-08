# Reference - real EZ Inspections page HTML

Captured DOM from a live EZ Inspections job page (job `330890154`). Kept here as
ground-truth for development: when changing the scraper (`extension/content.js`)
or verifying selectors, diff against these instead of guessing.

> These are **static HTML captures for reference only** - not loaded by the
> extension and not served anywhere. Safe to open in a browser to inspect structure.

## Files

| File | What it is |
| --- | --- |
| `sample-job-page-questions.html` | The `#Main_PanelMain` form - inspection **sections → questions → answers**. Scraped by `scrapeQuestions()`. |
| `sample-job-page-photos.html` | The `Main_RepeaterPic_DataListPic_*` photo grid - **photos, labels, category `<select>`, thumbnails**. Scraped by `scrapePhotos()`. |

## Key selectors (verified against these captures)

- **Questions** live in `#Main_PanelMain` as flat sibling `table.FieldGroup` rows.
  - `td.FieldGroupCaption > span` starts a new section (e.g. the first one is
    `Bad Address`); following groups belong to it until the next caption.
  - A question group has `td.FieldLabelHighlight` (label in `.labelNameFormat`)
    + `td.FieldControl` (radio group / `<select>` / `<textarea>`).
  - Selected radio = `input[checked]`; the page also tints the chosen control
    `background-color: rgb(204, 255, 204)`, but `checked` is the source of truth.
- **Photos**: each `span[attid="<id>"]` carries a `<label>Photo N</label>`; the
  category is the selected option of the sibling `select.stageClass`; the `<img>`
  `src` is a `/thumbnail/` URL (full-res = same URL without `/thumbnail/`).
- **Property address**: `#Main_LabelTitleAddress`
  (e.g. `22531 JOHN ROLFE LN, Katy, TX, 77449 (13511164) (Sys ID 330919755)`).

See `extension/CONTEXT.md` for the full page-structure notes and the
request/response contract.
