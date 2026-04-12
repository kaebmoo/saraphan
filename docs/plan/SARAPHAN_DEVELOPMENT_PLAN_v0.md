# Saraphan (สารพันธ์) — Enterprise RAG Knowledge Base

**GitHub:** `kaebmoo/saraphan`  
**Date:** 2026-03-29  
**Project:** Saraphan — ระบบฐานความรู้องค์กร NT  
**Approach:** Hybrid Build — RAGFlow engine + Custom FastAPI orchestration  
**Timeline:** 24 สัปดาห์ (6 เดือน) ถึง Production  
**Team:** 5-6 คน  
**Effort:** 14-20 developer-months

> สารพันธ์ = สาร (ความรู้, สาระ) + พันธ์ (เชื่อมโยง, ผูกพัน)  
> Enterprise RAG knowledge base for Thai organizations. Ask questions, get answers from your documents.

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
  Infrastructure + RAGFlow Deploy
       │
       ▼
Phase 1 ────────────────────────────── (สัปดาห์ 3-6)
  Core RAG Pipeline + Basic UI
       │
       ├──→ Phase 2: Thai Language Optimization (สัปดาห์ 7-10)
       │         │
       │         └──→ OCR Pipeline + Word Segmentation + Chunking
       │
       └──→ Phase 3: Security & Access Control (สัปดาห์ 11-14)
                 │
                 ├──→ Department Isolation (Bridge Model)
                 ├──→ Hybrid Cloud/On-Prem Routing
                 └──→ Audit Logging + PDPA Compliance
                          │
                          ▼
                 Phase 4: Production Hardening (สัปดาห์ 15-18)
                          │
                          ├──→ Load Testing + Caching
                          ├──→ Observability (Langfuse)
                          └──→ Admin Dashboard
                                   │
                                   ▼
                          Phase 5: Advanced Features (สัปดาห์ 19-24)
                                   │
                                   ├──→ Agentic RAG
                                   ├──→ Feedback Loop + Self-Learning
                                   └──→ Cross-Department Search
```

---

## Technology Stack (ตัดสินใจแล้ว)

| Layer | เลือกใช้ | เหตุผลหลัก |
|-------|---------|-----------|
| RAG Engine | RAGFlow (self-hosted, Apache 2.0) | DeepDoc + PaddleOCR Thai, chunking template สำหรับกฎหมาย/ระเบียบ |
| Orchestration API | FastAPI (Python) | สร้างใหม่เป็น service แยก, จัดการ auth/routing/audit |
| Frontend | React + Ant Design | สร้างใหม่, สอดคล้องกับ tech stack ที่ NT คุ้นเคย |
| Embedding (local) | BAAI/bge-m3 | #1 บน Thai retrieval benchmark ทุกตัว, MIT license |
| Embedding (cloud) | Google gemini-embedding-001 | MTEB multilingual สูงสุด (ห้ามใช้ text-embedding-004 มี bug กับภาษาไทย) |
| OCR | PaddleOCR-VL (via RAGFlow DeepDoc) | Apache 2.0, รองรับ 109 ภาษารวมไทย |
| OCR เสริม | ThaiTrOCR | CER ต่ำสุดสำหรับภาษาไทย ใช้เป็น fallback |
| Vector DB | Qdrant (Hybrid Cloud) | vector DB เดียวที่มี native Thai tokenizer, RBAC, hybrid deploy |
| Thai NLP | PyThaiNLP (newmm/nlpo3) + TLTK | word segmentation, sentence boundary, custom dictionary |
| LLM (sensitive) | Llama 3 / Typhoon Thai (on-prem Ollama) | ข้อมูลไม่ออกนอก NT network |
| LLM (general) | Claude / GPT-4o / Gemini | คุณภาพการตอบสูงสุด |
| Observability | Langfuse (self-hosted) | PDPA compliant, audit trail ครบ |
| Auth/ACL | Cerbos (open-source) + NT's existing auth | policy-based access control, pre-filtering |

---

## ทีมที่ต้องการ

| บทบาท | จำนวน | ความรับผิดชอบหลัก |
|--------|--------|------------------|
| Tech Lead / Architect | 1 | ออกแบบระบบ, ตัดสินใจ technical, review code |
| Backend Developer | 2 | FastAPI orchestration, RAGFlow integration, Thai NLP pipeline |
| Frontend Developer | 1 | React UI, chat interface, admin dashboard |
| DevOps / Infra | 1 | Docker/K8s, Qdrant cluster, monitoring, CI/CD |
| QA / Evaluation | 0.5-1 | RAGAS evaluation, Thai accuracy testing, load testing |

**หมายเหตุ:** Pornthep ทำหน้าที่ Tech Lead / Architect ได้ โดยใช้ Claude Code ช่วย implement ในส่วน backend

---

## Security, Privacy & Access Control Design

ส่วนนี้เป็น design document ที่ Phase 3 จะ implement ตาม  
ออกแบบให้สอดคล้องกับโครงสร้างองค์กร NT จริง และข้อกำหนด PDPA

### 1. ประเภทผู้ใช้ (User Roles & Clearance Hierarchy)

ระบบแบ่งผู้ใช้เป็น **5 ระดับ** ตามลำดับชั้นในองค์กร แต่ละระดับมี clearance level ที่กำหนดว่าเข้าถึงเอกสารได้ถึงชั้นความลับไหน

```
Clearance Level 5 ── System Admin (ผู้ดูแลระบบ)
        │              ไม่ได้ "อ่าน" เอกสารลับ แต่จัดการระบบได้ทั้งหมด
        │              ตั้งค่า, backup, monitor, จัดการ user/document
        │
Clearance Level 4 ── Executive (ผู้บริหารระดับสูง)
        │              กรรมการผู้จัดการ, รองกรรมการผู้จัดการ, ผู้ช่วย กจ.
        │              เข้าถึงเอกสาร: ทุกระดับ รวม "ลับมาก"
        │              เห็นเอกสารข้ามทุกฝ่าย
        │
Clearance Level 3 ── Director / VP (ผู้อำนวยการฝ่าย / AVP)
        │              เข้าถึงเอกสาร: ถึง "ลับ"
        │              เห็นเอกสารฝ่ายตัวเอง + เอกสารทั่วไป
        │              ขอเข้าถึงเอกสารฝ่ายอื่นได้ (ต้องอนุมัติ)
        │
Clearance Level 2 ── Manager (ผู้จัดการ / หัวหน้าส่วน)
        │              เข้าถึงเอกสาร: ถึง "ใช้ภายใน"
        │              เห็นเอกสารส่วนงานตัวเอง + เอกสารทั่วไป
        │              อัปโหลดเอกสารได้ (classification ≤ 2)
        │
Clearance Level 1 ── Staff (พนักงานทั่วไป)
                       เข้าถึงเอกสาร: เฉพาะ "เปิดเผย"
                       เห็นเฉพาะเอกสารทั่วไป + เอกสารที่ได้รับอนุญาตเพิ่มเติม
                       อ่านอย่างเดียว ไม่อัปโหลด
```

**หลักสำคัญ: Clearance สืบทอดลงล่าง**  
ผู้บริหาร (Level 4) เห็นทุกอย่างที่ Director (Level 3) เห็น  
Director เห็นทุกอย่างที่ Manager (Level 2) เห็น  
Manager เห็นทุกอย่างที่ Staff (Level 1) เห็น  

**Grant เฉพาะราย (Explicit Grant):**  
นอกเหนือจาก clearance level ปกติ admin สามารถ grant สิทธิ์เพิ่มเติมให้ user คนใดคนหนึ่งเห็นเอกสารเฉพาะเรื่องได้ เช่น พนักงานฝ่ายบัญชีที่ต้องดูเอกสารฝ่ายจัดซื้อเพื่อทำ audit

### 2. ชั้นความลับเอกสาร (Document Classification)

เอกสารทุกชิ้นในระบบต้องถูกจัดชั้นความลับ 1 ใน 4 ระดับ:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Level │ ชั้นความลับ    │ ใครเข้าถึง         │ LLM Routing  │ ตัวอย่างเอกสาร │
├───────┼───────────────┼───────────────────┼─────────────┼───────────────────┤
│   1   │ เปิดเผย       │ ทุกคน             │ Cloud OK     │ ระเบียบทั่วไป,   │
│       │ (Public)      │ (Staff ขึ้นไป)    │              │ คู่มือปฏิบัติงาน, │
│       │               │                   │              │ ประกาศ, SOP      │
│       │               │                   │              │ ทั่วไป           │
├───────┼───────────────┼───────────────────┼─────────────┼───────────────────┤
│   2   │ ใช้ภายใน      │ Manager ขึ้นไป    │ Cloud OK     │ แผนงานประจำปี,   │
│       │ (Internal)    │ ในฝ่ายที่เกี่ยวข้อง│ (PII scan)  │ รายงานผลดำเนิน   │
│       │               │ + ผู้ได้รับอนุญาต  │              │ งาน, คู่มือเฉพาะ │
│       │               │                   │              │ ทาง              │
├───────┼───────────────┼───────────────────┼─────────────┼───────────────────┤
│   3   │ ลับ           │ Director ขึ้นไป   │ On-Prem     │ ข้อมูลการเงิน    │
│       │ (Confidential)│ ในฝ่ายที่เกี่ยวข้อง│ เท่านั้น    │ ละเอียด, สัญญา,  │
│       │               │ + ผู้ได้รับอนุญาต  │              │ ข้อมูลบุคลากร,   │
│       │               │ เฉพาะราย          │              │ ผลประเมิน, คดี   │
├───────┼───────────────┼───────────────────┼─────────────┼───────────────────┤
│   4   │ ลับมาก        │ Executive เท่านั้น │ On-Prem     │ แผนกลยุทธ์,     │
│       │ (Secret)      │ + ผู้ได้รับอนุญาต  │ เท่านั้น    │ M&A, ข้อมูลเงิน  │
│       │               │ จาก Executive     │ + Encrypted │ เดือนผู้บริหาร,   │
│       │               │                   │ collection  │ ข้อมูลสอบสวน     │
└─────────────────────────────────────────────────────────────────────────────┘
```

