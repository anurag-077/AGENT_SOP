# Universal Real Estate Intelligence Agent — SOP & Implementation Guide

> **For the coding agent:** Read this document fully before writing a single line of code.
> Every section maps directly to a file or module you must create. Follow the order. Do not skip steps.

---

## Overview

This system replaces the current monolithic `RealEstateAgent` class with a **7-stage pipeline** where every stage is an independent, LLM-driven module. There is **zero hardcoded business logic**. The system must be able to answer any question answerable from the internal database — rate trends, inventory, comparisons, geo-proximity, project analysis, buyer patterns — without any human writing `if query_type == "trend"` anywhere.

### Directory Structure to Create

```
<img src = 'https://github.com/user-attachments/assets/124991a5-aa40-45f9-8e25-cd8dc2ed76b7'/>


backend/
├── agent/
│   ├── __init__.py
│   ├── pipeline.py              # Main orchestrator — runs all 7 stages
│   ├── stages/
│   │   ├── __init__.py
│   │   ├── s1_intent.py         # Stage 1: Intent + entity extraction
│   │   ├── s2_planner.py        # Stage 2: Planning agent
│   │   ├── s3_tool_selector.py  # Stage 3: Tool selection
│   │   ├── s4_query_builder.py  # Stage 4: SQL query builder
│   │   ├── s5_executor.py       # Stage 5: Execution engine
│   │   ├── s6_postprocess.py    # Stage 6: Post-processing
│   │   └── s7_formatter.py      # Stage 7: Response formatter
│   └── metrics.py               # AgentMetrics class (unchanged from current)
│
├── tools/
│   ├── __init__.py
│   ├── registry.py              # Central tool registry — all tools registered here
│   ├── transactions/
│   │   ├── __init__.py
│   │   └── filters.py           # All transaction filter tools
│   ├── projects/
│   │   ├── __init__.py
│   │   └── filters.py           # All project filter tools
│   ├── compute/
│   │   ├── __init__.py
│   │   └── engines.py           # Aggregation, trend, derived-metric engines
│   ├── amenity_tool.py          # KEEP AS IS (existing)
│   └── text_to_sql.py           # KEEP AS IS (existing)
│
└── database/
    └── db.py                    # KEEP AS IS (existing)
```

---

## Stage 1 — Intent + Entity Extraction (`s1_intent.py`)

### Purpose
Convert the raw user query string into a machine-readable JSON object. This is the only place where natural language is parsed. Everything downstream is structured.

### What to build

```python
# backend/agent/stages/s1_intent.py

import json
from openai import OpenAI

INTENT_EXTRACTION_PROMPT = """
Extract structured intent for a real-estate analytics pipeline.
Use ONLY values explicitly present in the user query.
Do NOT guess city/project/metric/ranges. If unknown, omit the field.
Do NOT create `entities.geo` unless query explicitly asks proximity/radius/nearby
or directly provides coordinates.
Do NOT set default `radius_m` values.
Return valid JSON only.

{{
  "intent": string,             // trend | comparison | ranking | count | summary | amenity_search | unknown
  "data_source": string,        // transactions | projects | listings | amenities | mixed
  "entities": {{
    "metric": string,
    "group_by": [string],
    "analysis_type": string,    // trend | comparison | ranking | summary | count
    "sort_by": string,
    "limit": number,
    "filters": {{
      "city": string, "location_name": string, "village_name": string,
      "micro_market": string, "sub_locality": string, "pincode": string,
      "project_name": string,
      "project_id": string, "internal_index_id": string, "project_registration_id": string,
      "transaction_type": string, "transaction_category": string, "sale_type": string,
      "property_type": string, "property_type_raw": string, "project_type": string,
      "unit_configuration": string,
      "floor_number": number, "unit_number": string, "tower_name": string,
      "facing_direction": string, "view_type": string,
      "furnishing_status": string, "condition_status": string,
      "developer_name": string, "construction_status": string, "approval_status": string, "encumbrance_status": string,
      "sro_code": string, "sro_name": string, "document_number": string,
      "buyer_name": string, "seller_name": string, "buyer_locality": string, "buyer_district": string, "buyer_state": string, "buyer_pincode": string,
      "bank_type": string, "party_code": string,
      "data_source": string, "data_type": string, "source_accessibility": string, "source_accessibility_way": string,
      "price_range": {{"min": number, "max": number}},
      "guideline_value_range": {{"min": number, "max": number}},
      "ppsf_range": {{"min": number, "max": number}},
      "area_range_sqft": {{"min": number, "max": number}},
      "net_area_range_sq_m": {{"min": number, "max": number}},
      "total_units_range": {{"min": number, "max": number}},
      "booked_units_range": {{"min": number, "max": number}},
      "booked_pct_range": {{"min": number, "max": number}},
      "total_fsi_range": {{"min": number, "max": number}},
      "total_plot_area_range_sq_m": {{"min": number, "max": number}}
    }},
    "geo": {{
      "type": string,           // radius | coordinate | project_name
      "lat": number,
      "lon": number,
      "radius_m": number,
      "reference_project": string
    }},
    "time": {{
      "period": string,         // all | specific | range | last_year | last_2_years
      "year": number,
      "quarter": string,
      "year_range": {{"start": number, "end": number}},
      "start_date": string,
      "end_date": string
    }}
  }}
}}

User query: {user_query}
"""
```

