# Saraphan (สารพันธ์) — Master Plan

**GitHub:** `kaebmoo/saraphan`  
**Date:** 2026-04-12  
**Project:** Saraphan — ระบบฐานความรู้องค์กร NT  
**Approach:** Hybrid Build — RAGFlow engine + Custom FastAPI orchestration  
**Timeline:** 24 สัปดาห์ (6 เดือน) ถึง Production  
**Team:** 5-6 คน  
**Effort:** 14-20 developer-months

> สารพันธ์ = สาร (ความรู้, สาระ) + พันธ์ (เชื่อมโยง, ผูกพัน)  
> Enterprise RAG knowledge base for Thai organizations.

---

## เอกสารทั้งหมด

| เอกสาร | เนื้อหา |
|--------|--------|
| **[[00_MASTER_PLAN]]** | (เอกสารนี้) ภาพรวม, tech stack, ทีม, timeline, risks, budget |
| **[[01_SECURITY_DESIGN]]** | ลำดับชั้นผู้ใช้ 5 ระดับ, ชั้นความลับ 4 ระดับ, ACL matrix, PDPA, PII, audit, encryption |
| **[[02_DATABASE_SCHEMA]]** | PostgreSQL schema ทั้งหมด (11 tables), Qdrant collections, Neo4j graph schema |
| **[[03_PHASE_0_1_CORE]]** | Phase 0: Infrastructure + Phase 1: Core RAG Pipeline + Basic UI |
| **[[04_PHASE_2_THAI]]** | Phase 2: Thai language optimization, OCR, hybrid search, query expansion, cross-ref extraction |
| **[[05_PHASE_3_SECURITY]]** | Phase 3: Security implementation — ACL, approval workflow, hybrid routing, audit |
| **[[06_PHASE_4_PRODUCTION]]** | Phase 4: Load testing, observability, admin dashboard, HA |
| **[[07_PHASE_5_GRAPHRAG]]** | Phase 5: GraphRAG, feedback loop, advanced RAG, pilot + rollout |
| **[[08_FRONTEND_PLAN]]** | Frontend spec ครบทุกหน้า — user UI + admin UI + components |

---

## สรุปภาพรวมโปรเจกต์

ระบบนี้แยกจาก NT AI Assistant (ที่ทำอยู่) โดยสิ้นเชิง  
NT AI Assistant = ถาม-ตอบข้อมูลการเงินจาก SQL database  
Saraphan = ถาม-ตอบจากเอกสารองค์กร (ระเบียบ, คำสั่ง, SOP, รายงาน)

**ผู้ใช้เป้าหมาย:** พนักงาน NT ทุกฝ่าย (~1,000+ คน)  
**ประเภทเอกสาร:** PDF/Word (ระเบียบ, นโยบาย, คำสั่ง), คู่มือปฏิบัติงาน, เอกสารสแกน, รายงานการเงิน  
**นโยบายข้อมูล:** Hybrid — เอกสารทั่วไปใช้ Cloud API ได้, เอกสารลับต้อง On-Premise

---

## Dependency Map

```
Phase 0 ────────────────────────────── (สัปดาห์ 1-2)
  Infrastructure + RAGFlow Deploy          → [[03_PHASE_0_1_CORE]]
       │
       ▼
Phase 1 ────────────────────────────── (สัปดาห์ 3-6)
  Core RAG Pipeline + Basic UI             → [[03_PHASE_0_1_CORE]]
       │
       ├──→ Phase 2: Thai Language Optimization (สัปดาห์ 7-10)
       │         │                                → [[04_PHASE_2_THAI]]
       │         └──→ OCR + Word Segmentation + Query Expansion
       │              + Cross-reference extraction (เตรียมสำหรับ GraphRAG)
       │
       └──→ Phase 3: Security & Access Control (สัปดาห์ 11-14)
                 │                                → [[05_PHASE_3_SECURITY]]
                 ├──→ Department Isolation         ← ใช้ [[01_SECURITY_DESIGN]]
                 ├──→ Hybrid Cloud/On-Prem Routing
                 └──→ Audit Logging + PDPA
                          │
                          ▼
                 Phase 4: Production Hardening (สัปดาห์ 15-18)
                          │                       → [[06_PHASE_4_PRODUCTION]]
                          ├──→ Load Testing + Caching
                          ├──→ Observability (Langfuse)
                          └──→ Admin Dashboard     ← ใช้ [[08_FRONTEND_PLAN]]
                                   │
                                   ▼
                          Phase 5: Advanced Features (สัปดาห์ 19-24)
                                   │              → [[07_PHASE_5_GRAPHRAG]]
                                   ├──→ GraphRAG สำหรับเอกสารระเบียบ/คำสั่ง
                                   ├──→ Feedback Loop + Self-Learning
                                   ├──→ Cross-Department Search
                                   └──→ Pilot + Full Rollout
```