**กฎการจัดชั้นความลับ:**
- ทุกเอกสารที่อัปโหลดต้องระบุชั้นความลับ (ไม่มี default เป็น public เพื่อป้องกันความผิดพลาด)
- ผู้อัปโหลดเสนอชั้นความลับ → ผู้มีสิทธิ์อนุมัติตาม classification level
- เอกสาร Level 3-4 ต้องได้รับอนุมัติจาก Director/Executive ก่อนเข้าระบบ
- เอกสาร Level 1-2 ผู้อัปโหลดระดับ Manager ขึ้นไปจัดชั้นได้เอง
- ชั้นความลับเปลี่ยนแปลงได้ (ยกระดับ/ลดระดับ) โดยผู้มีอำนาจ ทุกการเปลี่ยนแปลงบันทึก audit log

### 3. Access Control Matrix (ใครทำอะไรได้)

```
┌─────────────────────────────┬───────┬─────────┬──────────┬──────────┬───────────┐
│ สิทธิ์                       │ Staff │ Manager │ Director │Executive │ SysAdmin  │
├─────────────────────────────┼───────┼─────────┼──────────┼──────────┼───────────┤
│ ถามคำถาม (chat)              │  ✅   │   ✅    │    ✅    │    ✅    │    ✅     │
│ ดูเอกสาร Level 1 (เปิดเผย)   │  ✅   │   ✅    │    ✅    │    ✅    │    -      │
│ ดูเอกสาร Level 2 (ใช้ภายใน)  │  -    │   ✅*   │    ✅    │    ✅    │    -      │
│ ดูเอกสาร Level 3 (ลับ)       │  -    │   -     │    ✅*   │    ✅    │    -      │
│ ดูเอกสาร Level 4 (ลับมาก)    │  -    │   -     │    -     │    ✅*   │    -      │
│ อัปโหลดเอกสาร Level 1-2     │  -    │   ✅    │    ✅    │    ✅    │    -      │
│ อัปโหลดเอกสาร Level 3-4     │  -    │   -     │    ✅    │    ✅    │    -      │
│ อนุมัติเอกสาร Level 3       │  -    │   -     │    ✅    │    ✅    │    -      │
│ อนุมัติเอกสาร Level 4       │  -    │   -     │    -     │    ✅    │    -      │
│ ลบเอกสาร                    │  -    │   -     │ ฝ่ายตัวเอง│    ✅    │    -      │
│ จัดการ user/role             │  -    │   -     │    -     │    -     │    ✅     │
│ จัดการ department/collection │  -    │   -     │    -     │    -     │    ✅     │
│ ดู audit log                │  -    │   -     │ ฝ่ายตัวเอง│    ✅    │    ✅     │
│ ดู usage analytics          │  -    │   -     │ ฝ่ายตัวเอง│    ✅    │    ✅     │
│ ตั้งค่าระบบ                  │  -    │   -     │    -     │    -     │    ✅     │
│ Grant สิทธิ์เพิ่มเติม        │  -    │   -     │ ฝ่ายตัวเอง│   ทุกฝ่าย │    ✅     │
├─────────────────────────────┴───────┴─────────┴──────────┴──────────┴───────────┤
│ ✅* = เฉพาะฝ่ายตัวเอง + เอกสารที่ได้รับ grant เพิ่ม                               │
│ SysAdmin ไม่เข้าถึงเนื้อหาเอกสารลับ แต่จัดการ infrastructure ได้                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 4. Department Scope & Cross-Department Access

```
NT Organization (ตัวอย่าง)
├── สำนักกรรมการผู้จัดการ (CEO Office) ── Level 4 default
├── สายงานการเงิน (Finance)
│   ├── ฝ่ายบัญชี (Accounting)
│   ├── ฝ่ายการเงิน (Treasury)
│   └── ฝ่ายงบประมาณ (Budget)
├── สายงานเทคโนโลยี (Technology)
│   ├── ฝ่ายโครงข่าย (Network)
│   ├── ฝ่ายไอที (IT)
│   └── ฝ่ายพัฒนาระบบ (Development)
├── สายงานทรัพยากรบุคคล (HR)
│   ├── ฝ่ายบุคคล (Personnel)
│   └── ฝ่ายพัฒนาบุคลากร (Training)
├── สายงานกฎหมาย (Legal)
├── ...
```

**กฎการมองเห็นเอกสารข้ามฝ่าย:**

1. **ภายในสายงานเดียวกัน** (เช่น ฝ่ายบัญชี กับ ฝ่ายการเงิน อยู่ใต้สายงานการเงินเหมือนกัน)
   - Manager/Director: เห็นเอกสาร Level 1-2 ของฝ่ายอื่นในสายงานเดียวกัน
   - Level 3+ ต้อง grant เฉพาะราย

2. **ข้ามสายงาน** (เช่น ฝ่ายบัญชี กับ ฝ่ายบุคคล)
   - ไม่เห็นเอกสารของกันเลย (นอกจาก Level 1)
   - ต้อง grant เฉพาะรายเท่านั้น
   - Grant ข้ามสายงานต้องอนุมัติจาก Director ขึ้นไปของทั้งสองฝ่าย

3. **เอกสารร่วม (Shared Documents)**
   - เอกสารบางชิ้นเกี่ยวข้องหลายฝ่าย (เช่น ระเบียบจัดซื้อ → การเงิน + จัดซื้อ)
   - ตั้ง `allowed_departments` หลายฝ่ายได้
   - ใช้ JSONB array ใน metadata

### 5. Document Lifecycle & Approval Workflow

```
[ผู้อัปโหลดเลือกไฟล์ + ระบุชั้นความลับ + ระบุฝ่ายเจ้าของ]
        │
        ├── Level 1-2 + ผู้อัปโหลดเป็น Manager+
        │   → เข้าระบบทันที (status: processing → ready)
        │
        ├── Level 3
        │   → status: pending_approval
        │   → แจ้ง Director ของฝ่ายให้อนุมัติ
        │   → Director อนุมัติ → processing → ready
        │   → Director ปฏิเสธ → rejected (พร้อมเหตุผล)
        │
        └── Level 4
            → status: pending_approval
            → แจ้ง Executive ให้อนุมัติ
            → Executive อนุมัติ → processing → ready
            → Executive ปฏิเสธ → rejected (พร้อมเหตุผล)

ทุกขั้นตอนบันทึก audit log: ใครอัปโหลด, ใครอนุมัติ/ปฏิเสธ, เมื่อไร
```

**Document Versioning:**
- เมื่อระเบียบ/คำสั่งมีฉบับใหม่ → archive ฉบับเก่า (ยังค้นหาได้ แต่ระบุว่า "ยกเลิก/แทนที่แล้ว")
- ลิงก์ระหว่าง version: เอกสารใหม่ชี้ไปเอกสารเก่า และกลับกัน
- ค้นหา default แสดงฉบับล่าสุด, มี option ค้นรวม archived

**Document Retention & Deletion:**
- เอกสารทั่วไป: ไม่มี expiry (เก็บจน admin ลบ)
- เอกสาร Level 3-4: ตั้ง review date ได้ (เช่น ทบทวนทุก 1 ปีว่ายังต้อง classify สูงอยู่ไหม)
- การลบ: soft delete ก่อน (30 วัน) → hard delete (ลบ vectors + file + metadata)
- การลบเอกสาร Level 3-4 ต้องอนุมัติจาก Director/Executive
- ทุกการลบบันทึก audit log ถาวร (แม้เอกสารถูกลบแล้ว)

### 6. Data Handling Rules ตามชั้นความลับ

```
┌──────────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ กฎการจัดการ            │ Level 1      │ Level 2      │ Level 3      │ Level 4      │
│                      │ เปิดเผย       │ ใช้ภายใน      │ ลับ          │ ลับมาก       │
├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ ส่ง Cloud LLM ได้     │ ✅           │ ✅ + PII scan│ ❌           │ ❌           │
│ เก็บ Vector          │ shared       │ shared       │ dept silo    │ encrypted    │
│                      │ collection   │ collection   │ collection   │ silo         │
│ Embedding            │ Cloud or     │ Cloud or     │ On-Prem      │ On-Prem      │
│                      │ Local        │ Local        │ เท่านั้น      │ เท่านั้น      │
│ แสดง source text     │ ✅ เต็ม      │ ✅ เต็ม      │ ✅ snippet   │ ❌ reference │
│ ในคำตอบ              │              │              │ สั้นเท่านั้น   │ only         │
│ Copy/Export คำตอบ     │ ✅           │ ✅           │ ❌           │ ❌           │
│ ปรากฏใน analytics    │ ✅ ชื่อเอกสาร │ ✅ ชื่อเอกสาร │ ❌ anonymized│ ❌ anonymized│
│ Backup encryption    │ Standard     │ Standard     │ AES-256      │ AES-256      │
│                      │              │              │              │ + separate key│
│ Admin ดูเนื้อหาได้    │ ✅           │ ✅           │ ❌           │ ❌           │
└──────────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

**หมายเหตุสำคัญ:** SysAdmin จัดการ infrastructure ได้ (restart, backup, config) แต่ไม่ควรเข้าถึงเนื้อหาเอกสาร Level 3-4 ตรงๆ ระบบแยก encryption key ให้ business owner (Director/Executive) ถือ ไม่ใช่ IT admin

### 7. LLM Routing ตามชั้นความลับ (Decision Tree)

```
User Query เข้ามา
    │
    ▼
[Step 1: ตรวจสอบ user clearance level]
    │
    ▼
[Step 2: Retrieve documents ที่ user มีสิทธิ์ (pre-filter)]
    │
    ▼
[Step 3: ตรวจสอบ classification สูงสุดของ retrieved documents]
    │
    ├── retrieved docs ทั้งหมดเป็น Level 1-2
    │   │
    │   ▼
    │   [Step 4a: PII scan บน retrieved context]
    │   │
    │   ├── ไม่พบ PII → ส่ง Cloud LLM (Claude/GPT-4o)
    │   └── พบ PII → ส่ง On-Prem LLM (Ollama)
    │
    ├── retrieved docs มี Level 3 อยู่ด้วย
    │   → ส่ง On-Prem LLM เท่านั้น
    │   → Response ไม่แสดง source text เต็ม (snippet เท่านั้น)
    │
    └── retrieved docs มี Level 4 อยู่ด้วย
        → ส่ง On-Prem LLM เท่านั้น
        → Response แสดงเฉพาะ reference (ชื่อเอกสาร + หน้า)
        → ไม่แสดงเนื้อหาต้นฉบับ ต้องไปเปิดดูต้นฉบับเอง
```

### 8. PII ที่ต้องตรวจจับ (สำหรับ PDPA compliance)

ก่อนส่ง context ไป Cloud LLM ต้อง scan และ mask PII ต่อไปนี้:

