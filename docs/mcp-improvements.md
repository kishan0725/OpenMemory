# OpenMemory MCP Improvements

> High-level action items for enhancing the MCP (Model Context Protocol) implementation

---

## 🔴 Critical Priority

### 1. PgVector Post-Filtering Performance Issue
**Problem:** PgVector HNSW doesn't support pre-filtering - searches all vectors then filters by user_id post-hoc  
**Impact:** Inefficient multi-tenant queries - searches 100x more vectors than needed, slower as data grows  
**Action:** Implement table partitioning by user_id so HNSW index searches only user's partition  
**Files:** `packages/openmemory-js/src/core/db.ts`, `packages/openmemory-js/src/core/vector/pgvector.ts`  
**Effort:** 4 hours  
**Note:** PgVector limitation - WHERE clause filtering happens after HNSW neighbor search completes

### 2. User Level Isolation for Temporal Facts
**Problem:** Temporal facts are not filtered by user_id, causing data leakage between users  
**Impact:** Security vulnerability - users can see other users' facts in multi-tenant scenarios  
**Action:** Add `user_id` column to `temporal_facts` table and filter all queries by user  
**Files:** `packages/openmemory-js/src/core/db.ts`, `packages/openmemory-js/src/temporal_graph/query.ts`, `packages/openmemory-js/src/ai/mcp.ts`  
**Effort:** 3 hours  
**Status:** 🟡 In Progress

### 3. Temporal Facts Pagination
**Problem:** No limit on temporal fact queries - can return thousands of results causing timeouts  
**Impact:** MCP timeouts, memory exhaustion with large datasets  
**Action:** Add `limit` and `offset` parameters to all temporal query functions  
**Files:** `packages/openmemory-js/src/temporal_graph/query.ts`, `packages/openmemory-js/src/ai/mcp.ts`  
**Effort:** 30 minutes

### 4. Missing Object Index
**Problem:** Queries filtering by fact `object` field perform full table scans  
**Impact:** Severe performance degradation on large fact tables  
**Action:** Add database index: `CREATE INDEX temporal_facts_object_idx ON temporal_facts(object)`  
**Files:** `packages/openmemory-js/src/core/db.ts`  
**Effort:** 5 minutes

### 5. Excessive Database Calls in HSG Query
**Problem:** Query loop makes 40+ sequential DB calls (4 per result for 10 results)  
**Impact:** High latency on every search operation  
**Action:** Batch fetch all memories and vectors in 2 queries instead of N*4  
**Files:** `packages/openmemory-js/src/memory/hsg.ts` (line ~680)  
**Effort:** 2 hours

---

## 🟡 High Priority

### 6. Batch Operations Support
**Problem:** All MCP operations are single-item only (store/delete/reinforce one at a time)  
**Impact:** Inefficient bulk imports - 100 memories require 100 sequential MCP calls  
**Action:** Add `openmemory_store_batch`, `openmemory_delete_batch` tools  
**Files:** `packages/openmemory-js/src/ai/mcp.ts`  
**Effort:** 2 hours  
**Status:** 🟡 Planned

### 7. No Temporal Fact Caching
**Problem:** Every temporal query hits database directly, no caching layer  
**Impact:** Unnecessary database load for repeated queries  
**Action:** Implement TTL cache (60s) similar to HSG contextual cache  
**Files:** `packages/openmemory-js/src/temporal_graph/query.ts`  
**Effort:** 1 hour

### 8. No Metadata Filtering
**Problem:** Cannot filter queries by tags, date ranges, or custom metadata fields  
**Impact:** Limited query expressiveness, post-fetch filtering is inefficient  
**Action:** Add `tags`, `startTime`, `endTime`, `metadata` filters to `openmemory_query` tool  
**Files:** `packages/openmemory-js/src/ai/mcp.ts`, `packages/openmemory-js/src/memory/hsg.ts`  
**Effort:** 3 hours

---

## 🟠 Medium Priority

### 9. No Temporal Fact Update Tool
**Problem:** Cannot modify fact confidence, metadata, or manually close facts via MCP  
**Impact:** Temporal facts are immutable once created  
**Action:** Add `openmemory_update_fact` tool with confidence/metadata updates  
**Files:** `packages/openmemory-js/src/ai/mcp.ts`, `packages/openmemory-js/src/temporal_graph/store.ts`  
**Effort:** 1.5 hours

### 10. Minimal Error Context
**Problem:** Errors return only message string, no codes/traces/suggestions  
**Impact:** Difficult to debug MCP failures  
**Action:** Enhance error responses with error codes, stack traces, actionable suggestions  
**Files:** `packages/openmemory-js/src/ai/mcp_tools.ts`  
**Effort:** 1 hour

### 11. Date Format Inconsistency
**Problem:** Mix of Date objects, timestamps, and ISO strings across codebase  
**Impact:** Potential timezone bugs in point-in-time queries  
**Action:** Standardize on ISO 8601 strings everywhere, document timezone handling  
**Files:** Multiple files across temporal graph and MCP layers  
**Effort:** 3 hours

---

## Key Files Reference

```
MCP Tools:              packages/openmemory-js/src/ai/mcp.ts
Tool Registry:          packages/openmemory-js/src/ai/mcp_tools.ts
Temporal Queries:       packages/openmemory-js/src/temporal_graph/query.ts
Temporal Storage:       packages/openmemory-js/src/temporal_graph/store.ts
HSG Query Logic:        packages/openmemory-js/src/memory/hsg.ts
Database Schema:        packages/openmemory-js/src/core/db.ts
PgVector Store:         packages/openmemory-js/src/core/vector/pgvector.ts
Vector Store Interface: packages/openmemory-js/src/core/vector_store.ts
```

---

## Performance Impact Estimates

| Improvement | Current | Target | Gain |
|-------------|---------|--------|------|
| Object-based fact queries | 500ms (full scan) | 5ms (indexed) | **100x faster** |
| HSG query with 10 results | 400ms (40 queries) | 50ms (2 queries) | **8x faster** |
| Repeated temporal queries | Direct DB hit | Cache hit | **~100x faster** |
| Bulk import 100 memories | 100 MCP calls | 1 MCP call | **100x fewer round-trips** |

---

## Implementation Checklist

### 🔴 Critical Priority
- [ ] **#1** - Implement table partitioning by user_id for PgVector
- [ ] **#2** - Add user_id column to temporal_facts and filter all queries
- [ ] **#3** - Add limit/offset pagination to temporal fact queries
- [ ] **#4** - Create object index on temporal_facts table
- [ ] **#5** - Optimize HSG query with batch DB fetching (internal)

### 🟡 High Priority
- [ ] **#6** - Add batch MCP tools for bulk operations (API-level)
- [ ] **#7** - Implement temporal fact query caching with 60s TTL
- [ ] **#8** - Add metadata filtering (tags, dates) to openmemory_query

### 🟠 Medium Priority
- [ ] **#9** - Create openmemory_update_fact tool for confidence/metadata updates
- [ ] **#10** - Enhance error responses with codes, traces, and suggestions
- [ ] **#11** - Standardize date handling to ISO 8601 strings

---
