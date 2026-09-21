# Gift Stitch Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align the gift management admin pages and backend aggregate APIs with the Stitch prototype.

**Architecture:** Add aggregate VO/query/service methods beside the existing CRUD layer, keeping existing entity tables and `org_id` isolation. Update the admin pages to consume the aggregate APIs and render the prototype sections.

**Tech Stack:** Spring Boot, MyBatis Plus, Maven, Vue 3, TypeScript, Ant Design Vue.

---

### Task 1: Backend Aggregate Contract Tests
- Add failing tests for new route contracts (`/summary`, `/business-page`, `/profile`).
- Add assertions that `GiftPersonInfoTController` exposes `/summary`, `/business-page`, and `/profile`.

### Task 2: Backend Aggregate API Implementation
- `GiftDashboardSummaryVo.java`
- `GiftAmountTrendVo.java`
- `GiftRankingItemVo.java`
- `GiftRelationDistributionVo.java`
- `GiftPersonBusinessVo.java`

### Task 3: Frontend Alignment
- Aligned dashboard, person, event, record, and analysis pages in `alex_miaosha_front`.
