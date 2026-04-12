# Saraphan — Phase 5: GraphRAG + Advanced Features + Rollout

← กลับไป [[00_MASTER_PLAN]]  
← ก่อนหน้า [[06_PHASE_4_PRODUCTION]]  
→ ใช้ cross-ref data จาก [[04_PHASE_2_THAI#Task 2.5]]  
→ Graph schema ดูที่ [[02_DATABASE_SCHEMA#Neo4j Graph Schema]]  
→ Frontend pages ดูที่ [[08_FRONTEND_PLAN#Phase 5 Pages]]

---

## Phase 5: Advanced Features + Rollout (สัปดาห์ 19-24)

**เป้าหมาย:** GraphRAG สำหรับเอกสารระเบียบ, feature ขั้นสูง, pilot launch, full rollout

### Task 5.1: Feedback Loop + Self-Learning (สัปดาห์ 19-20)
- [ ] thumbs up/down บนทุกคำตอบ
- [ ] Admin review: ดู low-rated answers, แก้ไข golden examples
- [ ] Auto-detect knowledge gaps: queries ที่ตอบไม่ได้ → alert admin ให้เพิ่มเอกสาร
- [ ] Weekly RAGAS evaluation: track quality over time

### Task 5.2: GraphRAG สำหรับเอกสารระเบียบ/คำสั่ง (สัปดาห์ 19-22)

Standard RAG ค้นหาแบบ similarity ทำงานได้ดีกับคำถามตรงๆ แต่พังกับคำถามที่ต้อง:
- ไล่ตาม cross-reference ข้ามเอกสาร ("ตามระเบียบ X ข้อ 5 ที่อ้างถึงระเบียบ Y หมวด 3...")
- สรุปภาพรวมข้ามเอกสารหลายฉบับ ("นโยบายทั้งหมดที่เกี่ยวกับ data privacy มีอะไรบ้าง")
- ติดตามลำดับชั้นอำนาจ ("ใครมีอำนาจอนุมัติเรื่อง X ตามคำสั่งล่าสุด")
- ดูว่าระเบียบฉบับไหนยกเลิก/แก้ไขฉบับไหน

GraphRAG เสริม (ไม่แทนที่) standard RAG โดยเพิ่ม knowledge graph layer เฉพาะเอกสารที่ได้ประโยชน์จริง

**ทำเฉพาะเอกสารระเบียบ/คำสั่ง (~200-500 ฉบับ) ไม่ทำกับคู่มือ/SOP/รายงานการเงิน**
เหตุผล: ระเบียบ/คำสั่งมี cross-reference หนาแน่นที่สุด, คู่มือ/SOP ใช้ standard RAG เพียงพอ

#### Task 5.2.1: Neo4j Setup + Schema Design (สัปดาห์ 19)

```
Knowledge Graph Schema:

Nodes:
├── Regulation (ระเบียบ)
│   properties: doc_id, title, number, issue_date, effective_date, status, classification
├── Order (คำสั่ง)
│   properties: doc_id, title, number, issue_date, issuer, status, classification
├── Section (หมวด/ข้อ)
│   properties: section_number, title, content_summary, parent_doc_id
├── Organization (หน่วยงาน)
│   properties: name, code, level
├── Position (ตำแหน่ง)
│   properties: title, department
└── Topic (หัวข้อ/เรื่อง)
    properties: name, description

Edges:
├── CITES (อ้างอิง) ─── Regulation → Regulation/Order
├── SUPERSEDES (ยกเลิก) ─── Regulation → Regulation
├── AMENDS (แก้ไข) ─── Order → Regulation
├── CONTAINS (ประกอบด้วย) ─── Regulation → Section
├── DELEGATES_TO (มอบอำนาจให้) ─── Regulation → Position/Organization
├── APPLIES_TO (บังคับใช้กับ) ─── Regulation → Organization
├── RELATES_TO (เกี่ยวข้องกับ) ─── Regulation → Topic
└── ISSUED_BY (ออกโดย) ─── Order → Position
```

- [ ] Deploy Neo4j Community Edition (Docker)
- [ ] สร้าง schema ตามด้านบน
- [ ] ทดสอบ Cypher queries พื้นฐาน
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** Neo4j Community Edition (GPLv3, free)
- **ต้องเขียนเอง:** schema setup + test queries ~100 lines

#### Task 5.2.2: Entity & Relation Extraction (สัปดาห์ 19-20)

```
เอกสารระเบียบ/คำสั่ง
    │
    ▼
[Phase 2 cross-reference data] ← ใช้ข้อมูลที่ extract ไว้แล้วจาก Task 2.5
    │
    ▼
[LLM-based entity extraction] ← ใช้ Claude/GPT-4 (ทำครั้งเดียวต่อเอกสาร)
    │   Extract: Organizations, Positions, Topics, Sections
    │   ไม่ใช้ open-source LLM ตัวเล็ก — accuracy ต่ำเกินไปสำหรับภาษาไทย
    │
    ▼
[Merge: regex references + LLM entities]
    │
    ▼
[Load into Neo4j]
```

- [ ] สร้าง entity extraction prompt (ภาษาไทย):
  - ออกแบบ prompt ที่ extract: หน่วยงาน, ตำแหน่ง, หัวข้อ/เรื่อง, หมวด/ข้อ
  - Include few-shot examples จากเอกสาร NT จริง
  - Output format: JSON structured triples
- [ ] Merge กับ cross-reference data จาก Task 2.5:
  - regex references (cites, supersedes, amends) → edges ใน graph
  - LLM entities → nodes ใน graph
- [ ] Entity resolution (deduplication):
  - "ฝ่ายการเงิน" = "ฝ่ายการเงินและบัญชี" = "Finance Department"
  - ใช้ Thai string similarity (PyThaiNLP) + manual mapping table
- [ ] Batch processing: extract ทีละ 500 เอกสาร
  - ประมาณ cost: ~$100-300 สำหรับ 500 เอกสาร (Claude/GPT-4, one-time)
- [ ] Load into Neo4j: nodes + edges + properties
- [ ] Validation: สุ่ม 50 เอกสาร ตรวจว่า entities + relations ถูกต้อง > 90%
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** Claude API / GPT-4 (extraction), py2neo (Neo4j Python driver)
- **ต้องเขียนเอง:** extraction pipeline + entity resolver + loader ~600-800 lines

#### Task 5.2.3: Query Router — Standard RAG vs GraphRAG (สัปดาห์ 20-21)

```
User Query เข้ามา
    │
    ▼
[Query Classifier]
    │
    ├── คำถามตรงๆ (fact lookup)
    │   "ระเบียบการลาพักร้อนมีเงื่อนไขอะไร"
    │   → Standard RAG (Qdrant vector + BM25)
    │
    ├── คำถาม multi-hop / cross-reference
    │   "ระเบียบจัดซื้อฉบับล่าสุดแก้ไขอะไรจากฉบับเดิม"
    │   → GraphRAG: Cypher query → subgraph → LLM
    │
    ├── คำถามภาพรวม (global / summarization)
    │   "นโยบายทั้งหมดที่เกี่ยวกับ cybersecurity มีอะไรบ้าง"
    │   → GraphRAG: Topic node → connected regulations → community summary
    │
    └── คำถาม hybrid
        "ระเบียบจัดซื้อข้อ 5 บอกว่าอะไร และอ้างอิงระเบียบอื่นตรงไหน"
        → GraphRAG (หา relations) + Standard RAG (หา content ข้อ 5)
        → Merge results → LLM
```

- [ ] สร้าง query classifier:
  - Rule-based (keyword patterns): "แก้ไข", "ยกเลิก", "อ้างอิง", "ทั้งหมดที่เกี่ยวกับ",
    "ความเชื่อมโยง", "ผลกระทบ", "ฉบับล่าสุด" → route to GraphRAG
  - Default → Standard RAG (majority of queries)
- [ ] สร้าง GraphRAG retrieval service:
  - Input: user query + user permissions (ACL ยังคงใช้)
  - Step 1: LLM แปลง query → Cypher query (หรือ entity keywords สำหรับ graph search)
  - Step 2: Execute Cypher → retrieve subgraph (nodes + edges + properties)
  - Step 3: Convert subgraph → text context (structured summary)
  - Step 4: ส่ง context + original query → LLM → response
- [ ] Hybrid mode: GraphRAG context + Standard RAG chunks → merge → LLM
- [ ] ACL enforcement: filter graph results ตาม user clearance + department
  - Level 3-4 documents ที่อยู่ใน graph → ซ่อน nodes/edges ถ้า user ไม่มีสิทธิ์
  - ใช้ Qdrant ACL filter เดิมกับ graph node ที่มี doc_id mapping
- **ผู้รับผิดชอบ:** Backend 1 + Backend 2
- **ต้องเขียนเอง:** classifier + graph retrieval + merge logic ~500-700 lines

#### Task 5.2.4: GraphRAG Evaluation + Tuning (สัปดาห์ 21-22)

- [ ] สร้าง GraphRAG-specific golden set: 30 คำถาม multi-hop
  - 10 คำถาม cross-reference ("ระเบียบ X อ้างอิงอะไรบ้าง")
  - 10 คำถาม supersession/amendment ("คำสั่ง X แก้ไขอะไรจากฉบับเดิม")
  - 5 คำถาม global/summary ("นโยบายทั้งหมดเรื่อง X")
  - 5 คำถาม authority chain ("ใครมีอำนาจอนุมัติเรื่อง X ตามลำดับ")
- [ ] เปรียบเทียบ:
  - Standard RAG only vs GraphRAG only vs Hybrid (Standard + Graph)
  - วัด: accuracy, completeness, citation correctness
- [ ] ตัดสินใจ: GraphRAG คุ้มค่าไหม?
  - ถ้า accuracy improvement > 15% บน multi-hop queries → keep + expand
  - ถ้า < 15% → evaluate ว่าปัญหาอยู่ที่ extraction quality หรือ query routing
- [ ] Incremental update pipeline:
  - เมื่อเอกสารระเบียบใหม่ถูกอัปโหลด → auto-extract entities + relations → update graph
  - ไม่ต้อง rebuild graph ทั้งหมด
- **ผู้รับผิดชอบ:** QA + Tech Lead
- **ต้องเขียนเอง:** evaluation script + incremental updater ~300 lines

#### GraphRAG — สิ่งที่ไม่ทำ (ขอบเขตชัดเจน)
- ❌ ไม่ทำ GraphRAG กับคู่มือ/SOP (ไม่มี cross-reference มากพอ)
- ❌ ไม่ทำ GraphRAG กับรายงานการเงิน (ใช้ standard RAG + table extraction เพียงพอ)
- ❌ ไม่ใช้ Microsoft GraphRAG full pipeline (community summarization แพงเกินไปสำหรับ pilot)
  → ใช้ LazyGraphRAG approach แทน (defer summarization to query time)
- ❌ ไม่แทนที่ Qdrant ด้วย Neo4j — ทั้ง 2 ทำงานคู่กัน
- ❌ ไม่ใช้ open-source LLM ตัวเล็กสร้าง knowledge graph — entity extraction ภาษาไทยต้องใช้ LLM คุณภาพสูง

### Task 5.3: Advanced RAG Features (สัปดาห์ 20-22)
- [ ] Multi-turn conversation: จำบริบทสนทนาได้
- [ ] Query decomposition: คำถามซับซ้อน → แตกเป็นหลาย sub-queries
- [ ] Cross-department authorized search: ค้นข้าม department ที่มีสิทธิ์
- [ ] Document versioning: เมื่อระเบียบอัปเดต → archive เก่า, index ใหม่
  (เชื่อมกับ GraphRAG: อัปเดต SUPERSEDES edge อัตโนมัติ)
- [ ] Table extraction + structured Q&A: ถามข้อมูลจากตารางในเอกสารได้

### Task 5.4: Pilot Launch (สัปดาห์ 21-22)
- [ ] เลือก 2-3 ฝ่ายสำหรับ pilot (แนะนำ: ฝ่ายที่มีเอกสารเยอะ + ถามบ่อย)
- [ ] อัปโหลดเอกสารจริงของ pilot departments
- [ ] GraphRAG: โหลดระเบียบ/คำสั่งของ pilot departments เข้า graph
- [ ] Training session สำหรับ pilot users
- [ ] รวบรวม feedback 2 สัปดาห์ — แยกวัด standard RAG vs GraphRAG queries
- [ ] ปรับปรุงจาก feedback

### Task 5.5: Full Rollout (สัปดาห์ 23-24)
- [ ] อัปโหลดเอกสารทุกฝ่าย (อาจ phase เป็นรอบ)
- [ ] GraphRAG: โหลดระเบียบ/คำสั่งที่เหลือทั้งหมดเข้า graph
- [ ] Training session สำหรับทั้งบริษัท
- [ ] Launch announcement
- [ ] Dedicated support channel สัปดาห์แรก
- [ ] Monitor closely: performance, accuracy, user satisfaction

### Milestone 5: Full Production
- [ ] ทุกฝ่ายใช้งานได้
- [ ] RAGAS metrics คง > 0.75 (standard RAG)
- [ ] GraphRAG multi-hop accuracy > 85% บน golden set (30 คำถาม)
- [ ] Query router ส่งไป GraphRAG ถูก > 90% ของ multi-hop queries
- [ ] User satisfaction > 4.0/5.0
- [ ] p95 latency < 5s ที่ full load (standard RAG), < 8s (GraphRAG queries)

---