| ประเภท PII | Pattern / ตัวอย่าง | Action |
|-----------|-------------------|--------|
| เลขบัตรประชาชน | 13 หลัก (X-XXXX-XXXXX-XX-X) | Mask → [NATIONAL_ID] |
| เลขหนังสือเดินทาง | AA1234567 | Mask → [PASSPORT] |
| เบอร์โทรศัพท์ | 0X-XXX-XXXX, 0XX-XXX-XXXX | Mask → [PHONE] |
| Email | xxx@xxx.com | Mask → [EMAIL] |
| เลขบัญชีธนาคาร | 10-12 หลัก | Mask → [BANK_ACCOUNT] |
| เลขบัตรเครดิต | 16 หลัก | Mask → [CREDIT_CARD] |
| เงินเดือน/ค่าตอบแทน | "เงินเดือน xxx บาท" | Mask → [SALARY] |
| ที่อยู่บ้าน | ที่อยู่ส่วนตัว | Mask → [ADDRESS] |
| ข้อมูลสุขภาพ | ใบรับรองแพทย์, วันลาป่วย | Route to On-Prem |
| ชื่อ-นามสกุล (ในบริบท HR) | รายชื่อพนักงาน + ข้อมูลส่วนตัว | Route to On-Prem |

**กฎ:** ถ้าพบ PII แม้แต่ 1 รายการใน retrieved context → route ทั้ง query ไป On-Prem LLM ไม่ mask แล้วส่ง Cloud เพราะ context อาจ leak ข้อมูลได้ทางอ้อม

### 9. Audit Log Requirements (ตาม PDPA)

ทุก action ต่อไปนี้ต้องบันทึกแบบ immutable (append-only, ห้ามแก้ไข/ลบ):

**User Actions:**
- Login / logout (+ IP, device info)
- ถามคำถาม (query text, retrieved doc IDs + scores, response summary, LLM used, routing decision)
- เอกสารที่ถูก filter ออกเพราะ ACL (บันทึกว่า doc ไหนถูกซ่อน เพื่อ audit ว่า ACL ทำงานถูก)
- ให้ feedback (thumbs up/down)
- Copy/export response

**Document Actions:**
- อัปโหลด (ใคร, ไฟล์อะไร, classification อะไร)
- อนุมัติ/ปฏิเสธ (ใครอนุมัติ, เมื่อไร)
- เปลี่ยน classification (จาก → เป็น, ใครเปลี่ยน)
- ลบ (soft/hard, ใครลบ, เหตุผล)
- Re-process / re-index

**Admin Actions:**
- เปลี่ยนสิทธิ์ user (ใครเปลี่ยน, user ไหน, จาก → เป็น)
- Grant/revoke สิทธิ์เอกสาร
- เปลี่ยน system config (LLM settings, routing rules)
- เข้าถึง audit log (บันทึกว่า admin คนไหนดู log เรื่องอะไร)

**Retention:** เก็บ audit log ขั้นต่ำ 2 ปี (ตาม PDPA) แยก partition ตามปี เพื่อ archive ได้ง่าย

### 10. Session Security

- JWT token expiry: 8 ชั่วโมง (ตาม working hours)
- Refresh token: 30 วัน (ถ้าใช้ SSO ตาม SSO policy)
- Concurrent session: จำกัด 3 devices ต่อ user
- Rate limiting per user: 60 queries/hour (ป้องกัน data scraping)
- Rate limiting per IP: 120 queries/hour
- Failed login lockout: 5 ครั้ง → lock 15 นาที
- Session terminate: เมื่อ user ถูก deactivate ให้ revoke ทุก session ทันที

### 11. Encryption Requirements

| ส่วนของข้อมูล | At Rest | In Transit |
|-------------|---------|------------|
| PostgreSQL (metadata, audit) | AES-256 (TDE หรือ LUKS) | TLS 1.3 |
| Qdrant vectors (Level 1-2) | Disk encryption | TLS 1.3 |
| Qdrant vectors (Level 3-4) | AES-256 per-collection key | TLS 1.3 + mTLS |
| Document files (Level 1-2) | Disk encryption | TLS 1.3 |
| Document files (Level 3-4) | AES-256 per-file | TLS 1.3 |
| Backup | AES-256 | TLS 1.3 |
| API keys / credentials | HashiCorp Vault or SOPS | N/A |
| LLM API calls (Cloud) | N/A | TLS 1.3 |
| LLM calls (On-Prem) | N/A | TLS 1.3 (internal) |

### 12. PDPA Compliance Checklist

- [ ] **สิทธิ์เจ้าของข้อมูล (Data Subject Rights):**
  - ถ้ามี PII ของพนักงานในเอกสาร → ต้องสามารถค้นหาได้ว่า PII อยู่ในเอกสารไหนบ้าง
  - ถ้าพนักงานขอลบข้อมูลส่วนตัว → ต้องสามารถลบ/anonymize PII จาก vectors + source ได้
  - ถ้าพนักงานขอสำเนาข้อมูล → ต้องสามารถ export ข้อมูลที่เกี่ยวข้องกับเขาได้
- [ ] **วัตถุประสงค์การใช้ (Purpose Limitation):**
  - ระบุชัดว่าระบบใช้เพื่อ "ค้นหาและตอบคำถามจากเอกสารองค์กร" เท่านั้น
  - ไม่ใช้ข้อมูลในระบบเพื่อวัตถุประสงค์อื่น (เช่น ไม่เอา query log ไปประเมินผลงานพนักงาน)
- [ ] **Data Protection Impact Assessment (DPIA):**
  - ทำ DPIA ก่อนเริ่ม Phase 3 เพราะระบบประมวลผลข้อมูลส่วนบุคคลในปริมาณมาก
  - ส่ง DPO ของ NT ตรวจสอบ
- [ ] **Cross-border data transfer:**
  - เอกสาร Level 1-2 ที่ส่ง Cloud LLM → ข้อมูลออกนอกประเทศ → ต้องมี safeguard ตาม PDPA Section 28-29
  - ตรวจสอบว่า Cloud LLM providers มี DPA (Data Processing Agreement) กับ NT
  - เอกสาร Level 3-4 → ไม่ส่งออกนอกประเทศ (on-prem เท่านั้น)
- [ ] **Consent & Notice:**
  - แจ้ง privacy notice ให้พนักงานทราบเมื่อใช้ระบบครั้งแรก
  - ระบุว่า query และ response จะถูกบันทึก audit log
  - ระบุว่าข้อมูลบางส่วนอาจถูกส่งไป Cloud LLM (เฉพาะ Level 1-2)

---

## Hardware / Infrastructure Requirements

### Development Environment
- Server 1 (RAGFlow + Dependencies): 8 vCPU, 32GB RAM, 200GB SSD
  - RAGFlow (Docker Compose: Elasticsearch + Redis + MinIO + RAGFlow API)
- Server 2 (Vector DB + Embedding): 8 vCPU, 32GB RAM, 100GB SSD, GPU optional
  - Qdrant (Docker)
  - BGE-M3 embedding service (ต้องการ GPU 2GB+ สำหรับ inference เร็ว หรือ CPU ถ้ารับ latency ได้)
- Server 3 (On-Prem LLM): 16 vCPU, 64GB RAM, GPU 24GB+ (A100/A10G/RTX 4090)
  - Ollama + Llama 3 70B หรือ Typhoon Thai

### Production Environment (เพิ่มเติมจาก Dev)
- Load Balancer (Nginx/Traefik)
- PostgreSQL (metadata, user management, audit logs)
- Redis (caching, session, rate limiting)
- Langfuse (self-hosted, Docker)
- Backup storage (S3-compatible หรือ NAS)

---

## Phase 0: Infrastructure + RAGFlow Deploy (สัปดาห์ 1-2)

**เป้าหมาย:** ระบบพื้นฐานพร้อมรัน, RAGFlow ทำงานได้, Qdrant พร้อม, ทีม dev สามารถ develop ได้

### Task 0.1: Server Provisioning (Day 1-2)
- [ ] จัดเตรียม server ตาม spec ด้านบน (ขอ IT จัดหา หรือใช้ cloud)
- [ ] ติดตั้ง Docker + Docker Compose บนทุก server
- [ ] ตั้ง private network ระหว่าง server (VLAN หรือ VPN)
- [ ] ตั้ง domain/subdomain สำหรับ dev environment (เช่น kb-dev.nt.co.th)
- **ผู้รับผิดชอบ:** DevOps
- **เครื่องมือ:** Docker, Docker Compose, Ansible (optional)
- **ต้องเขียนเอง:** ไม่ (infrastructure setup)

### Task 0.2: RAGFlow Deployment (Day 2-3)
- [ ] Clone RAGFlow repo, ปรับ docker-compose.yml
- [ ] Deploy RAGFlow (Elasticsearch + Redis + MinIO + API server)
- [ ] ทดสอบ RAGFlow admin UI เข้าถึงได้
- [ ] อัปโหลดเอกสารทดสอบ 10-20 ไฟล์ (ภาษาไทย PDF/Word)
- [ ] ทดสอบ chunking + retrieval พื้นฐาน
- **ผู้รับผิดชอบ:** DevOps + Backend 1
- **เครื่องมือ:** RAGFlow Docker image (สำเร็จรูป)
- **ต้องเขียนเอง:** แก้ config เท่านั้น

### Task 0.3: Qdrant Deployment (Day 3-4)
- [ ] Deploy Qdrant (Docker, single node สำหรับ dev)
- [ ] สร้าง collection ทดสอบ, ตั้ง vector dimension = 1024 (BGE-M3)
- [ ] ทดสอบ CRUD operations
- [ ] ทดสอบ metadata filtering
- **ผู้รับผิดชอบ:** DevOps
- **เครื่องมือ:** Qdrant Docker image (สำเร็จรูป)
- **ต้องเขียนเอง:** ไม่

### Task 0.4: BGE-M3 Embedding Service (Day 4-5)
- [ ] Deploy BGE-M3 ผ่าน sentence-transformers หรือ TEI (Text Embeddings Inference by HuggingFace)
- [ ] สร้าง FastAPI wrapper endpoint: POST /embed → vector
- [ ] ทดสอบกับข้อความภาษาไทย 100 ประโยค
- [ ] Benchmark: latency per query, throughput
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** HuggingFace TEI (สำเร็จรูป Docker) หรือ sentence-transformers (Python)
- **ต้องเขียนเอง:** FastAPI wrapper ~50 lines

