README.md — Scraplist
EBRO Factory Scrap List Collection & Reporting

A browser-based tool that turns photographed or scanned EBRO "Control Calidad" forms into clean, structured Excel reports — powered by Google Gemini or DeepSeek vision models.

📖 Overview
Scraplist is a single-page web application that:

Accepts images of EBRO quality-control forms (JPG / PNG / WEBP / HEIC).

Normalizes, rotates (EXIF-aware), converts (HEIC → JPEG), and compresses them.

Sends each image to an AI vision model with a strict extraction prompt.

Validates, de-duplicates, and post-processes the returned JSON.

Presents a per-row quality audit with editable fields, image viewer, and zoom.

Learns from your manual edits (Correction Library).

Exports a merged, multi-sheet Excel workbook plus a separate error list.

No backend, no build step, no tracking. Everything runs in the browser.

✨ Key Features
Category	Highlights
AI extraction	Gemini 2.x/3.x (multiple models) or DeepSeek — switchable in the UI
Input modes	📁 Local Folder (File System Access API) · 📤 Upload / Drag-&-Drop
Auto-processing	Starts the AI consumer before compression finishes — zero idle time on big batches
Image pipeline	EXIF orientation fix · HEIC → JPEG · configurable max dimension · JPEG re-encode
File lifecycle	On success → readed/, on failure → error/ (Local Folder mode)
Rate limiting	Client-side token bucket (default 15 req/min)
Retry logic	3 attempts, exponential backoff (5s → 10s → 20s), transient-error detection
Quality audit	Per-form score, key-parameter completeness, missing / suspicious / format issues
Error summary	Grouped by type and by field, with counts across the whole batch
Manual editing	Modal with image pan/zoom/rotate (↺ ↻ Fit W Fit H 1:1) side-by-side with fields
Correction Library	Learns (path, AI value) → user value triplets; auto-applies after 2 confirmations
Excel export	3 sheets: EBRO Merged, Extraction Results, Parameter Summary
Error export	Separate workbook listing every failed file with reason and metadata
Wake Lock	Screen stays awake while processing — safe for long unattended batches
Persistence	Settings, API keys, folder handles, and corrections stored in localStorage / IndexedDB
🗂 Project Structure
text
scraplist/
├── index.html      # Markup, CDN imports (XLSX, heic-normalize)
├── style.css       # All styling (light theme, responsive 2-column layout)
├── config.js       # Platform configs, field schema, format rules, prompt builder
└── app.js          # Application logic (queue, AI calls, UI, exports)
config.js
PLATFORMS — Gemini / DeepSeek endpoint, models, hints, badge classes.

FIELD_SCHEMA — every field the AI must return, with a natural-language description.

FORMAT_RULES — regex + hint per field for client-side validation.

KEY_PARAMETERS — the fields that gate a form's "green" status.

OPTIONAL_FIELDS, HIDDEN_FIELDS — fields that may be empty or hidden from output.

buildPromptText() / buildJsonSchema() — generate the AI prompt and Gemini response schema.

app.js
Bootstrap — restores platform, API key, folder handles, input method.

Image pipeline — readExifOrientation, loadImageOriented, resizeImage, normalizeImageFormat.

Producer/Consumer — producer compresses the queue; consumer continuously drains it with AI calls.

Quality engine — computeQualityScore, buildKeyParameterSummary, groupIssuesByType.

Correction Library — recordCorrectionsFromEdit, applyLearnedCorrections, UI panel.

Exporters — onDownloadExcel, onDownloadErrorList.

File System Access — pickLocalFolder, loadLocalFiles, moveLocalFile, pullFileBackFromFolder.

🚀 Getting Started
1. Browser requirements
Feature	Chrome	Edge	Opera	Safari	Firefox
Upload mode	✅	✅	✅	✅	✅
Local Folder mode	✅	✅	✅	❌	❌
Wake Lock	✅	✅	✅	✅	⚠️ partial
Local Folder mode requires the File System Access API — Chrome, Edge, or Opera on desktop. On unsupported browsers, the Folder tab is disabled and the app falls back to Upload mode.

2. Serve the files
Because the app uses ES modules and the File System Access API, it must be served over HTTP(S) — not file://.

bash
# Any static server works, e.g.
python3 -m http.server 8080
# or
npx serve .
Then open http://localhost:8080.

3. Get an API key
Gemini → https://aistudio.google.com (free tier available)

DeepSeek → https://platform.deepseek.com (paid, with live balance widget)

Paste the key in the AI Configuration card. It's stored per-platform in localStorage (api_key_gemini, api_key_deepseek).

🧭 Usage
Upload mode (default everywhere)
Select 📤 Upload.

Drag & drop images onto the drop zone (or click to browse).

Optionally set a Path Prefix — prepended to the _file column in the export.

Leave Auto-process checked → the AI starts as soon as the first image is compressed.

Watch the queue, edit rows as needed, then click 📊 Download Merged Excel.

Local Folder mode (Chrome / Edge / Opera)
Select 📁 Folder.

Authorize three folders:

Input — where unprocessed forms live.

Readed — where successfully processed files are moved (as .jpg).

Error — where failures land so you can inspect and retry them.

Click 📥 Load Images — the folder is scanned, files are normalized and queued.

Click 🚀 Process & Move (or leave Auto-process on).

Each file is written to readed/ or error/ with a .jpg extension, and the original is deleted.

Folder handles are persisted in IndexedDB (ebro_fs_handles). On your next visit, just click Load — the browser will ask to re-grant permission once.

🧠 What the AI Extracts
The prompt enforces strict box-bounded reading, an anti-duplication rule, and an independent signature check. The schema is:

