# Smart Fill - EZ inspections - Extension

A Chrome side-panel extension that reads an EZ Inspections job page, sends its
questions/answers and labeled photos to a backend for AI verification, and shows
you where the AI agrees or disagrees so you can apply corrections with one click.

**v0.2.0** rebuilds the side panel in the **SmartFill design system**: a left
nav rail, an auto-detecting detection card, an animated status ring, and an
**Accept / Reject / Reconsider** review queue. Only the look & UX are shared with
the sibling SmartFill extension - the scrape/apply logic is EZ-specific.

This README covers **what each part does** and **the history of how it was
built**. For a copy-paste project brief to start a fresh chat, see
`CONTEXT.md` in this same folder.

---

## What it does (functionality)

### 1. Detect + scrape the page
`content.js` runs on the inspection page. The panel auto-detects the job (on
open and when you switch tabs) and, on Sync, reads:
- **Sections, questions, and current answers.** It walks every
  `table.FieldGroup` inside `#Main_PanelMain`. A `td.FieldGroupCaption` begins a
  new section; the question groups after it (label cell `td.FieldLabelHighlight`
  + control cell `td.FieldControl`) belong to that section. The current answer
  is taken from the checked radio, the selected dropdown option, or the
  textarea/input value.
- **Photos.** From the `Main_RepeaterPic_DataListPic_*` tables it pulls each
  photo's `attid`, its `Photo#` label, its category (from the green
  `select.stageClass` dropdown, e.g. "Street Scene"), and the image URL.
- **Hidden questions are skipped** - conditional fields that the page keeps at
  `display:none` until triggered are ignored.

### 2. Sync & verify (one button)
`sidepanel.js` runs the whole pipeline behind the **Sync & Verify with AI**
button, with progress shown on the status ring:
- Fetches each photo into a Blob using your logged-in session
  (`credentials: "include"`). Photos on the page are **thumbnails**
  (`.../thumbnail/<file>.jpg`); the panel tries the **full-res** version first
  (same URL without `/thumbnail/`) and falls back to the thumbnail. The full-res
  toggle lives in the **Config** tab.
- Builds a `FormData` with each photo appended under the `images` field, named
  `<image-id>.<ext>` so the backend maps bytes back to `photos[].id`, plus a `payload` JSON
  field with all the scraped Q&A and photo metadata.
- POSTs it to the fixed `VERIFY_URL` constant
  (`http://101.53.137.140/ez/api/inspections/verify`).
- Once the run completes the **Sync & Verify** button **greys out** (label →
  "Verified") so the operator can't re-fire the same job and repeat the API call.
  Use the **↻** button to reset the active job and deliberately re-run.
- On completion the status card shows an **end-of-sync summary**
  (`N to review · M matched`, plus `· K photos not loaded` if any image fetch
  failed), also written to the Activity log.

### 3. Review queue (Accept / Reject / Reconsider)
- The backend replies with one AI answer per question.
- Each AI-answered question becomes a **card** showing the **Current** page
  answer next to the **AI Suggestion** (plus confidence/reasoning when provided).
- Filter the queue by **All / Different / Matched** (*Different* = AI differs
  from the page; *Matched* = AI agrees). Counts update live.