### Task 0.5: Project Scaffold (Day 5-7)
- [ ] สร้าง Git repo: `saraphan` (github.com/kaebmoo/saraphan)
- [ ] โครงสร้างไฟล์เริ่มต้น (ดู Section "Project Structure" ด้านล่าง)
- [ ] ตั้ง .env.example, Docker Compose สำหรับ dev
- [ ] ตั้ง CI pipeline พื้นฐาน (lint + test)
- [ ] เขียน README.md
- **ผู้รับผิดชอบ:** Tech Lead
- **ต้องเขียนเอง:** scaffold code

### Task 0.6: On-Prem LLM Setup (Day 7-10)
- [ ] ติดตั้ง Ollama บน GPU server
- [ ] ดาวน์โหลด Llama 3 70B (หรือ Typhoon Thai ถ้ามี)
- [ ] ทดสอบ inference ภาษาไทย: ความเร็ว, คุณภาพ
- [ ] เปรียบเทียบ: Llama 3 70B vs Typhoon Thai vs Llama 3 8B (trade-off คุณภาพ/ความเร็ว)
- [ ] ตัดสินใจ model สำหรับ on-prem
- **ผู้รับผิดชอบ:** Backend 2
- **เครื่องมือ:** Ollama (สำเร็จรูป)
- **ต้องเขียนเอง:** benchmark script ~100 lines

### Milestone 0: Infrastructure Ready
- [ ] RAGFlow UI เข้าถึงได้ + อัปโหลดเอกสารไทยได้
- [ ] Qdrant รับ vector + query ได้
- [ ] BGE-M3 endpoint ตอบ embedding ภาษาไทยได้
- [ ] On-Prem LLM ตอบภาษาไทยได้
- [ ] Git repo + CI พร้อม

---

## Phase 1: Core RAG Pipeline + Basic UI (สัปดาห์ 3-6)

**เป้าหมาย:** End-to-end ถาม-ตอบจากเอกสารภาษาไทยได้ ผ่าน web UI, ยังไม่มี access control

### Task 1.1: FastAPI Orchestration Service (สัปดาห์ 3)

สร้าง API server กลางที่เป็น "สมอง" ของระบบ เชื่อมทุก component เข้าด้วยกัน

```
User → FastAPI Gateway → RAGFlow (retrieval) → LLM (generation) → User
```

- [ ] POST /api/v1/chat — รับคำถาม, เรียก RAGFlow search, ส่ง context + question ไป LLM, ตอบกลับ
- [ ] POST /api/v1/documents/upload — รับไฟล์, ส่งเข้า RAGFlow ingestion pipeline
- [ ] GET /api/v1/documents — list เอกสารทั้งหมด (จาก RAGFlow API)
- [ ] DELETE /api/v1/documents/{id} — ลบเอกสาร + vectors
- [ ] GET /api/v1/health — health check ทุก component
- **ผู้รับผิดชอบ:** Backend 1 + Backend 2
- **เครื่องมือ:** FastAPI, httpx (async HTTP client)
- **ต้องเขียนเอง:** ทั้งหมด ~800-1,200 lines
- **หมายเหตุ:** เรียก RAGFlow ผ่าน REST API, ไม่ import library ตรง

### Task 1.2: RAGFlow Configuration สำหรับภาษาไทย (สัปดาห์ 3)
- [ ] สร้าง Knowledge Base ใน RAGFlow, เลือก chunking template: "Laws" สำหรับระเบียบ, "Naive" สำหรับเอกสารทั่วไป
- [ ] ตั้ง embedding model → ชี้ไปที่ BGE-M3 endpoint (Task 0.4)
- [ ] ตั้ง chunk size: 512 tokens, overlap: 64 tokens
- [ ] ทดสอบ ingestion กับเอกสารจริง 5 ประเภท:
  - ระเบียบ/คำสั่ง NT (PDF)
  - คู่มือปฏิบัติงาน (Word)
  - เอกสารสแกน (PDF image)
  - รายงานการเงิน (PDF/Excel)
  - นโยบายทั่วไป (PDF)
- [ ] ตรวจสอบคุณภาพ chunking แต่ละประเภท
- **ผู้รับผิดชอบ:** Backend 1
- **ต้องเขียนเอง:** config เท่านั้น

### Task 1.3: LLM Router Service (สัปดาห์ 4)

เลือก LLM อัตโนมัติตามประเภทคำถาม (cloud vs on-prem)

- [ ] สร้าง LLM provider abstraction (pattern เดียวกับ OpenMiniCrew provider registry)
  - CloudProvider: Claude / GPT-4o / Gemini
  - LocalProvider: Ollama (Llama 3 / Typhoon)
- [ ] Default route: Cloud (Phase 3 จะเพิ่ม sensitive routing)
- [ ] Retry + fallback logic
- [ ] Token counting + cost tracking
- **ผู้รับผิดชอบ:** Backend 2
- **ต้องเขียนเอง:** ~400-600 lines (reuse pattern จาก OpenMiniCrew)

### Task 1.4: Prompt Engineering สำหรับ RAG (สัปดาห์ 4)
- [ ] เขียน system prompt ภาษาไทย:
  - บทบาท: ผู้ช่วยตอบคำถามจากเอกสาร NT
  - กฎ: ตอบจาก context เท่านั้น, ถ้าไม่มีข้อมูลให้บอกตรงๆ
  - รูปแบบ: อ้างอิงชื่อเอกสาร + หน้า/section ที่มา
  - ภาษา: ตอบภาษาเดียวกับคำถาม
- [ ] ทดสอบกับ 50 คำถามจริง (เตรียม golden set)
- [ ] วัดผล: accuracy, citation correctness, hallucination rate
- [ ] ปรับ prompt จนได้ accuracy > 80% บน golden set
- **ผู้รับผิดชอบ:** Tech Lead + Backend 1
- **ต้องเขียนเอง:** prompt templates + evaluation script

### Task 1.5: React Frontend — Chat UI (สัปดาห์ 5-6)
- [ ] หน้า Chat: พิมพ์คำถาม → แสดงคำตอบ + citation
  - Streaming response (SSE)
  - Citation panel: คลิกดูเอกสารต้นฉบับ
  - Chat history (session-based, ยังไม่ persist)
- [ ] หน้า Documents: อัปโหลด, list, ลบเอกสาร
  - Drag & drop upload
  - แสดง processing status (processing → ready)
  - Filter by ประเภท/วันที่
- [ ] หน้า Login (placeholder, ใช้ mock auth ก่อน)
- **ผู้รับผิดชอบ:** Frontend Developer
- **เครื่องมือ:** React, Ant Design, EventSource (SSE)
- **ต้องเขียนเอง:** ทั้งหมด ~3,000-4,000 lines

### Task 1.6: Database Schema — PostgreSQL (สัปดาห์ 3)

```sql
-- Users & Auth
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     VARCHAR(20) UNIQUE NOT NULL,  -- รหัสพนักงาน NT
    display_name    VARCHAR(100),
    email           VARCHAR(255),
    department_id   UUID REFERENCES departments(id),
    role            VARCHAR(20) DEFAULT 'staff',  -- sysadmin | executive | director | manager | staff
    clearance_level INTEGER DEFAULT 1,            -- 1-5 (ดู Security Design section)
    is_active       BOOLEAN DEFAULT true,
    max_sessions    INTEGER DEFAULT 3,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Department hierarchy (สะท้อนโครงสร้าง NT จริง)
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) UNIQUE,           -- รหัสหน่วยงาน NT
    name            VARCHAR(200) NOT NULL,
    name_en         VARCHAR(200),
    parent_id       UUID REFERENCES departments(id),  -- สร้าง tree: สายงาน → ฝ่าย → ส่วน
    dept_level      INTEGER DEFAULT 1,            -- 1=สายงาน, 2=ฝ่าย, 3=ส่วน/แผนก
    default_classification INTEGER DEFAULT 1,     -- default ชั้นความลับเอกสารของหน่วยงาน
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Documents metadata
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ragflow_doc_id  VARCHAR(100),                 -- ID ใน RAGFlow
    title           VARCHAR(500) NOT NULL,
    file_name       VARCHAR(255),
    file_type       VARCHAR(20),                  -- pdf, docx, xlsx, scan
    file_size_bytes BIGINT,
    classification  INTEGER NOT NULL,             -- 1=เปิดเผย, 2=ใช้ภายใน, 3=ลับ, 4=ลับมาก (ต้องระบุเสมอ)
    department_id   UUID REFERENCES departments(id),
    uploaded_by     UUID REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),    -- ผู้อนุมัติ (Level 3-4 ต้องมี)
    approved_at     TIMESTAMPTZ,
    rejection_reason TEXT,                        -- เหตุผลถ้าปฏิเสธ
    ocr_processed   BOOLEAN DEFAULT false,
    chunk_count     INTEGER DEFAULT 0,
    status          VARCHAR(20) DEFAULT 'pending_classification',
                    -- pending_classification → pending_approval → processing → ready
                    -- หรือ → rejected, error, archived, deleted
    version         INTEGER DEFAULT 1,
    replaces_doc_id UUID REFERENCES documents(id), -- ชี้ไปเอกสารฉบับเก่าที่ถูกแทนที่
    review_date     DATE,                         -- วันที่ต้องทบทวนชั้นความลับ (Level 3-4)
    deleted_at      TIMESTAMPTZ,                  -- soft delete timestamp
    metadata        JSONB,                        -- custom fields
    allowed_departments UUID[],                   -- ฝ่ายที่อนุญาตให้เข้าถึง (นอกเหนือจาก department_id)
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Document access permissions (Explicit Grants)
CREATE TABLE document_access (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID REFERENCES documents(id) ON DELETE CASCADE,
    grantee_type    VARCHAR(20) NOT NULL,         -- department | role | user
    grantee_id      TEXT NOT NULL,                -- department.id | role name | user.id
    permission      VARCHAR(20) DEFAULT 'read',   -- read | manage
    granted_by      UUID REFERENCES users(id),    -- ใครให้สิทธิ์
    expires_at      TIMESTAMPTZ,                  -- สิทธิ์ชั่วคราว (optional)
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Document approval workflow
CREATE TABLE document_approvals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID REFERENCES documents(id) ON DELETE CASCADE,
    requested_by    UUID REFERENCES users(id),
    requested_classification INTEGER NOT NULL,     -- ชั้นความลับที่ขอ
    approver_id     UUID REFERENCES users(id),
    status          VARCHAR(20) DEFAULT 'pending', -- pending | approved | rejected
    reason          TEXT,                          -- เหตุผลอนุมัติ/ปฏิเสธ
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ
);

-- Cross-department access requests
CREATE TABLE access_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requester_id    UUID REFERENCES users(id),
    document_id     UUID,                         -- specific document หรือ NULL ถ้าขอทั้ง department
    target_department_id UUID REFERENCES departments(id),
    reason          TEXT NOT NULL,
    status          VARCHAR(20) DEFAULT 'pending', -- pending | approved | rejected | expired
    approved_by_requester_dept UUID REFERENCES users(id), -- Director ฝ่ายผู้ขอ
    approved_by_target_dept UUID REFERENCES users(id),    -- Director ฝ่ายเจ้าของ
    expires_at      TIMESTAMPTZ,                  -- สิทธิ์หมดอายุเมื่อไร
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ
);

-- Chat conversations
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    title           VARCHAR(500),
    max_classification_seen INTEGER DEFAULT 1,    -- ชั้นความลับสูงสุดที่ปรากฏใน conversation นี้
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Chat messages
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL,         -- user | assistant
    content         TEXT NOT NULL,
    citations       JSONB,                        -- [{doc_id, chunk_id, page, text_snippet, classification}]
    llm_provider    VARCHAR(50),
    llm_model       VARCHAR(100),
    tokens_used     INTEGER,
    retrieval_scores JSONB,                       -- [{chunk_id, score}]
    filtered_doc_ids UUID[],                      -- เอกสารที่ถูก ACL filter ออก (สำหรับ audit)
    routing         VARCHAR(20),                  -- cloud | onprem
    routing_reason  VARCHAR(100),                 -- เหตุผลที่ route (classification | pii | keyword)
    pii_detected    BOOLEAN DEFAULT false,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Audit log (immutable, append-only, partitioned by year)
CREATE TABLE audit_log (
    id              BIGSERIAL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
                    -- query, upload, approve, reject, delete, access_denied,
                    -- config_change, role_change, grant_access, revoke_access,
                    -- login, logout, login_failed, export, view_audit
    resource_type   VARCHAR(50),                  -- document, conversation, user, system, access_request
    resource_id     TEXT,
    details         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Partitions สำหรับ audit log (สร้างล่วงหน้า)
CREATE TABLE audit_log_2026 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
CREATE TABLE audit_log_2027 PARTITION OF audit_log
    FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');

-- User sessions (สำหรับ concurrent session control)
CREATE TABLE user_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    token_hash      VARCHAR(255) NOT NULL,        -- hash ของ JWT (ไม่เก็บ token ตรง)
    device_info     VARCHAR(255),
    ip_address      INET,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Feedback
CREATE TABLE feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID REFERENCES messages(id),
    user_id         UUID REFERENCES users(id),
    rating          INTEGER CHECK (rating BETWEEN 1 AND 5),
    comment         TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_department ON users(department_id);
CREATE INDEX idx_users_clearance ON users(clearance_level);
CREATE INDEX idx_departments_parent ON departments(parent_id);
CREATE INDEX idx_documents_department ON documents(department_id);
CREATE INDEX idx_documents_classification ON documents(classification);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_uploaded_by ON documents(uploaded_by);
CREATE INDEX idx_documents_allowed_depts ON documents USING GIN(allowed_departments);
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);
CREATE INDEX idx_audit_user_time ON audit_log(user_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_log(action, created_at DESC);
CREATE INDEX idx_document_access_doc ON document_access(document_id);
CREATE INDEX idx_document_access_grantee ON document_access(grantee_type, grantee_id);
CREATE INDEX idx_document_approvals_status ON document_approvals(status, approver_id);
CREATE INDEX idx_access_requests_status ON access_requests(status);
CREATE INDEX idx_user_sessions_user ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expiry ON user_sessions(expires_at);
```

- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** PostgreSQL 16+, Alembic (migration)
- **ต้องเขียนเอง:** migration scripts + SQLAlchemy models

### Milestone 1: End-to-End Demo
- [ ] อัปโหลดเอกสารภาษาไทย → ระบบ chunk + embed → ถามคำถาม → ได้คำตอบ + citation
- [ ] Demo ให้ stakeholder ดูได้
- [ ] Golden set 50 คำถาม accuracy > 80%

---

## Phase 2: Thai Language Optimization (สัปดาห์ 7-10)

**เป้าหมาย:** คุณภาพการค้นหาและตอบคำถามภาษาไทยอยู่ในระดับ production

### Task 2.1: Thai Word Segmentation Pipeline (สัปดาห์ 7)

ปัญหา: ภาษาไทยไม่มีช่องว่างระหว่างคำ การ chunk แบบนับ character จะตัดคำผิด

- [ ] สร้าง preprocessing service:
  1. รับ raw text จาก RAGFlow
  2. PyThaiNLP `newmm` tokenizer ตัดคำ
  3. TLTK ตัด sentence boundary
  4. Chunk ที่ sentence boundary, target 256-512 tokens
- [ ] สร้าง custom dictionary สำหรับ NT:
  - ศัพท์โทรคมนาคม (FTTx, MPLS, OLT, กสทช. ฯลฯ)
  - ชื่อหน่วยงานภายใน NT
  - ศัพท์การเงิน/บัญชีที่ใช้บ่อย
  - ชื่อระเบียบ/คำสั่งที่เป็นชื่อเฉพาะ
- [ ] เปรียบเทียบ chunk quality: ก่อน vs หลัง word segmentation
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** PyThaiNLP (open-source), TLTK (open-source)
- **ต้องเขียนเอง:** preprocessing pipeline ~300-400 lines, custom dictionary ~200 terms

### Task 2.2: OCR Pipeline สำหรับเอกสารสแกน (สัปดาห์ 7-8)

```
Scanned PDF → Page Extraction → PaddleOCR-VL → Text + Layout
                                      ↓
                              ThaiTrOCR (fallback for low-confidence regions)
                                      ↓
                              Post-processing → Clean Thai Text → Chunking
```

- [ ] ตั้ง PaddleOCR-VL (ผ่าน RAGFlow DeepDoc หรือ standalone)
  - ทดสอบกับเอกสารสแกนจริงของ NT 20 ไฟล์
  - วัด CER (Character Error Rate) เฉพาะภาษาไทย
- [ ] ตั้ง ThaiTrOCR เป็น second-pass สำหรับ regions ที่ PaddleOCR confidence ต่ำ
- [ ] สร้าง post-processing pipeline:
  - แก้ไข common OCR errors ภาษาไทย (สระลอย, วรรณยุกต์หาย)
  - ลบ header/footer ซ้ำ
  - รวม text จากหลาย column ตามลำดับอ่าน
- [ ] เปรียบเทียบกับ Google Document AI (ถ้า budget อนุญาต) เพื่อตั้ง baseline
- **ผู้รับผิดชอบ:** Backend 2
- **เครื่องมือ:** PaddleOCR-VL (Apache 2.0), ThaiTrOCR (research model)
- **ต้องเขียนเอง:** post-processing + orchestration ~500-700 lines

### Task 2.3: Hybrid Search — BM25 + Vector (สัปดาห์ 8-9)

Vector search อย่างเดียวไม่พอสำหรับเอกสารกฎหมาย/ระเบียบ ต้องผสม keyword search

- [ ] ตั้ง Qdrant BM25 full-text search (ใช้ native Thai tokenizer จาก v1.15+)
- [ ] สร้าง hybrid search endpoint:
  1. Vector search (BGE-M3) → top 20
  2. BM25 keyword search → top 20
  3. Reciprocal Rank Fusion (RRF) → merge → top 10
- [ ] ทดสอบ 3 strategies: vector only, BM25 only, hybrid
- [ ] วัด MRR@10, Recall@10 บน golden set
- [ ] ปรับ weight ระหว่าง vector/BM25 จนได้ผลดีที่สุด
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** Qdrant (built-in BM25 + Thai tokenizer)
- **ต้องเขียนเอง:** hybrid search logic + RRF ~200-300 lines

### Task 2.4: Re-ranking (สัปดาห์ 9)
- [ ] เพิ่ม cross-encoder re-ranker หลัง hybrid search:
  - ตัวเลือก 1: BGE-reranker-v2-m3 (multilingual, open-source)
  - ตัวเลือก 2: Cohere Rerank (cloud API, ง่ายกว่า)
- [ ] ทดสอบ: search without reranker vs with reranker
- [ ] วัด improvement บน golden set
- **ผู้รับผิดชอบ:** Backend 1
- **ต้องเขียนเอง:** reranker integration ~100-200 lines

### Task 2.5: Evaluation Framework — RAGAS (สัปดาห์ 9-10)

ตั้งระบบวัดคุณภาพ RAG อัตโนมัติ ไม่ต้องวัดด้วยคนทุกครั้ง

- [ ] ติดตั้ง RAGAS (open-source RAG evaluation framework)
- [ ] สร้าง golden test set: 100+ คู่ (คำถาม, คำตอบที่ถูกต้อง, เอกสารที่ควรอ้างอิง)
  - 30 คำถามจากระเบียบ/คำสั่ง
  - 20 คำถามจากคู่มือ SOP
  - 20 คำถามจากเอกสารสแกน
  - 15 คำถามจากรายงานการเงิน
  - 15 คำถามที่ "ไม่มีคำตอบในเอกสาร" (ทดสอบ hallucination)
- [ ] วัด metrics:
  - Faithfulness (ตอบตรงกับ context ไหม)
  - Answer Relevancy (ตอบตรงคำถามไหม)
  - Context Precision (ดึง context ที่เกี่ยวข้องมาไหม)
  - Context Recall (ดึงมาครบไหม)
- [ ] ตั้ง baseline: ทุก metric > 0.75 ถือว่าผ่าน
- [ ] สร้าง CI job: รัน RAGAS ทุกครั้งที่แก้ prompt/chunking/search config
- **ผู้รับผิดชอบ:** QA + Tech Lead
- **เครื่องมือ:** RAGAS (open-source)
- **ต้องเขียนเอง:** golden set curation + CI integration ~300 lines

### Milestone 2: Thai Quality Baseline
- [ ] RAGAS metrics ทุกตัว > 0.75
- [ ] OCR สามารถอ่านเอกสารสแกนภาษาไทยได้ CER < 5%
- [ ] Hybrid search MRR@10 > 0.85 บน golden set
- [ ] คำถามที่ "ไม่มีคำตอบ" ระบบตอบว่าไม่พบข้อมูล > 90% ของกรณี

---

## Phase 3: Security & Access Control (สัปดาห์ 11-14)

**เป้าหมาย:** ทุกคนเห็นเฉพาะเอกสารที่มีสิทธิ์, sensitive data ไม่ออกนอก NT, audit log ครบ

