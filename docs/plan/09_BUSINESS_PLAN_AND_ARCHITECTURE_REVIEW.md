# Saraphan — Business Plan & Architecture Review

**Date:** 2026-05-18  
**Stage:** Planning  
**Source docs:** `README.md`, `docs/plan/00_MASTER_PLAN.md` ถึง `08_FRONTEND_PLAN.md`

---

## 1. Executive Summary

Saraphan คือระบบ Enterprise RAG Knowledge Base สำหรับองค์กรไทย โดยควรวางตลาดแรกเป็น **Government Knowledge AIaaS** สำหรับหน่วยงานรัฐและรัฐวิสาหกิจ: ให้เจ้าหน้าที่ถามคำถามจากเอกสารราชการ ระเบียบ คำสั่ง มติ คู่มือ SOP รายงาน และเอกสารสแกน โดยตอบเป็นภาษาไทยพร้อม citation, version awareness, GraphRAG สำหรับความเชื่อมโยงของเอกสาร และควบคุมสิทธิ์ตามชั้นความลับ

ข้อเสนอทางธุรกิจที่เหมาะกับระยะนี้คือ **Private/Hybrid AIaaS สำหรับภาครัฐก่อน แล้วค่อยขยายเป็น Enterprise Knowledge Product** ไม่ควรวางเป็น public SaaS ตั้งแต่แรก เพราะ requirement หลักคือข้อมูลลับ, PDPA, on-prem/hybrid deployment, SSO/LDAP, department ACL, audit และเอกสารภาษาไทยคุณภาพหลากหลาย

เป้าหมาย 6 เดือนตามแผนเดิมยังสมเหตุสมผลในเชิง prototype-to-production แต่ควรปรับลำดับสถาปัตยกรรมบางส่วน: security boundary, document governance, evaluation, ingestion pipeline, GraphRAG และ local LLM cost control ควรเป็น foundation ตั้งแต่ MVP ไม่ใช่รอ Phase 3-5 ทั้งหมด เพราะเป็น trust requirement ของภาครัฐ

---

## 2. Business Plan

### 2.1 Problem

องค์กรขนาดใหญ่มีเอกสารจำนวนมาก กระจายตามฝ่ายและรูปแบบไฟล์ พนักงานเสียเวลา:

- ค้นหาเอกสารที่ถูกต้อง
- ตรวจว่าเอกสารเป็นฉบับล่าสุดหรือไม่
- ไล่ cross-reference ระหว่างระเบียบ/คำสั่ง
- ขอคำตอบจากคนกลางแล้วได้คำตอบไม่ครบ
- ระวังชั้นความลับและ PDPA ด้วยตัวเอง

ปัญหานี้ไม่ใช่แค่ search problem แต่เป็น **knowledge access + governance problem**: ต้องตอบได้เร็ว, อ้างอิงได้, จำกัดสิทธิ์ได้ และ audit ได้

### 2.2 Target Users

กลุ่มเริ่มต้นควรเป็นหน่วยงานรัฐ/รัฐวิสาหกิจที่มีเอกสารระเบียบและคำสั่งจำนวนมาก:

- งานสารบรรณ/นิติการ/เลขานุการกรม: ค้นระเบียบ คำสั่ง มติ หนังสือเวียน และฉบับที่ถูกแก้ไข/ยกเลิก
- งานจัดซื้อ/การเงิน/พัสดุ: ค้นข้อกำหนด อำนาจอนุมัติ และเอกสารอ้างอิง
- งาน HR/ฝึกอบรม: ค้นสิทธิ์ ระเบียบพนักงาน คู่มือปฏิบัติงาน
- ผู้บริหาร: ดูภาพรวม policy, authority chain, cross-reference ระหว่างเอกสาร
- IT/DPO/Internal Audit: ดูแล SSO, ACL, audit, retention, PDPA, usage analytics

สำหรับ pilot ควรเลือก 1 หน่วยงานรัฐหรือรัฐวิสาหกิจที่มีเอกสารพร้อม และเลือก 2-3 กอง/ฝ่ายที่คำถามซ้ำสูง เช่น นิติการ, สารบรรณ, พัสดุ/จัดซื้อ, HR

### 2.3 Value Proposition

Saraphan ควรสื่อสารคุณค่าเป็น 4 แกน:

- **ลดเวลาค้นหาเอกสาร:** ถามเป็นภาษาไทยและได้คำตอบพร้อมที่มา
- **เข้าใจความเชื่อมโยงของเอกสารราชการ:** GraphRAG ไล่ความสัมพันธ์ เช่น อ้างอิง, แก้ไข, ยกเลิก, อำนาจอนุมัติ, หน่วยงานที่เกี่ยวข้อง
- **เพิ่มความถูกต้อง:** citation, versioning, no-answer behavior, evaluation golden set
- **ลดความเสี่ยงข้อมูลรั่ว:** ACL pre-filter, hybrid cloud/on-prem routing, PII scan, audit log
- **คุมต้นทุนและ data residency:** local LLM สำหรับงานซ้ำ/ข้อมูลลับ และ cloud LLM เฉพาะงานที่ต้องใช้คุณภาพสูง

### 2.4 Product Strategy

Phase แนะนำด้านธุรกิจ:

| Stage | เป้าหมาย | Outcome |
|-------|----------|---------|
| Gov Discovery | ยืนยัน pain point เอกสารราชการจริง | top 50 questions, document inventory, graph relation types |
| GraphRAG MVP | ถาม-ตอบ + ไล่ cross-reference จากระเบียบ/คำสั่ง | demo ที่แสดง citation, version, relation path |
| Secure Pilot | เพิ่ม SSO/LDAP, ACL, audit, local LLM routing | ใช้งานจริงใน 2-3 กอง/ฝ่ายโดยคุมข้อมูล |
| Production AIaaS | แพ็ก deployment แบบ private cloud/on-prem | subscription + support contract |
| Repeatable Gov Package | template สำหรับหน่วยงานรัฐอื่น | implementation time ลดลง, margin ดีขึ้น |

### 2.5 Revenue / Funding Model

ระยะ NT/internal reference customer:

- วัดผลแบบ cost saving และ risk reduction ไม่ใช่ revenue โดยตรง
- งบหลักคือ infra, GPU, LLM API, ทีมพัฒนา, operation
- Business case ควรผูกกับเวลาที่ลดได้ต่อพนักงานและจำนวนคำถามที่ตอบเองได้

ระยะขายให้หน่วยงานรัฐ:

- **Paid pilot:** 300k-1.5M บาท ต่อ 6-8 สัปดาห์ สำหรับ 1 หน่วยงาน/2-3 กอง
- **Private AIaaS:** 80k-500k บาท/เดือน ตาม users, documents, queries, SLA
- **On-prem enterprise:** setup 2M-10M บาท + annual maintenance 15-25%
- **Professional services:** document onboarding, OCR cleanup, graph schema, taxonomy, SSO/LDAP, ACL, PDPA/DPIA support
- **LLM usage tier:** local LLM included quota สำหรับงานซ้ำ/ข้อมูลลับ; cloud LLM เป็น usage pass-through + margin สำหรับงานยาก

ยังไม่ควรทำ public self-serve SaaS จนกว่าจะมี tenant isolation, billing, data residency, admin automation และ security certification พร้อม

### 2.6 ROI Model

ตัวแปรที่ควรวัดตั้งแต่ pilot:

- จำนวนผู้ใช้ active ต่อเดือน
- จำนวน query ต่อวัน
- เวลาเฉลี่ยที่ลดได้ต่อ query
- % คำถามที่ตอบได้พร้อม citation
- % คำถามที่ระบบตอบว่าไม่พบข้อมูลอย่างถูกต้อง
- จำนวน knowledge gaps ที่ถูกปิด
- จำนวน incident ด้าน access/PII เป็น 0

สูตร ROI ภายในแบบง่าย:

```text
monthly_saving =
  monthly_queries
  x minutes_saved_per_query
  / 60
  x average_staff_cost_per_hour

net_value = monthly_saving - monthly_operating_cost
```

จากเอกสารเดิม operating cost ประมาณ $2,000-5,200/เดือน ไม่รวม hardware one-time ดังนั้น pilot ควรวัดให้ได้ว่า query volume และ time saving เพียงพอหรือไม่ ก่อนขยาย full rollout

### 2.7 Go-To-Market ภาครัฐ

แผน rollout ที่ควรใช้:

1. เริ่มจากหน่วยงานที่มีเอกสารพร้อม เช่น ระเบียบ คำสั่ง มติ หนังสือเวียน และระบบสารบรรณ
2. ขาย paid pilot ด้วย use case ชัด: "ถามระเบียบ", "หาเอกสารฉบับล่าสุด", "เอกสารนี้แก้ไข/ยกเลิกอะไร", "ใครมีอำนาจอนุมัติ"
3. สร้าง golden set 100+ คำถามจากงานจริงก่อน build production flow
4. เปิดใช้เฉพาะเอกสาร Level 1-2 ก่อน หาก Level 3-4 ยังไม่ผ่าน DPIA
5. ใช้ local LLM เป็น default สำหรับข้อมูลภายใน และใช้ cloud LLM เฉพาะ query ยาก/งาน extraction ที่ต้องแม่น
6. ทำ training แบบ workflow-based ให้เจ้าหน้าที่และ admin

### 2.8 Success Metrics

ควรกำหนด milestone เชิงธุรกิจคู่กับ technical milestone:

| ช่วง | Metric ผ่านขั้นต่ำ |
|------|--------------------|
| MVP Demo | ตอบ top 50 questions ได้ accuracy > 80%, citation ถูกต้อง |
| GraphRAG Demo | ตอบ multi-hop/cross-reference ได้ > 80%, แสดง relation path ได้ |
| Thai Quality | RAGAS > 0.75, MRR@10 > 0.85, no-answer ถูก > 90% |
| Secure Pilot | PII/cloud leak = 0, ACL test pass 100%, audit ครบ |
| Production | p95 standard query < 5s, GraphRAG query < 8s, satisfaction > 4/5 |
| Rollout | active users > 40% ของกลุ่มเป้าหมาย pilot, unanswered query trend ลดลง |

---

## 3. Architecture Review

### 3.1 ภาพรวม

สถาปัตยกรรมเดิมเลือกทิศทางถูก: FastAPI orchestration ครอบ RAGFlow, Qdrant, local embedding, cloud/on-prem LLM, PostgreSQL metadata, Cerbos ACL, Langfuse observability และ Neo4j สำหรับ GraphRAG เฉพาะระเบียบ/คำสั่ง

อย่างไรก็ตามใน planning phase ควรปรับจาก "list of components" ให้กลายเป็น "operating architecture" ที่ชัดเรื่อง trust boundary, ownership, data lifecycle, failure mode, evaluation loop, graph lifecycle และ local/cloud model economics

### 3.2 Architecture Improvements ที่ควรทำก่อนเริ่ม

#### A. ย้าย Security Boundary เข้ามาใน MVP

แผนเดิมให้ Phase 1 ไม่มี access control แล้วค่อยทำ Phase 3 security แต่สำหรับ enterprise RAG ความเสี่ยงคือทีมจะ build retrieval/generation path ที่ไม่ได้คิด ACL ตั้งแต่ต้น แล้วต้อง refactor ใหญ่ภายหลัง

ข้อเสนอ:

- ตั้ง auth mock ได้ แต่ contract ต้องมี `user_id`, `department_id`, `role`, `clearance_level` ตั้งแต่วันแรก
- ทุก search API ต้องรับ permission context และ metadata filter ตั้งแต่ MVP
- เอกสารทุกไฟล์ต้องมี classification ตั้งแต่ upload แม้ยังไม่ enforce เต็ม
- audit event schema ควรถูกสร้างตั้งแต่ Phase 1 แม้ log บาง field ยังเป็น mock

#### B. แยก Document Ingestion Pipeline ออกจาก Chat API

Ingestion เป็นงานหนักและล้มเหลวง่าย: OCR, chunking, embedding, indexing, cross-reference extraction, graph extraction, approval, reprocess

ข้อเสนอ:

- เพิ่ม background worker + queue ตั้งแต่แรก เช่น Celery/RQ/Arq + Redis
- สร้าง state machine ของ document processing ให้ชัด: uploaded, pending_approval, extracting, chunking, embedding, graph_extracting, indexed, ready, failed, archived
- แยก object storage สำหรับ source file ออกจาก metadata DB และ vector DB
- ทุกขั้นตอนต้อง idempotent เพื่อ reprocess ได้

#### C. กำหนด Source of Truth ให้ชัดระหว่าง RAGFlow, PostgreSQL, Qdrant

เอกสารปัจจุบันใช้ RAGFlow ingestion และมี PostgreSQL documents table พร้อม Qdrant collections อาจเกิด metadata drift ได้

ข้อเสนอ:

- PostgreSQL เป็น source of truth สำหรับ document metadata, permissions, lifecycle, versions
- Object storage เป็น source of truth สำหรับ original files
- Qdrant เป็น derived index ลบ/สร้างใหม่ได้
- Neo4j เป็น derived graph index ลบ/สร้างใหม่ได้
- RAGFlow ใช้เป็น ingestion/chunking engine หรือ retrieval backend แต่ไม่ควรเป็น authority ด้าน permission/lifecycle

#### D. เพิ่ม Model Gateway / Policy Gateway

LLM routing เป็น core differentiator ของระบบนี้ จึงควรเป็น service boundary ที่ชัด ไม่ใช่ helper function กระจายใน code

ข้อเสนอ:

- สร้าง `ModelGateway` สำหรับ provider registry, retry, timeout, fallback, token/cost tracking, local-first routing
- สร้าง `PolicyGateway` สำหรับ Cerbos, permission resolver, PII decision, response redaction
- decision ทุกครั้งต้อง log เป็น structured audit: provider, model, routing_reason, max_classification, pii_detected

#### E. Evaluation-First Development

เอกสารมี RAGAS ใน Phase 2 แต่ควรเริ่ม golden set ก่อน implementation หลัก เพราะ RAG system ไม่มี correctness ชัดแบบ CRUD

ข้อเสนอ:

- เริ่ม golden set 50 คำถามตั้งแต่ Phase 0/1
- แยกชุดทดสอบเป็น retrieval, generation, citation, no-answer, ACL leakage, PII routing
- ตั้ง quality gate ใน CI สำหรับ prompt/search config changes
- เก็บ failed queries จาก pilot เข้า feedback loop อย่างเป็นระบบ

#### F. Data Governance / PDPA ต้องมี Artifact แยก

Security doc ดีแล้ว แต่ยังควรมี governance artifact ที่ใช้คุยกับ legal/DPO:

- data inventory: ข้อมูลอะไรอยู่ที่ไหน
- data flow diagram: อะไรออก cloud ได้/ไม่ได้
- retention schedule: source, vectors, logs, backups
- data subject request workflow: ค้นหา/ลบ/anonymize PII ใน source + vectors
- provider DPA checklist สำหรับ cloud LLM

#### G. ทำ GraphRAG เป็น Core แต่จำกัดโดเมน

GraphRAG คือ differentiator หลักสำหรับภาครัฐ เพราะเอกสารราชการมีความสัมพันธ์แบบอ้างอิง แก้ไข ยกเลิก มอบอำนาจ และใช้บังคับกับหลายหน่วยงาน แต่ไม่ควรทำ graph กับเอกสารทุกชนิด

ข้อเสนอ:

- ทำ GraphRAG เฉพาะระเบียบ คำสั่ง มติ หนังสือเวียน และเอกสาร policy ที่มี cross-reference
- ทำ cross-reference extraction ตั้งแต่ MVP เพราะเป็นฐานของ graph และช่วย retrieval ทันที
- ใช้ standard RAG กับ SOP, คู่มือ, FAQ, รายงานทั่วไป เพื่อคุมต้นทุนและ complexity
- GraphExplorer ใน frontend ยัง optional; สิ่งจำเป็นคือ response ต้องแสดง relation path และ citation ได้

#### H. ใช้ Local LLM เป็น Cost/Control Layer

Local LLM ไม่ควรใช้แทน cloud LLM ทุกงาน แต่ควรเป็น default สำหรับงานที่ซ้ำและข้อมูลอ่อนไหว

ข้อเสนอ:

- Local LLM: query expansion, classification, PII-sensitive answering, draft summary, routine Q&A
- Cloud LLM: graph/entity extraction รอบแรก, query ยาก, answer synthesis ที่ต้องคุณภาพสูง
- Cache aggressive: query expansion, embedding, graph retrieval result, prompt prefix
- เก็บ cost ต่อ tenant/หน่วยงาน/กอง เพื่อใช้ทำ pricing และ renewal

#### I. เพิ่ม Deployment Architecture 3 ระดับ

ควรกำหนด target deployment profile ตั้งแต่ planning:

| Profile | ใช้สำหรับ | ลักษณะ |
|---------|-----------|--------|
| Dev | ทีมพัฒนา | docker compose single-node, mock auth |
| Pilot | 2-3 ฝ่าย | real auth, ACL, audit, backup, single/limited HA |
| Production | ทั้งองค์กร | HA Qdrant/Postgres/API, DR, monitoring, runbook |