**เอกสารประกอบที่ใช้ตลอดทุก Phase:**
- [[01_SECURITY_DESIGN]] — อ้างอิงเรื่อง user roles, classification, ACL rules
- [[02_DATABASE_SCHEMA]] — อ้างอิง table structure
- [[08_FRONTEND_PLAN]] — อ้างอิง UI spec สำหรับ frontend developer

---

## Technology Stack

| Layer | เลือกใช้ | เหตุผลหลัก |
|-------|---------|-----------|
| RAG Engine | RAGFlow (self-hosted, Apache 2.0) | DeepDoc + PaddleOCR Thai, chunking template สำหรับกฎหมาย/ระเบียบ |
| Orchestration API | FastAPI (Python) | สร้างใหม่เป็น service แยก, จัดการ auth/routing/audit |
| Frontend | React + Ant Design | สร้างใหม่ → ดู [[08_FRONTEND_PLAN]] |
| Embedding (local) | BAAI/bge-m3 | #1 บน Thai retrieval benchmark ทุกตัว, MIT license |
| Embedding (cloud) | Google gemini-embedding-001 | MTEB multilingual สูงสุด (ห้ามใช้ text-embedding-004 มี bug กับภาษาไทย) |
| OCR | PaddleOCR-VL (via RAGFlow DeepDoc) | Apache 2.0, รองรับ 109 ภาษารวมไทย |
| OCR เสริม | ThaiTrOCR | CER ต่ำสุดสำหรับภาษาไทย ใช้เป็น fallback |
| Vector DB | Qdrant (Hybrid Cloud) | vector DB เดียวที่มี native Thai tokenizer, RBAC, hybrid deploy |
| Thai NLP | PyThaiNLP (newmm/nlpo3) + TLTK | word segmentation, sentence boundary, custom dictionary |
| LLM (sensitive) | Llama 3 / Typhoon Thai (on-prem Ollama) | ข้อมูลไม่ออกนอก NT network |
| LLM (general) | Claude / GPT-4o / Gemini | คุณภาพการตอบสูงสุด |
| Observability | Langfuse (self-hosted) | PDPA compliant, audit trail ครบ |
| Auth/ACL | Cerbos (open-source) + NT's existing auth | policy-based access control → ดู [[01_SECURITY_DESIGN]] |
| Graph DB (Phase 5) | Neo4j Community Edition (GPLv3) | GraphRAG → ดู [[07_PHASE_5_GRAPHRAG]] |
| GraphRAG (Phase 5) | LazyGraphRAG approach | entity extraction + query-time summarization |

---

## ทีมที่ต้องการ

| บทบาท | จำนวน | ความรับผิดชอบหลัก |
|--------|--------|------------------|
| Tech Lead / Architect | 1 | ออกแบบระบบ, ตัดสินใจ technical, review code |
| Backend Developer | 2 | FastAPI orchestration, RAGFlow integration, Thai NLP pipeline |
| Frontend Developer | 1 | React UI ตาม [[08_FRONTEND_PLAN]] |
| DevOps / Infra | 1 | Docker/K8s, Qdrant cluster, monitoring, CI/CD |
| QA / Evaluation | 0.5-1 | RAGAS evaluation, Thai accuracy testing, load testing |

---

## Hardware / Infrastructure Requirements

### Development Environment
- Server 1 (RAGFlow + Dependencies): 8 vCPU, 32GB RAM, 200GB SSD
- Server 2 (Vector DB + Embedding): 8 vCPU, 32GB RAM, 100GB SSD, GPU optional
- Server 3 (On-Prem LLM): 16 vCPU, 64GB RAM, GPU 24GB+ (A100/A10G/RTX 4090)

### Production Environment (เพิ่มเติม)
- Load Balancer (Nginx/Traefik)
- PostgreSQL → ดู [[02_DATABASE_SCHEMA]]
- Redis (caching, session, rate limiting)
- Langfuse (self-hosted, Docker)
- Neo4j Community (Phase 5) → ดู [[07_PHASE_5_GRAPHRAG]]
- Backup storage (S3-compatible หรือ NAS)

---

## Project Structure