### Task 3.1: Authentication — เชื่อม NT's Existing Auth (สัปดาห์ 11)
- [ ] สำรวจระบบ auth ที่ NT ใช้อยู่ (AD/LDAP? SSO? OAuth?)
- [ ] เชื่อม FastAPI กับ NT auth:
  - ถ้า AD/LDAP → ใช้ python-ldap
  - ถ้า OAuth/OIDC → ใช้ authlib
  - ถ้ายังไม่มี → สร้าง JWT-based auth ก่อน, เชื่อม SSO ทีหลัง
- [ ] Auto-sync department + role จาก NT's directory
- [ ] Session management + token refresh
- **ผู้รับผิดชอบ:** Backend 2
- **ต้องเขียนเอง:** auth middleware ~300-500 lines

### Task 3.2: Document-Level Access Control — Pre-filtering (สัปดาห์ 11-12)

Implement ตาม Security Design Section 3-4 (Access Control Matrix + Department Scope)

```
User Query → Auth Check → Get User Permissions
    → Resolve clearance: user.clearance_level + explicit grants + department hierarchy
    → Build Metadata Filter:
        - classification <= user.clearance_level
        - department_id IN (user's department + parent departments + granted departments)
        - document NOT in 'archived' or 'deleted' status
    → Qdrant Search WITH Filter (เฉพาะเอกสารที่มีสิทธิ์)
    → LLM Generation (จาก filtered context เท่านั้น)
```

- [ ] Deploy Cerbos (policy engine, Docker)
- [ ] เขียน access policies ตาม Access Control Matrix:
  ```yaml
  # Staff: เห็นเฉพาะ Level 1 + explicit grants
  # Manager: Level 1-2 ในฝ่ายตัวเอง + same business unit Level 1-2
  # Director: Level 1-3 ในฝ่ายตัวเอง + Level 1-2 ข้ามฝ่าย
  # Executive: Level 1-4 ทุกฝ่าย
  # SysAdmin: ไม่เข้าถึงเนื้อหาเอกสาร Level 3-4
  ```
- [ ] สร้าง permission resolver:
  - Input: user_id → Output: list of accessible (department_ids, max_classification, explicit_doc_ids)
  - ใช้ department hierarchy tree (parent_id) คำนวณ "same business unit"
  - Cache permissions ใน Redis (TTL 5 min, invalidate เมื่อ role/grant เปลี่ยน)
- [ ] สร้าง Qdrant metadata filter builder จาก resolved permissions
- [ ] ทดสอบ 20 scenarios:
  - Staff ค้นไม่เจอ Level 2-4 (ยกเว้น explicit grant)
  - Manager ค้นเจอ Level 2 ฝ่ายตัวเอง แต่ไม่เจอ Level 2 ฝ่ายอื่น
  - Director ค้นเจอ Level 3 ฝ่ายตัวเอง แต่ไม่เจอ Level 3 ฝ่ายอื่น
  - Executive ค้นเจอ Level 4 ทุกฝ่าย
  - SysAdmin ค้นเจอ Level 1-2 เท่านั้น (ไม่เห็น content Level 3-4)
  - Explicit grant ทำงานถูก: Staff ที่ได้ grant เห็นเอกสารเฉพาะที่ grant
  - สิทธิ์ชั่วคราวหมดอายุแล้ว → เข้าไม่ได้
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** Cerbos (open-source, Apache 2.0)
- **ต้องเขียนเอง:** policy files + permission resolver + filter builder ~600-800 lines

### Task 3.3: Document Approval Workflow (สัปดาห์ 12)

Implement ตาม Security Design Section 5 (Document Lifecycle)

- [ ] Upload flow:
  - ผู้อัปโหลดต้องเลือก classification (ไม่มี default — บังคับเลือก)
  - Level 1-2 โดย Manager+ → เข้าระบบทันที
  - Level 3 → status: pending_approval → แจ้ง Director ฝ่ายนั้น
  - Level 4 → status: pending_approval → แจ้ง Executive
  - Staff อัปโหลดไม่ได้ (ต้องให้ Manager อัปโหลดให้)
- [ ] Approval UI:
  - Admin/Director เห็น pending documents ในหน้า dashboard
  - Preview เอกสาร + approve/reject + เหตุผล
  - Email/notification แจ้งผู้อัปโหลดเมื่ออนุมัติ/ปฏิเสธ
- [ ] Classification change workflow:
  - เปลี่ยนจาก Level สูง → ต่ำ: ต้องอนุมัติจากระดับเดียวกับ Level เดิม
  - เปลี่ยนจาก Level ต่ำ → สูง: ต้องอนุมัติจากระดับ Level ใหม่
  - ทุกการเปลี่ยน → audit log
- **ผู้รับผิดชอบ:** Backend 2 + Frontend
- **ต้องเขียนเอง:** workflow engine + approval UI ~500-700 lines

### Task 3.4: Cross-Department Access Request (สัปดาห์ 12-13)

Implement ตาม Security Design Section 4 (Cross-Department Access)

- [ ] Access request flow:
  1. User ขอเข้าถึงเอกสาร/department อื่น + ระบุเหตุผล
  2. Director ฝ่ายผู้ขออนุมัติ (ยืนยันว่าจำเป็นจริง)
  3. Director ฝ่ายเจ้าของเอกสารอนุมัติ (ยินยอมให้เข้าถึง)
  4. ถ้าทั้ง 2 ฝ่ายอนุมัติ → สร้าง explicit grant + ตั้ง expiry date
- [ ] สิทธิ์ชั่วคราว: ตั้ง expires_at ได้ (เช่น 30 วัน, 90 วัน)
- [ ] Scheduled job: ตรวจสิทธิ์หมดอายุ ลบ grant อัตโนมัติ + แจ้ง user
- **ผู้รับผิดชอบ:** Backend 2
- **ต้องเขียนเอง:** request flow + dual-approval logic ~400-500 lines

### Task 3.5: Department Isolation — Bridge Model (สัปดาห์ 13)

```
Qdrant Collections:
├── saraphan_public       ← Level 1: ทุกคนค้นได้ (ระเบียบ, นโยบาย, คู่มือทั่วไป)
├── saraphan_internal     ← Level 2: Manager+ ค้นได้ (metadata filter by department)
├── saraphan_finance      ← Level 3: Director+ ฝ่ายการเงิน
├── saraphan_hr           ← Level 3: Director+ ฝ่าย HR
├── saraphan_legal        ← Level 3: Director+ ฝ่ายกฎหมาย
├── saraphan_executive    ← Level 4: Executive เท่านั้น (encrypted)
└── saraphan_{dept}       ← สร้างเพิ่มตาม department ที่มีเอกสาร Level 3+
```

- [ ] สร้าง collection management service:
  - Auto-create collection ตาม department config
  - Route document → correct collection ตาม classification + department
  - Level 1 → saraphan_public
  - Level 2 → saraphan_internal (+ metadata: department_id, allowed_departments)
  - Level 3 → saraphan_{dept} (siloed collection)
  - Level 4 → saraphan_executive (encrypted collection)
- [ ] Multi-collection search: query ข้าม collection ที่ user มีสิทธิ์พร้อมกัน
- [ ] Encryption: Level 4 collection ใช้ app-level encryption ก่อน store
- **ผู้รับผิดชอบ:** Backend 1 + DevOps
- **ต้องเขียนเอง:** collection router + multi-search + encryption ~400-500 lines

### Task 3.6: Hybrid Cloud/On-Prem Routing (สัปดาห์ 13-14)

Implement ตาม Security Design Section 6-7 (Data Handling Rules + LLM Routing)

```
User Query
    │
    ▼
[Retrieve + ACL Filter]
    │
    ▼
[ตรวจ classification สูงสุดของ retrieved docs]
    │
    ├── ทั้งหมด Level 1-2
    │   ├── [PII Scan context] → ไม่พบ PII → Cloud LLM
    │   └── [PII Scan context] → พบ PII → On-Prem LLM
    │
    ├── มี Level 3 → On-Prem LLM + response แสดง snippet สั้นเท่านั้น
    │
    └── มี Level 4 → On-Prem LLM + response แสดง reference only (ชื่อ+หน้า)
```

- [ ] PII detection ตาม Security Design Section 8:
  - เลขบัตรประชาชน, เบอร์โทร, email, เลขบัญชี, เงินเดือน ฯลฯ
  - ใช้ regex + keyword matching (เร็ว, ไม่ต้องใช้ LLM)
  - ถ้าพบ PII → route ทั้ง query ไป on-prem (ไม่ mask แล้วส่ง cloud)
- [ ] Response formatting ตาม classification:
  - Level 1-2: แสดง source text เต็ม + citation
  - Level 3: แสดง snippet สั้น (50 คำ) + document reference
  - Level 4: แสดงเฉพาะ document name + page number (ต้องไปเปิดดูต้นฉบับเอง)
- [ ] Disable copy/export สำหรับ response ที่มี Level 3-4 content
- **ผู้รับผิดชอบ:** Backend 2
- **ต้องเขียนเอง:** PII detector + router + response formatter ~600-800 lines

### Task 3.7: Audit Logging (สัปดาห์ 14)

Implement ตาม Security Design Section 9 (Audit Log Requirements)

- [ ] Audit middleware (FastAPI middleware):
  - ทุก request → log user, action, resource, IP, user-agent
  - Immutable: ใช้ INSERT only, ไม่มี UPDATE/DELETE endpoint
  - Partitioned by year (สร้าง partition ล่วงหน้า 2 ปี)
- [ ] Log ตาม action types:
  - User actions: query, login/logout, feedback, export
  - Document actions: upload, approve/reject, classify, delete
  - Admin actions: role change, grant/revoke, config change
  - Security events: access_denied, login_failed, session_exceeded
- [ ] สิ่งพิเศษ: log เอกสารที่ถูก ACL filter ออก (filtered_doc_ids ใน messages table)
  → เพื่อ audit ว่า ACL ทำงานถูกต้อง + ไม่มี data leak
- [ ] Audit log viewer (admin UI):
  - Director: ดู log เฉพาะฝ่ายตัวเอง
  - Executive: ดู log ทั้งหมด
  - SysAdmin: ดู log ทั้งหมด + system events
  - การเข้าดู audit log → บันทึกใน audit log อีกชั้นหนึ่ง (view_audit action)
- [ ] Retention: auto-archive partitions เก่ากว่า 2 ปี → cold storage
- **ผู้รับผิดชอบ:** Backend 2
- **ต้องเขียนเอง:** audit middleware + viewer API ~400-500 lines

