# LAVA-B V1 — Build Field Guide

## จาก Ubuntu Server สู่ Safe Auth + Read-only Google Form Discovery

> สถานะ ณ 16 กันยายน 2569 (2026): **SAFE AUTH + WEB INTEGRATION V1 = PASS**
>
> เอกสารนี้บันทึกสิ่งที่ทดลองจริง ปัญหาที่พบ วิธีแก้ เหตุผลของการตัดสินใจ และแผนเดินหน้าต่อ เพื่อใช้เป็นตำราเรียนภาคสนามสำหรับผู้เริ่มทำ browser automation บน Linux

---

## 1. เป้าหมาย

สร้าง LAVA-B (Larval Automation Bot) เป็น automation ขนาดเล็กบน Ubuntu Server สำหรับลดงานกรอก Google Form ที่ทำซ้ำ โดยแบ่งหน้าที่ชัดเจน:

```text
Human = ส่ง URL + Authentication เมื่อ Google ร้องขอ + รับผล
LAVA-B = ตรวจ URL → เปิด Form → อ่านโครงสร้าง → เตรียมข้อมูล/ภาพ → ตรวจ → Submit 1 ครั้ง → ยืนยันผล
```

หลักสำคัญคือ **automation ลดงานซ้ำ แต่ไม่ข้ามระบบรักษาความปลอดภัย** และ **Predict ≠ Permission to Submit**

---

## 2. Guardrails ที่ปักไว้ก่อนเขียนระบบ

- ไม่ฝัง username/password/OTP ลงใน bot
- ไม่ข้าม 2FA หรือ CAPTCHA
- ไม่ใช้ browser ที่ปิด sandbox สำหรับ credential จริง
- URL หรือโครงสร้างฟอร์มไม่ตรงสิ่งที่ยืนยันไว้ → STOP
- ชุดภาพต้องครบตามกติกา มิฉะนั้น STOP
- หนึ่งงานส่งได้เพียงหนึ่งครั้ง: `ONE JOB → ONE SUBMIT → ONE PASS`
- การกด Submit ยังไม่ถือว่า PASS; ต้องเห็น Google confirmation ก่อน
- ระหว่าง Form Discovery: `NO FILL / NO UPLOAD / NO SUBMIT`
- ข้อมูล authentication profile ถือเป็นข้อมูลอ่อนไหว: จำกัด permission และห้ามนำขึ้น Git

---

## 3. สิ่งที่สร้างสำเร็จแล้ว

### 3.1 โครงสร้าง LAVA-B

ใช้ project directory แยกส่วนงาน เช่น `app/`, `config/`, `logs/`, `data/` และ Python virtual environment เพื่อไม่ปะปน dependency กับระบบหลัก

### 3.2 LAVA Core

สร้าง core test ให้เขียน log และรายงานสถานะ ALIVE ได้ เพื่อพิสูจน์ว่า runtime, path และ permission ขั้นพื้นฐานทำงานก่อนเพิ่ม browser automation

### 3.3 Web UI

สร้าง Flask UI ขนาดเล็กบน port 8080 สำหรับรับ Google Form URL และแสดงสถานะอย่างตรงไปตรงมา เช่น URL PASS, FORM DISCOVERED, AUTH PASS/REQUIRED โดยหน้าปัจจุบันยังเป็น READ ONLY

### 3.4 systemd service

นำ Web UI ไปรันด้วย systemd ให้เริ่มอัตโนมัติและ restart เมื่อ process ล้ม พร้อมตรวจด้วย service status และ HTTP 200

### 3.5 Image Pipeline V1

วางหลักว่า original ต้องไม่ถูกแก้/ลบ และสร้าง processed copy ด้วยขั้นตอน:

```text
Original → Auto-orient → max 1600 px → Strip metadata → JPEG Q75 → Upload copy
```

### 3.6 Lightweight GUI

ติดตั้ง XFCE + Xorg + input แบบ minimal และพิสูจน์ว่า graphical session เปิดได้จริงด้วย `startx` เพื่อให้มีทางสำหรับ human authentication เมื่อจำเป็น

### 3.7 Chrome Stable + Sandbox

ติดตั้ง Google Chrome Stable และทดสอบ headless launch แบบ sandbox ปกติสำเร็จ ก่อนอนุญาตให้ใช้ credential จริง

### 3.8 Persistent Authentication Profile

สร้าง browser profile แยกที่:

```text
~/lava-bot/data/auth-profile
```