- Per card: **Accept** writes the AI answer into the page (it triggers the real
  radio/select so the page's own change handlers and green highlight fire);
  **Reject** keeps the current answer; **Reconsider** reopens the card and
  reverts the page if the answer had been applied. **Accept all / Reject all**
  act on pending cards only.
- **Click a card body** to scroll the page to that question, then flash a
  yellow highlight over it that fades out after ~1.5s (no persistent box).
  Cards stay in page order and keep their place when accepted/rejected.
- **Reference photos.** When the backend returns `referenceImages` for an
  answer, the card shows them as thumbnails. Click one to open an **in-page
  image viewer** on the inspection page (`imageModal.js`): zoom (wheel / + −),
  drag to pan, prev/next through that card's photos (‹ › or ← →), fit (⤢ or `0`),
  close (✕ / `Esc` / backdrop). It renders on the page (not the panel) because
  the photos are same-origin there and load with your session cookies.
- **Send Feedback.** After you Accept/Reject suggestions, the **Send Feedback**
  button POSTs your decisions - with the verify run's `result_id` - to the
  backend's `/api/inspections/feedback` endpoint, so corrections can train the
  model. (It uses the fixed `FEEDBACK_URL` constant, a sibling of `VERIFY_URL`.)
  It's usable **once per job** (greys out after sending), and is **auto-sent**
  when you click ↻ or close the side panel (`navigator.sendBeacon`).

**Multiple jobs at once:** results are retained per job id, so you can open two
job tabs in one window and each keeps its own review queue - switch tabs and the
right queue (with your Accept/Reject decisions) restores automatically. Syncing
the *same* job again replaces its results. A single window shows one job at a
time; for two truly side-by-side, use two browser windows (each has its own panel
state).

### 4. Activity, Config, and footer
- **Activity** tab logs each scrape / sync / accept / reject with a timestamp.
- **Config** tab: the Verify/Feedback endpoints shown masked and **read-only**
  (fixed constants, information only), the full-resolution-image toggle, and
  answer-block colour pickers - the toggle and colours persist via
  `chrome.storage.local`.
- The **footer** shows a connection status dot, the **DhanInfo** logo, and the
  version chip.

### 5. Backend (`backend/` folder)
Express + Multer. `upload.any()` captures all the image files; the `payload`
field is parsed as JSON. It returns one AI answer per question, each with an
optional `referenceImages` list (the photos cited as evidence). It currently
returns a **mock** result (echoes the page answer but flips the first Yes/No
question so you can see a mismatch); real AI replaces the `mockVerify()`
function, which already receives `photoFiles` - a map from each image **id** to
its saved file path (the upload is named `<id>.<ext>`).

**Request logging:** every request writes the input JSON (payload + file
metadata) to `backend/logs/input/` and the response JSON to
`backend/logs/output/`, under a matching timestamped filename so an
input/output pair is easy to correlate.

---

## File-by-file

| File | Role |
|------|------|
| `manifest.json` | MV3 config (v0.2.0). Declares the side panel, the content script (matches `ezinspections.com/inspManager/*`), permissions, and host permissions including `localhost:3000`. |
| `background.js` | Service worker. Opens the side panel on toolbar-icon click, and handles `ENSURE_CONTENT_SCRIPT` - injects `imageModal.js` + `content.js` on demand via `chrome.scripting.executeScript` so pages open before the extension loaded still work without a reload. |
| `content.js` | The scraper. Reads the job id (the Work Order number from `span#Main_LabelWorkOrder`, falling back to the URL `Id=` param), sections/questions/answers and photos, applies an answer back to the page, scrolls/highlights a question, and opens the image viewer. It also emits `EZ_PAGE_READY` when the job form appears so the panel auto-detects without a manual refresh. Messages in: `DETECT`, `SCRAPE`, `APPLY_ANSWER`, `FOCUS_QUESTION`, `CLEAR_HIGHLIGHT`, `SHOW_IMAGE_MODAL`, `PING`. Guards against double-injection so it's safe to load both via the manifest match and on demand. |
| `imageModal.js` | On-page full-resolution image viewer (zoom / pan / prev-next / fit / close). Renders on the inspection page so photos load with the session cookies. Exposes `window.EZ_IMAGE_MODAL.show(images, index)`. |
| `sidepanel.html` | Panel layout: nav rail, detection card, status canvas, review queue, Activity + Config tabs, footer, toast. |
| `sidepanel.css` | The SmartFill design system (copied verbatim) plus a small EZ-specific override for the accepted-card state. |
| `sidepanel.js` | Controller: auto-detection, the scrape → fetch photos → POST pipeline, the Accept/Reject/Reconsider queue and filters, activity log, and config persistence. |
| `icons/` | Toolbar icons. |
| `CONTEXT.md` | Paste-into-new-chat project brief. |
| `README.md` | This file. |

---

## How to run

**Backend** (from the `backend/` folder):
```bash
npm install
npm start          # http://localhost:3000
```

**Extension:**
1. `chrome://extensions` → enable **Developer mode**.
2. **Load unpacked** → select this `extension/` folder.
3. Open an EZ Inspections job page - **no reload needed**. If the page was
   already open before you loaded the extension, the side panel injects its
   content script on demand the first time it talks to the page.
4. Click the extension icon. The **Home** tab auto-detects the job and shows it
   as **SUPPORTED**; click **Sync & Verify with AI**, then review the queue.

---

## Project history

1. **Goal set.** Build an extension to scrape an ASP.NET inspection form
   (header + question + answer + images from a particular area), send images to a
   backend alongside the Q&A, have the backend answer the questions, then compare
   against the page answers and let the user apply the right answer from a side
   panel with a "Sync & Verify with AI" button.

2. **Requirements gathered.** Decided: images sent as **files** for **Multer**;
   backend contract designed from scratch; **no auth** yet; **all visible
   questions** sent at once; results reviewed by the user (no auto-fill).

3. **Real DOM analyzed.** Two HTML snippets from the live page were inspected to
   confirm exact selectors: `FieldGroup`/`FieldGroupCaption`/`FieldLabelHighlight`/
   `FieldControl` structure, `checked` radios as the answer source, the
   `ctl00$Main$<fieldKey>` naming, and the photo `span[attid]` + `select.stageClass`
   structure. Confirmed the images are **thumbnails** with a likely full-res URL
   pattern (drop `/thumbnail/`).

4. **Built and tested.** The extension and an Express+Multer backend stub were
   written. The backend was smoke-tested end to end: payload parsed, image file
   captured by Multer, mock AI flipped one answer to demonstrate a mismatch.

5. **UI rebuilt to the SmartFill design system (v0.2.0).** The plain
   scrape/compare panel was replaced with the SmartFill look & UX: nav rail
   (Home / Activity / Config), auto-detecting detection card, animated status
   ring, and an Accept / Reject / Reconsider review queue with All / Different /
   Matched filters, bulk actions, toasts, and click-to-scroll-and-flash. The
   backend contract and `content.js` scrape/apply logic were preserved;
   `content.js` gained `DETECT` and `FOCUS_QUESTION` messages. Detection now
   degrades to a "NOT SUPPORTED" card instead of throwing when the content script
   isn't present.

6. **On-demand content-script injection.** The side panel no longer depends on a
   page reload. When a message to `content.js` fails, `sidepanel.js` asks the
   service worker to inject it via `chrome.scripting.executeScript`, then retries
   - so pages open before the extension loaded work immediately. `content.js`
   guards against double-injection (`window.__ezVerifierContentLoaded`).

7. **Reference images, in-page viewer, multi-job, backend logging.** Answers now
   carry an optional `referenceImages` list, rendered as thumbnails on each card
   and opened in an on-page zoom/pan/prev-next viewer (`imageModal.js`, the
   `SHOW_IMAGE_MODAL` message). Completed runs are retained per job id so two job
   tabs in one window each keep their own queue (with a 15-job memory cap). The
   backend logs every request's input/output JSON to `backend/logs/input/` and
   `backend/logs/output/`. Detection/restore is keyed on job id so it survives
   ASP.NET postbacks that change the URL.

8. **Feedback API + simplified image contract.** The verify response now returns
   a `result_id`; a **Send Feedback** button POSTs the operator's Accept/Reject
   decisions to `/api/inspections/feedback` (logged to `backend/logs/feedback/`).
   Photos are sent as `{ id, category }` with files named `<id>.<ext>` (mapped by
   id), and `referenceImages` returns `{ id, category }` only - the extension
   rebuilds image URLs from the id. `confidence` / `reasoning` are now optional.
   The Sync button greys out after a completed run (↻ resets a job to re-run).

9. **Detection recovers on window focus; steadier click-to-flash.** Detection now
   also re-runs (debounced) on `chrome.windows.onFocusChanged`. Opening a photo in
   EZ's own viewer (a separate tab/window) pointed the panel's active-tab query at
   that page, flipping the card to **NOT SUPPORTED**; returning focus to the job
   window fired no tab event, so it stayed stuck until a reload. The focus listener
   recovers it automatically. The click-to-scroll flash now holds ~1.5s (was ~2s)
   and removes any in-flight overlay before drawing a new one, so repeated clicks
   no longer stack translucent layers into an ever-darker box over the question.

---

## Known things to verify / do next

- **Confirm the full-res image URL** on the live site (is removing `/thumbnail/`
  correct?). Adjust `fullResFromThumb()` in `content.js` if not.
- **Replace `mockVerify()`** with a real model call: pick photos by `category`
  relevant to each question, send question text + images, constrain the answer
  to `question.options`.
- **Image-fetch auth:** if photos sit on a host that needs separate auth, fetch
  them from inside the content script instead of the panel.
- **Spot-check `applyAnswer()`** field-key mapping against a few real fields.
