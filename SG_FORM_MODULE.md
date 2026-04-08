# SG Form Generator Module

## Overview

Build a module within the existing project that lets users generate a filled-in Singapore application form as a downloadable PDF. The user pastes in a raw schedule (plain text), fills in a few additional fields, uploads a passport photo, and downloads the completed form.

**The PDF is generated directly from code — no PowerPoint, no LibreOffice, no template files.** The layout is rendered programmatically to match the exact format described below.

**Zero data persistence.** No user input, uploaded images, or generated files are saved to any database, S3 bucket, or permanent storage. All processing happens in memory or via temporary files that are deleted immediately after the PDF is served to the browser. No logs of form content. This is a stateless, fire-and-forget pipeline. If the server crashes mid-generation, stale temp files should be cleaned up on next startup or via a TTL-based sweep.

---

## PDF Layout Specification

Single landscape page (roughly 10" x 5.6" or equivalent A4/widescreen landscape). White background. All text is black. Font: a clean sans-serif (Arial, Helvetica, or system equivalent). The page has two visual columns:

### Left Column (approx 40% width)

**Top-left: Passport Photo**
- Positioned in the upper-left area with ~0.5" margin from top and left edges
- Image sized to approximately 3.5" wide x 2" tall
- Maintain aspect ratio; centre-crop if the uploaded image doesn't match
- No border

**Bottom-left: Witness Section**
- Starts roughly 60% down the page
- Bold header: `Witness`
- Then three numbered lines, regular weight, with generous line spacing (~1.5x):
  ```
  1. GM1: [NAME]
  2. GM2: [NAME]
  3. GM3: [NAME]
  ```

### Right Column (approx 55% width, starting ~45% from left edge)

**Top-right: Header Block**
- Starts at the top with ~0.3" margin from top edge
- Five lines, regular weight, standard line spacing (~1.3x):
  ```
  Date of Application: [SG DATE]
  Date of Journey: [START DATE] - [END DATE]
  Application Name: [APPLICANT NAME] [CHINESE NAME]
  Sponsor Name: [SPONSOR NAME]
  Direct GM: [DIRECT GM NAME]
  ```
- Font size: ~14pt

**Middle-right: Schedule Block**
- Starts after a blank line gap below the header block
- Eight lines, regular weight, same font size:
  ```
  DD/MM/YY: HHMMhr City tour [DRIVER]
  DD/MM/YY: HHMMhr KK [NAME]
  DD/MM/YY: HHMMhr GJ1 [NAME]
  DD/MM/YY: HHMMhr Numbers [NAME]
  DD/MM/YY: HHMMhr GJ2 [NAME]
  DD/MM/YY: HHMMhr GJ3 [NAME]
  DD/MM/YY: HHMMhr GJ4 [NAME]
  DD/MM/YY: HHMMhr FK [NAME(S)]
  ```

**Bottom-right: Footer Block**
- Positioned in the lower portion of the right column, vertically aligned roughly with the witness section
- Two lines:
  ```
  Applied Share: [NUMBER] shares
  China Unicom Number: [PHONE]
  ```

### Visual Reference

Use the reference image `SG_LAYOUT_REFERENCE.jpg` (included in this repo at `[INSERT PATH]`) as the pixel-accurate target. The generated PDF must match this layout closely.

---

## Input Format

Users paste a raw text block in this format (the "schedule input"):

```
Name of Sponsor: Tan Geok Yian
Name of Guest: Harrison Phong (Guest)
13/4 1pm Time of Flight Land✅
13/4 2:30pm Airport Transfer & City Tour: 芳芳✅
14/4 2pm KK-Mag Yeo✅
14/4 5:30pm GJ1-Shaun Goh✅
15/4 9am URA✅
15/4 2pm Numbers-Alex Lim✅
15/4 6pm GJ2-Agnes Oon✅
15/4 7pm Family Dinner✅
15/4 9:30pm 3rd Night: Wendy and Joenna GM
16/4 10:30am GJ3-Carina Ng✅
16/4 12:30pm Register phone Card 🔄 Wendy will arrange**
16/4 3:30pm GJ4-Daniel Png✅
16/4 6pm FK: Wendy and Joenna GM✅
16/4 6pm SG: Moxhan (GM-Rachell,Joenna,Wendy)
Name of Direct GM: Wendy
16/4 7pm AP Networking Dinner✅
18/4 11:30am Airport Transfer✅
18/4 2:00pm Time of Flight home✅
```

