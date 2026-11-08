# Model Inspector & Layer Extractor (Safetensors & GGUF)

A browser-native binary header inspector and layer extractor for **Safetensors** and **GGUF (v1, v2, v3)** AI model files. It reads only the initial metadata header bytes via HTTP Range requests (for Hugging Face URLs) or `File.slice()` (for multi-gigabyte local files) to extract layer names, shapes, precision dtypes, parameter counts, quantization helper keys, and architectural block groups into structured JSON without downloading the full model weights.

### User Review & Critical Decisions

> [!IMPORTANT]
> The following product and technical decisions incorporate your Phase 1 preferences alongside the mandatory requirements from your prompt and attached reference implementation:

- **Confirmed Decision 1 — Bilingual Interface (ID / EN Toggle)**: Added an interactive **ID / EN** language toggle button in the top header alongside the **Dark Mode / Light Mode** toggle so every label, filter control, table header, tooltip, and error/status message seamlessly switches between Bahasa Indonesia and English.
- **Confirmed Decision 2 — Exact Reference UI Structure**: Preserved the exact layout, workflow, and controls from the reference `App.tsx.txt` without adding extra preset clutter.
- **Confirmed Decision 3 — Optional CORS Proxy Fallback**: Included an optional CORS fallback mechanism for remote Hugging Face HTTP `Range` requests so restricted redirects or cross-origin endpoints work reliably when deployed on static hosts like GitHub Pages (`github.io`).
- **Confirmed Decision 4 — 100% Binary Parser Fidelity**: Preserved `HybridBrowserReader`, `parseGgufStream`, `parseSafetensorsStream`, `analyzeArchitectureGroups`, `GGML_TYPES` (IDs 0–40 including `BF16`, `TQ1_0`, `MXFP4`, `NVFP4`), `GGUF_VAL_SIZES`, and `QUANT_HELPER_SUFFIXES` intact.

---

### 1. Overview & Core Concept

- **What It Does**: Inspects `.safetensors` and `.gguf` neural network weight files either directly from a Hugging Face link (`/blob/`, `/resolve/`, or `user/repo/file` shorthand) or from local disk via Drag & Drop / File Picker, parsing binary headers in milliseconds while consuming only a few kilobytes of bandwidth.
- **Target Audience / Persona**: ML engineers, quantization practitioners, ComfyUI/llama.cpp developers, and model researchers who need to inspect tensor shapes, mixed-precision distributions, quantization helper keys, and architecture blocks before downloading 10–100+ GB checkpoints.
- **Key Value**: Zero-upload local inspection and ultra-low-bandwidth remote inspection with instant JSON export (`Lengkap / Full`, `Ringkas / Compact`, `Minimal`) and regex-capable layer filtering.

---

### 2. User Experience & Visual Design

- **Key User Flows**:
  1. **Header Controls**: Switch between **Bahasa Indonesia (`ID`)** and **English (`EN`)**, and toggle between **Dark Mode** and **Light Mode** at any time.
  2. **Source Selection**:
     - *Hugging Face Link Tab*: Enter a Hugging Face model URL or `user/repo/file` path, optionally provide an `HF_TOKEN` for private/gated repositories, and optionally enable the CORS proxy fallback if a direct browser `Range` request is blocked by cross-origin redirect policies.
     - *Local File Tab*: Drag & drop or pick a `.safetensors` or `.gguf` file from disk (`0 Quota` instant `File.slice()` reading).
  3. **JSON Formatting & Parser Options**: Choose JSON structure mode (`Lengkap / Full`, `Ringkas / Compact`, `Minimal`), toggle separating quantization helper keys (`.weight_scale`, `.comfy_quant`, etc.), and toggle summarizing large GGUF metadata arrays with a configurable max item limit.
  4. **Interactive Inspection & Export**: View top-level metrics (Total Parameters in Billions, Main Layers, Quantization Tensors, Original File Size vs. KB Read), copy or download the generated `.json` file, and explore the **5 Interactive View Tabs**:
     - *Ringkasan & Blok Arsitektur / Summary & Architecture Blocks*: Precision/dtype distribution progress bars and auto-detected indexed (`blocks.[0..27]`) vs. non-indexed architecture block table.
     - *Daftar Layer Utama / Main Layers List*: Search by substring or `.* Regex`, filter by Precision/Dtype, filter by Block Group, sort by File Order, Name (A–Z), Largest Parameters, or Smallest Parameters, and navigate with configurable pagination (`25`, `50`, `100`, `250` rows per page).
     - *Tensor Kuantisasi / Quantization Tensors*: Dedicated table of separated quantization helper tensors.
     - *Metadata*: Searchable key-value viewer with automatic JSON pretty-printing for nested Safetensors/GGUF metadata objects.
     - *Preview Raw JSON*: Live preview of the first 200 lines of formatted JSON output matching the selected structure mode.
- **Visual Identity & Theme**:
  - *Aesthetic Direction*: High-craft developer console with dual theme support (Dark Slate `#020617` / `#0F172A` and Crisp Light `#F8FAFC` / `#FFFFFF`).
  - *Color Palette & Mood*: Deep slate surfaces paired with Indigo (`indigo-600` / `indigo-400`) primary interactive accents, Emerald (`emerald-500` / `emerald-400`) for format badges, shapes, and JSON download actions, and Amber (`amber-400`) for quantization scale indicators.
  - *Typography & Hierarchy*: Clean sans-serif UI headings paired with tabular monospace (`JetBrains Mono` / `font-mono tabular-nums`) for tensor names, shapes, byte counts, and parameter totals.
  - *Component Styling & Layout*: Responsive 12-column / 3-column desktop grid layout (`max-w-7xl`), crisp 1px borders, and high-density scannable data tables.

