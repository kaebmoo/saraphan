# Saraphan — Phase 0-1: Infrastructure + Core RAG Pipeline

← กลับไป [[00_MASTER_PLAN]]  
→ ถัดไป [[04_PHASE_2_THAI]]  
→ Database schema ดูที่ [[02_DATABASE_SCHEMA]]  
→ Frontend spec ดูที่ [[08_FRONTEND_PLAN#Phase 1 Pages]]

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