#### J. Frontend ควรสะท้อน Trust มากกว่า Feature จำนวนมาก

Frontend plan มี 21 หน้า ซึ่งดีในภาพเต็ม แต่สำหรับ pilot ควรลดเหลือ workflow ที่สร้าง trust:

- Chat พร้อม citation และ no-answer behavior
- Relation path สำหรับ GraphRAG เช่น เอกสาร A อ้างถึง/แก้ไข/ยกเลิกเอกสาร B
- Document upload พร้อม classification
- Document browser/preview ตามสิทธิ์
- Approval queue
- Audit/analytics แบบ minimum สำหรับ admin

หน้าอื่น เช่น GraphExplorer, advanced analytics, full SystemConfig ควรเลื่อนได้

---

## 4. Revised Delivery Plan

### Phase A: Gov Planning Validation (2 สัปดาห์)

- Document inventory ของหน่วยงาน/กองนำร่อง
- เลือก pilot use cases และ top 100 questions รวม multi-hop/cross-reference
- นิยาม graph relation types ขั้นต่ำ: cites, amends, supersedes, delegates_to, applies_to, issued_by
- DPIA draft และ data flow diagram
- ตัดสินใจ auth, infra, GPU/local LLM, cloud LLM provider policy
- ADR สำหรับ RAGFlow/Qdrant/Postgres source-of-truth

### Phase B: Secure GraphRAG MVP (4-6 สัปดาห์)

- FastAPI orchestration + model gateway
- Upload/ingestion worker แบบ async
- PostgreSQL metadata + object storage + Qdrant derived index
- Mock/real auth contract + classification + basic ACL filter
- Cross-reference extraction + Neo4j graph index สำหรับระเบียบ/คำสั่งชุด pilot
- Chat UI + citation + document preview
- GraphRAG answer พร้อม relation path
- Golden set 50-100 questions

### Phase C: Thai Quality + Governance (4 สัปดาห์)

- Thai segmentation/custom dictionary
- OCR pipeline + quality threshold
- Hybrid search, reranker, query expansion
- Local LLM routing + cache เพื่อลดต้นทุนต่อ query
- Audit middleware, PII routing, privacy notice
- RAGAS/quality CI + GraphRAG multi-hop evaluation

### Phase D: Secure Pilot (4 สัปดาห์)

- SSO/LDAP integration
- Approval workflow
- Cross-department grants
- Admin minimum dashboard
- Load test pilot scale
- Pilot training + feedback review

### Phase E: Production AIaaS Package (6-8 สัปดาห์)

- HA/backup/restore/runbook
- Langfuse dashboards
- Tenant isolation, usage metering, billing report
- GraphRAG production hardening
- Full rollout by department/agency waves

---

## 5. Key Decisions Needed

ต้องตัดสินใจก่อนเริ่ม coding จริง:

1. หน่วยงานรัฐ/รัฐวิสาหกิจนำร่อง และเอกสารจริงชุดแรก
2. Graph relation types ขั้นต่ำที่ต้องรองรับใน pilot
3. จะใช้ auth แบบใด: AD/LDAP, OIDC/SSO หรือ JWT ชั่วคราว
4. ข้อมูล Level 2 ส่ง cloud LLM ได้จริงหรือไม่ และต้องมี DPA provider ไหน
5. มี GPU/local LLM พร้อมหรือไม่ และ target cost/query เท่าไร
6. RAGFlow จะเป็น ingestion engine เท่านั้น หรือเป็น retrieval authority ด้วย
7. Object storage จะใช้ MinIO, NAS, S3-compatible หรือ storage ภายในองค์กร
8. ใครเป็น business owner ของ classification และ encryption keys สำหรับ Level 3-4

---

## 6. Recommended Next Actions

1. ทำ workshop 1 วันกับหน่วยงานนำร่องเพื่อเก็บ top questions, document inventory, pain points และ relation examples
2. เขียน ADR 5 ฉบับ: data source of truth, auth strategy, local/cloud LLM routing, deployment profile, graph schema
3. ปรับ master plan ให้ Phase 1 เป็น Secure GraphRAG MVP แทน MVP ที่ไม่มี ACL
4. สร้าง `docs/architecture/` สำหรับ data flow, deployment, ingestion lifecycle, graph lifecycle, security boundary
5. เริ่ม golden set ก่อนเริ่ม build pipeline โดยแยก standard RAG และ GraphRAG multi-hop