### Rules
- Always use `response_format={"type": "json_object"}` and `temperature=0`
- If JSON parsing fails, return `{"intent": "unknown", "entities": {}, "data_source": "transactions"}`
- Do NOT add fallback business logic here — unknown intents get handled in Stage 2

---

## Stage 2 — Planning Agent (`s2_planner.py`)

### Purpose
Convert the structured intent into an **ordered list of abstract steps**. This is the brain of the system. The plan tells every downstream stage what to do without knowing *how* to do it.

### What to build

```python
# backend/agent/stages/s2_planner.py

PLANNING_PROMPT = """
Create a short, deterministic execution plan from intent JSON.
Use only step types listed below. Do not invent new steps.

Available step types:
- apply_geo_filter
- apply_time_filter
- apply_space_filter
- apply_project_filter
- apply_price_filter
- apply_area_filter
- apply_config_filter
- apply_transaction_filter
- apply_property_filter
- apply_developer_filter
- apply_construction_filter
- compute_derived_metric
- aggregate_data
- compute_trend
- compute_ranking
- fetch_amenities
- fetch_project_coords
- join_tables
- format_output

Rules:
1) Include a step only if required by intent fields.
2) `apply_geo_filter` only when explicit coordinates exist (lat and lon).
   Place names like city/location/micro_market/sub_locality alone must use `apply_space_filter`, not `apply_geo_filter`.
3) Use join_tables only when query needs columns from multiple tables.
4) Keep steps unique and ordered.
5) format_output must be last.
6) If amenity search needs a project name and lat/lon is missing, include fetch_project_coords before fetch_amenities.

Return JSON only:
{{
  "steps": [string],
  "explanation": string
}}

Intent JSON: {intent_json}
"""

```

### Rules
- Never hardcode `if intent == "rate_trend" then steps = [...]`
- The LLM decides the steps. Your code just runs them.
- `format_output` must always be last

---

## Stage 3 — Tool Selection Agent (`s3_tool_selector.py`)

### Purpose
Map each abstract plan step to a concrete tool from the registry. The tool registry is the single source of truth for what tools exist and what they do.

### What to build

```python
# backend/agent/stages/s3_tool_selector.py

TOOL_SELECTION_PROMPT = """
Select tools for each plan step using the registry.
Use exact tool names from registry only.
Use only arguments found in intent JSON. Do not invent values.

Available tools:
{tool_registry_json}

Rules:
1) Skip steps that do not need a tool.
2) At most one tool per step.
3) Prefer the most specific tool for the step.
4) tool_args keys should match intent field names.

Return JSON only:
{{
  "selections": [
    {{
      "step": string,
      "tool_name": string,
      "tool_args": {{}}
    }}
  ]
}}

Plan steps: {plan_steps}
Intent: {intent_json}
"""
```

---

## Stage 4 — Query Builder (`s4_query_builder.py`)

### Purpose
Generate the SQL query (or queries) needed to execute the plan. This is schema-aware and uses self-healing retry on failure.

### What to build

