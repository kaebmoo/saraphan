# Saraphan — Phase 2: Thai Language Optimization

← กลับไป [[00_MASTER_PLAN]]  
← ก่อนหน้า [[03_PHASE_0_1_CORE]]  
→ ถัดไป [[05_PHASE_3_SECURITY]]  
→ Cross-ref data จะถูกใช้ต่อใน [[07_PHASE_5_GRAPHRAG#Task 5.2.2]]

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

### Task 2.3: Hybrid Search + Query Expansion (สัปดาห์ 8-9)

Vector search อย่างเดียวไม่พอสำหรับเอกสารกฎหมาย/ระเบียบ ต้องผสม keyword search + query expansion เพื่อเพิ่ม recall

```
User Query: "ระเบียบการลาพักร้อน"
    │
    ▼
[Query Expansion — ใช้ On-Prem LLM สร้าง 2 variants]
    │
    ├── Original: "ระเบียบการลาพักร้อน" (weight ×2)
    ├── Variant 1: "สิทธิ์วันลาพักผ่อนประจำปี เงื่อนไขการลา"
    └── Variant 2: "การอนุมัติลาพักร้อน จำนวนวันลา"
    │
    ▼
[แต่ละ query ค้นทั้ง 2 backends]
    │
    ├── BM25 (Qdrant FTS, Thai tokenizer) → top 20 per query
    └── Vector (BGE-M3 cosine) → top 20 per query
    │
    ▼
[RRF Fusion — Reciprocal Rank Fusion]
    score = Σ(1/(k+rank+1)) where k=60
    Original query weight ×2 (ให้ความสำคัญกับ query เดิม)
    Top-rank bonus: doc ที่ rank #1 ใน list ใดก็ตาม +0.05
    → เก็บ top 30 candidates
```

- [ ] ตั้ง Qdrant BM25 full-text search (ใช้ native Thai tokenizer จาก v1.15+)
- [ ] สร้าง query expansion service:
  - ใช้ on-prem LLM (Ollama) สร้าง 2 variants จาก original query
  - Prompt: "สร้างคำค้นหา 2 แบบที่ต่างจากต้นฉบับ เน้นคำ synonym และมุมมองต่าง"
  - Cache: query เดิม → ใช้ variants ที่ cache ไว้ (Redis, TTL 1 hr)
  - ถ้า LLM ช้า/ไม่ตอบ → fallback ใช้ original query อย่างเดียว
- [ ] สร้าง hybrid search endpoint:
  1. Query expansion → 3 queries (original ×2 + 2 variants)
  2. แต่ละ query ค้นทั้ง Vector search (BGE-M3) + BM25
  3. รวม 6 result lists ด้วย RRF (k=60)
  4. Top-rank bonus: +0.05 สำหรับ doc ที่ rank #1 ใน list ใดก็ตาม
  5. เก็บ top 30 candidates ส่งต่อให้ reranker
- [ ] ทดสอบ 4 strategies: vector only, BM25 only, hybrid (no expansion), hybrid + expansion
- [ ] วัด MRR@10, Recall@10 บน golden set
- [ ] ปรับ weight: original query multiplier, RRF k value, top-rank bonus
- **ผู้รับผิดชอบ:** Backend 1
- **เครื่องมือ:** Qdrant (built-in BM25 + Thai tokenizer), Redis (expansion cache)
- **ต้องเขียนเอง:** hybrid search + query expansion + RRF ~400-500 lines

### Task 2.4: Re-ranking + Position-Aware Blend (สัปดาห์ 9)

```
[Top 30 candidates จาก Task 2.3 RRF]
    │
    ▼
[Cross-Encoder Re-ranker (BGE-reranker-v2-m3)]
    แต่ละ doc ได้ rerank score 0.0-1.0
    │
    ▼
[Position-Aware Blend — ผสม RRF score กับ reranker score]
    │
    ├── RRF rank 1-3:   final = 0.75 × RRF + 0.25 × reranker
    │   (เชื่อ retrieval มากกว่า — exact match ที่ RRF ให้ rank สูง ไม่ควรถูก reranker ดึงลง)
    │
    ├── RRF rank 4-10:  final = 0.60 × RRF + 0.40 × reranker
    │   (balanced — ให้ reranker ช่วยปรับลำดับ)
    │
    └── RRF rank 11-30: final = 0.40 × RRF + 0.60 × reranker
        (เชื่อ reranker มากกว่า — doc ที่ RRF ให้ rank ต่ำ ถ้า reranker เห็นว่าดีจริง ให้ขึ้นมาได้)
    │
    ▼
[Sort by final score → top 10 → ส่ง LLM generate]
```

**ทำไมต้อง Position-Aware Blend:**
Pure reranking อาจทำลาย exact match ที่ดีอยู่แล้ว เช่น user ถามชื่อระเบียบตรงๆ BM25 ให้ rank #1 ถูกต้อง แต่ reranker อาจให้คะแนนต่ำกว่า doc อื่นที่เนื้อหาคล้ายกว่า Position-aware blend ป้องกันกรณีนี้

- [ ] เพิ่ม cross-encoder re-ranker:
  - ตัวเลือก 1: BGE-reranker-v2-m3 (multilingual, open-source) — แนะนำ
  - ตัวเลือก 2: Cohere Rerank (cloud API, ง่ายกว่า แต่ต้อง route ผ่าน cloud)
- [ ] Implement position-aware blend ตาม formula ด้านบน
- [ ] ทดสอบ 3 configs:
  - Hybrid search (no reranker)
  - Hybrid + pure reranker (100% reranker score)
  - Hybrid + position-aware blend
- [ ] วัด improvement บน golden set — target: MRR@10 > 0.85
- [ ] ปรับ blend weights ตาม evaluation results
- **ผู้รับผิดชอบ:** Backend 1
- **ต้องเขียนเอง:** reranker integration + position-aware blend ~200-300 lines

### Task 2.5: Cross-Reference Extraction จากเอกสารระเบียบ/คำสั่ง (สัปดาห์ 10)

**เตรียมข้อมูลสำหรับ GraphRAG ใน Phase 5** — ทำตอนนี้เพราะต้องทำพร้อมกับ chunking pipeline

ระเบียบ/คำสั่ง NT มักมี pattern การอ้างอิงที่แยกออกได้ด้วย regex + rules:
- "ตามคำสั่งที่ XXX/25XX"
- "ให้เป็นไปตามระเบียบว่าด้วย..."
- "ยกเลิกคำสั่งที่ XXX/25XX"
- "แก้ไขเพิ่มเติม ข้อ X ของระเบียบ..."
- "อาศัยอำนาจตาม..."

- [ ] สร้าง cross-reference extractor (regex + pattern matching):
  - Input: chunked text จาก regulatory documents
  - Output: list of references [{source_doc, target_doc_ref, relation_type, section}]
  - relation_type: cites | supersedes | amends | delegates_to | refers_to
- [ ] เก็บ extracted references ลง `document_references` table:
  ```sql
  CREATE TABLE document_references (
      id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      source_doc_id   UUID REFERENCES documents(id),
      target_doc_ref  TEXT NOT NULL,        -- "คำสั่งที่ 123/2568" (raw text reference)
      target_doc_id   UUID REFERENCES documents(id),  -- resolved (NULL ถ้ายังไม่ resolve)
      relation_type   VARCHAR(20) NOT NULL, -- cites, supersedes, amends, delegates_to, refers_to
      source_section  TEXT,                 -- "ข้อ 5", "หมวด 3"
      target_section  TEXT,                 -- section ที่อ้างถึง
      confidence      FLOAT DEFAULT 1.0,    -- 1.0 = regex match, <1.0 = LLM inferred
      created_at      TIMESTAMPTZ DEFAULT NOW()
  );
  CREATE INDEX idx_doc_refs_source ON document_references(source_doc_id);
  CREATE INDEX idx_doc_refs_target ON document_references(target_doc_id);
  ```
- [ ] สร้าง reference resolver: จับคู่ target_doc_ref กับ documents ที่มีอยู่ในระบบ
  - exact match: เลขคำสั่ง/ระเบียบ
  - fuzzy match: ชื่อระเบียบ (ใช้ Thai word similarity)
  - unresolved: เก็บ raw text ไว้ resolve ทีหลังเมื่อเอกสารถูกเพิ่มเข้ามา
- [ ] Enrich chunk metadata: เพิ่ม `references_out` และ `referenced_by` ลง Qdrant metadata
  → ใช้ปรับปรุง retrieval ได้ทันที (boost chunks ที่ถูกอ้างอิงบ่อย)
  → Phase 5 จะนำ reference graph นี้ไปสร้าง knowledge graph ใน Neo4j
- **ผู้รับผิดชอบ:** Backend 2
- **เครื่องมือ:** regex + PyThaiNLP (สำหรับ normalization)
- **ต้องเขียนเอง:** extractor + resolver ~400-500 lines
- **หมายเหตุ:** ข้อมูลนี้มีประโยชน์ทันทีแม้ไม่มี GraphRAG — ใช้ boost retrieval score ของเอกสารที่ถูกอ้างอิงมากได้

### Task 2.6: Evaluation Framework — RAGAS (สัปดาห์ 10)

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

