# Smart Fill - EZ inspections

A Chrome side-panel extension (**SmartFill**) that scrapes an EZ Inspections job page (ASP.NET
WebForms), sends the questions/answers + labeled photos to a backend, gets
AI-suggested answers, and highlights matches/mismatches so you can click to apply.

## Project layout

```
extension/          # the Chrome MV3 extension (load this unpacked)
  manifest.json
  background.js     # opens the side panel
  content.js        # scrapes the page + applies answers back
  sidepanel.html/.css/.js   # the UI
  icons/
backend/            # Express + Multer stub
  server.js
  package.json
reference/          # real captured EZ page HTML (questions + photos) for development
```

## How it works

1. **Scrape** (`content.js`): walks every `table.FieldGroup` in `#Main_PanelMain`.
   A `td.FieldGroupCaption` starts a new section; following groups with a
   `td.FieldLabelHighlight` + `td.FieldControl` are questions. The selected
   answer is read from the `checked` radio (or select / textarea value).
   Hidden conditional questions (`display:none`) are skipped.
   Photos come from `span[attid]` inside `Main_RepeaterPic_DataListPic_*` tables:
   each gives an `attid`, a `Photo#` label, a category from the `<select>`, and
   the thumbnail `<img src>`.

2. **Image URLs**: the page shows thumbnails at
   `.../I<job>/thumbnail/<file>.jpg`. The extension guesses full-res by removing
   the `/thumbnail/` segment and fetches that first, falling back to the
   thumbnail if it 404s. **Verify this URL pattern on the live site** - if full-res
   lives elsewhere, adjust `fullResFromThumb()` in `content.js`.

3. **Sync** (`sidepanel.js`): fetches each image into a Blob (with
   `credentials: "include"` so your logged-in session is used), builds a
   `FormData` with `photo_0..photo_N` files plus a `payload` JSON field, and
   POSTs to the backend.

4. **Compare**: backend returns `{ answers: [{ questionId, aiAnswer, ... }] }`.
   The panel shows Page answer vs AI answer, flags ✓ Match / ✗ Mismatch, and
   each option is a button - click to write it back into the page (`APPLY_ANSWER`
   triggers the real radio so the page's own highlight/onChange fires).

## Run the backend

```bash
cd backend
npm install
npm start        # http://localhost:3000
```

The stub saves uploaded images to `backend/uploads/` and returns a **mock** AI
response that echoes the page answer but flips the first Yes/No question so you
can see a mismatch. Replace `mockVerify()` with a real model call.

## Load the extension

1. Chrome → `chrome://extensions` → enable **Developer mode**.
2. **Load unpacked** → select the `extension/` folder.
3. Open an EZ Inspections job page (`ezinspections.com/inspManager/...`) and
   **reload it** (content scripts only inject on pages loaded after install).
4. Click the extension icon to open the side panel. The **Home** tab
   auto-detects the job and shows it as **SUPPORTED**.
5. Click **Sync & Verify with AI** - one button runs scrape → fetch photos →
   verify. Review each suggestion with **Accept / Reject / Reconsider**, filter
   by **All / Different / Matched**, and click a card to scroll the page to that
   question and flash it.

### UI (v0.2.0)

The side panel uses a SmartFill-style layout: a left nav rail with **Home**
(the Sync + review workflow), **Activity** (event log), and **Config**
(read-only backend endpoints shown masked for reference, full-resolution-image
toggle, and answer-block colour pickers). Detection runs automatically on open,
when you switch tabs, and when a browser window regains focus (so viewing a
photo in EZ's own viewer and closing it recovers to **SUPPORTED** without a
reload); the toolbar **↻** button re-detects on demand.

## Request / response contract

**POST** `/api/inspections/verify` - `multipart/form-data`

- `payload` (text): JSON
  ```json
  {
    "jobId": "330890154",
    "work_code": "Cyprexx Occupancy Verification Inspection",
    "sections": [
      { "header": "Gain Access",
        "questions": [
          { "id": "21_StreetSign", "text": "Is Street Sign Present?",
            "type": "radio", "options": ["Yes","No"], "currentAnswer": "Yes" }
        ] }
    ],
    "photos": [
      { "id": "958095971", "category": "Street Scene" }
    ]
  }
  ```
- `images` (files, repeated): the binary images. **Each file is named
  `<id>.<ext>`** (e.g. `958095971.jpg`) so the backend maps the bytes back to the
  matching `photos[].id`. Photos in the payload carry only `id` + `category`.
- `work_code` is resolved by matching the page title against a known list of
  inspection types (case-insensitive, substring match so leading/trailing words
  and the trailing `(id)` are ignored). If none match, `work_code` is `null`.
  The known codes (see `KNOWN_WORK_CODES` in `extension/content.js`):
  - `Cyprexx Occupancy Verification Inspection`
  - `Cyprexx Aspen Grove V2 Property Condition`
  - `Cyprexx Sales Date Inspection Instructions`
  - `M&T Door Knock Inspection`
  - `Foreclosure (Contact)`
  - `CONTACT INSPECTION`
  - `Cyprexx Aspen Grove V2 REO`
  - `Cyprexx Interior/Exterior`
  - `Vacancy Task`
  - `NFR-WT2`

**Response** - JSON
```json
{
  "result_id": "2026-06-29T14-30-05-123Z__job-330890154__a1b2",
  "answers": [
    { "questionId": "21_StreetSign", "aiAnswer": "No",
      "referenceImages": [ { "id": "958095971", "category": "Street Scene" } ] }
  ]
}
```

- `result_id` ties a later **feedback** POST back to this run.
- `aiAnswer` for radio/select must be one of that question's `options`.
- `referenceImages` (optional) cites evidence photos by **id + category only**;
  the extension rebuilds the image URL from the id (matching it to the scraped
  photo) and shows each as a thumbnail that opens the on-page viewer.
- `confidence` and `reasoning` are **optional** - send them if you have them,
  omit them otherwise.

**POST** `/api/inspections/feedback` - `application/json`

```json
{
  "result_id": "...",
  "jobId": "330890154",
  "feedback": [
    { "questionId": "21_StreetSign", "section": "Gain Access",
      "question": "Is Street Sign Present?", "currentAnswer": "Yes",
      "aiAnswer": "No", "decision": "accept", "finalAnswer": "No" }
  ]
}
```
→ `{ "ok": true }`. Sent by the **Send Feedback** button with the operator's
Accept/Reject decisions. `decision` is `accept` | `reject` | `matched`;
`finalAnswer` is the value left on the page. The side panel uses a fixed
`FEEDBACK_URL` constant (`…/api/inspections/feedback`). It's sent **once per job** (the
button greys out after), and **auto-sent** when the operator clicks ↻ or closes
the side panel (`navigator.sendBeacon`).

## Wiring up real AI (later)

In `mockVerify()` you already receive `photoFiles` mapping each image `id` → saved path.
For each question, select photos whose `category` is relevant, read the bytes,
send question text + images to your model, and constrain the answer to
`question.options`. Return the same answer shape.

## Things to verify on the live site

- Full-res image URL pattern (remove `/thumbnail/`?).
- Whether image fetch from the extension keeps the session cookie (it uses
  `credentials: "include"`; if the images are on a different host that needs
  separate auth, you may need to fetch via the content script instead).
- Field-key mapping: `applyAnswer()` rebuilds the radio group name as
  `ctl00$Main$<fieldKey>`. Confirmed against the provided HTML, but spot-check
  a few fields.