```
saraphan/
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env.example
├── README.md
├── Makefile
│
├── backend/                        # FastAPI orchestration
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── api/v1/                 # REST endpoints
│   │   │   ├── chat.py, documents.py, auth.py, admin.py
│   │   │   ├── access.py, audit.py, health.py
│   │   ├── services/               # Business logic
│   │   │   ├── rag_service.py, ragflow_client.py, embedding_service.py
│   │   │   ├── vector_service.py, llm_router.py, ocr_service.py
│   │   │   ├── thai_nlp.py, approval_service.py, reference_extractor.py
│   │   │   └── graph/             # GraphRAG (Phase 5)
│   │   │       ├── neo4j_client.py, entity_extractor.py, entity_resolver.py
│   │   │       ├── graph_retriever.py, query_router.py, graph_updater.py
│   │   ├── security/               # Auth + ACL + PII → ดู [[01_SECURITY_DESIGN]]
│   │   │   ├── auth.py, permissions.py, acl.py, pii_detector.py
│   │   │   ├── query_classifier.py, response_filter.py
│   │   │   ├── session_manager.py, encryption.py, audit.py
│   │   ├── models/                 # SQLAlchemy → ดู [[02_DATABASE_SCHEMA]]
│   │   ├── providers/              # LLM providers (pattern จาก OpenMiniCrew)
│   │   └── utils/
│   ├── alembic/
│   ├── tests/
│   ├── Dockerfile
│   └── requirements.txt
│
├── frontend/                       # React + Ant Design → ดู [[08_FRONTEND_PLAN]]
│   ├── src/pages/
│   ├── src/components/
│   ├── src/services/
│   ├── Dockerfile
│   └── package.json
│
├── evaluation/                     # Quality assurance
│   ├── golden_set/
│   ├── graphrag_test/
│   ├── security_test/
│   └── run_*.py
│
├── config/                         # Cerbos policies, RAGFlow, Qdrant, Neo4j, Langfuse configs
├── scripts/                        # seed_departments, sync_users, import_documents, build_knowledge_graph
└── docs/                           # ARCHITECTURE, API, DEPLOYMENT, SECURITY, PDPA, USER_GUIDE, ADMIN_GUIDE
```

---

## Timeline Summary

| สัปดาห์ | Phase | Deliverable หลัก | เอกสาร |
|---------|-------|-----------------|--------|
| 1-2 | Phase 0: Infrastructure | RAGFlow + Qdrant + BGE-M3 + On-Prem LLM ทำงานได้ | [[03_PHASE_0_1_CORE]] |
| 3-6 | Phase 1: Core Pipeline | End-to-end Q&A + Basic UI, accuracy > 80% | [[03_PHASE_0_1_CORE]] |
| 7-10 | Phase 2: Thai Optimization | Thai OCR + hybrid search + query expansion + cross-ref extraction | [[04_PHASE_2_THAI]] |
| 11-14 | Phase 3: Security | ACL 5 ระดับ + approval workflow + hybrid routing + audit | [[05_PHASE_3_SECURITY]] |
| 15-18 | Phase 4: Production | Load test, monitoring, admin dashboard, HA | [[06_PHASE_4_PRODUCTION]] |
| 19-22 | Phase 5a: GraphRAG + Pilot | Knowledge graph ระเบียบ/คำสั่ง, feedback loop, pilot 2-3 departments | [[07_PHASE_5_GRAPHRAG]] |
| 23-24 | Phase 5b: Full Rollout | ทุกฝ่ายใช้งาน, support, monitoring | [[07_PHASE_5_GRAPHRAG]] |

---

## ตารางสรุป: เขียนเอง vs เครื่องมือสำเร็จรูป

| Component | เขียนเอง | เครื่องมือสำเร็จรูป | หมายเหตุ |
|-----------|---------|-------------------|---------|
| Document ingestion & chunking | ไม่ | RAGFlow DeepDoc | ปรับ config |
| OCR | orchestration | PaddleOCR-VL + ThaiTrOCR | post-processing |
| Thai word segmentation | pipeline | PyThaiNLP + TLTK | custom dictionary |
| Embedding | wrapper | BGE-M3 (HuggingFace TEI) | Docker deploy |
| Vector search + query expansion | hybrid search + expansion | Qdrant | BM25 + vector + RRF + position-aware blend |
| Re-ranking | integration | BGE-reranker-v2-m3 | ~100 lines |
| Cross-reference extraction | ทั้งหมด | regex + PyThaiNLP | Phase 2, prep for GraphRAG |
| LLM routing | ทั้งหมด | — | reuse pattern จาก OpenMiniCrew |
| PII detection | ทั้งหมด | — | regex + rules |
| Access control | integration | Cerbos | policy + filter builder |
| Authentication | integration | NT's existing auth | ขึ้นกับระบบ NT |
| Audit logging | ทั้งหมด | — | middleware + PostgreSQL |
| Observability | instrumentation | Langfuse | ~200 lines |
| Frontend ทั้งหมด | ทั้งหมด | — | React + Ant Design → [[08_FRONTEND_PLAN]] |
| Evaluation | golden set | RAGAS | framework สำเร็จรูป |
| Database | schema + models | PostgreSQL + Alembic | → [[02_DATABASE_SCHEMA]] |
| Graph DB (Phase 5) | config | Neo4j Community | Docker deploy |
| GraphRAG pipeline (Phase 5) | ทั้งหมด | Claude/GPT-4 API | entity extraction + query routing |

