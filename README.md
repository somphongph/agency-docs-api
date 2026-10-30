# Agency Dashboard — API collection & docs

## Bruno collection

โฟลเดอร์ `agency-docs-api/` เป็น Bruno collection ในรูปแบบ **OpenCollection YAML** (Bruno 4.x)
ใช้ทั้งยิง API ด้วยมือและรันชุดเทสต์สัญญา API

```
agency-docs-api/
├─ opencollection.yml          # collection: "Agency Dashboard API" + default environment = local
├─ environments/local.yml      # baseUrl = http://localhost:8080
├─ dashboard/get-dashboard.yml # GET  /api/v1/dashboard          + 8 assertions
├─ dashboard/refresh-dashboard.yml # POST /api/v1/dashboard/refresh + 4 assertions
├─ manual/stream-dashboard.yml # GET  /api/v1/dashboard/stream (SSE, app only — ไม่รันใน CI)
├─ README.md
└─ screenshots/
```

> เดิม collection นี้อยู่ในรูปแบบ legacy `.bru` (Bruno v1) แล้วถูก migrate เป็น YAML
> เพื่อให้เปิด/แก้ไขได้ใน Bruno 4.x — ตัว request และ assertions ยังคงเดิมทุกข้อ

### ใช้งานผ่าน Bruno app

1. เปิด Bruno → **Open Collection** → เลือกโฟลเดอร์ `agency-docs-api`
2. เลือก environment **local**
3. กด **Send** ที่ request `dashboard / Get Dashboard`
4. ดูผล assertions ที่แท็บ **Tests**

### ใช้งานผ่าน CLI (ไม่ต้องเปิด app)

```bash
# ต้องรันที่ root ของ collection และ backend ต้องรันอยู่ (port 8080)
cd agency-docs-api
npx @usebruno/cli run dashboard            # รันเฉพาะ request ที่จบได้ (get + refresh) — ใช้ใน CI
npx @usebruno/cli run dashboard --env local # ระบุ environment เองก็ได้ ผลเหมือนกัน
```

### สิ่งที่ collection ตรวจ (`dashboard/get-dashboard.yml`)

| Test | สิ่งที่ยืนยัน |
|---|---|
| responds with 200 and JSON | endpoint พร้อมใช้งานและ content-type ถูกต้อง |
| is not cached | `Cache-Control: no-store` (กัน browser/CDN cache ข้อมูลเก่า) |
| standard envelope | ทุก response ห่อด้วย `{ success, type, message, data }` และ payload อยู่ใน `data` |
| Asia/Bangkok offset | `data.updatedAt` ลงท้าย `+07:00` และ parse เป็นวันที่ได้ |
| regions + OVERALL | 8 region ตามลำดับ BKK-A … Other และมีแถว OVERALL |
| cases / ANP | case เป็นจำนวนเต็ม ≥ 0, ANP ≥ 0 และมีทศนิยม 2 ตำแหน่ง |
| daily ≤ mtd ≤ ytd | ค่าที่เล็กกว่ารวมอยู่ในค่าที่ใหญ่กว่าเสมอ (ทั้ง Case และ ANP) |
| OVERALL = Σ region | ทุกกลุ่ม × ทุกช่วง ตรงกับผลรวมของ 8 region แบบ exact (คิดเป็นสตางค์/ร้อยละของ mb) |

และ `dashboard/refresh-dashboard.yml` (POST) ตรวจว่า: ตอบ 200 + envelope, snapshot ครบ 8 region,
`updatedAt` เป็น +07:00 และ invariant `OVERALL = Σ region` ยังถูกต้องหลังสุ่มใหม่ (4 assertions)

> `GET` คืน snapshot เดิมทุกครั้ง (กด Send ซ้ำตัวเลขจะเท่าเดิม) — ตัวเลขจะเปลี่ยนเมื่อยิง
> `POST /api/v1/dashboard/refresh` ซึ่งจะ push event ไปยังทุกเบราว์เซอร์ที่ต่อ
> `GET /api/v1/dashboard/stream` (SSE) ด้วย

## SSE (live updates)

หน้าเว็บไม่ poll แล้ว แต่ต่อ `GET /api/v1/dashboard/stream` ค้างไว้ (event name `dashboard`,
payload เป็น envelope เดียวกัน) ลองดูด้วย curl:

```bash
curl -N -H 'Accept: text/event-stream' http://localhost:8080/api/v1/dashboard/stream
# อีก terminal: ยิง refresh แล้วดู event ใหม่โผล่ใน stream
curl -X POST http://localhost:8080/api/v1/dashboard/refresh
```

- **Bruno app**: เปิด `manual / Stream Dashboard (SSE)` → Send → จะเห็น event ไหลเข้ามา realtime
  (กด Cancel เพื่อปิด) ไฟล์นี้อยู่ใน `manual/` และไม่มี assertion เพราะ stream ไม่มีจุด "จบ"
- **Bruno CLI 4.3.0**: ไม่ skip request ที่เป็น SSE (ทดสอบแล้ว: รันแล้วค้างจน timeout) จึงต้องรัน
  เฉพาะ `dashboard` — `npx @usebruno/cli run dashboard`
- **CI/สคริปต์**: ใช้ curl ข้างบน หรือ parseInt จาก `POST /refresh` แทน

## ภาพหน้าจอ

| สถานะปกติ (อัปเดตแบบ live ผ่าน SSE) | เมื่อ API ล่ม: คงข้อมูลล่าสุด + แจ้งเตือน |
|---|---|
| ![Dashboard](screenshots/dashboard.png) | ![Error state](screenshots/dashboard-error.png) |

## Decision log (สรุปเหตุผลการออกแบบ)

- **คำนวณ OVERALL ที่ Backend** — เป็น single source of truth, ทุก client ได้ค่าเดียวกัน,
  กติกาปัดเศษอยู่ที่เดียว และเทสต์ invariant ฝั่ง server ได้ตรง
- **ANP เก็บเป็น integer hundredths แล้วแปลงเป็น `BigDecimal` scale 2** — ผลรวมจึงไม่มี
  floating-point drift และค่าที่แสดงบนจอรวมกันได้เท่ากับ OVERALL เป๊ะ
- **`daily ≤ mtd ≤ ytd`** — สุ่ม YTD ก่อน แล้วให้ MTD/Daily เป็นสัดส่วนของช่วงที่ใหญ่กว่า
  ทำให้ข้อมูลจำลองยังดูสมเหตุสมผลโดยไม่ต้องมี DB
- **Polling ด้วย `timer` + `exhaustMap`** — กัน request ซ้อนเมื่อ API ช้า และ `catchError`
  อยู่ภายใน inner observable ทำให้ stream หลักไม่ตาย เมื่อ API กลับมาจึงอัปเดตต่อเอง
- **`takeUntilDestroyed` + route-level provider** — service ถูกสร้างและถูกทำลายพร้อม route
  Dashboard ออกจากหน้าจึงหยุด poll ทันที (มี unit test ยืนยัน)
- **เก็บข้อมูลล่าสุดที่สำเร็จไว้** — error ไม่ล้าง `data`/`lastUpdated` แต่ขึ้น banner และ
  เคลียร์ banner อัตโนมัติเมื่อรอบถัดไปสำเร็จ
