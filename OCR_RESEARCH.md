# OCR_RESEARCH.md — IDP OCR/Document Extraction Module

**Status:** Research phase complete. No implementation has started yet (see `README.md` for current project status and next steps).
**Date of research:** 2026-08-29
**Scope:** OCR/document-extraction layer only, per the IDP OCR Module master prompt. This document does not cover medical interpretation, translation, or any downstream IDP module.

**Method note:** This research was conducted as five parallel, targeted investigations (OCR engines, PDF extraction, table extraction/layout analysis, image preprocessing, and privacy/architecture), each required to cite sources for factual claims and to flag anything it could not verify rather than guess. Vendor marketing claims are treated skeptically and cross-checked against independent benchmarks/discussions where possible. All sources are listed in [§12 References](#12-references). Legal claims (India's DPDP Act) are explicitly flagged where a detail could not be confirmed — treat those as unverified, not authoritative.

---

## 1. Executive Summary

**Recommended stack for the MVP:**

| Layer | Recommendation | Rejected alternatives (MVP) |
|---|---|---|
| PDF native extraction | **PyMuPDF** (text + page rendering) | Apache Tika |
| PDF table extraction (native-text PDFs only) | **pdfplumber** / **Camelot** (fast path) | — |
| Digital-vs-scanned page classification | Custom heuristic: text density + image coverage + font/encoding sanity + glyphless-font detection | — |
| OCR engine (primary) | **PaddleOCR (PP-OCRv5 + PP-StructureV3)**, self-hosted | Surya (license risk), general VLMs (hallucination risk) |
| OCR engine (fallback/cross-check) | **Tesseract 5** | — |
| Table structure recognition (image/OCR path) | **PP-StructureV3 table module**, with bbox row/column clustering as a free fallback | Donut, LayoutLMv3 (generative hallucination risk), Table Transformer (v2 candidate only) |
| Image preprocessing | **Conditional pipeline**, gated by quality checks — nothing applied unconditionally except orientation fix and the blur-quality gate | Default binarization, generative super-resolution |
| Cloud OCR | **Optional, pluggable, consent-gated provider** behind an abstraction — not the default | Using cloud OCR as the default path |
| Async processing | **FastAPI + ARQ/Redis** (or a minimal DB-status worker if the team wants to avoid running Redis) | Bare `BackgroundTasks` alone, Celery+Redis |
| Storage of OCR output | **One JSON blob per document** (page array, per the master prompt's schema) | Fully normalized relational schema (premature) |
| Privacy default | **Self-hosted/offline OCR**, treat all medical documents as high-sensitivity regardless of legal classification | Cloud OCR as default for real documents |

**Why this stack:** every candidate technology was evaluated primarily against the project's non-negotiable constraints — numeric fidelity (no invented digits), traceability (bounding boxes + confidence), table structure preservation, and not sending patient data to third parties by default. Several popular options were rejected specifically because they violate one of these constraints even though they are otherwise capable (see [§9 Rejected Alternatives](#9-rejected-alternatives)).

---

## 2. Problem Analysis

What makes this OCR problem harder than generic document OCR (see master prompt §3, §6):

1. **Numeric fidelity is the primary success metric, not prose readability.** A lab value like `10.2` silently becoming `102` or `0.5` becoming `5` is a correctness failure with real-world consequence downstream, even though it might look like a "minor" OCR slip on ordinary text. This constraint rules out several techniques (generative VLM OCR, super-resolution, aggressive binarization) that are acceptable for general-purpose OCR but unacceptable here.
2. **Input quality is highly variable and often adversarial-by-accident:** phone photos with tilt, shadows, uneven lighting, and blur are a first-class input, not an edge case — most real users will photograph a report rather than scan it.
3. **Tables carry the highest-value information** (test/result/unit/reference-range rows) and are also the structure most easily corrupted by both bad OCR and naive preprocessing (merged cells, lost row/column alignment).
4. **The system must preserve uncertainty rather than resolve it.** Most OCR/document-AI tooling is built to produce a clean, confident-looking output; this project explicitly needs the opposite — confidence scores and low-confidence flags must survive to the output, not be smoothed away.
5. **Documents mix extraction regimes.** A single upload can be a clean digital PDF, a scanned PDF, or a multi-page PDF with some of each — the pipeline must classify and route per page, not per document.
6. **Privacy stakes are high** even without a directly-applicable law: these are medical documents, often containing patient-identifying information, being processed by a small team without institutional compliance infrastructure.

---

## 3. PDF Extraction — Technology Comparison

### Library comparison

| Library | Speed | Layout/coordinates | Table support | License | Verdict |
|---|---|---|---|---|---|
| **PyMuPDF (fitz)** | Fastest (~180 pages/sec plain text; ~8–12x faster than pdfplumber) | Yes — per-span coordinates, fonts, sizes; also renders pages to images natively (no external binary) | Basic | **AGPL** (or paid commercial license) | **Primary engine** for extraction + rendering |
| **pdfplumber** | Slow (~18 pages/sec) | Yes, geometry-based | **Best-in-class** table detection | MIT | **Use specifically for table-shaped regions** |
| **pypdf** | Fast for simple text | Minimal | Weak | BSD (permissive) | Not selected — no advantage over PyMuPDF for our needs |
| **pdfminer.six** | Slower, low-level | Very detailed (underlies pdfplumber) | N/A directly | MIT | Not used directly (used via pdfplumber) |
| **Apache Tika** | N/A — JVM service | N/A | N/A (shells out to Tesseract for OCR) | Apache-2.0, but requires a JVM | **Rejected** — adds a Java runtime dependency for no accuracy gain over pure-Python libraries; its own OCR path just wraps Tesseract anyway |

**Decision:** PyMuPDF for native text extraction and page rendering (speed, built-in rasterization for the OCR-fallback path, positional metadata for bounding boxes). pdfplumber (or Camelot as a faster first pass) specifically for table-shaped regions in native-text PDFs, since PyMuPDF's own table support is weaker.

**Open licensing question (recorded, not yet resolved):** PyMuPDF is AGPL-licensed (or requires a paid commercial license from Artifex). This is acceptable for a self-hosted, open-source student project but must be revisited if the project ever needs closed-source or SaaS-without-source-disclosure distribution. Fallback if that becomes a problem: pypdf + pdfplumber only (both permissively licensed), at some cost to speed and rendering convenience.

### Digital-vs-scanned page classification

No single signal is reliable; the recommended heuristic combines several, evaluated **per page** (documents are frequently mixed):

- **Text density:** extracted character count below a threshold relative to page area → suspect.
- **Image coverage:** one or two images covering >~80–85% of the page rectangle → almost certainly a scan/photo even if some machine-added text (e.g. a header) is present.
- **Font embedding check:** real digital text has embedded/referenced fonts; a pure-image page has none.
- **Garbled-text / encoding sanity check:** if extracted text is non-sensible or highly repetitive, it may be a prior bad OCR pass baked into the PDF — should route to fresh OCR, not be trusted as ground truth.
- **Glyphless/invisible font detection:** PDFs previously processed by tools like OCRmyPDF embed an invisible "GlyphLessFont" text layer under the scanned image. Naive extraction will "succeed" and silently return someone else's old, possibly low-quality OCR output as if it were native text. This must be detected and tagged distinctly (`extraction_method: "prior_ocr_layer"`), not conflated with genuine digital text.

**Recommended rule:** `has_usable_text = (char_count_per_page > threshold) AND (max_image_coverage_ratio < ~0.85) AND (text passes sanity check) AND (fonts are not glyphless)`. If false → render that page and route it to OCR.

### Rendering pages to images (OCR fallback)

Use PyMuPDF's `get_pixmap(dpi=...)` rather than pulling in `pdf2image`/Poppler as a separate dependency. **300 DPI is the consensus default** — higher rarely improves OCR accuracy meaningfully but increases memory (~25MB/page at 300 DPI) and processing time; DPI should be configurable per document so a low-confidence result can trigger a re-render/retry at higher DPI.

### Other gotchas to design around

- **Text-as-curves PDFs:** some generators render characters as vector outlines with no real text objects — `get_text()` returns nothing despite the page "looking digital." Caught by the same low-char-count + high-vector-coverage heuristic.
- **Encrypted/password-protected PDFs:** attempt decrypt with an empty password (common for owner-only restrictions); otherwise fail explicitly (`processing_status: "failed"`) rather than crash.
- **Rotated pages:** read the PDF's `/Rotate` metadata and apply it before rendering for OCR.

---

## 4. OCR Engine — Technology Comparison

Evaluated: Tesseract 5, PaddleOCR (PP-OCRv4/v5/v6 + PP-StructureV3), EasyOCR, docTR, Surya OCR, cloud APIs (AWS Textract, Google Document AI, Azure AI Document Intelligence), and vision-language-model approaches (general multimodal LLMs and purpose-built OCR-VLMs like GOT-OCR2.0, olmOCR, Nougat, DeepSeek-OCR).

| Engine | License | Runs offline | Table/layout model | Notable finding | Verdict |
|---|---|---|---|---|---|
| **Tesseract 5** | Apache-2.0 | Yes, CPU-only | No structural table model; word boxes only | ~18% CER on one benchmark; accuracy "plummets" past ~5° skew | Lightweight fallback/cross-check, not primary |
| **PaddleOCR (PP-OCRv5/v6 + PP-StructureV3)** | Apache-2.0 | Yes (CPU-capable, GPU recommended for structure module) | **Yes** — integrated layout + table-structure + cell OCR pipeline | PP-OCRv5 claims parity with much larger VLMs on OCR benchmarks at 5M params; weaker on handwriting (24% CER vs EasyOCR's 16% in one test) — acceptable since our brief is primarily printed documents | **Primary/self-hosted engine** |
| **EasyOCR** | Apache-2.0 | Yes | No table/layout model, boxes only | Easiest prototyping API; better than PaddleOCR on handwriting in one test | Good for a quick spike, not selected for production pipeline |
| **docTR (Mindee)** | Apache-family | Yes | Word/line boxes, no dedicated table model | No verified head-to-head vs PaddleOCR found — only self-published comparisons vs cloud APIs | Credible alternative if PaddleOCR's dependency weight becomes a problem; not selected for MVP |
| **Surya OCR** | **GPL-3.0** code; model weights under a revenue-gated "Rail-M" license (free under $2M revenue) | Yes | Yes, strong layout analysis (88% Publaynet) | 87.2% pass rate across 91 languages; capable, but license risk | **Rejected for MVP** — copyleft + revenue-gated model license is a real risk for a project that may become a public/commercial product |
| **Cloud APIs** (AWS Textract, Google Document AI, Azure Document Intelligence) | Commercial | No (sends data off-device) | Yes, strong (tables/forms add-ons) | Production accuracy realistically 80–95%, below vendor-advertised 95–99%; ~$1.50/1,000 pages raw, up to ~$50/1,000 pages for structured extraction; AWS Textract is HIPAA-eligible | **Optional, pluggable provider — not MVP default** (cost + sends medical data to a third party) |
| **General VLMs** (Claude/GPT-4o/Gemini as OCR) | Commercial | No | N/A | **Documented to hallucinate** — invent plausible-looking values for fields they can't read clearly (arXiv 2506.20168), because instruction-tuning rewards confident answers over admitting uncertainty | **Excluded from MVP** — directly conflicts with the "never silently fix uncertain OCR results" requirement |
| **Purpose-built OCR-VLMs** (GOT-OCR2.0, olmOCR, Nougat, DeepSeek-OCR) | Mixed | Mostly yes, GPU-heavy | Yes | Promising research direction, some rival larger general VLMs, but immature/heavy for a small team to productionize now | Not selected for MVP; revisit later behind an explicit verification layer |

**Ranked recommendation:**
1. **PaddleOCR (PP-OCRv5 + PP-StructureV3)** — only open-source candidate with an integrated table-structure model matching the lab-report use case; self-hosted (no data leaves the server); CPU-runnable for MVP scale with a GPU upgrade path.
2. **Tesseract** — lightweight secondary engine: useful as a fallback where PaddlePaddle can't be installed, and as a cross-check signal for confidence scoring.
3. **A cloud API (Azure Document Intelligence or AWS Textract)**, behind the same provider abstraction — architected as swappable, not used by default, gated behind explicit user consent and (if ever used on real documents) a signed BAA-equivalent agreement.

---

## 5. Table Extraction & Layout Analysis

Target case:
```
TEST       RESULT       UNIT       REFERENCE RANGE
Hb         10.2         g/dL       12–16
WBC        7800         /µL        4000–11000
```

| Approach | Works on scanned images? | Output | Verdict |
|---|---|---|---|
| **PaddleOCR PP-StructureV3 table module** | Yes | Structured HTML/LaTeX table markup with cell text + row/col/span structure | **Primary path** — already in the same library as the chosen OCR engine; scored 96.3% on OmniDocBench v1.6 (leading on text/formula/table sub-tasks) |
| **Camelot / Tabula** | **No** — reads the PDF's internal vector/text-position data, not pixels | Exact tables from native PDFs | Use only on the **native-text PDF branch**, as a fast path before falling back to the image pipeline |
| **Bbox row/column clustering heuristic** (cluster OCR word boxes by y then x) | Yes (works on whatever OCR already produced) | Reconstructed rows/columns from geometry | **Free fallback/cross-check** — works well on clean, aligned, single-line tables; documented failure modes: merged header cells, multi-line cell text, borderless tables, skew |
| **Table Transformer (TATR, Microsoft)** | Yes | HTML/CSV via a two-stage detection+structure DETR model | Solid and well-benchmarked (PubTables-1M), but trained on financial/scientific tables, not medical lab reports — accuracy on our layouts is unverified. **v2 candidate**, not MVP |
| **LayoutLMv3 / Donut** | Donut: yes (OCR-free); LayoutLMv3: needs a separate OCR pass | Field/value extraction or document QA | **Rejected.** Donut's decoder is generative and can paraphrase/hallucinate digits rather than copy exact characters — directly incompatible with the numeric-fidelity requirement. Neither gives bounding boxes/confidence per token. Fine-tuning either needs a labeled dataset the team doesn't have. |

**Decision:** PP-StructureV3's table module is the primary table-extraction path for the image/OCR branch; bbox clustering is kept as a zero-cost fallback (useful when PP-Structure misses a table, and reusable on the native-PDF branch). Camelot is tried first on native-text PDFs. Table extraction is architected as a **distinct stage after OCR/layout detection**, not folded into free-text OCR — raw OCR text/boxes are retained regardless, for traceability and fallback. Table Transformer is a recorded v2 evaluation candidate if PP-Structure proves insufficient on real Indian lab-report layouts.

---

## 6. Image Preprocessing Research

Central constraint from the master prompt: preprocessing must never risk turning `10.2` into `102`. This shaped every verdict below.

| Technique | Verdict | Trigger condition |
|---|---|---|
| Blur detection (variance of Laplacian) | **Always run**, as a gate, not a fix | Always — cheap, CPU-only; flags/warns rather than hard-rejects, since the heuristic can misfire |
| Orientation fix (90/180/270°) | **Always run** | Always — OCR fails outright on unrotated 90°+ pages |
| Deskew (sub-90° tilt) | Conditional | Only if measured skew exceeds ~2–3° |
| Perspective correction (document-boundary detection + homography) | Conditional | Only if a photographed (non-flat) document is detected with sufficient contour confidence — a no-op/risk on an already-flat scan |
| Shadow/uneven-lighting correction (background subtraction or CLAHE) | Conditional | Only if a lighting-uniformity check fails — CLAHE on an already-clean image amplifies noise/JPEG artifacts |
| Denoising/sharpening | Conditional, light-touch only | Only if a noise metric is high; mild bilateral filtering only — **never hard morphological erosion/dilation**, since decimal points, minus signs, and punctuation are exactly the thinnest strokes on a page and the first casualty of aggressive denoising (well-supported instance of the documented "thin-stroke loss vs. background-noise survival" tradeoff in binarization/denoising literature) |
| Binarization/thresholding | **Not a default** for the deep-learning OCR path | PaddleOCR's models are trained on RGB images; converting to grayscale/binary collapses information they rely on. Reserve thresholding for the legacy Tesseract fallback path only, which tolerates it better |
| Super-resolution / upscaling | **Not recommended**, or deterministic-only with extreme caution | GAN-based super-resolution (e.g. Real-ESRGAN) is documented to hallucinate fine detail on low-resolution/text inputs — trading fidelity for "plausibility," which is the exact failure mode this project must prevent. If a resolution floor must be met, use deterministic bicubic/Lanczos upscaling only, never generative/diffusion super-resolution on text regions |

**Recommended pipeline (steps only run when their trigger condition fires; every step taken is logged per-document for traceability):**

```
1. Quality gate — blur score + exposure check → tag quality, warn if poor (don't block)
2. Orientation fix — always
3. Document boundary detection → perspective correction — only if photographed doc detected
4. Deskew — only if skew > ~2-3°
5. Shadow/illumination correction — only if lighting non-uniform
6. Light denoise (mild bilateral only) — only if noise metric high
7. No default binarization for deep-learning OCR path (RGB/grayscale in)
8. No generative super-resolution; deterministic upscaling only if below a resolution floor
```

---

## 7. Medical-Document-Specific Considerations

Synthesized from the OCR-engine and table-extraction research, applied specifically to the document types in master prompt §6/§20:

- **Numeric character confusion** (0/O, 1/l/I, 5/S, 8/B) is the single highest-value error class to test for and monitor via confidence scores, since it silently corrupts lab values without producing an obviously malformed string.
- **Units and symbols** (g/dL, µL, mmHg, %, <, >, –) must be preserved verbatim; several of these are exactly the "thin stroke" characters most at risk from aggressive preprocessing (see §6) — reinforcing the conditional/light-touch preprocessing decision.
- **Reference ranges** written as `12–16` or `4000–11000` are effectively small tables/pairs and should be treated as structured data (via the table pipeline), not flattened into free text where the range delimiter could be lost or misread as a hyphen-minus vs. en-dash.
- **Medical terminology and abbreviations** are not corrected or "spell-fixed" by the OCR layer — per the master prompt's uncertainty-preservation rule, this is explicitly out of scope for this module (it belongs to a downstream module with its own evidence-based correction process, if any).
- **Handwritten annotations** (common on prescriptions and doctor's notes) are a known weak point for every open-source engine evaluated (PaddleOCR: 24% CER; Tesseract: worse). No candidate technology solves this well within MVP constraints — the recommended handling is to flag low-confidence regions rather than attempt to force a clean transcription (Azure Document Intelligence was noted as comparatively stronger on handwriting if a cloud fallback is ever justified for this specific case).
- **Stamps, signatures, letterhead/branding, headers/footers** are layout noise relative to the clinically relevant content; PP-StructureV3's layout-analysis stage is the mechanism for separating these regions from data regions, rather than a bespoke rule set.

---

## 8. Privacy & Security Considerations

### Legal/regulatory context

- **India's DPDP Act 2023** does **not** create a separate "sensitive personal data" category the way GDPR/HIPAA do — it applies a uniform framework to all personal data with a risk-based approach (extra duties apply once an entity crosses a "Significant Data Fiduciary" volume threshold). Consent must be free, specific, informed, unconditional, and given via clear affirmative action; Section 7 exempts certain uses (data voluntarily provided, medical emergencies, public-health situations, legal obligations) from fresh-consent requirements. **Unverified/flagged:** specific DPDP Rules 2025 implementation-date details could not be confirmed in this research pass — treat any such date claims as unconfirmed until checked directly against a primary government source before relying on them for compliance decisions.
- **HIPAA (US)** is not directly binding but is the standard reference benchmark cited across the industry: encryption at rest/in transit, minimum-necessary access, audit logging, retention/deletion policy. All three major cloud OCR vendors offer HIPAA-eligible tiers under a signed BAA (AWS Textract confirmed; Azure Document Intelligence covered under Microsoft's HIPAA BAA; Google Document AI's BAA specifics were less clearly documented in this research pass and should be verified directly before any real use).

**Practical implication:** regardless of what's strictly legally mandated, this project should treat all uploaded medical documents as high-sensitivity by default — explicit consent before upload/processing, data minimization, and a defined deletion/retention policy.

### Local vs. cloud OCR

Consistent finding across sources: self-hosted OCR (PaddleOCR/Tesseract run locally) keeps documents from leaving the project's infrastructure entirely — no third-party trust requirement — at the cost of DevOps overhead and no automatic accuracy improvements from vendor updates. Cloud OCR is more convenient/accurate in some cases but requires trusting an external vendor with medical payloads, which is only acceptable under a proper BAA/DPA-equivalent agreement.

**Decision:** self-hosted OCR (PaddleOCR primary, Tesseract fallback) is the default and only path for the MVP. Cloud OCR is designed into the architecture as an optional, explicitly consent-gated provider behind the `OCRProvider` abstraction — not wired in or enabled by default.

### File upload security

- Validate uploads via **magic-byte/content-sniffing** (`%PDF-`, JPEG/PNG/WEBP signatures), not just file extension or client-supplied `Content-Type` — both are trivially spoofable.
- Enforce hard file-size limits before reading the full file into memory.
- Treat PDF parsing defensively — a parser exception means "reject," not "retry harder"; cap decompressed size/page count/render resolution to bound resource use (zip-bomb-style concerns apply to any compressed internal streams).
- Never use a user-supplied filename as a storage path component (path traversal risk) — generate a UUID-based storage name server-side.
- Store uploads outside the web root; serve back only via an authenticated endpoint.
- No medical document content or patient-identifying information in application logs.

---

## 9. Rejected Alternatives

Consolidated list of technologies explicitly considered and rejected, with the reason recorded so the decision isn't silently re-litigated later:

| Technology | Rejected for | Reason |
|---|---|---|
| Apache Tika | PDF/document extraction | JVM dependency for no accuracy gain; its OCR path just wraps Tesseract anyway |
| Surya OCR | Primary OCR engine | GPL-3.0 code + revenue-gated model license — real risk if the project becomes public/commercial |
| General-purpose VLMs (Claude/GPT-4o/Gemini) as OCR | OCR engine | Documented hallucination on unclear text — directly violates the "never silently fix uncertain OCR" requirement |
| Purpose-built OCR-VLMs (GOT-OCR2.0, olmOCR, Nougat, DeepSeek-OCR) | MVP OCR engine | Immature/GPU-heavy for a small team to productionize now; revisit later behind an explicit verification layer |
| Donut | Table/layout extraction | Generative decoder can paraphrase/hallucinate digits; no bounding boxes/confidence |
| LayoutLMv3 | Table/layout extraction | Needs a separate OCR pass anyway; designed for closed-schema field extraction, not faithful transcription |
| Table Transformer (TATR) | MVP table extraction | Trained on financial/scientific tables, not medical lab reports — unverified fit; kept as a recorded v2 evaluation candidate |
| Cloud OCR APIs as default | Default OCR path | Sends medical data to a third party by default; recurring per-page cost; kept as an optional, consent-gated provider instead |
| Default image binarization | Preprocessing default (for the deep-learning OCR path) | Deep-learning OCR models are trained on RGB/grayscale; binarization collapses information they rely on and risks deleting thin strokes (decimal points, punctuation) |
| Generative super-resolution (e.g. Real-ESRGAN) | Preprocessing | Documented to hallucinate fine detail on low-resolution/text inputs — trades fidelity for plausibility |
| Celery + Redis | MVP async processing | Right tool once multiple workers/retries/horizontal scaling are needed, but overkill for current MVP scale |
| Bare FastAPI `BackgroundTasks` (alone, for OCR itself) | Async job execution | In-process, fire-and-forget, no status tracking, lost on crash/restart — insufficient for multi-page CPU-bound OCR jobs that need status tracking; acceptable only for trivial sub-5-second actions like writing an initial "uploaded" row |

---

## 10. Risks

- **Numeric OCR errors** (digit confusion, lost decimal points/range delimiters) — the highest-priority risk class; mitigated by engine choice (PaddleOCR over VLMs), conditional/light-touch preprocessing, and confidence-score retention, but not eliminated. Must be caught by dedicated numeric-accuracy testing (§11, and master prompt §15/§21).
- **Table misalignment** (merged headers, multi-line cells, borderless tables) — a known failure mode of both the structure model and the bbox-clustering fallback; requires realistic test documents to quantify.
- **Poor input quality** (blur, extreme skew, low light) — mitigated by the quality gate, but some images will remain genuinely unreadable and must produce an explicit low-confidence/failure signal rather than a fabricated result.
- **Handwriting** — a known weak point across all evaluated open-source engines; must be surfaced as low-confidence, not silently transcribed.
- **Privacy/compliance risk** — mitigated by defaulting to self-hosted OCR and gating cloud OCR behind explicit consent, but the DPDP Rules 2025 implementation details are unverified and should be checked before the project scales or handles real patient data.
- **PyMuPDF AGPL licensing** — acceptable for an open-source self-hosted student project now; a real constraint if the project later needs closed-source/SaaS distribution without source disclosure.
- **Dependency footprint** — PaddleOCR pulls in the PaddlePaddle framework, a heavier install than Tesseract; relevant to deployment simplicity for a small team.
- **Processing time/scalability** — CPU-only inference for PP-StructureV3 will be slow on multi-page documents; DPI/quality tradeoffs and async processing are designed in specifically to absorb this, but real measurement is needed once implemented.
- **Vendor benchmark inflation** — independent sources put cloud OCR production accuracy meaningfully below vendor-advertised figures (80–95% vs. 95–99% claimed); any accuracy claim (ours or a vendor's) must be backed by our own measurement against ground truth, not taken on faith.

---

## 11. Testing Strategy

Per master prompt §20/§21, evaluation must be measured, not asserted. Planned approach for the implementation phase:

- **Text accuracy:** character/word error rate against ground-truth transcriptions for a representative test set (clean pathology PDF, scanned pathology PDF, multi-page report, photographed report, tilted photo, low-light photo, blurry image, table-heavy report, prescription, receipt, mixed PDF, poor/unreadable document — the twelve categories in master prompt §20).
- **Numerical accuracy:** a dedicated test pass, separate from general text accuracy, specifically exercising decimals, ranges, percentages, units, and dates (e.g. `10.2`, `10.02`, `0.5`, `12–16`, `<5`, `>10`, `98%`, `120/80`) with exact-match assertions — this is the test class most likely to catch the `10.2 → 102` failure mode called out in master prompt §15.
- **Medical terminology accuracy:** test names, medicine names, and abbreviations checked against source documents for unintended alteration (the OCR layer must not "correct" spelling).
- **Table accuracy:** row/column/cell-relationship correctness on table-heavy reports, checked separately from raw text accuracy since a table can have perfect character-level OCR and still be structurally wrong.
- **Layout accuracy:** headings, sections, tables, and columns correctly separated/ordered.
- **Performance:** processing time, memory, and CPU/GPU usage measured per document type, to validate the DPI and async-processing assumptions made in this research.
- **Preprocessing ablation:** for the conditional preprocessing pipeline (§6), test with each conditional step forced on/off against the same images to confirm each step actually improves accuracy for its trigger condition and doesn't regress clean inputs — this validates the research recommendations empirically before they're trusted in production.

A concrete test dataset (legally appropriate, no real patient data without authorization, per master prompt §20) is a Step 4/6 (Prototype/Test) deliverable, not part of this research document.

---

## 12. References

**OCR engines:**
- OCR comparison: Tesseract vs EasyOCR vs PaddleOCR vs MMOCR — https://toon-beerten.medium.com/ocr-comparison-tesseract-versus-easyocr-vs-paddleocr-vs-mmocr-a362d9c79e66
- PaddleOCR vs Tesseract vs EasyOCR: OCR Speed and Accuracy — https://www.codesota.com/ocr/paddleocr-vs-tesseract
- OmniDocBench GitHub (CVPR 2025) — https://github.com/opendatalab/OmniDocBench
- PP-OCRv5 Technical Report — https://arxiv.org/pdf/2603.24373
- PP-OCRv6 Technical Report — https://arxiv.org/pdf/2606.13108
- PaddleOCR 3.0 Technical Report — https://arxiv.org/pdf/2507.05595
- PP-StructureV3 Documentation — https://paddlepaddle.github.io/PaddleOCR/main/en/version3.x/algorithm/PP-StructureV3/PP-StructureV3.html
- docTR benchmarking discussion — https://github.com/mindee/doctr/discussions/1576
- Surya OCR GitHub — https://github.com/datalab-to/surya
- Best Open Source OCR Tools & Models 2026 — https://unstract.com/blog/best-opensource-ocr-tools/
- AWS Textract vs Google Document AI vs Azure Document Intelligence (2026) — https://invoicedataextraction.com/blog/aws-textract-vs-google-document-ai-vs-azure-document-intelligence
- Document AI/OCR API Price Comparison 2026 — https://soceton.com/blogs/document-ai-ocr-pricing-comparison
- Seeing is Believing? Mitigating OCR Hallucinations in MLLMs — https://arxiv.org/abs/2506.20168
- Guide to Improving OCR Accuracy — https://www.docsumo.com/blog/improving-ocr-accuracy
- Pharma Document AI & OCR Accuracy Benchmark — https://intuitionlabs.ai/articles/pharma-document-ai-ocr-benchmarks
- olmOCR: Unlocking Trillions of Tokens in PDFs with VLMs — https://arxiv.org/pdf/2502.18443

**PDF extraction:**
- Best Python PDF Libraries comparison — https://www.nutrient.io/blog/best-python-pdf-libraries/
- PyMuPDF vs pdfplumber — https://pdfmux.com/blog/pymupdf-vs-pdfplumber/
- pdfplumber vs PyMuPDF vs PyPDF2 comparison — https://subhajitbhar.com/blog/pdf-extraction/pdfplumber-vs-pymupdf-vs-pypdf2/
- PyMuPDF official feature docs — https://pymupdf.readthedocs.io/en/latest/about.html
- Apache Tika (Wikipedia) — https://en.wikipedia.org/wiki/Apache_Tika
- TikaOCR docs — https://cwiki.apache.org/confluence/display/tika/tikaocr
- Best document parsing tools 2026 — https://mixpeek.com/curated-lists/best-document-parsing-tools
- PyMuPDF GitHub discussion #1653 (scanned vs digital detection) — https://github.com/pymupdf/PyMuPDF/discussions/1653
- Programmatically determine if a PDF is scanned or digital (Oct 2025) — https://jamesmccaffreyblog.com/2025/10/03/programmatically-determine-if-a-pdf-document-is-scanned-or-digital-using-python/
- OCR vs text PDF guide — https://docs.bswen.com/blog/2026-03-16-ocr-vs-text-pdf-python/
- Scanned vs native PDFs — https://openpreservation.org/blogs/scanned-vs-native-pdfs-how-to-differentiate-them/
- PDF OCR guide (DPI tradeoffs) — https://ploomber.io/blog/pdf-ocr/
- pdf2ocr PyPI — https://pypi.org/project/pdf2ocr/1.1.2/
- OCRmyPDF cookbook — https://ocrmypdf.readthedocs.io/en/latest/cookbook.html
- OCRmyPDF advanced features docs — https://ocrmypdf.readthedocs.io/en/latest/advanced.html
- OCRmyPDF GitHub — https://github.com/ocrmypdf/OCRmyPDF

**Table extraction & layout analysis:**
- PP-StructureV3 docs — https://paddlepaddle.github.io/PaddleOCR/main/en/version3.x/algorithm/PP-StructureV3/PP-StructureV3.html
- PaddleX table_structure_recognition docs — https://paddlepaddle.github.io/PaddleX/3.3/en/module_usage/tutorials/ocr_modules/table_structure_recognition.html
- Extract Tables from PDF (Nanonets) — https://nanonets.com/blog/extract-tables-from-pdf/
- Tabula vs Camelot vs pdfplumber 2026 — https://dev.to/martin_pdfexcel/tabula-vs-camelot-vs-pdfplumber-in-2026-which-python-library-actually-wins-22kn
- Table Transformer GitHub — https://github.com/microsoft/table-transformer
- Table Transformer HF model card — https://huggingface.co/microsoft/table-transformer-structure-recognition-v1.1-all
- LayoutLMv3 explained — https://www.kungfu.ai/blog-post/engineering-explained-layoutlmv3-and-the-future-of-document-ai
- Donut paper — https://arxiv.org/pdf/2111.15664
- Multi-column table OCR (PyImageSearch) — https://pyimagesearch.com/2022/02/28/multi-column-table-ocr/
- ClusterTabNet paper — https://arxiv.org/html/2402.07502v2
- Invoice table pipeline paper — https://arxiv.org/pdf/2507.07029

**Image preprocessing:**
- OCR Preprocessing: How to Improve Extraction Accuracy — https://www.docuclipper.com/blog/ocr-preprocessing/
- PaddleOCR image channel discussion — https://github.com/PaddlePaddle/PaddleOCR/discussions/14510
- PaddleOCR preprocessing suggestion issue — https://github.com/PaddlePaddle/PaddleOCR/issues/12089
- Impact of image preprocessing on PaddleOCR performance — https://www.sciencedirect.com/science/article/pii/S1877050925027383
- PaddleOCR Document Image Preprocessing Pipeline (official) — https://paddlepaddle.github.io/PaddleOCR/main/en/version3.x/pipeline_usage/doc_preprocessor.html
- Constructing a Document Scanner and OCR with OpenCV — https://reintech.io/blog/construct-document-scanner-ocr-opencv
- Text skew correction with OpenCV and Python — https://pyimagesearch.com/2017/02/20/text-skew-correction-opencv-python/
- Smart Document Scanning with Live OCR using OpenCV.js — https://opencv.org/smart-document-scanning-with-live-ocr-using-opencv-js/
- Blur image detection using Laplacian operator and OpenCV — https://www.researchgate.net/publication/315919131_Blur_image_detection_using_Laplacian_operator_and_Open-CV
- Edges Before Embeddings: A Confidence-Aware Blur Gate for VLM Pipelines — https://arxiv.org/html/2606.25838
- OCR Low Accuracy on Scanned Documents — https://imagetotable.ai/blog/ocr-low-accuracy-scanned-documents
- Adaptive-interpolative binarization with stroke preservation — https://www.sciencedirect.com/science/article/abs/pii/S1047320315001236
- Shadow detection/removal using ML and morphological operations — https://www.researchgate.net/publication/330219259_Shadow_detection_and_removal_from_images_using_machine_learning_and_morphological_operations
- Removing Shadows from Images of Documents (UCSB/ACCV16) — https://web.ece.ucsb.edu/~psen/Papers/ACCV16_RemovingShadows.pdf
- Real-ESRGAN overview — https://www.aimodels.fyi/models/replicate/real-esrgan-lucataco
- Hallucination Score: Mitigating Hallucinations in Generative Super-Resolution — https://arxiv.org/html/2507.14367v2

**Privacy & architecture:**
- DPDP Act 2023 Explained — https://www.matters.ai/compliance/dpdp/dpdp-act-2023
- Decoding the DPDP Act 2023 (EY) — https://www.ey.com/en_in/insights/cybersecurity/decoding-the-digital-personal-data-protection-act-2023
- Health Data and the DPDP Act — Practical Guide — https://amlegals.com/health-data-and-the-dpdp-act-a-practical-guide/
- DPDP Act 2023 (Wikipedia) — https://en.wikipedia.org/wiki/Digital_Personal_Data_Protection_Act,_2023
- AWS Textract HIPAA-eligible announcement — https://aws.amazon.com/blogs/machine-learning/amazon-textract-is-now-hipaa-eligible
- HIPAA-Compliant AI Document Processing Companies 2026 — https://data.folio3.com/blog/hipaa-compliant-ai-document-processing-companies/
- Mindee: On-premise vs Cloud OCR — https://www.mindee.com/blog/on-premise-ocr-vs-cloud-ocr
- ScanLens: On-Device vs Cloud OCR — https://scanlens.io/blog/on-device-vs-cloud-ocr
- Veryfi: Cloud vs On-Prem OCR — https://www.veryfi.com/technology/cloud-vs-on-premise-bank-check-ocr/
- Magic Bytes / Content Sniffing for secure uploads — https://habtesoft.medium.com/beyond-the-extension-securing-file-uploads-with-content-sniffing-and-magic-bytes-b0622bf0679d
- Secure API file uploads with magic numbers — https://transloadit.com/devtips/secure-api-file-uploads-with-magic-numbers/
- Understanding File Upload Vulnerabilities — https://aardwolfsecurity.com/understanding-file-upload-vulnerabilities/
- ARQ vs Celery for FastAPI background tasks — https://www.bithost.in/blog/tech-3/how-to-run-fastapi-background-tasks-arq-vs-celery-11
- FastAPI Background Tasks vs Celery — https://dev.to/uaslimcreate/fastapi-background-tasks-vs-celery-for-ai-feature-processing-when-to-queue-and-when-to-345l
- BackgroundTasks vs ARQ+Redis — https://davidmuraya.com/blog/fastapi-background-tasks-arq-vs-built-in/

---

## 13. Next Steps

This document completes **Step 2 (Research)** of the master prompt's implementation process. Per that process, the next steps — not yet started — are:

- **Step 3 — Architecture:** formalize the recommendations above into `OCR_ARCHITECTURE.md` (component responsibilities, data flow, API design, error handling, storage schema).
- **Step 4 — Prototype:** a small proof-of-concept validating the highest-uncertainty decisions from this research (PaddleOCR numeric accuracy on real-looking lab values; the conditional preprocessing triggers; the digital-vs-scanned page heuristic) against representative documents.
- **Step 5 onward — Implement/Test/Improve**, with every change recorded in `OCR_IMPLEMENTATION_LOG.md` as it happens.

No code has been written yet. See `README.md` for current project status.