**สรุป: ~55% ใช้เครื่องมือสำเร็จรูป, ~45% เขียนเอง**

---

## Risk Register

| Risk | ผลกระทบ | ความน่าจะเป็น | แผนรับมือ |
|------|---------|-------------|---------|
| OCR ภาษาไทยไม่แม่นพอ | สูง | ปานกลาง | Fallback: Google Document AI → [[04_PHASE_2_THAI]] |
| On-Prem LLM คุณภาพภาษาไทยต่ำ | สูง | ปานกลาง | ทดสอบ Typhoon Thai, Qwen 2.5 Thai → [[03_PHASE_0_1_CORE]] |
| NT auth ซับซ้อนกว่าคาด | ปานกลาง | สูง | เริ่ม JWT ก่อน → [[05_PHASE_3_SECURITY]] |
| เอกสารเก่าคุณภาพต่ำ (สแกนเบลอ) | ปานกลาง | สูง | quality threshold + admin review |
| RAGFlow breaking change | ปานกลาง | ต่ำ | Pin version |
| User adoption ต่ำ | สูง | ปานกลาง | Pilot ก่อน → [[07_PHASE_5_GRAPHRAG]] |
| GPU server ไม่พอ | สูง | ปานกลาง | CPU-only mode หรือ cloud GPU |
| PDPA compliance issues | สูง | ต่ำ | DPIA ก่อน Phase 3 → [[01_SECURITY_DESIGN]] |
| GraphRAG entity extraction ไม่แม่นพอ | ปานกลาง | ปานกลาง | ใช้ Claude/GPT-4 เท่านั้น → [[07_PHASE_5_GRAPHRAG]] |
| GraphRAG ไม่คุ้มค่า (< 15% improvement) | ต่ำ | ปานกลาง | ใช้ cross-ref boost แทน → [[04_PHASE_2_THAI#Task 2.5]] |

---

## Budget Estimation

| รายการ | ค่าใช้จ่ายต่อเดือน | หมายเหตุ |
|--------|-----------------|---------|
| Cloud LLM API (Claude/GPT-4o) | ~$500-2,000 | เฉลี่ย ~1,000 queries/day |
| Cloud Embedding API (Gemini) | ~$50-200 | เฉพาะ document ใหม่ |
| Server (3 machines) | ขึ้นกับ NT infra | cloud ~$1,500-3,000/mo |
| GPU Server (on-prem LLM) | ขึ้นกับ NT infra | ซื้อ GPU ~$10,000-15,000 one-time |
| GraphRAG extraction (one-time) | ~$100-300 | 500 เอกสาร ครั้งเดียว |
| GraphRAG incremental | ~$20-50/mo | เอกสารระเบียบใหม่ |
| Software licenses | $0 | ทั้งหมด open-source |
| **รวมต่อเดือน** | **$2,000-5,200** | ไม่รวม hardware one-time |

---

## สิ่งที่ต้องตัดสินใจก่อนเริ่ม

1. **Server:** จัดหา on-premise หรือใช้ cloud? → กระทบ [[03_PHASE_0_1_CORE]]
2. **NT Auth:** ระบบ auth ปัจจุบันใช้อะไร? → กระทบ [[05_PHASE_3_SECURITY]]
3. **Pilot departments:** เลือกฝ่ายไหน? → กระทบ [[07_PHASE_5_GRAPHRAG]]
4. **GPU:** มี GPU server พร้อมใช้ไหม? → กระทบ on-prem LLM quality
5. **Team:** มีทีม dev พร้อมกี่คน? → timeline อาจยืด
6. **เอกสาร:** มี digital archive หรือต้องสแกนใหม่? → กระทบ [[04_PHASE_2_THAI]]
