# Architecture & Engineering Guide: Natural Language to SQL (NL-to-SQL) & Database Agents

## 1. Overview & Core Philosophy
Natural Language to SQL (NL-to-SQL) agents translate ambiguous human queries into executable database operations. In enterprise environments, failure to design strict boundaries results in severe risks: SQL injection, accidental database corruption (`DROP`/`DELETE`), runaway full-table scans, schema hallucination, and exorbitant LLM token burn during query retries.

This document formalizes the **Three-Layer Architecture** for NL-to-SQL agents across SQLite and PostgreSQL, defining exactly where to reuse industry-standard libraries versus where to build custom, token-efficient supervisory agents.

---

## 2. The Three-Layer Architecture

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Layer 3: Agent Orchestration & Self-Correction (ReAct Loop)                           │
│ - Prompt-caching friendly DDL context                                                  │
│ - Bounded retry loop (max 3-5 loops) on execution errors                               │
│ - Pluggable Choice: Lightweight Core Engine (Recommended) vs. Frameworks (Vanna/LC)   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Layer 2: SQL AST Parsing, Validation & Guardrails (MANDATORY STANDARD LIBRARY)         │
│ - Engine: `sqlglot`                                                                    │
│ - AST statement validation (Whitelisted: SELECT, WITH; Forbidden: DROP, DELETE, etc.) │
│ - Injected LIMIT / paging guarantees & dialect transpilation (SQLite <-> Postgres)     │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ Layer 1: Database Connection & Schema Introspection (MANDATORY STANDARD LIBRARY)       │
│ - Engine: `SQLAlchemy`                                                                 │
│ - Standardized connection pooling, read-only session modes                             │
│ - Reflection of tables, columns, data types, foreign key relationships, & sample rows  │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Layer Breakdown & Decision Matrix

### Layer 1: Schema Introspection & Database Engine
* **Rule**: **Always reuse `SQLAlchemy`**. Never write custom dialect reflection queries (e.g. SQLite `PRAGMA table_info` or Postgres `pg_catalog.pg_attribute`).
* **Capabilities Reused**:
  - `sqlalchemy.inspect(engine)` to reflect tables, columns, primary keys, foreign keys, and column nullability.
  - Uniform connection abstractions across SQLite (`sqlite:///...`) and PostgreSQL (`postgresql+psycopg2://...`).
  - Read-only execution configurations (SQLite URI `mode=ro`, PostgreSQL `SET TRANSACTION READ ONLY`).

### Layer 2: SQL AST Validation & Sanitization
* **Rule**: **Always reuse `sqlglot`**. Never use custom regex or string patterns to detect queries or validate permissions.
* **Capabilities Reused**:
  - `sqlglot.parse_one(sql, read=dialect)` for dialect-aware AST generation.
  - Statement type whitelisting: only allow `sqlglot.exp.Select` or CTEs (`sqlglot.exp.With`).
  - Strict blocking of mutation / DDL operations (`Drop`, `Delete`, `Update`, `Insert`, `Alter`, `Truncate`).
  - Safe dialect transpilation (e.g., transpiling PostgreSQL date truncations or ILIKE into SQLite equivalents).
  - Deterministic AST modification: automatically injecting a safety `LIMIT 100` if the generated query omits a limit clause.

### Layer 3: Agent Orchestration & Error Recovery Loop
* **Rule**: **Provide a Pluggable Choice**. Default to our in-core token-efficient supervisory wrapper, but abstract it via a base interface (`BaseDatabaseAgent`).
* **Why Core ReAct Orchestration is Default**:
  - **Prompt Caching Efficiency**: Static DDL schemas are fixed at the top of the prompt payload; dynamic user queries and error states follow.
  - **Targeted Error Recovery**: When database execution fails (e.g., missing column, ambiguous join), send only the query diff and the exact database exception string, avoiding full context duplication.
  - **Loop Bounds**: Hard circuit breaker (`max_loops = 3`) prevents infinite retry loops and runaway token burn.
  - **Workspace Integration**: Direct coupling with `OutputGuard` (auto-offloading multi-megabyte result sets to disk pointers) and `HITL` approval workflows.

---

## 4. Dialect Considerations: SQLite vs. PostgreSQL

| Feature / Behavior | SQLite | PostgreSQL |
| :--- | :--- | :--- |
| **Typing System** | Dynamic (Type affinity: INTEGER, TEXT, REAL, BLOB). | Strict static types, custom ENUMs, composite types, arrays. |
| **Date/Time Functions** | `strftime()`, `datetime('now')`. | `NOW()`, `DATE_TRUNC()`, intervals (`interval '1 day'`). |
| **Case Sensitivity** | `LIKE` is case-insensitive for ASCII characters. | `LIKE` is case-sensitive; `ILIKE` is required for case-insensitivity. |
| **Namespaces / Schemas** | Single default namespace (`main`). | Multi-schema support (`public`, custom schemas). Requires schema-qualified names. |
| **Read-Only Mode** | `file:...uri=true?mode=ro`. | `SET TRANSACTION READ ONLY;` or dedicated read-only role (`GRANT SELECT ON ALL TABLES...`). |

---

## 5. Standard Error Recovery Pattern

When a query fails during execution, the Layer 3 agent executes the following recovery protocol:

1. **Capture Exact Error**: Retain the DB engine's exception message (e.g. `psycopg2.errors.UndefinedColumn: column "created_date" does not exist`).
2. **Retrieve Contextual Schema Snippet**: Pull the specific column definitions for the referenced table.
3. **Targeted Correction Prompt**:
   ```
   The previously generated SQL failed execution:
   Error: {error_message}
   Query: {failed_sql}
   Relevant Table Schema:
   {table_ddl}

   Generate a corrected SQL query addressing the exact error above.
   ```
4. **Counter Increment**: Increment loop counter. If `attempt >= max_attempts`, halt and return diagnostic error to the user.
