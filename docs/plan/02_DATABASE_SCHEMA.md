# Saraphan — Database Schema

← กลับไป [[00_MASTER_PLAN]]  
→ User roles & classification ดูที่ [[01_SECURITY_DESIGN]]  
→ GraphRAG schema (Neo4j) ดูที่ [[07_PHASE_5_GRAPHRAG#Task 5.2.1]]

---

## PostgreSQL Schema

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

### เพิ่มเติม: Cross-Reference Table (สร้างใน Phase 2, ใช้ใน Phase 5 GraphRAG)

```sql
CREATE TABLE document_references (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_doc_id   UUID REFERENCES documents(id),
    target_doc_ref  TEXT NOT NULL,        -- "คำสั่งที่ 123/2568" (raw text reference)
    target_doc_id   UUID REFERENCES documents(id),  -- resolved (NULL ถ้ายังไม่ resolve)
    relation_type   VARCHAR(20) NOT NULL, -- cites, supersedes, amends, delegates_to, refers_to
    source_section  TEXT,                 -- "ข้อ 5", "หมวด 3"
    target_section  TEXT,
    confidence      FLOAT DEFAULT 1.0,    -- 1.0 = regex match, <1.0 = LLM inferred
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_doc_refs_source ON document_references(source_doc_id);
CREATE INDEX idx_doc_refs_target ON document_references(target_doc_id);
```

---

## Qdrant Collections

ดู [[01_SECURITY_DESIGN#6. Data Handling Rules]] สำหรับกฎการเก็บ vector ตามชั้นความลับ

```
Qdrant Collections:
├── saraphan_public       ← Level 1: ทุกคนค้นได้
├── saraphan_internal     ← Level 2: metadata filter by department
├── saraphan_finance      ← Level 3: Director+ ฝ่ายการเงิน
├── saraphan_hr           ← Level 3: Director+ ฝ่าย HR
├── saraphan_legal        ← Level 3: Director+ ฝ่ายกฎหมาย
├── saraphan_executive    ← Level 4: Executive เท่านั้น (encrypted)
└── saraphan_{dept}       ← สร้างเพิ่มตาม department ที่มี Level 3+
```

Vector dimension: 1024 (BGE-M3)  
Distance metric: Cosine  
Metadata per chunk: doc_id, department_id, classification, allowed_departments, references_out, referenced_by_count

---

## Neo4j Graph Schema (Phase 5)

ดูรายละเอียดเพิ่มที่ [[07_PHASE_5_GRAPHRAG#Task 5.2.1]]

ใช้เฉพาะเอกสารระเบียบ/คำสั่ง ไม่ใช้กับ SOP/รายงานการเงิน

```
Nodes: Regulation, Order, Section, Organization, Position, Topic
Edges: CITES, SUPERSEDES, AMENDS, CONTAINS, DELEGATES_TO, APPLIES_TO, RELATES_TO, ISSUED_BY
```