### Parsing Rules

Extract the following from the raw text:

1. **Sponsor Name** — line starting with `Name of Sponsor:`
2. **Guest/Applicant Name** — line starting with `Name of Guest:` (strip any parenthetical like "(Guest)")
3. **Direct GM first name** — line starting with `Name of Direct GM:`
4. **Schedule events** — each line matching pattern `DD/M[M] H:MMam/pm <event description>`

From the schedule events, extract specifically these items by matching keywords:

| Keyword in line | Template field | What to extract |
|---|---|---|
| `City Tour` or `Airport Transfer & City Tour` | City tour line | date, time, name after `:` (the driver) |
| `KK-` or `KK -` | KK line | date, time, name after `-` |
| `GJ1-` | GJ1 line | date, time, name after `-` |
| `Numbers-` | Numbers line | date, time, name after `-` |
| `GJ2-` | GJ2 line | date, time, name after `-` |
| `GJ3-` | GJ3 line | date, time, name after `-` |
| `GJ4-` | GJ4 line | date, time, name after `-` |
| `FK:` or `FK -` | FK line | date, time, name(s) after `:` (strip "GM" suffix) |
| `SG:` | Used to derive SG Date | the date of this line = Date of Application |
| `Time of Flight Land` | Start Date | the date of this line |
| `Time of Flight home` | (informational only) | NOT used for End Date |

### Date Logic

**Date of Application (SG Date):** The date from the line containing `SG:`.

**Start Date:** The date from the `Time of Flight Land` line.

**End Date — 5-Day Rule:** The journey must always span exactly **5 days inclusive** (Start Date = Day 1, End Date = Day 5). Therefore:
- End Date = Start Date + 4 calendar days. Always.
- Example: Start Date 13/04 -> End Date 17/04 (13, 14, 15, 16, 17 = 5 days).
- The flight home date and SG date are irrelevant to this calculation.

All dates on the PDF use `DD/MM/YYYY` format (e.g. `16/04/2026`).
Schedule line dates use `DD/MM/YY` format (e.g. `16/04/26`).
Assume current year if no year is provided in the input.

**Time format conversion:** Input times like `2:30pm` -> 24hr military format without colon: `1430`. Examples: `5:30pm` -> `1730`, `9am` -> `0900`, `10:30am` -> `1030`.

**All names must be fully UPPERCASED** in the final output.

Strip emoji (✅, 🔄) and trailing markers like `**` from parsed content.

Lines that don't match any template field (e.g. `URA`, `Family Dinner`, `3rd Night`, `Register phone Card`, `AP Networking Dinner`, `Airport Transfer` without City Tour) are informational only and are NOT placed into the PDF.

---

## Additional Form Fields (user fills in manually via UI)

These fields are NOT parsed from the schedule text. The user fills them in separately:

| Field | Type | Notes |
|---|---|---|
| Applicant Chinese Name | text input | Optional. If blank, omit entirely (no trailing space after the English name). |
| Applied Shares | select: `66` or `108` | Required. |
| China Unicom Number | text input | Required. The applicant's China phone number. |
| Witness GM1 Full Name | text input | Required. Must be full name. |
| Witness GM2 Full Name | text input | Required. Must be full name. |
| Witness GM3 Full Name | text input | Required. Must be full name. |
| Direct GM Full Name | text input | Required. The schedule only gives a first name (e.g. "Wendy"). User provides the full name here (e.g. "WENDY TAN"). |
| Passport Photo | file upload | Required. JPEG or PNG. |

All name fields are uppercased in the final output regardless of how the user types them.

---

## PDF Generation

### Recommended Approach

Use a PDF generation library that supports:
- Precise x/y text placement
- Image embedding (JPEG/PNG)
- Unicode/CJK characters (for Chinese names and characters like 芳芳)

