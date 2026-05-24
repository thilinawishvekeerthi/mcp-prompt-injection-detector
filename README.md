# Lightweight Explainable ML for MCP Tool Poisoning & Prompt Injection Detection

**Author:** Thilina Wishvakeerthi | 20232930 / W2084838
**Programme:** MSc Advanced Software Engineering — IIT / University of Westminster
**Supervisor:** Mr. Achala Aponso
**Submission:** May 2026

---

## What This Project Does

The Model Context Protocol (MCP) lets AI agents connect to external tools by reading tool descriptions. Attackers can poison those descriptions — embedding hidden instructions that make the agent exfiltrate data, execute system commands, or bypass safety rules — all before any user interaction occurs.

This project builds a **lightweight, explainable, pre-invocation detector** that screens every MCP tool description before the agent calls it. If the description looks malicious, it is blocked. If it is safe, it is allowed through. Every decision comes with a SHAP explanation showing exactly why.

---

## Key Results

| Metric | Value |
|---|---|
| ROC-AUC | 0.9142 |
| F1-Score | 0.7848 |
| Recall | 0.8009 |
| Precision | 0.7693 |
| False Positive Rate | 0.1556 |
| P50 Latency (GPU) | 22.9 ms |
| P95 Latency (GPU) | 41.3 ms |
| Classifier Size | 1.4 MB |
| Detection Threshold | 0.390 (F1-optimal) |
| Training Corpus | 403,215 samples · 7 sources |

---

## Files in This Submission

```
CODE/
│
├── mcp_prompt_injection_prevention.ipynb   ← Main notebook (141 cells)
├── mcp_server_only.ipynb                   ← FastAPI deployment server (standalone)
├── mcp-security-gateway.html               ← Browser-based tool registration portal
│
└── McpPromptInjectionEvaluation/
    └── augmentation_data/
        ├── synthetic_templates.json        ← Source D: 6,000 synthetic MCP tool descriptions
        ├── hard_negative_attacks.json      ← Source E: 80 targeted hard-negative attacks
        └── benign_fp_fixes.json            ← Source F: 80 benign false-positive corrections
```

---

## File Descriptions

### `mcp_prompt_injection_prevention.ipynb`

The main research notebook. Contains the complete pipeline from raw data to final evaluation.

**Sections:**

| Section | What it does |
|---|---|
| 0 — Setup | Install dependencies, configure paths, set `FORCE_*` flags |
| 1 — Raw data loading | Download BIPIA, HackAPrompt, ahsanayub datasets from Hugging Face |
| 2 — Evidence analysis | Justify design decisions with data (truncation at 4,000 chars, label fixes) |
| 3 — Dataset pipeline | Apply 7 label quality fixes, deduplicate, merge all sources |
| 4 — Augmentation | Load synthetic MCP templates, hard negatives, benign FP fixes |
| 5 — Train/val/test split | 70/15/15 stratified split — 282,250 / 60,482 / 60,483 samples |
| 6 — Embeddings | Encode with all-mpnet-base-v2 (768-d), cache to Google Drive |
| 7 — FeatureEngineer v2 | Extract 16 hand-crafted structural features from raw text |
| 8 — Model training | XGBoost on 784-d vectors (768 emb + 16 eng), GPU T4, 23.7s |
| 9 — Calibration | Platt scaling (sigmoid), F1-optimal threshold sweep → t = 0.390 |
| 10 — Cross-model comparison | XGB+MPNet vs XGB+MiniLM vs TF-IDF+LR vs DistilBERT |
| 11 — SHAP explainability | TreeSHAP beeswarm, scatter, contrastive decision plots |
| 12 — Obfuscation robustness | 7 techniques × 23,780 samples — base64, homoglyph, leetspeak, etc. |
| 13 — Domain transfer | 4 MCP domains: email, API, code execution, database ops |
| 14 — Failure mode analysis | TP / TN / FP / FN breakdown, McNemar's test, shared blind spots |
| 15 — Final results table | Full 4-model comparison at F1-optimal thresholds |

**To run from cache (recommended):**
Set all three flags to `False` in the master config cell and run sequentially:
```python
FORCE_RERUN            = False
FORCE_REBUILD_DATASET  = False
FORCE_REBUILD_SPLIT    = False
```

**To re-run from scratch:**
Set the relevant flag to `True`. The notebook will re-download data, rebuild splits, or retrain as needed.

---

### `mcp_server_only.ipynb`

A standalone deployment notebook. Starts the FastAPI detection server on Google Colab and exposes it via an ngrok tunnel. Does not contain training code — loads the pre-trained model from cache.

**Endpoints:**

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Returns model status, threshold, and model name |
| `/detect` | POST | Fast single-tool classification (BLOCK / ALLOW + probability) |
| `/detect/explain` | POST | Classification + SHAP top-5 feature attributions |
| `/detect/batch` | POST | Batch classification for multiple tools at once |
| `/docs` | GET | Auto-generated Swagger UI |

