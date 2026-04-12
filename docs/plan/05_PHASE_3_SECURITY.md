# Saraphan — Phase 3: Security & Access Control Implementation

← กลับไป [[00_MASTER_PLAN]]  
← ก่อนหน้า [[04_PHASE_2_THAI]]  
→ ถัดไป [[06_PHASE_4_PRODUCTION]]  
→ Design spec ดูที่ [[01_SECURITY_DESIGN]]  
→ Database tables ดูที่ [[02_DATABASE_SCHEMA]]  
→ Frontend pages ดูที่ [[08_FRONTEND_PLAN#Phase 3 Pages]]

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