Good options depending on your stack:
- **Node.js:** `pdf-lib` (lightweight, no native deps) or `pdfkit`
- **Python:** `reportlab` or `fpdf2`
- **Browser-side:** `jsPDF` (if you want fully client-side generation with zero server involvement — ideal for the no-persistence requirement)
- **HTML-to-PDF:** Render an HTML page matching the layout, convert via `puppeteer` or `playwright`. This is the easiest path for pixel-accurate layout but requires a headless browser on the server.

### Fully Client-Side Option (Recommended)

If the existing project is a web app, **generate the PDF entirely in the browser** using `jsPDF` or `pdf-lib`. This eliminates all server-side processing and storage concerns by design — data never leaves the user's browser. No template files are needed; the layout is defined in code.

Flow:
1. User pastes text, fills form, uploads passport photo
2. JavaScript parses everything client-side
3. PDF is generated in-browser using the PDF library
4. Browser triggers a download of the generated blob
5. Nothing is sent to any server

### CJK Font Support

The layout may include Chinese characters (e.g. driver name 芳芳, applicant Chinese name). Ensure the PDF library is configured with a font that supports CJK characters. For `jsPDF`, this means embedding a CJK-capable font (e.g. Noto Sans SC). For `pdf-lib`, embed the font file. This is critical — missing CJK support will render boxes or blank spaces instead of characters.

### Passport Image Handling

- Accept JPEG or PNG uploads
- Resize to fit the designated area (~3.5" x 2") maintaining aspect ratio
- Centre-crop if aspect ratio doesn't match (don't stretch/distort)
- Embed directly in the PDF
- For client-side generation: read the file as a data URL / ArrayBuffer, pass to the PDF library

### Output

- Filename: `SG_[APPLICANT_NAME].pdf` (e.g. `SG_HARRISON_PHONG.pdf`)
- Single landscape page
- Trigger browser download immediately on generation

---

## UI Flow

1. **Step 1 — Paste Schedule:** Large textarea. User pastes the raw schedule text. A "Parse" button extracts and previews the parsed data (sponsor, guest, all schedule items with dates/times, SG date, start date, calculated end date). Show a confirmation summary so the user can verify before proceeding. Highlight any missing fields (e.g. no City Tour line found).

2. **Step 2 — Additional Details:** Form fields for: Applicant Chinese Name, Applied Shares (dropdown), China Unicom Number, Direct GM Full Name (pre-fill the first name from parsed data as a hint), Witness GM1/GM2/GM3 full names.

3. **Step 3 — Upload Passport:** File upload for passport photo (JPEG/PNG). Show a preview thumbnail.

4. **Step 4 — Generate & Download:** "Generate PDF" button. Show a brief loading state. On completion, trigger PDF download. Show a success message. No data is retained after this point.

Optionally: allow the user to go back and edit any step before generating. A single-page form with all steps visible is also fine — the step breakdown is a UX suggestion, not a hard requirement.

---

## Error Handling

- If the schedule text is missing required lines (no City Tour, no KK, etc.), highlight which fields are missing and let the user proceed with blanks or fix the input.
- If dates can't be parsed, flag the specific line.
- If no `SG:` line is found, prompt the user to manually enter the SG Date.
- Validate that all required form fields are filled before enabling the Generate button.
- If passport image is too small (< 200px on either dimension), warn but allow.
- If CJK characters are detected but the font doesn't support them, show a warning.

---

## Tech Notes

- **No database, no persistent storage.** Nothing is saved anywhere. Prefer fully client-side PDF generation to eliminate server involvement entirely.
- **No PPTX dependency.** The PDF is generated from code. No template files, no LibreOffice, no PowerPoint manipulation. The layout is defined programmatically.
- **No server-side file handling** if using the client-side approach. The passport image is read in-browser and embedded directly into the PDF blob.
- The reference image `SG_LAYOUT_REFERENCE.jpg` is provided for visual accuracy — the generated PDF should closely match this layout. It is a development reference only and is not used at runtime.
- Test with Chinese characters in both the driver name field and the applicant Chinese name field to verify CJK rendering.