จำกัด permission เป็น 700 และเตรียม `.gitignore` ไม่ให้นำ session/profile ขึ้น repository

มนุษย์ทำ Google Authentication ผ่าน Chrome จริงบน graphical session จากนั้นปิด Chrome แล้วให้ Playwright ใช้ profile เดิมแบบ headless

ผลคือ standalone scout เข้าถึง Google Form ได้และรายงาน `FORM_DISCOVERED` โดยไม่ต้องฝังรหัสผ่านใน source code

### 3.9 Web + Persistent Auth Integration

แก้ Web UI ให้ใช้ `launch_persistent_context()` และ auth profile เดียวกับ scout

ผลทดสอบสุดท้าย:

```text
URL     ✅ PASS
FORM    ✅ DISCOVERED
AUTH    ✅ PASS
STATE   FORM_DISCOVERED

MODE: READ ONLY
NO FILL • NO UPLOAD • NO SUBMIT
```

จุดนี้คือ milestone สำคัญ: **LAVA-B SAFE AUTH + WEB INTEGRATION V1 = PASS**

---

## 4. ปัญหาที่พบ และบทเรียนจากการแก้

### ปัญหา 1 — Playwright Chromium ขาด dependency

Browser launch แรกไม่ผ่านเพราะ library ของระบบไม่ครบ เช่น `libnspr4.so`

แก้โดยติดตั้ง dependency สำหรับ Chromium ผ่าน Playwright แล้วทดสอบ launch ใหม่

**บทเรียน:** แยกปัญหา application code ออกจาก OS/browser dependency ก่อนแก้ logic ของ bot

### ปัญหา 2 — Chromium ของ Playwright เจอ `No usable sandbox!`

ในช่วง Auth Bridge prototype Chromium บางชุดไม่สามารถใช้ sandbox ได้บนสภาพแวดล้อมนั้น การใส่ `--no-sandbox` ทำให้ browser เปิดได้ แต่ไม่ถือเป็น production fix

**การตัดสินใจ:** ใช้เพียงพิสูจน์เส้นทางโดยไม่กรอก credential จริง แล้วเปลี่ยนไปติดตั้ง Chrome Stable ที่ sandbox ทำงานปกติ

**บทเรียน:** `Workaround ≠ Production Fix` และ credential จริงต้องไม่เข้า browser ที่ลด security boundary

### ปัญหา 3 — Server เดิมไม่มี GUI สำหรับมนุษย์ทำ Auth

ทดลองแนวทาง Xvfb + x11vnc + SSH tunnel เป็น Auth Bridge และพิสูจน์ว่า remote display ทำงานได้ จากนั้นเมื่อมีจอจริง จึงติดตั้ง XFCE minimal และใช้ graphical session โดยตรง ซึ่งง่ายกว่าในสภาพแวดล้อมปัจจุบัน

**บทเรียน:** prototype มีไว้พิสูจน์สมมติฐาน ไม่จำเป็นต้องกลายเป็น architecture ถาวร

### ปัญหา 4 — Script เปิด Chrome จาก File Manager กลายเป็นเปิดไฟล์ข้อความ

การ double-click helper script ไม่ได้ execute ตามที่คาด จึงเรียก script จาก shell พร้อมกำหนด `DISPLAY=:0` เพื่อเปิด Chrome บนจอจริง

**บทเรียน:** เมื่อ debug Linux GUI ให้แยกเรื่อง executable permission, display environment และ file-manager behavior ออกจากกัน

### ปัญหา 5 — Scout ผ่าน Auth แต่ Web UI ยังขึ้น `AUTH_REQUIRED`

นี่เป็นปัญหาสำคัญที่สุดของรอบล่าสุด: `scout.py` ใช้ persistent profile แล้ว แต่ `web.py` ยังเปิด Chrome ด้วย temporary browser context

จึงเกิดสถานการณ์:

```text
Scout → auth-profile → FORM_DISCOVERED
Web   → temporary profile → AUTH_REQUIRED
```

แก้โดยให้ Web ใช้ profile เดียวกัน:

```python
context = p.chromium.launch_persistent_context(
    user_data_dir=str(PROFILE),
    executable_path=CHROME,
    headless=True
)
```

และปิดด้วย `context.close()`

หลัง restart systemd ผล Web UI เปลี่ยนเป็น `AUTH PASS` + `FORM DISCOVERED`

**บทเรียน:** authentication ไม่ได้ติดอยู่กับ “เครื่อง” อย่างเดียว แต่ browser session/profile ต้องเป็นตัวเดียวกับที่ automation ใช้

