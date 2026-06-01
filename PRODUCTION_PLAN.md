# แผนนำขึ้น Production — สถานะการปิดงาน (Production Plan)

**โครงการ:** Chanthasy Stock — ระบบจัดการสต็อกต้นไม้
**Stack:** React 18 + Vite + Supabase + Vercel
**แผนเดิมเขียนเมื่อ:** 2026-05-22
**อัปเดตสถานะล่าสุด:** 2026-06-01

> 📌 เอกสารนี้คือ **แผนตั้งต้น** ที่ใช้ไล่ปิดงานก่อนเปิด production
> สำหรับ **สถานะรายละเอียดปัจจุบัน + งานที่เหลือ + roadmap** ให้ดูที่ [`REVIEW.md`](./REVIEW.md) ซึ่งเป็นเอกสารสถานะหลัก
> เอกสารนี้เก็บไว้เพื่อบันทึกว่าแผนเดิมแต่ละข้อปิดไปแล้วหรือยัง

---

## สรุปความคืบหน้า

แผนเดิมมี 38 ข้อ (Critical 8 / High 10 / Medium 12 / Low 8)
**ปิดไปแล้วเกือบทั้งหมด** — เหลือเพียงงานที่ต้องพึ่งการจัดซื้อ/ตั้งค่าภายนอก และงานขัดเกลา

| ระดับ | ปิดแล้ว | เหลือ |
|-------|---------|-------|
| 🔴 Critical | 7 / 8 | C8 (domain + Resend) |
| 🟠 High | 10 / 10 | — |
| 🟡 Medium | 9 / 12 | M2, M4 (บางส่วน), M10/M11 (ตรวจซ้ำ) |
| 🟢 Low | 4 / 8 | L2, L4, L7, L8 (บางส่วน) |

---

## 🔴 Critical

| # | งาน | สถานะ | หลักฐาน |
|---|-----|-------|---------|
| C1 | UNIQUE per-owner (multi-tenant) | ✅ | migration `001_production_hardening.sql` |
| C2 | schema/migrations ตรงกับ DB จริง | ✅ | migrations 001–019 |
| C3 | CalendarPage ใช้ owner/store id | ✅ | multi-store cutover (mig 006–008), filter `store_id` |
| C4 | Realtime subscriptions กรอง owner/store | ✅ | ทุกหน้าใช้ `filter: store_id=eq.${ownerId}` |
| C5 | User-friendly error mapper | ✅ | `src/lib/errors.ts` (`userMessage`) |
| C6 | Server-side role enforcement (RLS) | ✅ | helpers `can_write`, `can_delete`, `is_admin`, `effective_owner_id` |
| C7 | Password strength | ✅ | `errors.ts` `validatePassword` (≥8 + ตัวเลข + ตัวอักษร) |
| **C8** | **Domain verification สำหรับอีเมล** | ⏳ **เหลือ** | sender ยัง fallback `onboarding@resend.dev` — ต้องซื้อโดเมน + verify Resend + ตั้ง `FROM_EMAIL` |

---

## 🟠 High — ปิดครบแล้ว ✅

| # | งาน | สถานะ | หลักฐาน |
|---|-----|-------|---------|
| H1 | Currency sync to DB | ✅ | sync ทั้งสองทางใน profiles |
| H2 | Topbar search ไม่หาย/ไม่ spam | ✅ | `<form onSubmit>` (Enter-to-submit) `Topbar.jsx:74` |
| H3 | Onboarding skip สำหรับ staff | ✅ | `Layout.jsx` เช็คสมาชิก store ก่อน |
| H4 | Image size limit / compression | ✅ | `src/lib/image.ts` + test |
| H5 | ลบรูป cleanup ใน Storage | ✅ | `StockPage.jsx` remove + log error |
| H6 | Dashboard realtime ครบ table | ✅ | subscribe plants + movements |
| H7 | Password reset link (dynamic) | ✅ | `LoginPage.jsx:46` dynamic redirect |
| H8 | ปุ่ม Confirm กันกดซ้ำ | ✅ | `Confirm.jsx` busy state + disabled |
| H9 | Role / member migration | ✅ | `store_members` (mig 006–008) |
| H10 | Admin เห็น email ของ user | ✅ | RPC join `auth.users` |