### Task 3.8: Session Security (สัปดาห์ 14)
- [ ] JWT with short expiry (8 hrs) + refresh token (30 days)
- [ ] Concurrent session control: max 3 devices per user (ตาม user_sessions table)
  - ถ้าเกิน → แจ้ง user เลือก terminate session เก่า
- [ ] Rate limiting:
  - Per user: 60 queries/hour
  - Per IP: 120 queries/hour
  - Burst: 10 queries/minute
  - เมื่อถึง limit → 429 + audit log
- [ ] Failed login lockout: 5 ครั้ง → lock 15 นาที → audit log
- [ ] Auto-revoke sessions เมื่อ: user ถูก deactivate, role เปลี่ยน, clearance ลดลง
- **ผู้รับผิดชอบ:** Backend 2
- **ต้องเขียนเอง:** session manager + rate limiter ~300-400 lines

### Task 3.9: DPIA (Data Protection Impact Assessment) (สัปดาห์ 14)
- [ ] จัดทำเอกสาร DPIA ตาม PDPA requirements
- [ ] ส่ง DPO ของ NT ตรวจสอบ
- [ ] จัดทำ Privacy Notice สำหรับผู้ใช้ระบบ Saraphan
- [ ] ตรวจสอบ DPA (Data Processing Agreement) กับ Cloud LLM providers
- **ผู้รับผิดชอบ:** Tech Lead + Legal team ของ NT
- **ต้องเขียนเอง:** เอกสาร (ไม่ใช่ code) + privacy notice UI component

### Milestone 3: Security Ready
- [ ] ACL ทดสอบผ่านครบ 20 scenarios (ดู Task 3.2)
- [ ] Staff ค้นไม่เจอเอกสาร Level 2+ (100%)
- [ ] Manager ค้นไม่เจอเอกสาร Level 3+ ของฝ่ายอื่น (100%)
- [ ] Document approval workflow: Level 3-4 ต้องผ่านอนุมัติก่อนเข้าระบบ
- [ ] Cross-department access request + dual-approval ทำงานครบ flow
- [ ] Query sensitive (Level 3-4 content หรือ PII) → route to on-prem LLM (100%)
- [ ] PII ไม่หลุดไป Cloud API (ทดสอบกับ 50 queries ที่มี PII)
- [ ] Response format ถูกต้องตาม classification (full/snippet/reference-only)
- [ ] Audit log บันทึกครบทุก interaction + immutable
- [ ] Session control: concurrent limit + rate limit + lockout ทำงาน
- [ ] เชื่อม NT auth สำเร็จ
- [ ] DPIA ผ่านการตรวจจาก DPO

---

## Phase 4: Production Hardening (สัปดาห์ 15-18)

**เป้าหมาย:** รองรับ 1,000+ users, มี monitoring, admin จัดการระบบได้

### Task 4.1: Load Testing + Performance Optimization (สัปดาห์ 15)
- [ ] ทดสอบด้วย Locust/k6:
  - 100 concurrent users, 10 queries/sec
  - Target: p95 latency < 5 seconds (รวม retrieval + generation)
- [ ] Bottleneck analysis + optimization:
  - Embedding cache (Redis): cache vector ของ query ที่ซ้ำ
  - Search result cache (Redis): cache top-k results, TTL 5 min
  - Connection pooling (RAGFlow, Qdrant, PostgreSQL)
  - Async everywhere (FastAPI + httpx)
- [ ] Qdrant optimization: HNSW parameters tuning, quantization ถ้าจำเป็น
- **ผู้รับผิดชอบ:** DevOps + Backend 1
- **เครื่องมือ:** Locust (open-source), Redis
- **ต้องเขียนเอง:** cache layer ~200-300 lines

### Task 4.2: Observability — Langfuse (สัปดาห์ 15-16)
- [ ] Deploy Langfuse (self-hosted Docker)
- [ ] Integrate: trace ทุก LLM call
  - Input prompt, output, tokens, latency, cost
  - Retrieval step: query, results, scores
  - User feedback linkage
- [ ] Dashboard:
  - Daily query volume
  - Avg latency breakdown (retrieval vs generation)
  - Token cost per department
  - Error rate
  - User satisfaction (feedback score)
- **ผู้รับผิดชอบ:** DevOps + Backend 2
- **เครื่องมือ:** Langfuse (open-source, self-hosted)
- **ต้องเขียนเอง:** instrumentation ~200 lines

### Task 4.3: Admin Dashboard (สัปดาห์ 16-18)
- [ ] Document Management:
  - อัปโหลด batch, จัด category/department
  - ดู processing status, re-process failed documents
  - ตั้ง classification level per document
- [ ] User Management:
  - Sync จาก NT directory
  - จัดสิทธิ์ per department/role
  - ดู usage per user
- [ ] System Config:
  - LLM provider settings
  - Routing rules (cloud/onprem)
  - Chunking parameters
  - Embedding model selection
- [ ] Analytics:
  - Top queries (popular topics)
  - Unanswered queries (knowledge gaps)
  - Document usage (เอกสารไหนถูกอ้างอิงบ่อย)
  - Cost tracking per LLM provider
- **ผู้รับผิดชอบ:** Frontend + Backend 2
- **ต้องเขียนเอง:** ทั้งหมด ~4,000-5,000 lines (React + API)

### Task 4.4: High Availability Setup (สัปดาห์ 17-18)
- [ ] Qdrant: 3-node cluster (HA mode)
- [ ] PostgreSQL: primary + replica
- [ ] FastAPI: 2+ instances behind load balancer
- [ ] RAGFlow: evaluate scaling options
- [ ] Automated backup:
  - PostgreSQL: daily pg_dump
  - Qdrant: snapshot to S3/NAS
  - Document storage: replicated
- [ ] Health check + auto-restart (Docker restart policy / K8s)
- [ ] Disaster recovery plan document
- **ผู้รับผิดชอบ:** DevOps
- **ต้องเขียนเอง:** Docker Compose (production) + backup scripts

### Milestone 4: Production Ready
- [ ] p95 latency < 5s ที่ 100 concurrent users
- [ ] Langfuse dashboard แสดงข้อมูลครบ
- [ ] Admin dashboard ใช้งานได้
- [ ] HA setup ทดสอบ failover สำเร็จ
- [ ] Backup + restore ทดสอบสำเร็จ

---

## Phase 5: Advanced Features + Rollout (สัปดาห์ 19-24)

**เป้าหมาย:** Feature ขั้นสูง, pilot launch, full rollout

### Task 5.1: Feedback Loop + Self-Learning (สัปดาห์ 19-20)
- [ ] thumbs up/down บนทุกคำตอบ
- [ ] Admin review: ดู low-rated answers, แก้ไข golden examples
- [ ] Auto-detect knowledge gaps: queries ที่ตอบไม่ได้ → alert admin ให้เพิ่มเอกสาร
- [ ] Weekly RAGAS evaluation: track quality over time

### Task 5.2: Advanced RAG Features (สัปดาห์ 20-22)
- [ ] Multi-turn conversation: จำบริบทสนทนาได้
- [ ] Query decomposition: คำถามซับซ้อน → แตกเป็นหลาย sub-queries
- [ ] Cross-department authorized search: ค้นข้าม department ที่มีสิทธิ์
- [ ] Document versioning: เมื่อระเบียบอัปเดต → archive เก่า, index ใหม่
- [ ] Table extraction + structured Q&A: ถามข้อมูลจากตารางในเอกสารได้

### Task 5.3: Pilot Launch (สัปดาห์ 21-22)
- [ ] เลือก 2-3 ฝ่ายสำหรับ pilot (แนะนำ: ฝ่ายที่มีเอกสารเยอะ + ถามบ่อย)
- [ ] อัปโหลดเอกสารจริงของ pilot departments
- [ ] Training session สำหรับ pilot users
- [ ] รวบรวม feedback 2 สัปดาห์
- [ ] ปรับปรุงจาก feedback

### Task 5.4: Full Rollout (สัปดาห์ 23-24)
- [ ] อัปโหลดเอกสารทุกฝ่าย (อาจ phase เป็นรอบ)
- [ ] Training session สำหรับทั้งบริษัท
- [ ] Launch announcement
- [ ] Dedicated support channel สัปดาห์แรก
- [ ] Monitor closely: performance, accuracy, user satisfaction

### Milestone 5: Full Production
- [ ] ทุกฝ่ายใช้งานได้
- [ ] RAGAS metrics คง > 0.75
- [ ] User satisfaction > 4.0/5.0
- [ ] p95 latency < 5s ที่ full load

---

## Project Structure (สำหรับ custom code)