```python
# backend/agent/stages/s4_query_builder.py

SCHEMA = """
Allowed tables and columns:

transactions:
  project_id, internal_index_id, project_name, location_name, village_name, year, quarter,
  city_id, city_name, transaction_category_id, sub_registrar_office_code, sub_registrar_office_name,
  document_number, transaction_type, transaction_category, agreement_price, guideline_value,
  transaction_date, date_of_agreement_execution, floor_number, unit_number, tower_name,
  property_type, property_type_raw, unit_configuration, net_carpet_area_sq_m, 
  balcony_sq_m, terrace_sq_m, stamp_duty_paid, registration_fee,
  seller_name, buyer_name, bank_type, party_code, project_latitude, project_longitude,
  location_latitude, location_longitude, buyer_pincode, buyer_locality, buyer_district, buyer_state,
  sale_type, project_type, country_name, state_name, micro_market, sub_locality, pincode,
  facing_direction, view_type, furnishing_status, condition_status, is_duplicate, is_llm_processed,
  is_manual_processed, source_accessibility, source_accessibility_way, sourcing_cost, sourcing_time,
  data_type, data_source

projects:
  project_id, internal_index_id, registered_project_name, project_name, location_id, location_name,
  city_name, city_id, project_latitude, project_longitude, location_latitude, location_longitude,
  plot_number, project_registration_id, total_units, booked_units, commencement_date,
  final_proposed_date_of_completion, project_bhk_summary, organization_individual_name,
  number_of_developers, pincode, total_fsi, total_plot_area_sq_m, bhk_wise_min_max_area,
  bhk_wise_carpet_area, project_type, bhk_wise_total_booked_units, carpet_wise_total_booked_units,
  total_building_count, project_tower_completion_date, number_of_sanctioned_floors,
  amenity_profile, age_of_project, construction_status, building_grade, zoning_type,
  encumbrance_status, country_name, state_name, sub_locality, micro_market, frontage,
  approval_status, data_source, source_accessibility, source_accessibility_way, sourcing_cost,
  sourcing_time, data_type
"""

SQL_BUILD_PROMPT = """
Write one PostgreSQL SELECT query from plan + intent + selected tools.
Use only tables/columns in SCHEMA.

SCHEMA (strict):
{schema}

Plan steps: {plan_steps}
Intent: {intent_json}
Tool selections: {tool_selections_json}

RULES:
1) Output must be one SELECT (or WITH ... SELECT).
2) Never use INSERT/UPDATE/DELETE/DDL.
3) Use only filters explicitly present in intent/tool_args.
4) Use ILIKE for text filters unless exact match is explicitly required.
5) Add LIMIT 1000 by default unless user asks another limit.
6) For geo filter use the 
7) Booked %: (booked_units::float / NULLIF(total_units, 0)) * 100
8) For trend analysis, include year and/or quarter in SELECT and GROUP BY.
10) Return only columns needed for answer.

Return raw SQL only. No markdown. No explanation.
"""

SQL_FIX_PROMPT = """
Fix the failed SQL query.
Keep original intent. Do not add unsupported columns/tables.
Output one corrected SELECT query only.

SQL:
{sql}

Error:
{error}

Schema:
{schema}
"""

```