Section	Field	Notes
header	_validity	valid / invalid — internal, stripped before export
header	ticket_category	Process Scrap Parts (green band) / Supplier Claim Parts (orange band)
header	ticket_id	Large red printed number
section_1	codigo_conjunto	Long alphanumeric code
section_1	codigo_componente	e.g. Pilar B/Sup/Der
section_1	codigo_rechaz	3–5 chars, mixed digits/letters (e.g. 221M)
section_1	cantidad	1–3 digits (guard line excluded)
section_1	origen_area_zona	AREA + ZONA concatenated (e.g. M1M3)
section_2	motivo_rechace	Short defect phrase
section_2	fecha	DD/MM/YY style
section_2	observaciones	Optional — often null
section_2	operario	3–4 digit ID — often null
signatures	inspector / visto_bueno_calidad / encargado_linea	Signed / Not Signed
Post-processing (runs on every AI response)
unwrapSchemaEcho — unwraps any {value, type, description} echoes.

coerceNullStrings — turns literal "null" strings into real nulls.

normalizeSignatures + sanitizeSignatures — canonicalizes signature values.

normalizeTicketCategory — maps color words to the two valid categories.

dedupeFields — clears duplicated values across sibling fields.

flagSuspiciousSignatures — warns when all three signatures are Signed.

removeHiddenFields — drops header.company, header.document_type.

applyLearnedCorrections — replays the Correction Library.

📊 Quality Audit
Every rendered batch gets:

Data Quality score — weighted (70% key params + 30% other fields), 0–100%.

Per-key-parameter completeness grid — how many forms had each key field.

Issue summary — grouped by Missing, Format mismatch, Suspicious value, Manually edited.

Per-row status light — 🟢 all good · 🟡 non-key issues · 🔴 key-parameter issue.

Cell highlighting — missing (red), suspicious (yellow), format (red border), edited (green border).

Key parameters (configurable in config.js):

text
section_1.codigo_conjunto · section_1.codigo_rechaz · section_1.cantidad
section_1.codigo_componente · section_1.origen_area_zona
section_2.fecha · section_2.operario · section_2.motivo_rechace
signatures.encargado_linea
📚 Correction Library
When you edit a field in the modal and click Save changes, Scraplist records the tuple:

json
{ "path": "section_1.codigo_rechaz", "aiValue": "22104", "userValue": "221M", "count": 1 }
Once the same correction is seen ≥ 2 times (CORRECTION_MIN_COUNT), it becomes ACTIVE and is applied automatically to future extractions.

Open the library with the 📚 View Corrections button or inspect it from the console:

js
showCorrectionLibrary();               // console.table of all entries
clearCorrectionLibrary();              // wipe everything
deleteCorrection(path, aiVal, userVal) // remove one entry
Corrections are stored in localStorage under the key correction_library.

📤 Excel Export
ebro-batch-<timestamp>.xlsx
Sheet	Contents
EBRO Merged	_file + every flattened field, one row per successful form
Extraction Results	# · Status (GREEN/YELLOW/RED) · Reason · <all fields>
Parameter Summary	Quality score + per-key-parameter missing/present counts
ebro-errors-<timestamp>.xlsx
One row per failed form: # · _file · _error · _invalidImage · _sizeOriginalKB · _sizeResizedKB · _status · _movedTo.

⚙️ Configuration Reference
Storage key	Purpose
ai_platform	Active platform (gemini / deepseek)
api_key_gemini, api_key_deepseek	Per-platform API keys
gemini_model	Selected Gemini model ID
source_path_prefix	Prefix prepended to _file in Upload mode
max_image_size	Max long-edge dimension (default 1600 px)
input_method	folder or upload
correction_library	Array of learned corrections
IndexedDB ebro_fs_handles	Directory handles for Input / Readed / Error
Tunables in config.js:

RATE_LIMIT_PER_MIN (default 15)

CORRECTION_MIN_COUNT (default 2)

🛠 Troubleshooting
Symptom	Cause / Fix
"Your browser does not support the File System Access API."	You're on Firefox/Safari or a non-secure origin. Switch to Upload mode, or use Chrome/Edge/Opera over http(s)://.
"INVALID_IMAGE" errors	The AI decided the photo isn't a Control Calidad form. Retries are disabled for these by design.
Repeated 429 / 503 errors	You hit the model's rate limit. The retry logic backs off, but consider a Flash-Lite model or a paid tier.
Folders must be re-authorized every session	Browsers reset requestPermission on reload for security. One click per folder per session.
Balance panel stays empty	Only DeepSeek exposes a balance endpoint. Gemini doesn't.
Queue stuck on "Waiting"	The consumer hasn't started. Click 🚀 Start Batch / Process & Move.
Wake Lock not acquired	Some browsers release it when the tab is hidden. Scraplist re-requests it on visibilitychange.
🔒 Privacy
Images are sent directly from your browser to the selected AI provider's API.

No intermediary server, no analytics, no telemetry.

API keys never leave your machine except in the Authorization header toward the provider.

Folder handles are stored locally in IndexedDB; file contents are never uploaded anywhere except the AI endpoint.

🗺 Roadmap / Known Gaps
□ Batch history across sessions
□ CSV export alongside Excel
□ Multi-language UI toggle (ES / EN)
□ Signature-crop preview in the edit modal
□ Per-provider cost estimation widget
📄 License
Internal EBRO tooling — see your department's usage policy.

Ready for your improvements. Tell me what you'd like to add, rename, restructure, or expand (e.g. section-by-section rewrites, screenshots placeholders, badges, a CONTRIBUTING/CHANGELOG, Spanish translation, deployment notes for GitHub Pages, etc.) and I'll patch the README accordingly.



