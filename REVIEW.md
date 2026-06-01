# รายงานสถานะโครงการ Chanthasy Stock

**โครงการ:** Chanthasy Stock — ระบบจัดการสต็อกต้นไม้ (Multi-tenant SaaS)
**Stack:** React 18 + Vite + Supabase (Postgres + Auth + Storage + Edge Functions) + Vercel (primary) + GitHub Pages (live mirror)
**อัปเดตล่าสุด:** 2026-06-01
**ผู้ตรวจ:** Claude (Opus 4.8)
**ขอบเขต:** เทียบสถานะปัจจุบันกับรายการงานเดิมใน REVIEW (2026-05-24) และ `PRODUCTION_PLAN.md` (2026-05-22)

---

## สรุปภาพรวม

โครงการเดินหน้ามาไกลมาก **งาน Critical / High / Business เกือบทั้งหมดจากรีวิวรอบก่อนถูกปิดไปแล้ว**
ฐานข้อมูลมี migration ถึง **019** และมีฟีเจอร์เชิงพาณิชย์ครบระดับ MVP (billing, VAT, invoice PDF, customers, PO, audit log, i18n, landing page)

สถานะปัจจุบัน: **"พร้อม production ในเชิงเทคนิค — เหลือแค่งานตั้งค่าภายนอก + ขัดเกลา + ฟีเจอร์ขยาย"**

ระดับความสำคัญที่ใช้ในรายงาน:

| ระดับ | ความหมาย |
|---|---|
| 🔴 Critical | ต้องแก้ก่อนรับผู้ใช้จริง — security / data loss risk |
| 🟠 High | กระทบการใช้งานรายวันหรือความน่าเชื่อถือ |
| 🟡 Medium | คุณภาพโค้ด / DX / UX ที่ควรปรับปรุง |
| 🟢 Low | Nice-to-have สำหรับเติบโตในระยะถัดไป |

---

## ✅ ส่วนที่ทำเสร็จแล้วตั้งแต่รีวิวรอบก่อน

### Code & Security
- **C1 — `adjust_stock` ตรวจสิทธิ์ tenant/role แล้ว** (migration `004_secure_adjust_stock.sql`) ✅
- **C2 — schema/migrations เป็นชุดเดียวที่ตรงกับ production** (migration 001–019) ✅
- **C3 — Dead code prototype ที่ root (`app.jsx`, `data.jsx`, ฯลฯ) ถูกลบแล้ว** ✅
- **C4 — Password reset ใช้ dynamic redirect** (`LoginPage.jsx:46` `${window.location.origin}/reset-password`) ✅
- **H1 — Topbar search ใช้ `<form onSubmit>` (Enter-to-submit)** ไม่ navigate ทุก keystroke แล้ว ✅
- **H2 — MovementsPage realtime เป็น `event: '*'` + filter `store_id`** ✅
- **H5 — Storage cleanup log error ผ่าน `console.error`** ไม่เงียบแล้ว ✅
- **H6 — OnboardingWizard ถูกทำให้เรียบง่าย** (ไม่ save DB เองแล้ว ลดความเสี่ยง error เงียบ) ✅
- **M6 — ย้ายไป `BrowserRouter` (ทิ้ง HashRouter)** URL สะอาด ✅
- **M7 — AAL gating ตาม event type** ✅
- **M8 — Supabase warn เป็น dev-only** ✅
- Lint สะอาด (zero warnings), RLS helper functions (`effective_owner_id`, `can_write`, `can_delete`, `is_admin`), MFA, PWA, skeleton loaders ✅

### Business Features
- **B1 — Billing / Subscription** (Stripe checkout + webhook + portal, migration 019) ✅
- **B2 — VAT settings** (migration 005) ✅
- **B3 — Invoice / Receipt PDF** (`@react-pdf/renderer`, lazy-loaded) ✅
- **B4 — Customer database** (migration 013, `CustomersPage`) ✅
- **B5 — Bulk Import** (`BulkImport.jsx`) ✅
- **B6 — Barcode / QR Scanner** (`BarcodeScanner.jsx`) ✅
- **B7 — Purchase Order workflow** (migration 014, `PurchaseOrdersPage`) ✅
- **B14 — Admin Audit Log** (migration 016/017, `AuditLogPage`) ✅
- **B16 — i18n** (`react-i18next`, `LanguageSwitcher`) ✅
- **B17 — Landing page** (`LandingPage` — pricing CTA, testimonials, FAQ, contact form) ✅
- เพิ่มเติม: Multi-store / สาขา (migration 006–008), Daily settlement (migration 009), report_stats RPC (migration 018) ✅

### Infrastructure
- **I2 — Sentry error monitoring** (`src/lib/sentry.js`, no-op ถ้าไม่มี DSN) ✅
- **I3/I4 — Vercel เป็น primary host** → preview deploy ต่อ PR + รองรับ SPA fallback; เก็บ GitHub Pages เป็น live mirror (deploy.yml auto บน push) ✅
- **M4 — CI workflow** (lint + test, GitHub Actions Node.js 24) ✅

---

## ⏳ งานที่ยังเหลือ

### 🔴 Critical — ต้องทำก่อนรับผู้ใช้จริงในวงกว้าง

#### C5. Email sender ยัง fallback เป็น `onboarding@resend.dev`

**ไฟล์:** `supabase/functions/auth-email-hook/index.ts:16`, `notify-low-stock/index.ts:6`, `submit-contact/index.ts:42`

โค้ดรองรับ env `FROM_EMAIL` แล้ว แต่ค่า default ยังเป็น `onboarding@resend.dev` (domain ทดสอบของ Resend) — Gmail/Outlook มักโยนเข้า spam หรือ block