### Rules
- Strip markdown fences (` ```sql ` etc.) before executing
- Validate: query must start with `SELECT` or `WITH` (case-insensitive)
- Reject any query containing `INSERT`, `UPDATE`, `DELETE`, `DROP`, `TRUNCATE`, `ALTER`
- Max retry attempts: 3

---

## Stage 5 — Execution Engine (`s5_executor.py`)

### Purpose
Execute the SQL query against PostgreSQL. On failure, call QueryBuilder.fix() and retry. Track retries in metrics.

### What to build

```python
# backend/agent/stages/s5_executor.py

import re
from sqlalchemy import text
from backend.database.db import engine

DANGEROUS_KEYWORDS = {"insert", "update", "delete", "drop", "truncate", "alter", "create"}

class ExecutionEngine:
    def __init__(self, query_builder: QueryBuilder, max_retries: int = 3):
        self.query_builder = query_builder
        self.max_retries = max_retries

    def _validate(self, sql: str) -> None:
        clean = sql.strip().lower()
        if not (clean.startswith("select") or clean.startswith("with")):
            raise ValueError(f"Query must start with SELECT or WITH. Got: {clean[:40]}")
        for kw in DANGEROUS_KEYWORDS:
            if re.search(rf"\b{kw}\b", clean):
                raise ValueError(f"Dangerous keyword detected: {kw}")

    def _strip_fences(self, sql: str) -> str:
        sql = re.sub(r"```(?:sql)?", "", sql, flags=re.IGNORECASE)
        return sql.replace("```", "").strip()

    def execute(self, sql: str) -> dict:
        sql = self._strip_fences(sql)
        last_error = None

        for attempt in range(self.max_retries):
            try:
                self._validate(sql)
                with engine.connect() as conn:
                    result = conn.execute(text(sql))
                    columns = list(result.keys())
                    rows = [dict(zip(columns, row)) for row in result.fetchall()]
                return {"status": "success", "data": rows, "columns": columns, "retries": attempt}
            except Exception as e:
                last_error = str(e)
                if attempt < self.max_retries - 1:
                    sql = self._strip_fences(self.query_builder.fix(sql, last_error))
                    continue

        return {"status": "error", "error": last_error, "data": [], "retries": self.max_retries}
```

---

## Stage 6 — Post-Processing Layer (`s6_postprocess.py`)

### Purpose
Perform computations that SQL cannot do natively or that are better handled in Python: trend percentages, YoY change, rankings, outlier flagging, data confidence scoring.

### What to build

```python
# backend/agent/stages/s6_postprocess.py

import pandas as pd
from typing import Optional

class PostProcessor:

    def process(self, data: list[dict], intent: dict) -> dict:
        if not data:
            return {"processed_data": [], "summary_stats": {}, "confidence": 0.0}

        df = pd.DataFrame(data)
        analysis_type = intent.get("entities", {}).get("analysis_type", "summary")
        group_by = intent.get("entities", {}).get("group_by", [])

        result = {"processed_data": data, "summary_stats": {}, "confidence": 1.0}

        # Trend computation
        if analysis_type == "trend" and "year" in df.columns:
            df = df.sort_values("year")
            metric_cols = [c for c in df.columns if c not in group_by and df[c].dtype in ["float64", "int64"]]
            for col in metric_cols:
                df[f"{col}_pct_change"] = df.groupby(
                    [g for g in group_by if g != "year" and g in df.columns]
                )[col].pct_change() * 100 if group_by else df[col].pct_change() * 100
            result["processed_data"] = df.to_dict(orient="records")

        # Summary stats
        numeric_cols = df.select_dtypes(include="number").columns.tolist()
        if numeric_cols:
            result["summary_stats"] = {
                col: {
                    "mean": round(df[col].mean(), 2),
                    "median": round(df[col].median(), 2),
                    "min": round(df[col].min(), 2),
                    "max": round(df[col].max(), 2),
                    "count": int(df[col].count())
                }
                for col in numeric_cols[:5]  # cap at 5 metrics for readability
            }

        # Confidence score: based on row count
        n = len(df)
        result["confidence"] = min(1.0, n / 30) if n < 30 else 1.0

        return result
```

---

## Stage 7 — Response Formatter (`s7_formatter.py`)

### Purpose
Convert the processed data into a user-readable markdown report with tables, insights, and a confidence note.

### What to build

```python
# backend/agent/stages/s7_formatter.py

FORMATTER_PROMPT = """
You are writing a concise markdown real-estate report.
Use only provided data. Do not invent facts, values, trends, or locations.

If data is empty:
- Say no matching records were found.
- Suggest 1-2 practical filter changes.

If data exists:
1) One short executive summary.
2) One markdown table (max 20 rows).
3) 2-3 evidence-based insights from table/stats.
4) One confidence line at end.

Never include raw JSON or SQL.
Data confidence: {confidence}

User's original question: {original_question}
Intent: {intent_json}
Processed data (first 20 rows): {data_json}
Summary stats: {stats_json}
{amenity_section}
"""

AMENITY_SECTION_TEMPLATE = """
Nearby Amenities Data:
{amenity_json}
Include a brief amenities section only if amenity data is present.
"""
```

---

## Tool Registry (`tools/registry.py`)

### Purpose
Single source of truth for all tools. Every tool is registered here with its name, description, and the SQL clause or Python logic it contributes.

### What to build

```python
# backend/tools/registry.py

from dataclasses import dataclass
from typing import Callable, Optional
import json

@dataclass
class Tool:
    name: str
    category: str           # "transaction_filter" | "project_filter" | "compute" | "amenity"
    description: str
    sql_fragment: Optional[str]         # parameterized SQL WHERE/JOIN clause template
    python_fn: Optional[Callable]       # for non-SQL tools
    required_intent_fields: list[str]   # intent fields this tool needs

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}
        self._register_all()

    def register(self, tool: Tool):
        self._tools[tool.name] = tool

    def get(self, name: str) -> Optional[Tool]:
        return self._tools.get(name)

    def get_schema_json(self) -> str:
        return json.dumps([
            {"name": t.name, "category": t.category, "description": t.description,
             "required_fields": t.required_intent_fields}
            for t in self._tools.values()
        ], indent=2)

    def _register_all(self):
        # ── TRANSACTION FILTERS ──────────────────────────────────────────────
        self.register(Tool(
            name="filter_by_space",
            category="transaction_filter",
            description="Filter transactions by city, micro_market, or sub_locality",
            sql_fragment="city_name = '{city}' AND micro_market = '{micro_market}'",
            python_fn=None,
            required_intent_fields=["entities.filters.city"]
        ))
        self.register(Tool(
            name="filter_by_time",
            category="transaction_filter",
            description="Filter transactions by year and/or quarter",
            sql_fragment="year BETWEEN {start_year} AND {end_year}",
            python_fn=None,
            required_intent_fields=["entities.time"]
        ))
        self.register(Tool(
            name="filter_by_project",
            category="transaction_filter",
            description="Filter transactions by project name",
            sql_fragment="project_name ILIKE '%{project_name}%'",
            python_fn=None,
            required_intent_fields=["entities.filters.project_name"]
        ))
        self.register(Tool(
            name="filter_by_transaction_type",
            category="transaction_filter",
            description="Filter by sale or rental transaction type",
            sql_fragment="transaction_type = '{transaction_type}'",
            python_fn=None,
            required_intent_fields=["entities.filters.transaction_type"]
        ))
        self.register(Tool(
            name="filter_by_price_range",
            category="transaction_filter",
            description="Filter transactions by agreement price range",
            sql_fragment="agreement_price BETWEEN {min} AND {max}",
            python_fn=None,
            required_intent_fields=["entities.filters.price_range"]
        ))
        self.register(Tool(
            name="filter_by_floor",
            category="transaction_filter",
            description="Filter by floor number",
            sql_fragment="floor_number = {floor_number}",
            python_fn=None,
            required_intent_fields=["entities.filters.floor_number"]
        ))
        self.register(Tool(
            name="filter_by_property_type",
            category="transaction_filter",
            description="Filter by property type: Flat, Shop, Office",
            sql_fragment="property_type = '{property_type}'",
            python_fn=None,
            required_intent_fields=["entities.filters.property_type"]
        ))
        self.register(Tool(
            name="filter_by_configuration",
            category="transaction_filter",
            description="Filter by BHK configuration e.g. 1BHK, 2BHK, 3BHK",
            sql_fragment="unit_configuration = '{unit_configuration}'",
            python_fn=None,
            required_intent_fields=["entities.filters.unit_configuration"]
        ))
        self.register(Tool(
            name="filter_by_area_range",
            category="transaction_filter",
            description="Filter by carpet area in sq ft",
            sql_fragment="gross_carpet_area_sq_ft BETWEEN {min} AND {max}",
            python_fn=None,
            required_intent_fields=["entities.filters.area_range_sqft"]
        ))
        self.register(Tool(
            name="filter_by_geo_bounds",
            category="transaction_filter",
            description="Filter transactions within a radius of a coordinate using haversine",
            sql_fragment="/* join with projects on project_name, then haversine filter */",
            python_fn=None,
            required_intent_fields=["entities.geo"]
        ))
        self.register(Tool(
            name="filter_by_sale_type",
            category="transaction_filter",
            description="Filter by sale type",
            sql_fragment="sale_type = '{sale_type}'",
            python_fn=None,
            required_intent_fields=["entities.filters.sale_type"]
        ))
        self.register(Tool(
            name="filter_by_project_type",
            category="transaction_filter",
            description="Filter by project type (residential, commercial, etc.)",
            sql_fragment="project_type = '{project_type}'",
            python_fn=None,
            required_intent_fields=["entities.filters.project_type"]
        ))
        self.register(Tool(
            name="filter_by_facing_direction",
            category="transaction_filter",
            description="Filter by facing direction of unit",
            sql_fragment="facing_direction = '{facing_direction}'",
            python_fn=None,
            required_intent_fields=["entities.filters.facing_direction"]
        ))
        self.register(Tool(
            name="filter_by_furnishing_status",
            category="transaction_filter",
            description="Filter by furnishing status (furnished, semi-furnished, unfurnished)",
            sql_fragment="furnishing_status = '{furnishing_status}'",
            python_fn=None,
            required_intent_fields=["entities.filters.furnishing_status"]
        ))

        # ── PROJECT FILTERS ──────────────────────────────────────────────────
        self.register(Tool(
            name="filter_project_by_space",
            category="project_filter",
            description="Filter projects by city, micro_market, sub_locality",
            sql_fragment="city_name = '{city}' AND micro_market = '{micro_market}'",
            python_fn=None,
            required_intent_fields=["entities.filters.city"]
        ))
        self.register(Tool(
            name="filter_project_by_geo_bounds",
            category="project_filter",
            description="Filter projects within radius using haversine on project_latitude/longitude",
            sql_fragment="/* haversine on project_latitude, project_longitude */",
            python_fn=None,
            required_intent_fields=["entities.geo"]
        ))
        self.register(Tool(
            name="filter_by_total_units",
            category="project_filter",
            description="Filter projects by total unit count range",
            sql_fragment="total_units BETWEEN {min} AND {max}",
            python_fn=None,
            required_intent_fields=["entities.filters"]
        ))
        self.register(Tool(
            name="filter_by_booked_units_pct",
            category="project_filter",
            description="Filter by percentage of booked units",
            sql_fragment="(booked_units::float / NULLIF(total_units,0)) * 100 >= {min_pct}",
            python_fn=None,
            required_intent_fields=["entities.filters"]
        ))
        self.register(Tool(
            name="filter_by_developer_name",
            category="project_filter",
            description="Filter projects by developer or organization name",
            sql_fragment="organization_individual_name ILIKE '%{developer_name}%'",
            python_fn=None,
            required_intent_fields=["entities.filters.developer_name"]
        ))
        self.register(Tool(
            name="filter_by_construction_status",
            category="project_filter",
            description="Filter by construction status",
            sql_fragment="construction_status = '{construction_status}'",
            python_fn=None,
            required_intent_fields=["entities.filters.construction_status"]
        ))
        self.register(Tool(
            name="filter_by_approval_status",
            category="project_filter",
            description="Filter by project approval status",
            sql_fragment="approval_status = '{approval_status}'",
            python_fn=None,
            required_intent_fields=["entities.filters.approval_status"]
        ))
        self.register(Tool(
            name="filter_by_bhk_summary",
            category="project_filter",
            description="Filter projects by available BHK types in project",
            sql_fragment="project_bhk_summary ILIKE '%{bhk}%'",
            python_fn=None,
            required_intent_fields=["entities.filters.unit_configuration"]
        ))
        self.register(Tool(
            name="filter_by_age_of_project",
            category="project_filter",
            description="Filter by project age using commencement_date",
            sql_fragment="EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM commencement_date) BETWEEN {min_age} AND {max_age}",
            python_fn=None,
            required_intent_fields=["entities.filters"]
        ))
        self.register(Tool(
            name="filter_by_total_fsi",
            category="project_filter",
            description="Filter by total FSI value",
            sql_fragment="total_fsi BETWEEN {min} AND {max}",
            python_fn=None,
            required_intent_fields=["entities.filters"]
        ))
        self.register(Tool(
            name="filter_by_total_plot_area",
            category="project_filter",
            description="Filter by total plot area in sq m",
            sql_fragment="total_plot_area_sq_m BETWEEN {min} AND {max}",
            python_fn=None,
            required_intent_fields=["entities.filters"]
        ))
        self.register(Tool(
            name="filter_by_amenity_profile",
            category="project_filter",
            description="Filter projects by amenity profile (gym, pool, etc.)",
            sql_fragment="amenity_profile ILIKE '%{amenity}%'",
            python_fn=None,
            required_intent_fields=["entities.filters"]
        ))
        self.register(Tool(
            name="fetch_project_coords",
            category="project_filter",
            description="Fetch project lat/lon by name — prerequisite for geo-based amenity search",
            sql_fragment="SELECT project_latitude, project_longitude FROM projects WHERE project_name ILIKE '%{project_name}%' LIMIT 1",
            python_fn=None,
            required_intent_fields=["entities.geo.reference_project"]
        ))

        # ── COMPUTE TOOLS ────────────────────────────────────────────────────
        self.register(Tool(
            name="aggregation_engine",
            category="compute",
            description="GROUP BY + aggregate: SUM, AVG, COUNT, MEDIAN on numeric columns",
            sql_fragment="GROUP BY {group_cols}",
            python_fn=None,
            required_intent_fields=["entities.group_by"]
        ))
        self.register(Tool(
            name="derived_metric_engine",
            category="compute",
            description="Compute derived metrics: rate_per_sqft, booked_pct, avg_price",
            sql_fragment="SUM(agreement_price) / NULLIF(SUM(gross_carpet_area_sq_ft), 0) AS avg_rate_per_sqft",
            python_fn=None,
            required_intent_fields=["entities.metric"]
        ))
        self.register(Tool(
            name="trend_engine",
            category="compute",
            description="Compute YoY or QoQ percentage change (handled in post-processing)",
            sql_fragment=None,
            python_fn=None,
            required_intent_fields=["entities.analysis_type"]
        ))
        self.register(Tool(
            name="ranking_engine",
            category="compute",
            description="Rank results by a metric using ORDER BY",
            sql_fragment="ORDER BY {sort_col} DESC LIMIT {limit}",
            python_fn=None,
            required_intent_fields=["entities.sort_by"]
        ))

        # ── AMENITY TOOLS ────────────────────────────────────────────────────
        self.register(Tool(
            name="get_amenities",
            category="amenity",
            description="Fetch nearby amenities from OSM using lat/lon + radius",
            sql_fragment=None,
            python_fn=None,   # handled by existing amenity_tool.py
            required_intent_fields=["entities.geo"]
        ))
