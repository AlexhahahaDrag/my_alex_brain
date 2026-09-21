# Gift Stitch Alignment Design

## Goal
Bring the gift management admin module back in line with the Stitch prototype for the five desktop pages: dashboard, person, event, record, and analysis. The implementation must keep the existing user, RBAC, organization, backend, and admin frontend frameworks, and must not add new foundation systems.

## Source Of Truth
1. 数据概览 - 礼尚往来管理
2. 亲友管理 - 礼尚往来管理
3. 事由管理 - 礼尚往来管理
4. 礼金记录 - 礼尚往来管理
5. 统计报表 - 礼尚往来管理

## Backend Design
No table split is required. The existing tables remain the source:
- `gift_person_info_t`
- `gift_event_info_t`
- `gift_record_info_t`
- 关系语义由 `gift_person_info_t.relation_type` + 关系词典表承载。
