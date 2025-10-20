# Semantic Search for Google Sheets — Design Overview

## 1. Approach to Semantic Understanding

Traditional spreadsheet search relies on keyword matching or rigid schema definitions, which fail to capture **semantic intent** — for example, understanding that “profit” might be computed as “revenue minus cost,” or that “growth rate” refers to a derived metric rather than a literal column name.

Our approach redefines spreadsheet understanding as a **semantic retrieval and reasoning problem**. We model both **what the data is** and **how it is computed** through a dual-index hybrid architecture that integrates **vector-based retrieval** and **LLM-driven reasoning**.

### 1.1 Conceptual Overview

At a high level, we separate the problem into two complementary layers:

1. **Knowledge Representation Layer (Offline):**  
   Extracts and encodes spreadsheet knowledge into two pre-computed semantic indices:
   - **Spec RAG Index** — captures *structural knowledge*: columns, data types, sheet organization, and metadata.
   - **Measure RAG Index** — captures *computational knowledge*: formulas, derived metrics, and relationships between cells.

2. **Reasoning & Query Layer (Online):**  
   A **LLM Agent** interprets user intent, retrieves relevant semantic units from the indices, and synthesizes responses through reasoning and validation.

This hybrid retrieval-generation design enables the system to answer both descriptive queries (“What columns track revenue?”) and analytical queries (“How is gross margin calculated?”) — something neither classical search nor pure LLM approaches can achieve efficiently.

### 1.2 Semantic Representation of Spreadsheet Knowledge

Each spreadsheet is pre-processed into **semantic records** that preserve its meaning in a machine-understandable form. Instead of indexing every cell, we summarize *meaningful units of knowledge* such as columns, formulas, and labeled aggregates.

#### Spec RAG Index — Structural Understanding
- **Goal:** Represent how data is organized and labeled.
- **Features:**  Flexible Header Identification, Precise Addressing, Type and Context Mapping  
- **Structure:**
```  
    Sheet {sheet_name}
    Column {column_name}
    Address {A, B, ...}
    Description {col_description if available}
    Has values of type {numeric/text/date/etc.}
    Top sample values {v1, v2, v3, v4, v5}
```  
- **Scale:** ~20–50 entries per spreadsheet.  
- **Example queries:**  
  - “List all sheets with sales data.”  
  - “Show columns related to revenue.”