```

---

## Main Pipeline Orchestrator (`pipeline.py`)

### Purpose
Wire all 7 stages together in a streaming generator. This replaces `RealEstateAgent.execute_stream()`.

### What to build

```python
# backend/agent/pipeline.py

import json
import time
from openai import OpenAI

from backend.agent.stages.s1_intent import IntentExtractor
from backend.agent.stages.s2_planner import PlannerAgent
from backend.agent.stages.s3_tool_selector import ToolSelectorAgent
from backend.agent.stages.s4_query_builder import QueryBuilder
from backend.agent.stages.s5_executor import ExecutionEngine
from backend.agent.stages.s6_postprocess import PostProcessor
from backend.agent.stages.s7_formatter import ResponseFormatter
from backend.agent.metrics import AgentMetrics
from backend.tools.registry import ToolRegistry
from backend.tools.amenity_tool import get_amenities

def _sse(event_type: str, content, **kwargs) -> str:
    payload = {"type": event_type, "content": content, **kwargs}
    return f"data: {json.dumps(payload, default=str)}\n\n"

class UniversalRealEstateAgent:
    def __init__(self):
        self.client = OpenAI()
        self.registry = ToolRegistry()
        self.intent_extractor = IntentExtractor(self.client)
        self.planner = PlannerAgent(self.client)
        self.tool_selector = ToolSelectorAgent(self.client, self.registry)
        self.query_builder = QueryBuilder(self.client)
        self.executor = ExecutionEngine(self.query_builder)
        self.postprocessor = PostProcessor()
        self.formatter = ResponseFormatter(self.client)

    def execute_stream(self, question: str):
        metrics = AgentMetrics()
        yield _sse("start", f"Processing: {question}")

        # ── STAGE 1: Intent extraction ────────────────────────────────────
        yield _sse("stage", "Stage 1: Extracting intent and entities...")
        intent = self.intent_extractor.extract(question)
        yield _sse("intent", intent)

        # ── STAGE 2: Planning ─────────────────────────────────────────────
        yield _sse("stage", "Stage 2: Building execution plan...")
        plan = self.planner.plan(intent)
        yield _sse("plan", plan)

        # ── STAGE 3: Tool selection ───────────────────────────────────────
        yield _sse("stage", "Stage 3: Selecting tools...")
        tool_selections = self.tool_selector.select(plan, intent)
        yield _sse("tool_selections", tool_selections)

        # ── AMENITY SHORTCUT ──────────────────────────────────────────────
        # If plan includes fetch_amenities, we may need coords first
        needs_amenities = "fetch_amenities" in plan["steps"]
        needs_project_coords = "fetch_project_coords" in plan["steps"]
        project_coords = None

        if needs_project_coords:
            ref_project = intent.get("entities", {}).get("geo", {}).get("reference_project", "")
            if ref_project:
                yield _sse("stage", f"Fetching coordinates for: {ref_project}")
                coord_sql = f"SELECT project_latitude, project_longitude FROM projects WHERE project_name ILIKE '%{ref_project}%' LIMIT 1"
                coord_result = self.executor.execute(coord_sql)
                if coord_result["data"]:
                    project_coords = coord_result["data"][0]
                    yield _sse("observation", f"Found coords: {project_coords}")

        # ── STAGE 4: Query building ───────────────────────────────────────
        sql = None
        db_result = {"status": "skipped", "data": [], "columns": []}

        if "fetch_amenities" not in plan["steps"] or len(plan["steps"]) > 2:
            yield _sse("stage", "Stage 4: Building SQL query...")
            sql = self.query_builder.build(plan, intent, tool_selections)
            yield _sse("sql_query", sql)

            # ── STAGE 5: Execution ────────────────────────────────────────
            yield _sse("stage", "Stage 5: Executing query...")
            t0 = time.time()
            db_result = self.executor.execute(sql)
            duration = round(time.time() - t0, 2)
            metrics.tools_called += 1

            if db_result["status"] == "success":
                yield _sse("observation_preview", f"Retrieved {len(db_result['data'])} rows in {duration}s.",
                           data=db_result["data"][:3])
            else:
                yield _sse("error", f"Execution failed after retries: {db_result.get('error')}")

        # ── AMENITY FETCH ─────────────────────────────────────────────────
        amenity_result = None
        if needs_amenities:
            geo = intent.get("entities", {}).get("geo", {})
            lat = project_coords.get("project_latitude") if project_coords else geo.get("lat")
            lon = project_coords.get("project_longitude") if project_coords else geo.get("lon")
            radius = geo.get("radius_m", 500)
            amenity_type = intent.get("entities", {}).get("filters", {}).get("property_type", "")

            if lat and lon:
                yield _sse("action", f"Fetching amenities within {radius}m...")
                amenity_result = get_amenities(lat=lat, lon=lon, radius=radius, amenity_type=amenity_type)
                metrics.tools_called += 1
                yield _sse("observation", f"Found {amenity_result.get('count', 0)} amenities.")

        # ── STAGE 6: Post-processing ──────────────────────────────────────
        yield _sse("stage", "Stage 6: Post-processing data...")
        processed = self.postprocessor.process(db_result["data"], intent)
        if amenity_result:
            processed["amenity_data"] = amenity_result

        # ── STAGE 7: Formatting ───────────────────────────────────────────
        yield _sse("stage", "Stage 7: Generating report...")
        report = self.formatter.format(processed, intent, question)
        for chunk in report.split("\n"):
            yield _sse("report_chunk", chunk + "\n")

        yield _sse("done", "", metrics=metrics.finalize())