**แก้ (งานภายนอก + ตั้งค่า):**
1. ซื้อโดเมน (เช่น `chanthasy.com`)
2. Verify ที่ resend.com/domains
3. ตั้ง env `FROM_EMAIL=noreply@chanthasy.com` ใน Supabase Edge Function secrets

#### Stripe — ตรวจว่าใช้ live keys จริงหรือยัง

โค้ดอ่านจาก env (`STRIPE_SECRET_KEY`, `STRIPE_PRICE_PRO`, `STRIPE_PRICE_BUSINESS`) ไม่มี key hardcode (ดีแล้ว) — แต่ต้องยืนยันว่าตั้งค่า **live keys + price IDs จริง** ก่อนเปิดรับเงิน และทดสอบ webhook end-to-end

---

### 🟠 High

#### I1. ยังไม่มี Custom Domain
ยังใช้ URL เริ่มต้นของ Vercel/GitHub Pages — ไม่ professional + ผูกกับ C5 (ต้องมีโดเมนก่อนถึง verify Resend ได้)

#### I3. Backup / Restore Drill
Supabase Free มี daily backup 7 วัน แต่ยังไม่เคย restore จริงเพื่อทดสอบ — แนะนำ export ทุก table ลง R2/S3 รายสัปดาห์ + manual restore drill ไตรมาสละครั้ง

---

### 🟡 Medium

#### M2. TypeScript migration ทำแค่บางส่วน
- `src/lib/` เป็น `.ts` แล้ว (11 ไฟล์) ✅
- `src/pages/` (25 ไฟล์) และ `src/components/` ยังเป็น `.jsx` ทั้งหมด → ค่อยๆ migrate ต่อ + ใช้ `supabase gen types typescript`

#### M5. Test coverage ยังเน้น utility + บาง page
มี `errors.test.js`, `image.test.js`, `utils.test.js`, `AdminPage.test.jsx`, `StockPage.test.jsx`, `SuppliersPage.test.jsx` — ยังขาด integration test สำหรับ:
- Auth flow (sign-in, MFA challenge)
- Role boundary (viewer อ่านได้/เขียนไม่ได้)
- Multi-tenant isolation (เห็นเฉพาะข้อมูล store ตัวเอง)

#### Repo hygiene — โฟลเดอร์ค้าง
`tree-stock-enterprise/` และ `tree-stock-tmp/` ยังอยู่ใน working tree (**untracked — ไม่ได้อยู่ใน git**) เป็นโปรเจกต์เก่า/temp ลบทิ้งได้เลยเพื่อความสะอาด

#### L3. Rate Limiting บน Edge Function
`admin-manage-users` / billing functions ตรวจ JWT แล้ว แต่ยังไม่มี rate limit — แนะนำ Upstash Ratelimit

#### I5/I6. Status page + Cookie/PDPA consent banner
ถ้าเปิด analytics ต้องมี PDPA banner; status page ช่วยให้ผู้ใช้รู้ว่าระบบล่มหรือเฉพาะตัวเอง

---

### 🟢 Low / ฟีเจอร์ขยายในอนาคต (ยังไม่แตะ — optional)

- **B8** Multi-warehouse stock per location (ตอนนี้ stock เป็นตัวเลขเดียวต่อ plant ต่อ store)
- **B9** Batch / Lot tracking + FIFO/LIFO + aged inventory
- **B10** Currency conversion จริง (ตอนนี้เปลี่ยนแค่สัญลักษณ์ ไม่แปลงค่า)
- **B11/B12** Promotion/discount, Loyalty/membership
- **B13** Stock adjustment approval flow สำหรับ staff
- **B15** Plant care schedule (recurring) ผูกกับ plant_id
- **B19** Mobile app (Capacitor wrap → Play Store/App Store)
- **B20** LINE OA / Facebook Page integration
- **B21/B22** Image gallery หลายมุม, Plant knowledge base

---

## Roadmap แนะนำ (ปรับใหม่)

### Phase 1 — เปิดใช้จริง (พึ่งงานภายนอกเป็นหลัก)
- [ ] ซื้อ custom domain (I1)
- [ ] Verify Resend + ตั้ง `FROM_EMAIL` (C5)
- [ ] ใส่ Stripe live keys + ทดสอบ webhook end-to-end
- [ ] ตั้ง Sentry DSN + (ถ้าใช้ analytics) PDPA banner (I6)
- [ ] Backup/restore drill ครั้งแรก (I3)

### Phase 2 — ขัดเกลา
- [ ] ลบโฟลเดอร์ค้าง `tree-stock-enterprise/`, `tree-stock-tmp/`
- [ ] Integration tests: auth flow, role boundary, multi-tenant isolation (M5)
- [ ] TypeScript migration ต่อ: pages + components (M2)
- [ ] Rate limiting บน Edge Functions (L3)

### Phase 3 — ขยายเชิงธุรกิจ (ตามความต้องการลูกค้า)
- [ ] B8 Multi-warehouse, B9 Batch/Lot tracking
- [ ] B13 Approval flow, B10 currency conversion จริง
- [ ] B20 LINE OA integration
- [ ] B19 Mobile app

---

**สรุป:** งานพัฒนาหลักเสร็จแล้ว — เส้นทางสู่ production ตอนนี้คือ **งานตั้งค่า/จัดซื้อภายนอก** (domain, Resend, Stripe live) มากกว่างานเขียนโค้ด ส่วนงานโค้ดที่เหลือเป็นการขัดเกลา (tests, TypeScript) และฟีเจอร์ขยายที่เลือกทำตามความต้องการตลาดได้