---

### 3. Key Product Decisions & Trade-Offs

- **Decision 1: Client-Side Stream Slicing (`HybridBrowserReader`)**
  - *Chosen Approach*: Maintain a sliding `Uint8Array` buffer populated on-demand via 2 MB chunked HTTP `Range: bytes=start-end` requests for remote URLs or `File.slice(start, end)` for local files, with `skip()` advancing the remote offset without downloading skipped bytes.
  - *Why*: Allows skipping multi-megabyte GGUF tokenizer arrays and stopping immediately before raw tensor weight data, keeping bandwidth usage to a few hundred KB even on 20+ GB models.
- **Decision 2: Bilingual Dictionary State (`ID` / `EN`)**
  - *Chosen Approach*: Provide a lightweight reactive translation dictionary covering all UI strings, options, table headers, and error messages controlled by a header toggle (`ID` | `EN`).
  - *Why*: Fulfils the requirement that all features work seamlessly in both Bahasa Indonesia and English while preserving 100% of the reference layout and logic.
- **Decision 3: Optional CORS Fallback & Static GitHub Pages Build (`base: './'`)**
  - *Chosen Approach*: Configure `base: './'` in Vite so all asset paths are relative for `github.io` deployment, and allow `HybridBrowserReader` to retry or route through a CORS-friendly fetch when direct Hugging Face CDN redirects strip custom headers or block browser `Range` requests.
  - *Why*: Ensures 100% serverless static hosting compatibility on GitHub Pages while handling cross-origin quirks gracefully.

---

### 4. Technical Architecture & Data Strategy *(Technical Reference)*

- **Architecture & Component Diagram**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                Header Navbar (Brand + ID/EN Toggle + Theme)             │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
┌─────────────────────────────────────┐ ┌─────────────────────────────────┐
│        1. Model Source Panel        │ │   2. JSON & Parser Settings     │
│  • Hugging Face URL + HF_TOKEN      │ │  • Mode: Lengkap/Ringkas/Minimal│
│  • Optional CORS Fallback Toggle    │ │  • Separate Quant Helper Keys   │
│  • Local File Drag & Drop (.gguf /  │ │  • Summarize Large GGUF Arrays  │
│    .safetensors via File.slice)     │ │  • Max Array Items Limit        │
└──────────────────┬──────────────────┘ └────────────────┬────────────────┘
                   │                                     │
                   └──────────────────┬──────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         HybridBrowserReader                             │
│  • Remote: HTTP Range (bytes=offset-end) + 2MB Sliding Uint8Array       │
│  • Local:  File.slice(offset, end) -> ArrayBuffer                       │
│  • Binary Primitives: readUint32/64, readInt32/64, readString, skip()   │
└──────────────────┬─────────────────────────────────────┬────────────────┘
                   ▼                                     ▼
┌─────────────────────────────────────┐ ┌─────────────────────────────────┐
│          parseGgufStream()          │ │     parseSafetensorsStream()    │
│  • GGUF v1/v2/v3 Header & KV Parser │ │  • 64-bit Header Size + JSON    │
│  • GGML_TYPES (0..40: F32..NVFP4)   │ │  • Auto-parse __metadata__ JSON │
│  • 64-bit Array Summarizer & Skip   │ │  • QUANT_HELPER_SUFFIXES Split  │
└──────────────────┬──────────────────┘ └────────────────┬────────────────┘
                   └──────────────────┬──────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│             analyzeArchitectureGroups() & Summary Aggregator            │
│  • Detects Indexed Blocks (e.g. double_blocks.[0..18]) & Non-Indexed    │
│  • Computes Precision Distribution & Total Parameter Counts             │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     5 Interactive Inspector Views                       │
│  [1. Summary & Arch Blocks] [2. Main Layers + Regex/Dtype/Group Filter] │
│  [3. Quantization Tensors ] [4. Searchable Metadata] [5. Raw JSON View] │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Data Model & State**:
  - `TensorItem`: `{ name, shape, precision, parameters, blockGroup }`
  - `BlockGroupInfo`: `{ total_blocks?, total_tensors, parameters, parameters_M }`
  - `SummaryInfo`: `{ file_name, source_type, format, file_size_gb, file_size_mb, total_layers, quantization_helper_tensors, total_parameters, total_parameters_billion, downloaded_kb, precision_distribution, architecture_block_summary }`
- **Interactive Component & State Mapping**:
  - **Source & Extraction Handler (`handleExtract`)**: Normalizes Hugging Face URLs (`/blob/` -> `/resolve/`, `hf://` or `user/repo/file`), probes initial 16 bytes to detect `GGUF` magic bytes vs. Safetensors, initializes `HybridBrowserReader`, runs the appropriate binary stream parser, and populates `summary`, `mainTensors`, `quantTensors`, and `metadata`.
  - **Advanced Layer Filter (`filteredLayers` & `paginatedLayers`)**: Filters `mainTensors` in real time by `selectedPrecision`, `selectedBlockGroup`, and `searchQuery` (plain text or `RegExp`), sorts by file order, alphabetical name, or parameter count, and slices rows according to `currentPage` and `rowsPerPage`.
  - **JSON Generator & Export (`generatedJsonPayload`, `handleDownloadJson`, `handleCopyJson`)**: Dynamically constructs the output JSON according to `jsonMode` (`lengkap`, `ringkas`, `minimal`) and provides one-click `.json` file download and clipboard copy with visual confirmation.
