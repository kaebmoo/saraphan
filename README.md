# Saraphan (สารพันธ์)

Enterprise RAG Knowledge Base สำหรับองค์กรไทย  
ถามคำถาม ได้คำตอบจากเอกสารของคุณ

> สารพันธ์ = สาร (ความรู้, สาระ) + พันธ์ (เชื่อมโยง, ผูกพัน)

---

## ปัญหาที่แก้

องค์กรขนาดใหญ่มีเอกสารหลายพันฉบับ ระเบียบ คำสั่ง คู่มือปฏิบัติงาน นโยบาย กระจายอยู่ตามแผนก
พนักงานเสียเวลาค้นหาข้อมูลที่ต้องการ หรือถามคนอื่นแล้วได้คำตอบไม่ครบ

Saraphan ให้พนักงานพิมพ์คำถามเป็นภาษาธรรมชาติ ระบบค้นหาจากเอกสารทั้งหมดที่มีสิทธิ์เข้าถึง
แล้วตอบพร้อมอ้างอิงว่าคำตอบมาจากเอกสารไหน หน้าอะไร

## คุณสมบัติหลัก

- ถาม-ตอบภาษาไทยจากเอกสารองค์กร พร้อมอ้างอิงต้นฉบับ
- รองรับ PDF, Word, Excel, เอกสารสแกน (OCR ภาษาไทย)
- ค้นหาแบบ Hybrid: BM25 + Vector + Query Expansion + Re-ranking
- ชั้นความลับเอกสาร 4 ระดับ (เปิดเผย, ใช้ภายใน, ลับ, ลับมาก)
- ลำดับชั้นผู้ใช้ 5 ระดับ (Staff, Manager, Director, Executive, SysAdmin)
- Hybrid Cloud/On-Prem: เอกสารทั่วไปใช้ Cloud LLM ได้ เอกสารลับประมวลผลในองค์กรเท่านั้น
- GraphRAG สำหรับเอกสารระเบียบ/คำสั่ง ไล่ตาม cross-reference ข้ามเอกสารได้
- Audit log ครบทุก interaction สอดคล้อง PDPA

## สถาปัตยกรรม

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ผู้ใช้งาน (Browser)                          │
│                        React + Ant Design                           │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     FastAPI Orchestration Layer                      │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────────────┐  │
│  │   Auth   │  │   ACL    │  │    PII    │  │   LLM Router      │  │
│  │  (JWT/   │  │ (Cerbos) │  │ Detection │  │ Cloud ↔ On-Prem   │  │
│  │   SSO)   │  │          │  │           │  │                   │  │
│  └──────────┘  └──────────┘  └───────────┘  └───────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    RAG Pipeline                               │   │
│  │  Query Expansion → Hybrid Search → Re-ranking → Generation   │   │
│  │                    (Position-Aware Blend)                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────┬──────────────┬──────────────┬──────────────┬─────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────────┐
│  RAGFlow   │ │   Qdrant   │ │   Neo4j    │ │   LLM Providers    │
│  DeepDoc   │ │  (Vector   │ │ (GraphRAG) │ │                    │
│  + OCR     │ │   + BM25   │ │ ระเบียบ/   │ │  Cloud: Claude,    │
│  PaddleOCR │ │   + Thai   │ │ คำสั่ง     │ │  GPT-4o, Gemini    │
│            │ │  tokenizer)│ │            │ │                    │
│            │ │            │ │            │ │  On-Prem: Ollama   │
│            │ │            │ │            │ │  (Llama3/Typhoon)  │
└────────────┘ └────────────┘ └────────────┘ └────────────────────┘
       │
       ▼
