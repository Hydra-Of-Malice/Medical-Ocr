<div align="center">

# 🩺 Medical OCR

### Medical reports in, exact text out.

**Numbers kept exactly as printed · Unclear text flagged, never guessed · Runs on your own server by default**

[![Status](https://img.shields.io/badge/status-research%20phase-2f6fde?style=for-the-badge)](OCR_RESEARCH.md)
![Platform](https://img.shields.io/badge/platform-self--hosted%20Python%20%28planned%29-555?style=for-the-badge)
![Release](https://img.shields.io/badge/release-none%20yet-9a6700?style=for-the-badge)
![Cloud OCR](https://img.shields.io/badge/cloud%20OCR-off%20by%20default-00a86b?style=for-the-badge)

</div>

> **Status:** the research is done and no code has been written yet. There is nothing to install or run today. This README describes the planned design and says so wherever it matters.

Medical OCR is the text-extraction part of IDP (Intelligent Diagnostic Platform). When it is built, you will upload a lab report PDF, a scanned prescription, or a phone photo of a pathology report and get back its text, tables, and values exactly as written, with every word traceable to its place on the page. Today the repository holds the research that chose the tools: [OCR_RESEARCH.md](OCR_RESEARCH.md).

## 💡 Why you'll like it

These are the design goals set by the research. None of them are built yet.

| | |
|---|---|
| 🔢 **Numbers stay exact** | A lab value of 10.2 must never come out as 102, so tools known to invent characters were ruled out. |
| 🚩 **Doubts are flagged** | Each piece of text keeps a confidence score, and unclear text is marked instead of quietly "fixed". |
| 📊 **Tables keep their shape** | Test, result, unit, and reference range stay together in the same row. |
| 📱 **Phone photos welcome** | Tilted, shadowed, or blurry photos get a correction only when a quality check calls for it. |
| 🔒 **Documents stay with you** | OCR runs on your own server. A cloud OCR service would be optional and need explicit consent. |
| 🔍 **Every word checkable** | Each word keeps its page and position, so you can compare it with the original. |
| 📄 **Mixed PDFs handled** | Each page is checked on its own: digital text is read directly, scanned pages go to OCR. |

## 🧭 Three steps

This is the planned flow. It does not exist yet.

1. **Upload a document.** Send a PDF, JPG, JPEG, PNG, or WEBP file, one page or many.
2. **Let it process.** Pages are sorted, cleaned up if needed, and read in the background while you check the job status.
3. **Review the result.** Get one structured file per document, with low-confidence text flagged for you to check.

## 🌐 Get it

There is nothing to download yet. No release exists and no application code has been written. You can read the research now:

1. Open [OCR_RESEARCH.md](OCR_RESEARCH.md) on GitHub, or clone the repository (see Development below).
2. Watch the repository on GitHub to hear when the prototype lands.

Planned requirements, from the research:

| Requirement | Details |
|---|---|
| Input files | PDF, JPG, JPEG, PNG, WEBP. Digital or scanned, single or multi-page. |
| Documents | Mainly printed medical documents: lab and pathology reports, prescriptions, receipts |
| Server | Self-hosted Python service. Python 3.11 or newer is the current target. |
| Hardware | Runs on CPU. A GPU is recommended for the table and layout model, which is slow on CPU. |
| Internet | Not needed for OCR in the default setup |
| Not supported | Reliable handwriting reading. Medical interpretation or diagnosis, translation, summaries, doctor recommendations, and drug-interaction checks are out of scope for this module. |
| Not tested | Everything. Nothing has been built or measured yet. |

## 🔍 What it does

Planned stages, from the research:

| Stage | What happens |
|---|---|
| Upload check | The file type is checked from its contents, not its name. Size is capped. The file is stored under a generated name. |
| Page sorting | Each page is classified as digital text, scanned or photographed, or carrying an old OCR text layer that should not be trusted. |
| Digital pages | Text and its positions are read straight from the PDF. Tables are read from the PDF's own layout. |
| Image cleanup | A blur check and orientation fix always run. Deskew, perspective fix, shadow removal, and light denoising run only when a check calls for them. No black-and-white conversion by default and no AI upscaling. |
| OCR | Scanned and photographed pages are read by PaddleOCR, with Tesseract as a fallback and cross-check. |
| Tables | A table model rebuilds rows and columns. Grouping word boxes by position is the fallback. |
| Output | One JSON document per upload: pages, text, word positions, confidence scores, how each page was read, and which cleanup steps ran. |

## ⚙️ How it works

```text
 upload ─► file check ─► page sorter
                             │
             ┌───────────────┴───────────────┐
             ▼                               ▼
       digital page                   scanned / photo
             │                               │
     PDF text + tables           cleanup (only if needed)
             │                               │
             │                         OCR ─► tables
             │                               │
             └───────────────┬───────────────┘
                             ▼
                one JSON file per document
               (text, positions, confidence)
```

Processing is planned as a background job, so large documents do not block the upload. Full reasoning, rejected alternatives, and sources are in [OCR_RESEARCH.md](OCR_RESEARCH.md).

Planned components. None of them are installed or bundled yet.

| Component | Purpose | License |
|---|---|---|
| PyMuPDF | PDF text, positions, and page rendering | AGPL-3.0, or a paid Artifex commercial license |
| pdfplumber, Camelot | Tables in digital PDFs | MIT |
| PaddleOCR (PP-OCRv5, PP-StructureV3) | Main OCR engine, layout, and table structure | Apache-2.0 (code and published models) |
| PaddlePaddle | Framework PaddleOCR runs on | Apache-2.0 |
| Tesseract 5 | Fallback OCR and cross-check | Apache-2.0 |
| FastAPI | Upload and status API | MIT |
| ARQ with Redis, or a simple database-status worker | Background jobs | ARQ: MIT. Redis 8 and later: RSALv2, SSPLv1, or AGPLv3. Redis 7.2 and earlier: BSD-3-Clause. |

## 🛡️ Responsible use

- **Not a medical device.** This module transcribes documents. It does not diagnose, interpret results, or give medical advice.
- **Check every value.** OCR can misread digits, decimal points, and units. Compare important values with the original document before acting on them.
- **Patient privacy.** Treat every uploaded document as sensitive. The design keeps documents on your own server, keeps document content and patient details out of logs, and makes any cloud OCR provider opt-in with explicit consent.
- **No real patient data for testing** without proper authorization. Use synthetic or properly de-identified documents.
- **Legal checks are yours.** The research notes on India's DPDP Act and HIPAA are background, not legal advice, and some details are marked unverified.
- **Today no code runs,** so nothing is processed, stored, or sent anywhere.

## ⚠️ Known limits

- No application code exists. Nothing can be installed, run, or tested yet.
- Accuracy has not been measured. Figures in the research come from third-party benchmarks, not from tests on this project's documents.
- Handwriting is weak in every open-source engine reviewed. In one test PaddleOCR had about a 24% character error rate on handwriting. The plan is to flag it as low confidence, not to read it reliably.
- Tables with merged headers, multi-line cells, no borders, or skew are known failure modes.
- Some photos will be unreadable. The design reports low confidence or failure for them instead of guessing.
- Language support has not been decided or tested.
- PaddleOCR pulls in PaddlePaddle, a large install. Table and layout recognition on CPU will be slow for multi-page documents.
- PyMuPDF is AGPL-licensed. A closed-source product or a hosted service without source disclosure would need a commercial license or a switch to permissively licensed PDF libraries.
- There is no test dataset and there are no automated tests.
- Details of India's DPDP Rules 2025 were not confirmed in the research.
- This is a small student/team project. The design stays simple on purpose: one OCR engine at a time behind a swappable interface, one job queue, one output format.

## 🛠️ Development

Prerequisites: Git.

```bash
git clone https://github.com/Hydra-Of-Malice/Medical-Ocr.git
cd Medical-Ocr
```

| Folder / file | Contents |
|---|---|
| `OCR_RESEARCH.md` | Technology comparisons, decisions, rejected options, risks, testing plan, and sources |
| `README.md` | This file |
| `.gitignore` | Ignores Python caches, virtual environments, `.env`, and `node_modules/` |

There is nothing to build yet. The work follows a research-first plan:

| Stage | Output | Status |
|---|---|---|
| Research | `OCR_RESEARCH.md` | Done |
| Architecture | `OCR_ARCHITECTURE.md`: components, data flow, API, error handling, storage | Not started |
| Prototype | Test the riskiest decisions: numeric accuracy on realistic lab values, the digital-vs-scanned page check, the cleanup triggers | Not started |
| Implementation | Upload, PDF and image pipelines, OCR, structured output, API, a minimal test front end. Changes logged in `OCR_IMPLEMENTATION_LOG.md`. | Not started |
| Testing | Measured text, number, table, and layout accuracy on realistic documents | Not started |
| Final report | `OCR_FINAL_REPORT.md`: what was built, tested, and its limits | Not started |

## 📄 License

All rights reserved. There is no license file. The repository is public to read, but no license is granted to reuse its contents. No third-party code is included yet; the planned components and their licenses are listed in How it works above.