#### Measure RAG Index — Computational Understanding
- **Goal:** Represent how values are computed and how formulas connect data.
- **Features:** Cross sheet relations, generic relation finder, have proper cell address  
- **Structure:**  
  ```json
  {"id": "financial_ratios_net_profit_growth", "name": "Net Profit Growth", "sheet": "Financial Ratios", "cell_address": "C9", "row": 8, "column": 3, "formula": "=('3-Year Forecast'!C10-'3-Year Forecast'!B10)/'3-Year Forecast'!B10", "formula_semantic": "=('3-Year Forecast'!net_profit_after_tax_year_2-'3-Year Forecast'!net_profit_after_tax_year_1)/'3-Year Forecast'!net_profit_after_tax_year_1"}

  {"id": "cost_analysis_annual_cost_column_rule", "name": "Annual Cost (Column Rule)", "sheet": "Cost Analysis", "type": "column_rule", "column": 3, 
  "formula_semantic": "cost_analysis_yearly = cost_analysis * 12", "column_name": "Annual Cost", "formula_pattern": "={B}*12",}

    # Ragged on formula_semantic
  ````
- **Scale:** ~200–500 entries per spreadsheet.  
- **Example queries:**  
  - “Find where growth rate is calculated.”  
  - “Show me formulas that use total cost.”

### 1.3 Semantic Formula Translation

A key innovation is **semantic normalization of formulas**. Each formula is translated from its raw form (e.g., `=B2+B3`) to a human-readable, semantically labeled version (e.g., `=revenue_q1 + revenue_q2`). This process allows the embedding model to understand the business meaning of a formula instead of its positional syntax.

These translations make it possible to perform **natural language search over computations**, bridging the gap between symbolic spreadsheet logic and human reasoning.

### 1.4 Why This Matters

By combining structured embeddings with reasoning-oriented LLM orchestration, our system achieves:

- **Human-like understanding** of spreadsheet logic and relationships.  
- **Scalable performance** — embeddings are computed once, reused across queries.  
- **Explainable results**, since every answer traces back to semantic units.  
- **Domain adaptability**, since language models interpret terminology dynamically.

This approach transforms spreadsheets from static data containers into *semantic knowledge graphs* that can be queried conversationally.

---

## 2. Handling Business Domain Knowledge

We handle business-specific semantics through **contextual embeddings** and **LLM-led clarification** rather than fixed schemas.

- During pre-computation, the model embeds representative samples (column names, values, metadata), implicitly capturing domain semantics such as “ARR,” “MRR,” or “customer churn.”  
- During query time, the LLM agent interprets the domain vocabulary and aligns it with indexed entities.  
- If ambiguity remains, the agent asks for clarification (e.g., “Do you mean customer churn or revenue churn?”).

This design avoids manual ontology creation and automatically generalizes across domains — from finance to HR to sales analytics.

---

## 3. Query Processing and Result Ranking

### Step 1 — Query Analysis
The **LLM Agent** analyzes the user's query to understand intent and identify any ambiguity. If the query is unclear or could have multiple interpretations, the agent will ask clarifying questions to gather additional information before proceeding with the search.

### Step 2 — Tool Selection
Based on intent, the agent routes the query through specialized tools:

| Tool | Purpose | Backend |
|------|----------|----------|
| `search_spec()` | Retrieve structural / metadata insights | Spec RAG Index |
| `search_measures()` | Retrieve formulas and derived metrics | Measure RAG Index |
| `get_cell_value()` / `get_row_values()` | Verify specific cells via live API | Google Sheets API |

### Step 3 — Iterative Retrieval
The agent executes retrieval and validation up to three times:
1. Retrieve semantically similar units via FAISS (cosine similarity).  
2. Validate and refine keywords or embeddings.  
3. Optionally verify live cell values for factual accuracy.

### Step 4 — Ranking and Explanation
- **Ranking:** Weighted cosine similarity + contextual fit.  
- **Explanation:** LLM generates a natural-language rationale for each result.  
- **Output:** Top-k results with provenance (sheet name, column, formula, etc.).

---

## 4. Technical Architecture and Data Structures

### Architecture Diagram

        ┌────────────────────┐
        │ Google Sheets Data │
        └─────────┬──────────┘
                  │
    ┌─────────────▼────────────────┐
    │  Pre-Computation Pipeline    │
    │ (Spec + Measure Extraction)  │
    └─────────────┬────────────────┘
                  │
    ┌─────────────▼────────────────┐
    │   FAISS Vector Indices       │
    │ (Spec RAG + Measure RAG)     │
    └─────────────┬────────────────┘
                  │
    ┌─────────────▼────────────────┐
    │ LLM Agent (Gemini Pro)       │
    │ - Query Ambiguity Removal    │
    │ - Tool Orchestration         │
    │ - Iterative Refinement       │
    │ - Validation & Ranking       │
    └─────────────┬────────────────┘
                  │
    ┌─────────────▼────────────────┐
    │ Desired Output               │
    └──────────────────────────────┘
### Data Structures
| Type | Description | Example |
|------|--------------|----------|
| `column_spec` | Metadata for each column | `{name: "Revenue", dtype: "float", samples: [100, 120, 150]}` |
| `formula_measure` | Semantic formula translation | `profit_margin = (revenue - cost) / revenue` |
| `labeled_value` | Data cell with semantic context | `{row: "Q1", column: "Revenue", value: 100}` |

---

## 5. Performance Considerations and Trade-offs

| Metric | Value | Notes |
|--------|--------|-------|
| **Indexing cost** | <$0.002 per spreadsheet | One-time pre-computation |
| **Query cost** | ~$0.0003 | Includes LLM orchestration |
| **Latency** | 350–650 ms | With up to 3 refinement loops |
| **Scalability** | 10K+ cells, 50+ sheets | Independent of token limits |
| **Memory** | 50–300 MB/sheet | FAISS + embeddings |

### Optimizations
- **Pre-computation** eliminates token bottlenecks during runtime.  
- **Dual indexing** balances granularity and interpretability.  
- **Lazy API access** fetches real values only when necessary.

### Trade-offs
- Requires an initial indexing step per spreadsheet.  
- Slightly higher orchestration complexity.  
- No full-data storage for privacy (only summaries).

---

## 6. Challenges and Solutions

| Challenge | Description | Solution |
|------------|--------------|-----------|
| **Formula semantics loss** | Raw formulas lack context (`=B2+B3`). | Added semantic translation (`=revenue_q1 + revenue_q2`). |
| **Domain ambiguity** | Queries use business-specific language. | LLM clarification + iterative refinement. |
| **Cost efficiency** | LLM queries can be expensive. | Minimized via pre-computed embeddings + lightweight calls. |
| **Cross-sheet reasoning** | Formulas reference other sheets. | Track lineage in Measure index. |
| **Scalability** | LLMs have context limits. | Pre-computation bypasses token constraints entirely. |

---

## Summary

This design enables **semantic, scalable, and explainable natural language access** to spreadsheet data.  
By pre-indexing structure and computation separately and using the LLM purely as an **orchestrator**, not a data store, the system achieves:

- Deep understanding of both structure and formulas.  
- Real-time, low-cost query execution.  
- Domain adaptability through contextual learning.  
- Transparent, explainable results with provenance tracking.

In short, the system turns spreadsheets into **searchable, intelligent knowledge systems** that understand both *what the data says* and *what it means*.