```

---

## Error Handling Rules

1. Every stage must catch its own exceptions and yield an `_sse("error", ...)` event — never crash the generator
2. If Stage 1 returns `intent == "unknown"`, yield a clarification message and stop the pipeline early
3. If Stage 5 returns `status == "error"` after 3 retries, skip Stage 6 and go directly to Stage 7 with the error as context (formatter will explain what went wrong gracefully)
4. All LLM calls must have a 30-second timeout

---

## Testing Checklist (run these queries end-to-end)

- "What is the average rate per sqft for 2BHK flats in Wakad for the last 3 years, year-wise?"
- "Show me top 10 projects in Pune by number of transactions in 2023"
- "What bus stops are within 500m of Om Paradise project?"
- "Compare 1BHK vs 2BHK average prices in Baner micro-market"
- "Which developer registered the most projects in Pimpri-Chinchwad?"
- "Show me all projects with more than 80% booking and under-construction status"
- "What is the price trend for South-facing 3BHK flats in Kharadi from 2020 to 2024?"
- "Give me all listings in Hinjewadi with price below 80 lakhs"

Each query should produce a valid SQL query, execute successfully, and return a formatted markdown report without any code modification.

---

## Constraints for the Coding Agent

1. **No hardcoded intent routing.** Every conditional must be driven by LLM output.
2. **No hardcoded SQL fragments** except in tool `sql_fragment` fields, which are descriptive hints only — the final SQL is always LLM-generated.
3. **All prompts must be strings with `{placeholder}` format** — never f-strings with business logic embedded.
4. **Tool registry is the single source of truth.** Adding a new filter = add one entry to `_register_all()`. Nothing else changes.
5. **The pipeline generator must yield SSE events for every stage** — the frontend depends on these event types: `start`, `stage`, `intent`, `plan`, `tool_selections`, `sql_query`, `observation`, `observation_preview`, `action`, `report_chunk`, `error`, `done`.
6. **Keep existing files unchanged:** `backend/tools/amenity_tool.py`, `backend/tools/text_to_sql.py`, `backend/database/db.py`.
7. **Python version**: 3.10+. Use `list[dict]` not `List[Dict]`.
8. **Dependencies required:** `openai`, `sqlalchemy`, `pandas`, `python-dotenv`.
```