### ปัญหา 6 — การแก้ทีละบรรทัดเริ่มเสี่ยงผิดตำแหน่ง

เมื่อ patch หลายจุด เช่น import, profile path, launch และ close การแก้ด้วยมือทีละตำแหน่งเพิ่มความเสี่ยง จึงสำรองไฟล์เดิมแล้วแทนด้วยไฟล์ฉบับที่ตรวจครบ จากนั้นใช้ `py_compile` ก่อน restart service

**บทเรียน:** ก่อนแก้ต้อง backup; หลังแก้ต้อง syntax-check; หลัง deploy ต้องตรวจ service และ endpoint แยกกัน

---

## 5. รูปแบบการวินิจฉัยที่ใช้ได้ผล

เราไม่ได้ถามเพียงว่า “บอททำงานไหม” แต่แยกระบบเป็นชั้น:

```text
OS / Disk / Network
        ↓
Python + venv
        ↓
Playwright
        ↓
Browser dependency
        ↓
Chrome sandbox
        ↓
GUI / Human Auth
        ↓
Persistent browser profile
        ↓
Standalone Scout
        ↓
Web integration
        ↓
Form Discovery
```

แต่ละชั้นต้องมี PASS ที่สังเกตได้ก่อนเดินต่อ วิธีนี้ทำให้เมื่อ Web UI ยังขึ้น AUTH_REQUIRED เรารู้ว่าปัญหาไม่ใช่ Google account หรือ Chrome เพราะ standalone scout ผ่านแล้ว จึงจำกัดวงปัญหาเหลือ integration ของ Web ได้รวดเร็ว

---

## 6. สถาปัตยกรรมที่พิสูจน์แล้ว ณ ปัจจุบัน

```text
Human
  │
  │ Google Form URL
  ▼
LAVA-B Web UI
  │
  ├─ Validate URL
  │
  ▼
Playwright
  │
  ▼
Chrome Stable + normal sandbox
  │
  ▼
Persistent auth-profile
  │
  ├─ session valid ─────────► Google Form
  │
  └─ auth required ─────────► HUMAN ACTION REQUIRED
                                  │
                                  ▼
                              Human Auth
                                  │
                                  └────► resume
```

ปัจจุบันเส้นทางถึง Google Form ทำงานแล้ว แต่ระบบยังตั้งใจหยุดที่ READ ONLY

---

## 7. Roadmap จากจุดนี้ถึงเป้าหมาย

### Phase 1 — FORM DISCOVERY V2

เพิ่ม “ตา” ให้ bot อ่านโครงสร้างจริงของฟอร์มโดยไม่แก้ข้อมูล:

- section/page
- question/label
- input/control type
- required/not required
- navigation buttons
- file-upload control

ผลลัพธ์ต้องสร้าง fingerprint/schema ที่ตรวจสอบได้ และยังคง `NO FILL / NO UPLOAD / NO SUBMIT`

### Phase 2 — DRY RUN

เมื่อ schema ถูกยืนยันแล้ว จึงเพิ่ม “มือแบบล็อกไก”:

- map คำตอบไปยัง field ที่ยืนยันแล้ว
- ตรวจชุดภาพให้ครบ 5 ภาพ
- เตรียม processed copies
- fill ข้อมูลทดสอบ
- upload ภาพทดสอบ
- ตรวจ pre-submit state
- **หยุดก่อน Submit**

ถ้าฟอร์มเปลี่ยน, field หาย, required เพิ่ม, ภาพไม่ครบ หรือ destination ผิด → FAIL/STOP แทนการเดา

### Phase 3 — CONTROLLED SUBMIT

เปิด Submit เฉพาะ test form ที่ปักไว้แล้ว:

```text
PRECHECK PASS
      ↓
SUBMIT COUNT = 0 ?
      ↓ yes
CLICK SUBMIT ONCE
      ↓
VERIFY GOOGLE CONFIRMATION
      ↓
PASS + SUBMIT COUNT = 1
```

ห้ามถือ click สำเร็จเป็นผลสำเร็จ ต้องตรวจ confirmation page/message

### Phase 4 — WEEKLY OPERATION

เมื่อ V1 end-to-end ผ่านจริง จึงนำ workflow ไปใช้ตามรอบงาน:

