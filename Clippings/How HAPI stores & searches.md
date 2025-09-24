---
title: How HAPI stores & searches
source: https://chatgpt.com/c/68d13959-f398-8321-9086-55e8d8936944
author:
  - "[[ChatGPT]]"
published:
created: 2025-09-24
description: ChatGPT helps you get answers, find inspiration and be more productive. It is free to use and easy to try. Just ask and ChatGPT can help with writing, learning, brainstorming and more.
tags:
  - clippings
---
## How HAPI stores & searches

**1) Canonical copy in a big column (CLOB/JSON)**

- The full FHIR resource (JSON/XML) is stored “as-is” in a large text/JSON column (table often named like `HFJ_RESOURCE` / `HFJ_RES_VER` with fields such as `RES_TEXT`/`RES_JSON`), preserving the exact wire form.

**2) Search parameters extracted to side tables (indexed)**

- On create/update, HAPI parses the resource and extracts values for **FHIR SearchParameters** into typed index tables, e.g.:
	- `HFJ_SPIDX_STRING` (string params, e.g., `name`)
	- `HFJ_SPIDX_TOKEN` (coded/token params, e.g., `identifier`, `code`)
	- `HFJ_SPIDX_DATE` (dates, e.g., `birthdate`, `onset`)
	- `HFJ_SPIDX_QUANTITY` / `_QUANTITY_NRML` (quantities with UCUM)
	- `HFJ_SPIDX_COORDS` (geo)
	- `HFJ_SPIDX_URI`, `HFJ_SPIDX_NUMBER`, tags, presence, etc.
- These tables hold (resource\_id, param\_name, normalized\_value, …) with proper DB indexes so HAPI can answer the RESTful search API quickly without scanning CLOBs.

**3) Query execution**

- When you call `/Patient?name=smith&birthdate=ge2020-01-01`, HAPI:
	- Translates each criterion to SQL targeting the appropriate index table(s).
	- Intersects the matching `resource_id` sets.
	- Fetches the canonical resources from the CLOB/JSON column for the final bundle.

**4) Search caching/results tables**

- HAPI also keeps **search/session bookkeeping** (e.g., `HFJ_SEARCH`, `HFJ_SEARCH_RESULT`) so paging and repeated requests don’t recompute everything.

---

## Why this design (pros/cons)

**Pros**

- **Standards fidelity:** exact FHIR JSON is preserved.
- **Search speed:** common FHIR search params are **pre-indexed** in relational tables.
- **Operational maturity:** RDBMS gives ACID, backups, familiar ops.

**Cons (for analytics)**

- **Two representations:** truth in the CLOB, **queryability in side tables**. Ad-hoc analytics beyond the predefined SearchParameters gets awkward.
- **Evolving profiles/extensions:** new fields or custom profiles require configuring/extracting additional params or doing JSON-path queries (slower).
- **Complex joins:** cross-resource analytics often devolves into many joins or app-side assembly.
- **Reindexing cost:** changing SearchParameter definitions can trigger expensive reindex cycles.

---

## Implications for FHIR analytics

If you stay within FHIR’s **search parameter model**, HAPI JPA is fine. For broader analytics (cohorts, networks, longitudinal measures across many resource types), teams typically add one or more of:

- **Materialized views / ETL** from the CLOB into analytics tables (star schema for OLAP).
- **JSONB (PostgreSQL) + GIN indexes** to query arbitrary paths (still gets tricky at scale).
- **Search/BI engines**: Elasticsearch/OpenSearch for text & ad-hoc filters; warehouse/OLAP for cubes.
- **Event streams**: capture updates from HAPI, transform to an analytics store.

---

## Where D4M/Hazoo differs

- **Single uniform model**: decompose every resource into `(row, column, value)` **associative arrays**. No separate “index tables” vs “CLOB”—the AA is the queryable form.
- **Exploratory power**: sparse matrix/semiring algebra handles joins, filters, groupings as algebraic ops, well-suited to **cross-resource, graph-like** analytics.
- **Scale**: built on big-data stores (e.g., Accumulo/Hadoop), horizontally scalable.

**Bridge strategy:** keep HAPI for **FHIR-compliant CRUD & search**, and stream resources to D4M/Hazoo for **analytics**. Your current pipeline (D4M → CSV → Blender) is exactly that pattern.

---

## If you stick with HAPI but want more analytics

- **Enable JSONB & GIN (Postgres)**: index frequently used JSON paths for ad-hoc queries.
- **Add ETL to a warehouse**: flatten resources to facts/dimensions for OLAP dashboards.
- **Track extensions**: define search parameters for key extensions you care about, to avoid CLOB scans.
- **Partition & VACUUM discipline**: for large installs, manage bloat and reindex windows proactively.

If you want, I can sketch a minimal **change-feed** (e.g., outbox table or DB logical decoding) to mirror HAPI updates into D4M/Hazoo in near-real-time.