---

## 🟡 Medium

| # | งาน | สถานะ | หมายเหตุ |
|---|-----|-------|---------|
| M1 | ลบ BillingPage (dead code) | ✅ N/A | Billing ถูกนำกลับมาเป็นฟีเจอร์จริง (`BillingCard`, Stripe) |
| **M2** | **ลบ `tree-stock-enterprise/` + `tree-stock-tmp/`** | ⏳ **เหลือ** | ยังอยู่ใน working tree (untracked — ไม่อยู่ใน git) ลบได้เลย |
| M3 | แยก inline styles → utility classes | ✅ | 2 batches |
| **M4** | **TypeScript migration** | 🔶 **บางส่วน** | `src/lib/` เป็น `.ts` แล้ว (11 ไฟล์) — `pages/` (25) + `components/` ยังเป็น `.jsx` |
| M5 | Linter / Formatter | ✅ | ESLint zero warnings + CI |
| M6 | Custom hook ลด duplicate | ✅ | `usePagination`, `useFilteredList` |
| M7 | Loading skeletons | ✅ | `Skeleton.jsx` |
| M8 | ReportsPage กรองช่วงเวลา | ✅ | `report_stats` RPC (mig 018) |
| M9 | Empty state สำหรับ Calendar | ✅ | `EmptyState.jsx` |
| M10 | Topbar shopname ตาม admin view | 🔍 ตรวจซ้ำ | ควรยืนยันใน admin view mode |
| M11 | `useMemo` ใน DashboardPage | 🔍 ตรวจซ้ำ | minor perf |
| M12 | Realtime unsubscribe ตอน switch shop | ✅ | channel name ต่อ `ownerId` |

---

## 🟢 Low

| # | งาน | สถานะ | หมายเหตุ |
|---|-----|-------|---------|
| L1 | PWA / Add to Home Screen | ✅ | vite-plugin-pwa + manifest |
| **L2** | **Custom Domain** | ⏳ **เหลือ** | ผูกกับ C8 (ต้องมีโดเมนก่อน) |
| L3 | Error Tracking (Sentry) | ✅ | `src/lib/sentry.js` (no-op ถ้าไม่มี DSN) |
| **L4** | **Analytics (PostHog ฯลฯ)** | ⏳ เหลือ (optional) | ยังไม่ติดตั้ง |
| L5 | 2FA / MFA | ✅ | `MfaEnroll`, `MfaChallengePage` |
| L6 | Data Export (PDPA) | ✅ | Settings → ดาวน์โหลด JSON backup |
| **L7** | **Cookie consent / PDPA banner** | ⏳ เหลือ | จำเป็นถ้าเปิด analytics (L4) |
| **L8** | **Tests** | 🔶 บางส่วน | มี unit + บาง page test — ขาด integration (auth flow, role boundary, multi-tenant) |

---

## งานที่เหลือจริง (สรุป)

1. 🔴 **C8 / L2** — ซื้อ custom domain → verify Resend → ตั้ง `FROM_EMAIL` (งานภายนอก)
2. 🟡 **M2** — ลบโฟลเดอร์ค้าง `tree-stock-enterprise/`, `tree-stock-tmp/`
3. 🟡 **M4** — TypeScript migration ต่อ (pages + components)
4. 🟢 **L4 + L7** — analytics + cookie consent banner (ทำคู่กัน)
5. 🟢 **L8** — integration tests
6. 🔍 **M10/M11** — ตรวจซ้ำ (งานเล็ก)

นอกเหนือจากนี้คือฟีเจอร์ขยายเชิงธุรกิจ (B8 multi-warehouse, B9 batch/lot, B20 LINE OA ฯลฯ) — ดูรายละเอียดใน [`REVIEW.md`](./REVIEW.md)

---

**สรุป:** แผน production เดิมถูกปิดไปแล้วเกือบ 90% — เส้นทางสู่การเปิดใช้จริงตอนนี้เหลือ **งานจัดซื้อ/ตั้งค่าภายนอก** (โดเมน, Resend, Stripe live) เป็นหลัก ส่วนงานโค้ดที่เหลือเป็นการขัดเกลาที่ไม่บล็อกการเปิดใช้