```text
รับ URL
  ↓
Validate
  ↓
Auth valid?
  ├─ yes → continue
  └─ no  → Human Auth
  ↓
Schema/Fingerprint check
  ↓
Exactly 5 images?
  ↓
Fill + Upload
  ↓
Pre-submit validation
  ↓
Submit once
  ↓
Confirmation
  ↓
PASS / FAIL log
  ↓
กลับรัง
```

### Phase 5 — Learning Layer (หลัง V1 เท่านั้น)

ค่อยพิจารณาการจำ form ID, fingerprint, first/last seen, seen count หรือ pattern ของรอบงาน เพื่อช่วย discovery และลดงานซ้ำ

แต่กฎเดิมไม่เปลี่ยน:

> **Prediction may assist discovery; prediction never grants permission to submit.**

---

## 8. Definition of Done ของ LAVA-B V1

V1 จะถือว่าจบเมื่อทดสอบ end-to-end บนฟอร์มทดสอบแล้วได้ครบ:

```text
URL VALIDATION       PASS
SAFE AUTH            PASS
FORM DISCOVERY       PASS
SCHEMA CHECK         PASS
IMAGE CHECK 5/5      PASS
FILL                 PASS
UPLOAD 5/5           PASS
PRE-SUBMIT CHECK     PASS
SUBMIT 1/1           PASS
CONFIRMATION         PASS
FINAL JOB RESULT     PASS
```

และต้องพิสูจน์ failure path อย่างน้อยว่าเมื่อข้อมูล/ภาพ/form/auth ไม่ตรง ระบบหยุดโดยไม่ submit

---

## 9. สิ่งที่ยังไม่ทำ — โดยตั้งใจ

ยังไม่เพิ่มระบบเรียนรู้/ทำนาย, dashboard ใหญ่, feature เสริม, automatic CAPTCHA handling หรือ scope ใหม่ เพราะเป้าหมายปัจจุบันคือทำ V1 ให้เดินครบหนึ่งรอบอย่างถูกต้องก่อน

นี่เป็นบทเรียนสำคัญของงาน automation: **ระบบเล็กที่จบงานจริงและหยุดได้เมื่อไม่แน่ใจ มีค่ากว่าระบบใหญ่ที่ดูฉลาดแต่ควบคุมไม่ได้**

---

## 10. วิธีเรียนรู้ที่ได้จากโครงการนี้

สิ่งสำคัญไม่ใช่การจำ command หรือ code ทุกบรรทัด แต่คือการรู้ว่าจะถามระบบอย่างไร:

```text
INPUT
  ↓
PROCESS
  ↓
OUTPUT
  ↓
FEEDBACK
  ↓
แก้เฉพาะชั้นที่ไม่ผ่าน
```

เมื่อเจอปัญหา:

1. อย่าเดา
2. ทำให้ปัญหาเล็กลง
3. ตรวจทีละชั้น
4. เก็บ PASS ที่พิสูจน์แล้วไว้
5. แก้เฉพาะจุดที่ยัง FAIL
6. backup ก่อนเปลี่ยน
7. test ก่อน deploy
8. deploy แล้วตรวจผลอีกครั้ง
9. security boundary ไม่ใช่สิ่งที่ automation มีสิทธิ์ข้าม

นี่คือแก่นของ LAVA-B มากกว่าตัว bot เอง: **เรียนรู้วิธีเรียนรู้จากระบบจริง**

---

## 11. สรุป

LAVA-B เริ่มจากโจทย์เล็กมาก — งานซ้ำบน Google Form — แต่ระหว่างทางทำให้เห็นองค์ประกอบของระบบจริงทั้ง Linux, service management, browser automation, GUI/headless environment, authentication, persistent session, security guardrail, logging และการวินิจฉัยแบบเป็นชั้น

ณ จุดนี้ bot ยังไม่ได้รับอนุญาตให้ Submit และนั่นไม่ใช่ข้อด้อย แต่เป็นผลจากการออกแบบตามลำดับ: **ให้มันมองเห็นและเข้าใจสิ่งที่อยู่ตรงหน้าก่อน แล้วจึงค่อยให้มันลงมือ**

เป้าหมายถัดไปจึงชัดเจน: Form Discovery V2 → Dry Run → Controlled Submit → Confirmation → Weekly Operation

> **ไม่เดา • ไม่ยิงซ้ำ • ไม่ข้าม Auth • ไม่เสือกเมื่อไม่มีงาน**
>
> รับลิงก์ → Auth โดยคน → LAVA-B จัดการ → ยืนยันผล → จ๊วด → กลับรัง 🔥🐝
