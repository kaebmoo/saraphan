# Saraphan — Security, Privacy & Access Control Design

← กลับไป [[00_MASTER_PLAN]]  
→ Implement ใน [[05_PHASE_3_SECURITY]]  
→ Database tables ใน [[02_DATABASE_SCHEMA]]  
→ Frontend UI ใน [[08_FRONTEND_PLAN#Security & Admin Pages]]

---

เอกสารนี้เป็น design document ที่ [[05_PHASE_3_SECURITY]] จะ implement ตาม  
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