**Request format:**
```json
{
  "prompt": "weather_api: Retrieves current weather using lat/lng coordinates...",
  "tool_name": "weather_api"
}
```

**Response format:**
```json
{
  "tool_name": "weather_api",
  "action": "ALLOW",
  "probability": 0.031,
  "explanation": [
    { "feature": "prompt_length", "shap_value": -0.412, "direction": "BENIGN" },
    { "feature": "attack_pattern_count", "shap_value": -0.108, "direction": "BENIGN" }
  ]
}
```

---

### `mcp-security-gateway.html`

A self-contained browser UI for the MCP tool registration workflow. Open directly in any browser — no server required for the UI itself (the FastAPI backend must be running separately).

**Features:**
- Paste any MCP tool JSON payload and analyze all tools at once
- Connects to your live FastAPI backend via ngrok URL
- Displays real SHAP attributions, confidence scores, and attack type classification
- Two built-in presets: working cases (5 tools) and known failure modes (5 tools)
- Human-in-the-loop review: Accept / Reject each tool before registration
- Exports a final registration payload with audit log

**How to use:**
1. Start the FastAPI server from `mcp_server_only.ipynb`
2. Copy the ngrok URL into the "Detector Backend" field
3. Paste your MCP tool JSON or load a preset
4. Click **Analyze Tools**

---

### `McpPromptInjectionEvaluation/augmentation_data/`

Custom data files created for this project. These are the novel data contributions used to make the model MCP-aware.

#### `synthetic_templates.json` — Source D (6,000 samples)

Template pools for generating synthetic **benign** MCP tool descriptions across 6 deployment domains:

| Domain | Description |
|---|---|
| `file_ops` | File reading, writing, searching, summarising |
| `database` | SQL queries, record lookup, schema inspection |
| `email_tasks` | Send, read, draft, search emails |
| `api_integration` | REST calls, authentication, webhook handling |
| `code_execution` | Run scripts, lint code, test execution |
| `web_browsing` | Fetch URLs, extract content, navigate pages |

Each template uses `{slot}` placeholders filled from domain-specific slot lists, generating realistic tool descriptions that resemble real MCP documentation.

#### `hard_negative_attacks.json` — Source E (80 samples)

80 targeted attack samples designed to cover the five systematic failure modes identified during development:

| Failure mode | Description |
|---|---|
| `unix_command_injection` | Short imperative sentences with shell commands |
| `tool_chain_attack` | Instructions to automatically call secondary tools |
| `low_surface_injection` | Minimal attack surface — short, lowercase, no special chars |
| `base64_partial` | Partially encoded payloads (primary blind spot) |
| `multilingual_injection` | Non-English attack phrases |

#### `benign_fp_fixes.json` — Source F (80 samples)

80 benign prompts added specifically to reduce false positives found during smoke testing. Each entry includes the original false-positive probability before the fix was applied.

---

## System Architecture

```
MCP Tool Description (raw text)
         │
         ├──► all-mpnet-base-v2 (SentenceTransformer)
         │         └── 768-d semantic embedding
         │
         ├──► FeatureEngineer v2
         │         └── 16 engineered features:
         │               prompt_length, word_count, uppercase_ratio,
         │               special_char_ratio, jailbreak_count,
         │               instruction_count, override_count,
         │               roleplay_count, system_count,
         │               attack_pattern_count, homoglyph_ratio,
         │               base64_content_ratio, injection_signal_density,
         │               unix_cmd_count, tool_chain_count,
         │               max_sentence_injection
         │
         ├──► StandardScaler (engineered features only)
         │
         ├──► Concatenate → 784-d feature vector (768 + 16)
         │
         ├──► XGBoost classifier
         │         n_estimators=300, max_depth=6, lr=0.05
         │         scale_pos_weight=1.54, GPU (tree_method='hist')
         │
         ├──► CalibratedClassifierCV (sigmoid / Platt scaling)
         │
         ├──► Threshold t = 0.390 (F1-optimal on validation set)
         │
         └──► BLOCK / ALLOW + TreeSHAP explanation (< 5ms overhead)
```

---

## Cache Files (Google Drive)

Model weights, embeddings, and evaluation results are stored on Google Drive due to file size.

> **[Access cache files on Google Drive](https://drive.google.com/drive/folders/15Ap8BslIrpjE7tSzyV1hotsYO693ZRyi?usp=sharing)**

Set `PROJECT_DIR` in the master config cell to point to your Drive folder before running.

---

## Dependencies

```
python >= 3.9
torch
transformers
sentence-transformers
xgboost
scikit-learn
shap
fastapi
uvicorn
pyngrok
datasets          # Hugging Face
pandas
numpy
matplotlib
```

Install in Colab:
```python
# Cell 1 of the notebook installs all dependencies automatically
```

---

## Reproducibility

All results are fully reproducible from the cached artefacts on Google Drive.
Random seed: `42` throughout (except BIPIA cell which uses seed `2023` per the original paper).
No randomness is introduced after the train/val/test split is saved to `dataset_split.pkl`.