┌────────────┐    ┌────────────┐    ┌────────────┐
│ BGE-M3     │    │ PostgreSQL │    │  Langfuse  │
│ Embedding  │    │ (metadata, │    │ (observa-  │
│ (local)    │    │  users,    │    │  bility)   │
│            │    │  audit)    │    │            │
└────────────┘    └────────────┘    └────────────┘
```

## Tech Stack

| Layer | เครื่องมือ | License |
|-------|----------|---------|
| RAG Engine | RAGFlow | Apache 2.0 |
| API | FastAPI (Python) | MIT |
| Frontend | React + Ant Design | MIT |
| Embedding | BAAI/bge-m3 | MIT |
| OCR | PaddleOCR-VL | Apache 2.0 |
| Vector DB | Qdrant | Apache 2.0 |
| Graph DB | Neo4j Community | GPLv3 |
| Thai NLP | PyThaiNLP | Apache 2.0 |
| Auth/ACL | Cerbos | Apache 2.0 |
| Observability | Langfuse | MIT |

## โครงสร้างโปรเจกต์

```
saraphan/
├── backend/                    # FastAPI orchestration service
│   ├── app/
│   │   ├── api/v1/             # REST endpoints
│   │   ├── services/           # RAG pipeline, Thai NLP, OCR
│   │   │   └── graph/          # GraphRAG (Phase 5)
│   │   ├── security/           # Auth, ACL, PII detection, audit
│   │   ├── models/             # SQLAlchemy models
│   │   └── providers/          # LLM provider registry
│   ├── alembic/                # DB migrations
│   └── tests/
│
├── frontend/                   # React + Ant Design
│   ├── src/pages/              # 21 pages (chat, documents, admin, ...)
│   ├── src/components/         # Shared components
│   └── src/services/           # API client
│
├── evaluation/                 # RAGAS, search benchmarks, ACL tests
├── config/                     # Cerbos policies, RAGFlow, Qdrant configs
├── scripts/                    # Seed data, user sync, document import
├── docs/                       # Architecture, API, deployment guides
│
├── docker-compose.yml
├── docker-compose.dev.yml
├── Makefile
└── .env.example
```

## เอกสารแผนพัฒนา

แผนพัฒนาอยู่ใน `docs/plan/` ออกแบบสำหรับใช้กับ Obsidian (wiki-links เชื่อมโยงระหว่างเอกสาร)

| เอกสาร | เนื้อหา |
|--------|--------|
| [00_MASTER_PLAN](docs/plan/00_MASTER_PLAN.md) | ภาพรวม, tech stack, timeline, risks, budget |
| [01_SECURITY_DESIGN](docs/plan/01_SECURITY_DESIGN.md) | User roles, document classification, ACL, PDPA |
| [02_DATABASE_SCHEMA](docs/plan/02_DATABASE_SCHEMA.md) | PostgreSQL + Qdrant + Neo4j schemas |
| [03_PHASE_0_1_CORE](docs/plan/03_PHASE_0_1_CORE.md) | Infrastructure + Core RAG pipeline |
| [04_PHASE_2_THAI](docs/plan/04_PHASE_2_THAI.md) | Thai language optimization, OCR, hybrid search |
| [05_PHASE_3_SECURITY](docs/plan/05_PHASE_3_SECURITY.md) | Security implementation |
| [06_PHASE_4_PRODUCTION](docs/plan/06_PHASE_4_PRODUCTION.md) | Production hardening |
| [07_PHASE_5_GRAPHRAG](docs/plan/07_PHASE_5_GRAPHRAG.md) | GraphRAG + advanced features + rollout |
| [08_FRONTEND_PLAN](docs/plan/08_FRONTEND_PLAN.md) | Frontend spec ครบทุกหน้า |

## ความปลอดภัย

ระบบออกแบบให้รองรับเอกสารที่มีความลับหลายระดับ:

```
ชั้นความลับเอกสาร              ผู้ใช้ที่เข้าถึงได้           LLM Routing
──────────────────────────────────────────────────────────────────────
เปิดเผย (Level 1)              ทุกคน                       Cloud OK
ใช้ภายใน (Level 2)             Manager+ ในฝ่ายที่เกี่ยวข้อง   Cloud + PII scan
ลับ (Level 3)                  Director+ ในฝ่ายที่เกี่ยวข้อง  On-Prem เท่านั้น
ลับมาก (Level 4)               Executive + ผู้ได้รับอนุญาต    On-Prem + Encrypted
```

รายละเอียดเพิ่มเติมดูที่ [01_SECURITY_DESIGN](docs/plan/01_SECURITY_DESIGN.md)

## Requirements

### Development
- Docker + Docker Compose
- Python 3.11+
- Node.js 20+
- GPU (optional, สำหรับ embedding inference เร็วขึ้น)

### Production
- Server 3 เครื่อง (RAGFlow + Qdrant/Embedding + On-Prem LLM)
- GPU 24GB+ สำหรับ On-Prem LLM (A100/A10G/RTX 4090)
- PostgreSQL 16+
- Domain + HTTPS

## Quick Start (Development)

```bash
# Clone
git clone https://github.com/kaebmoo/saraphan.git
cd saraphan

# ตั้งค่า environment
cp .env.example .env
# แก้ค่าใน .env: API keys, database credentials

# รัน services ทั้งหมด
docker compose up -d

# รัน backend (development)
cd backend
pip install -r requirements.txt
python -m uvicorn app.main:app --reload --port 8000

# รัน frontend (development)
cd frontend
npm install
npm run dev
```

## Timeline

| สัปดาห์ | Phase | เป้าหมาย |
|---------|-------|---------|
| 1-2 | Infrastructure | RAGFlow + Qdrant + BGE-M3 + On-Prem LLM พร้อม |
| 3-6 | Core Pipeline | ถาม-ตอบจากเอกสารภาษาไทยได้ + Basic UI |
| 7-10 | Thai Optimization | OCR + hybrid search + query expansion |
| 11-14 | Security | ACL + approval workflow + hybrid routing + audit |
| 15-18 | Production | Load testing + monitoring + admin dashboard |
| 19-24 | Advanced + Rollout | GraphRAG + feedback loop + pilot + full rollout |

## สถานะ

**อยู่ระหว่างการวางแผน** — แผนพัฒนาเสร็จสมบูรณ์แล้ว กำลังเตรียม infrastructure

## License

[MIT](LICENSE)

## เกี่ยวกับชื่อ

**สารพันธ์** มาจาก สาร (ความรู้, สาระ, ข้อมูล) + พันธ์ (เชื่อมโยง, ผูกพัน, ความสัมพันธ์)
หมายถึง ระบบที่เชื่อมโยงความรู้ในองค์กรเข้าด้วยกัน ให้ค้นหาและเข้าถึงได้ง่าย
