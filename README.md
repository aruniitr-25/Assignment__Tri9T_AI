# Structural PDF AST Parser & Version Diffing Pipeline

An end-to-end Python pipeline and verification suite built to parse text-layer PDF technical manuals (`ct200_manual.pdf` v1 and `ct200_manual_v2.pdf` v2) directly into an Abstract Syntax Tree (AST)[cite: 1, 15]. The system extracts document structure, handles 11 complex visual/layout irregularities, and evaluates per-node content staleness and material changes across document revisions[cite: 1, 15].

---

## 1. Features & Capabilities

* **Font & Geometry-Based Heading Extraction:** Detects document hierarchy using line bounding boxes ($x_0$), bold font flags, and structural numbering patterns (`^\d+(\.\d+)*\.?\s`), rather than naive font size thresholds.
* **Layout Normalization & Deterministic Hashes:** Applies Unicode NFKC normalization, maps soft/non-breaking hyphens (`U+2011`, `U+2010`) to standard hyphens `-`, joins line-break hyphenations, and collapses whitespace. Content hashes remain strictly invariant when text shifts across page boundaries[cite: 1, 13].
* **Table Column & Cell Reconstruction:** Groups multi-line wrapped table cells (e.g., Table 4.2 row `E5`) into unified structured dictionaries using column $X$-coordinate alignments ($x_0 \approx 39.5, 79.0, 215.7$)[cite: 1, 12].
* **Token-Level Parameter Staleness Classifier:** Scans numeric quantities, error identifiers (`E1`–`E6`), and engineering units (`mmHg`, `bpm`, `cm`, `cycles`, `%`, `seconds`) to accurately distinguish `material` technical edits from minor rewording (`stale_minor`) and unchanged nodes (`fresh`)[cite: 1].
* **Zero-Network Unit Test Suite:** Executable `unittest` suite covering all 11 document edge cases and diff scenarios offline via `MockLLMClient`.

---

## 2. Handled Layout Edge Cases

| # | Edge Case / Irregularity | Resolution Mechanism in `parser.py` |
| :--- | :--- | :--- |
| **1** | **Skipped Hierarchy Level** (`2.1.1.1` follows `2.1` without `2.1.1`)[cite: 1, 10] | Assigned parent `node_2.1` with `level = parent.level + 1` (Level 3); recorded diagnostic warning without inventing fake nodes[cite: 1]. |
| **2** | **Out-of-Order Siblings** (`3.4` appears before `3.3`)[cite: 1, 11] | Preserved physical document stream reading order (`3.1` $\to$ `3.2` $\to$ `3.4` $\to$ `3.3`); logged `"number_label out of sequence"` warning on `node_3.3`[cite: 1]. |
| **3** | **Duplicate Heading Titles** (`Error Codes` at `4.2` & `7.1`)[cite: 1, 12, 14] | Assigned disambiguated logical IDs (`node_4.2` vs `node_7.1`) scoped to their respective parent nodes (`node_4` and `node_7`)[cite: 1, 12, 14]. |
| **4** | **Table-Only Sections & Wrapped Cells** (`2.1` & `4.2`)[cite: 1, 10, 12] | Tracked leading column cells ($x_0 < 50\text{pt}$) to group wrapped multi-line lines into row dicts; stored in `tables` list with empty `body_text`[cite: 1]. |
| **5** | **Indented Numbered Lists** (Section `3.3` list item 5 wrap)[cite: 1, 11] | Detected indentation ($x_0 \approx 50.8\text{pt}$) and non-bold weight to separate list items from headings; joined wrapped continuation lines into item 5[cite: 1]. |
| **6** | **Stranded Page-Bottom Headings** (Section `2` on Page 1) | Child sections attached strictly by document reading order sequence, never by page boundaries[cite: 1]. |
| **7** | **Cross-Page Paragraph Splitting** (Section `3.1` p2 $\to$ p3) | Joined split text fragments across page breaks into single cohesive string blocks[cite: 1]. |
| **8** | **Cross-References in Body Prose** (`see 2.1, 4.3`)[cite: 1, 11] | Scanned and extracted reference targets into a node `cross_refs` list without creating graph edges[cite: 1]. |
| **9** | **Normalization & Page-Break Hash Stability**[cite: 1] | NFKC normalization and hyphen mapping ensure sections shifting across pages (e.g., `5.1`) retain identical SHA-256 hashes[cite: 1, 13]. |
| **10** | **3-Line Wrapped Title Block**[cite: 1, 9] | Concatenated 22pt bold lines into a single `root` AST node[cite: 1, 9]. |
| **11** | **Empty Container Sections** (Sections 1–8)[cite: 1] | Stored container nodes as valid AST entries with `body_text = ""`[cite: 1]. |

---

## 3. Project Structure

```text
.
├── data/
│   ├── ct200_manual.pdf          # Version 1 Manual
│   └── ct200_manual_v2.pdf       # Version 2 Manual
├── parser.py                     # AST Node Definition & PDF Parser Pipeline
├── matcher.py                    # AST Matcher & Token-Level Staleness Detector
├── test_manual_parser.py         # Automated Unittest Suite (11 Edge Cases)
└── README.md                     # Documentation