```
saraphan/
├── docker-compose.yml              # All services
├── docker-compose.dev.yml          # Dev overrides
├── .env.example
├── .env                            # (ห้าม commit)
├── README.md
├── Makefile                        # shortcuts: make dev, make test, make deploy
│
├── backend/                        # FastAPI orchestration service
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                 # FastAPI app entry
│   │   ├── config.py               # Settings from .env
│   │   │
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── chat.py         # POST /chat
│   │   │   │   ├── documents.py    # CRUD documents + approval workflow
│   │   │   │   ├── auth.py         # Login/logout/session
│   │   │   │   ├── admin.py        # Admin endpoints
│   │   │   │   ├── access.py       # Cross-department access requests
│   │   │   │   ├── audit.py        # Audit log viewer
│   │   │   │   └── health.py       # Health check
│   │   │   └── deps.py             # Dependencies (auth, db session, permission check)
│   │   │
│   │   ├── services/
│   │   │   ├── rag_service.py      # Orchestrate: retrieve → generate
│   │   │   ├── ragflow_client.py   # RAGFlow API wrapper
│   │   │   ├── embedding_service.py # BGE-M3 / Cloud embedding
│   │   │   ├── vector_service.py   # Qdrant operations + collection routing
│   │   │   ├── llm_router.py       # Cloud/OnPrem LLM selection (classification-based)
│   │   │   ├── ocr_service.py      # PaddleOCR + ThaiTrOCR
│   │   │   ├── thai_nlp.py         # Word segmentation, chunking
│   │   │   └── approval_service.py # Document approval workflow
│   │   │
│   │   ├── security/
│   │   │   ├── auth.py             # JWT / LDAP / OAuth
│   │   │   ├── permissions.py      # Permission resolver (clearance + grants + dept hierarchy)
│   │   │   ├── acl.py              # Cerbos integration + filter builder
│   │   │   ├── pii_detector.py     # PII scanning (Thai national ID, phone, etc.)
│   │   │   ├── query_classifier.py # Classification-based routing decision
│   │   │   ├── response_filter.py  # Response formatting per classification level
│   │   │   ├── session_manager.py  # Concurrent session control + rate limiting
│   │   │   ├── encryption.py       # App-level encryption for Level 4 documents
│   │   │   └── audit.py            # Audit logging middleware
│   │   │
│   │   ├── models/                 # SQLAlchemy models
│   │   │   ├── user.py
│   │   │   ├── department.py
│   │   │   ├── document.py         # + approval workflow
│   │   │   ├── access.py           # document_access + access_requests
│   │   │   ├── conversation.py
│   │   │   ├── session.py
│   │   │   └── audit.py
│   │   │
│   │   ├── providers/              # LLM providers (pattern จาก OpenMiniCrew)
│   │   │   ├── base.py
│   │   │   ├── claude_provider.py
│   │   │   ├── openai_provider.py
│   │   │   ├── gemini_provider.py
│   │   │   ├── ollama_provider.py
│   │   │   └── registry.py
│   │   │
│   │   └── utils/
│   │       ├── thai_dict.py        # NT custom dictionary
│   │       └── text_processing.py
│   │
│   ├── alembic/                    # DB migrations
│   ├── tests/
│   │   ├── test_acl.py             # 20+ ACL scenarios
│   │   ├── test_pii_detector.py
│   │   ├── test_approval_workflow.py
│   │   ├── test_routing.py         # Classification-based LLM routing
│   │   └── test_audit.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── frontend/                       # React + Ant Design
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Chat.tsx            # Main chat interface
│   │   │   ├── Documents.tsx       # Document management + approval
│   │   │   ├── AccessRequest.tsx   # Cross-department access requests
│   │   │   ├── Admin.tsx           # Admin dashboard
│   │   │   ├── AuditLog.tsx        # Audit log viewer (role-scoped)
│   │   │   ├── Login.tsx
│   │   │   └── PrivacyNotice.tsx   # PDPA privacy notice (first login)
│   │   ├── components/
│   │   │   ├── ChatMessage.tsx     # + classification badge + copy restriction
│   │   │   ├── CitationPanel.tsx   # full/snippet/reference-only per level
│   │   │   ├── DocumentUpload.tsx  # + classification selector (mandatory)
│   │   │   ├── ApprovalQueue.tsx   # Pending approvals for Director/Executive
│   │   │   └── SearchFilter.tsx
│   │   └── services/
│   │       └── api.ts              # API client
│   ├── Dockerfile
│   └── package.json
│
├── evaluation/                     # Quality assurance
│   ├── golden_set/
│   │   ├── questions.json
│   │   └── expected_docs.json
│   ├── security_test/
│   │   ├── acl_scenarios.json      # 20+ ACL test scenarios
│   │   ├── pii_test_data.json      # PII patterns for testing
│   │   └── routing_test.json       # LLM routing test cases
│   ├── run_ragas.py
│   ├── run_acl_tests.py
│   ├── run_ocr_benchmark.py
│   └── run_search_benchmark.py
│
├── config/                         # External service configs
│   ├── cerbos/
│   │   └── policies/
│   │       ├── document_access.yaml    # Document read/manage policies
│   │       ├── upload_policy.yaml      # Who can upload what level
│   │       ├── approval_policy.yaml    # Who can approve what level
│   │       ├── admin_policy.yaml       # Admin action policies
│   │       └── audit_view_policy.yaml  # Who can view which audit logs
│   ├── ragflow/
│   │   └── config.yml
│   ├── qdrant/
│   │   └── config.yml
│   └── langfuse/
│       └── docker-compose.yml
│
├── scripts/
│   ├── seed_departments.py         # Import NT department structure
│   ├── seed_roles.py               # Setup initial roles + clearance levels
│   ├── sync_users.py               # Sync users from AD/LDAP
│   ├── import_documents.py         # Bulk document import
│   └── backup.sh                   # Backup all data
│
└── docs/
    ├── ARCHITECTURE.md
    ├── API.md
    ├── DEPLOYMENT.md
    ├── SECURITY.md                 # Security design (from this plan)
    ├── PDPA_COMPLIANCE.md          # PDPA compliance documentation
    ├── USER_GUIDE.md
    └── ADMIN_GUIDE.md              # Admin operations guide
```

---

## ตารางสรุป: เขียนเอง vs เครื่องมือสำเร็จรูป

| Component | เขียนเอง | เครื่องมือสำเร็จรูป | หมายเหตุ |
|-----------|---------|-------------------|---------|
| Document ingestion & chunking | ไม่ | RAGFlow DeepDoc | ปรับ config เท่านั้น |
| OCR | เขียน orchestration | PaddleOCR-VL + ThaiTrOCR | เขียน post-processing |
| Thai word segmentation | เขียน pipeline | PyThaiNLP + TLTK | เขียน custom dictionary |
| Embedding | เขียน wrapper | BGE-M3 (HuggingFace TEI) | Docker deploy |
| Vector search | เขียน hybrid search | Qdrant | BM25 + vector fusion |
| Re-ranking | เขียน integration | BGE-reranker-v2-m3 | ~100 lines |
| LLM routing | เขียนทั้งหมด | — | reuse pattern จาก OpenMiniCrew |
| PII detection | เขียนทั้งหมด | — | regex + rules |
| Access control | เขียน integration | Cerbos | policy files + filter builder |
| Authentication | เขียน integration | NT's existing auth | ขึ้นกับระบบ NT ปัจจุบัน |
| Audit logging | เขียนทั้งหมด | — | middleware + PostgreSQL |
| Observability | เขียน instrumentation | Langfuse | ~200 lines integration |
| Chat UI | เขียนทั้งหมด | — | React + Ant Design |
| Admin dashboard | เขียนทั้งหมด | — | React + Ant Design |
| Evaluation | เขียน golden set | RAGAS | framework สำเร็จรูป |
| Database | เขียน schema + models | PostgreSQL + Alembic | standard tooling |
| Infrastructure | เขียน config | Docker Compose / K8s | standard tooling |

**สรุป: ~60% ใช้เครื่องมือสำเร็จรูป, ~40% เขียนเอง (ส่วนใหญ่เป็น orchestration + Thai language + security)**

---

## Timeline Summary

| สัปดาห์ | Phase | Deliverable หลัก |
|---------|-------|-----------------|
| 1-2 | Phase 0: Infrastructure | RAGFlow + Qdrant + BGE-M3 + On-Prem LLM ทำงานได้ |
| 3-6 | Phase 1: Core Pipeline | End-to-end Q&A + Basic UI, accuracy > 80% |
| 7-10 | Phase 2: Thai Optimization | Thai OCR + word segmentation + hybrid search, RAGAS > 0.75 |
| 11-14 | Phase 3: Security | ACL + hybrid routing + audit, เชื่อม NT auth |
| 15-18 | Phase 4: Production | Load test OK, monitoring, admin dashboard, HA |
| 19-22 | Phase 5a: Advanced + Pilot | Feedback loop, multi-turn, pilot 2-3 departments |
| 23-24 | Phase 5b: Full Rollout | ทุกฝ่ายใช้งาน, support, monitoring |

---

## Risk Register

| Risk | ผลกระทบ | ความน่าจะเป็น | แผนรับมือ |
|------|---------|-------------|---------|
| OCR ภาษาไทยไม่แม่นพอ | สูง | ปานกลาง | Fallback: Google Document AI (cloud), ถ้าไม่ได้ให้คนทำ manual correction |
| On-Prem LLM คุณภาพภาษาไทยต่ำ | สูง | ปานกลาง | ทดสอบ Typhoon Thai, Qwen 2.5 Thai, ถ้าไม่ดีพอใช้ cloud + PII masking แทน |
| NT auth ซับซ้อนกว่าคาด | ปานกลาง | สูง | เริ่ม JWT-based ก่อน, เชื่อม SSO ทีหลัง |
| เอกสารเก่าคุณภาพต่ำ (สแกนเบลอ) | ปานกลาง | สูง | ตั้ง quality threshold, เอกสารที่ OCR ไม่ผ่าน → flag ให้ admin review |
| RAGFlow อัปเดตแล้ว breaking change | ปานกลาง | ต่ำ | Pin version, อ่าน changelog ก่อน upgrade |
| User adoption ต่ำ | สูง | ปานกลาง | Pilot ก่อน, เน้น use case ที่เห็นประโยชน์ชัด (ค้นระเบียบ) |
| GPU server ไม่พอ | สูง | ปานกลาง | ใช้ CPU-only mode ได้ (ช้ากว่า), หรือใช้ cloud GPU |
| PDPA compliance issues | สูง | ต่ำ | ทำ Data Protection Impact Assessment (DPIA) ก่อน Phase 3 |

---

## Budget Estimation (ค่าใช้จ่ายหลัก)

| รายการ | ค่าใช้จ่ายต่อเดือน | หมายเหตุ |
|--------|-----------------|---------|
| Cloud LLM API (Claude/GPT-4o) | ~$500-2,000 | ขึ้นกับ usage, เฉลี่ย ~1,000 queries/day |
| Cloud Embedding API (Gemini) | ~$50-200 | สำหรับ document ใหม่ที่ไม่ได้ embed locally |
| Server (3 machines) | ขึ้นกับ NT infra | ถ้าใช้ cloud ~$1,500-3,000/mo |
| GPU Server (on-prem LLM) | ขึ้นกับ NT infra | ถ้าซื้อ GPU ~$10,000-15,000 one-time |
| Software licenses | $0 | ทั้งหมด open-source |
| **รวมต่อเดือน (ประมาณ)** | **$2,000-5,000** | ไม่รวม hardware one-time |

---

## สิ่งที่ต้องตัดสินใจก่อนเริ่ม

1. **Server:** จัดหา on-premise หรือใช้ cloud? (กระทบ budget + timeline Phase 0)
2. **NT Auth:** ระบบ auth ปัจจุบันใช้อะไร? (กระทบ Phase 3 complexity)
3. **Pilot departments:** เลือกฝ่ายไหนเป็น pilot? (กระทบ Phase 5 document preparation)
4. **GPU:** มี GPU server พร้อมใช้ไหม? หรือต้องจัดหา? (กระทบ on-prem LLM quality)
5. **Team:** มีทีม dev พร้อมกี่คน? timeline อาจยืดถ้าทีมเล็กกว่า 5 คน
6. **เอกสาร:** มี digital archive อยู่แล้วไหม? หรือต้องสแกนใหม่? (กระทบ Phase 2 effort)
