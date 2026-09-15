# LAVA-B Auth Bridge Field Guide

## คู่มือภาคสนาม: ทำให้บอทบน Ubuntu Server แบบไม่มีจอ เปิด Browser ให้คนยืนยัน Google Authentication ได้อย่างปลอดภัย

> สถานะเอกสาร: Field Notes / Prototype
>
> เป้าหมาย: บันทึกเส้นทางทดลองจริง ปัญหาที่พบ วิธีวินิจฉัย และเหตุผลในการแก้ เพื่อให้ผู้อื่นนำแนวคิดไปประยุกต์ใช้ได้
>
> หลักสำคัญ: **ไม่เดา — ตรวจทีละชั้น — ผ่านแล้วค่อยไปต่อ — ห้ามเอา credential จริงไปเสี่ยงกับ browser ที่ปิด sandbox**

---

## 1. ปัญหาที่ต้องการแก้

LAVA-B เป็น automation ขนาดเล็กที่รันบน Ubuntu Server แบบ terminal-only ไม่มี Desktop Environment และไม่มีจอภาพจริง งานที่ต้องทำในอนาคตคือเปิด Google Form ผ่าน Chromium/Playwright โดยให้มนุษย์เป็นผู้ทำ Google Authentication เอง จากนั้น automation จึงใช้ browser session เดิมทำงานต่อ

โจทย์จึงไม่ใช่การฝัง username/password ลงใน bot แต่คือการสร้าง **Auth Bridge**:

```text
เครื่องผู้ใช้
    │
    │ SSH tunnel
    ▼
localhost:5900
    │
  x11vnc
    │
  Xvfb :99
    │
 Chromium / Playwright
    │
 Google Authentication
```

แนวคิดนี้แยกหน้าที่ชัดเจน:

- คน: Authentication และการตัดสินใจที่ต้องใช้สิทธิ์
- Bot: งานซ้ำหลัง Authentication
- Server: เก็บ browser profile/session ที่จำเป็น
- ห้าม bot เก็บรหัสผ่าน Google

---

## 2. หลักการก่อนลงมือ

### 2.1 ทำทีละชั้น

อย่าแก้หลายจุดพร้อมกัน ลำดับการตรวจที่ใช้จริงคือ:

```text
Playwright installed?
      ↓
Chromium exists?
      ↓
มี DISPLAY หรือไม่?
      ↓
Xvfb ทำงานหรือไม่?
      ↓
x11vnc มองเห็น Xvfb หรือไม่?
      ↓
VNC ถูกจำกัดไว้ที่ localhost หรือไม่?
      ↓
SSH tunnel ผ่านหรือไม่?
      ↓
VNC Viewer เห็นจอ :99 หรือไม่?
      ↓
Chromium เปิดบน :99 หรือไม่?
      ↓
Google Sign-in page แสดงหรือไม่?
      ↓
ค่อยพิจารณา Authentication จริง
```

### 2.2 PASS/FAIL ต้องเห็นชัด

UI ของ automation ควรแสดง workflow แบบสั้น ๆ เช่น:

```text
① ตรวจสอบ URL           PASS
② เปิด Google Form       PASS
③ Google Authentication WAITING
④ อ่านโครงสร้างฟอร์ม     WAIT
⑤ ตรวจชุดภาพ             WAIT
⑥ เตรียมข้อมูล            WAIT
⑦ ตรวจสอบก่อนส่ง          WAIT
⑧ Submit                 LOCKED
⑨ Confirmation           WAIT
```

หลักคือ **ปกติดู PASS; ผิดปกติค่อยเปิด log**

### 2.3 Predict ≠ Permission

แม้ bot จะจำ URL หรือเดารอบงานได้ในอนาคต การคาดการณ์ไม่ใช่สิทธิ์ในการ Submit งาน ต้องมี guardrail แยกต่างหาก

---

## 3. ตรวจ Chromium ของ Playwright

ใน Python virtual environment:

```bash
cd ~/lava-bot
source .venv/bin/activate

which chromium chromium-browser google-chrome
```

ถ้าไม่มี output ไม่ได้แปลว่า Playwright ไม่มี browser เพราะ Playwright อาจติดตั้ง Chromium ไว้ใน cache ของตัวเอง

ตรวจ executable ที่ Playwright ใช้จริง:

```bash
python - <<'PY'
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    print("Chromium :", p.chromium.executable_path)
PY
```

ตัวอย่าง path ที่พบจากการทดลอง:

```text
~/.cache/ms-playwright/chromium-XXXX/chrome-linux64/chrome
```

### ปัญหาที่พบ

อาจมีข้อความลักษณะ:

```text
Task was destroyed but it is pending!
TargetClosedError
```

ในกรณีที่เกิดตอนสคริปต์สั้น ๆ จบการทำงานหลังเพียงอ่าน executable path อย่าเพิ่งสรุปว่า browser เสีย จุดสำคัญคือเราได้ path ที่ต้องการแล้ว ให้ทดสอบ launch จริงแยกอีกขั้นหนึ่ง

**ทริก:** อย่าใช้ error จากขั้น discovery ไปสรุปความเสียหายของขั้น launch

---

## 4. ตรวจระบบแสดงผลของ Server

Server แบบ terminal-only มักไม่มี X/Wayland session:

```bash
echo "DISPLAY=$DISPLAY"
echo "WAYLAND_DISPLAY=$WAYLAND_DISPLAY"
which Xvfb
```

ผลที่เหมาะกับสถาปัตยกรรมนี้คือ:

```text
DISPLAY=
WAYLAND_DISPLAY=
/usr/bin/Xvfb
```

หมายความว่าไม่มี Desktop จริง แต่มี X virtual framebuffer พร้อมใช้งาน

---

## 5. สร้าง Virtual Display ด้วย Xvfb

เริ่มจอเสมือนหมายเลข `:99`:

```bash
Xvfb :99 -screen 0 1280x800x24 &
```

ตรวจ process:

```bash
ps aux | grep '[X]vfb :99'
```

ควรเห็น:

```text
Xvfb :99 -screen 0 1280x800x24
```

### Warning ที่พบ

Xvfb/xkbcomp อาจเตือน keysym เช่น:

```text
Could not resolve keysym XF86CameraAccessEnable
Could not resolve keysym XF86RadarOverlay
...
Errors from xkbcomp are not fatal to the X server
```

ถ้าบรรทัดท้ายระบุว่า `not fatal` และ process Xvfb ยังอยู่ ให้ถือว่าเป็น warning ของ keymap สำหรับปุ่มพิเศษ ไม่ต้องรื้อระบบเพื่อแก้สิ่งที่ไม่เกี่ยวกับงาน

**ทริก:** อ่านบรรทัดสรุปท้าย error block ก่อนเสมอ Warning จำนวนมากไม่ได้แปลว่า service ล้ม

---

## 6. ติดตั้ง x11vnc

ตรวจของเดิมก่อน:

```bash
dpkg -l | grep -E 'xvfb|x11vnc|novnc' || true
```

ถ้ายังไม่มี x11vnc:

```bash
sudo apt install x11vnc -y
```

ตรวจ:

```bash
x11vnc -version
```

---

## 7. ผูก x11vnc กับจอ :99 โดยไม่เปิด VNC ออกเครือข่าย

ใช้:

```bash
x11vnc -display :99 -localhost -forever -shared -rfbport 5900 &
```

จุดสำคัญคือ `-localhost` เพราะไม่ต้องการเปิด VNC server เปลือยให้ LAN หรือเครือข่ายอื่นเชื่อมตรง

ตรวจ socket:

```bash
ss -ltnp | grep 5900
```

ค่าที่ต้องการ:

```text
127.0.0.1:5900
[::1]:5900
```

ไม่ควรเห็น `0.0.0.0:5900` หากตั้งใจใช้เฉพาะ SSH tunnel

### Warning: x11vnc WITHOUT a password

x11vnc จะเตือนว่าไม่มี password ซึ่งเป็น warning ที่ต้องเข้าใจบริบท ใน prototype นี้เราไม่ได้ expose port 5900 ออก network แต่ bind เฉพาะ loopback แล้วครอบด้วย SSH tunnel

อย่างไรก็ตาม หากจะเปลี่ยน architecture หรือเปิด port ออก network ต้องทบทวน authentication/encryption ใหม่ ห้ามถือว่าการไม่มี VNC password ปลอดภัยในทุกสภาพแวดล้อม

---

## 8. สร้าง SSH Tunnel จากเครื่องผู้ใช้

ที่เครื่อง client เปิด terminal แยกหนึ่งหน้าต่าง:

```bash
ssh -N -L 5900:127.0.0.1:5900 USER@SERVER_TAILSCALE_IP
```

ถ้าคำสั่งสำเร็จ มันอาจ **เงียบและค้างอยู่** นั่นคือพฤติกรรมปกติ เพราะ process กำลังรักษา tunnel

อย่าปิด terminal นี้ระหว่างใช้งาน

แนวคิด:

```text
Client 127.0.0.1:5900
        │
        │ encrypted SSH tunnel
        ▼
Server 127.0.0.1:5900
        │
      x11vnc
```

**ทริก:** command ที่เงียบไม่ได้แปลว่าค้างเสมอไป สำหรับ `ssh -N` ความเงียบคือสัญญาณที่คาดไว้

---

## 9. ติดตั้ง VNC Viewer ที่เครื่อง Client

ตรวจ:

```bash
which remmina vncviewer tigervncviewer
```

ถ้าไม่มี ให้ติดตั้ง viewer ขนาดเล็ก เช่น TigerVNC:

```bash
sudo apt update
sudo apt install tigervnc-viewer -y
```

ตรวจ:

```bash
which vncviewer
```

เปิด:

```bash
vncviewer 127.0.0.1:5900
```

ถ้าเห็น **จอดำ** และชื่อ session ของ server/display แสดงถูกต้อง นี่คือ PASS ไม่ใช่ความล้มเหลว จอดำหมายถึง Xvfb ทำงานแต่ยังไม่มี application วาดอะไรบน display

**ทริก:** ในระบบ headless “จอดำ” อาจเป็นผลลัพธ์ที่ถูกต้องที่สุดของขั้น connectivity test

---

## 10. ทดลองเปิด Chromium บน Virtual Display

สร้าง persistent browser profile แยกสำหรับ bot:

```bash
mkdir -p ~/lava-bot/data/browser-profile
```

จากนั้นทดลอง launch แบบ foreground ก่อน เพื่อให้เห็น error จริง:

```bash
DISPLAY=:99 \
/path/to/playwright/chromium \
--user-data-dir=$HOME/lava-bot/data/browser-profile \
--no-first-run \
--no-default-browser-check \
https://accounts.google.com/
```

### บทเรียนสำคัญ: อย่าใส่ `&` ตอน debug รอบแรก

เมื่อ launch ด้วย `&` แล้ว browser ไม่โผล่ เราอาจไม่ทันเห็นสาเหตุ จึงควรรัน foreground เพื่ออ่าน stderr ก่อน

---

## 11. ปัญหา Chromium: `No usable sandbox!`

ปัญหาจริงที่พบ:

```text
FATAL: ... No usable sandbox!
If you are running on Ubuntu 23.10+ or another Linux distro that has disabled
unprivileged user namespaces with AppArmor ...
```

นี่เป็นจุดที่ต้องแยกให้ชัดว่า:

- Xvfb ไม่ได้เสีย
- VNC ไม่ได้เสีย
- SSH tunnel ไม่ได้เสีย
- Chromium ถูกหยุดก่อนสร้างหน้าต่าง เพราะ sandbox policy ของ Linux/AppArmor

เพื่อ **พิสูจน์ architecture เท่านั้น** สามารถทดลอง launch ด้วย:

```bash
DISPLAY=:99 \
/path/to/playwright/chromium \
--user-data-dir=$HOME/lava-bot/data/browser-profile \
--no-first-run \
--no-default-browser-check \
--no-sandbox \
https://accounts.google.com/
```

ผลการทดลอง: Chromium สามารถปรากฏบน VNC และหน้า Google Sign-in โหลดได้สำเร็จ

### แต่ `--no-sandbox` ไม่ใช่คำตอบ production

`--no-sandbox` ลดชั้นป้องกันสำคัญของ Chromium จึงควรใช้เฉพาะการพิสูจน์เส้นทางในสภาพแวดล้อมควบคุม ไม่ควรกรอก credential จริงหรือปัก flag นี้เป็น configuration ถาวรโดยไม่ประเมินความเสี่ยง

แนวทางถัดไปควรศึกษาวิธีให้ Chromium sandbox ทำงานอย่างถูกต้องกับ Ubuntu/AppArmor แทนการปิด sandbox

**ทริกสำคัญที่สุดของคืนนี้:** เมื่อ workaround ทำให้ระบบ “เปิดได้” อย่ารีบตีความว่า workaround นั้น “เหมาะสำหรับใช้งานจริง”

---

## 12. Error อื่นที่อาจเห็นจาก Chrome for Testing

อาจพบข้อความเกี่ยวกับ GCM/registration เช่น:

```text
PHONE_REGISTRATION_ERROR
DEPRECATED_ENDPOINT
Failed to log in to GCM
```

หากหน้า Google Sign-in แสดงได้ ให้แยก error เหล่านี้ออกจากปัญหา sandbox และ Authentication หลักก่อน อย่าแก้ทุก log line ที่เห็นโดยไม่มีผลต่อเป้าหมายปัจจุบัน

หลักวินิจฉัย:

> แก้เฉพาะสิ่งที่ block milestone ปัจจุบัน

---

## 13. Persistent Profile มีไว้ทำอะไร

`--user-data-dir` แยก browser profile ของ automation ออกจาก browser อื่น ทำให้ session/cookie ที่ได้รับหลัง Authentication สามารถอยู่ใน profile เดิมและถูก browser session ถัดไปนำมาใช้ได้

แต่ต้องถือ directory นี้เป็นข้อมูลอ่อนไหว:

- จำกัด permission
- ไม่ commit เข้า Git
- ไม่ copy แจก
- ไม่เก็บ password ลง source/config/log
- เพิ่ม path ที่เกี่ยวข้องลง `.gitignore`

ตัวอย่าง `.gitignore`:

```gitignore
.venv/
data/browser-profile/
logs/
*.log
```

---

## 14. Workflow Monitor ที่ควรสร้างต่อ

หน้า LAVA-B ควรลีนและแสดงเพียงสิ่งที่คนต้องรู้:

```text
LAVA-B

Google Form URL
[................................]
[ เริ่มทำงาน ]

WORKFLOW STATUS
--------------------------------
① ตรวจสอบ URL           PASS
② เปิด Google Form       PASS
③ Google Authentication WAITING
④ อ่านโครงสร้างฟอร์ม     WAIT
⑤ ตรวจชุดภาพ 5 รูป       WAIT
⑥ เตรียมข้อมูล            WAIT
⑦ ตรวจสอบก่อนส่ง          WAIT
⑧ Submit                 WAIT
⑨ Confirmation           WAIT
--------------------------------
MODE: SAFE / NO SUBMIT
```

สถานะหลักควรมีเพียง:

```text
WAIT → RUNNING/WAITING → PASS
                       ↘ FAIL → STOP
```

หลัง Submit ต้องตรวจ Confirmation แยกต่างหาก เพราะ “กดปุ่มได้” ไม่เท่ากับ “ปลายทางรับข้อมูลสำเร็จ”

---

## 15. Image Pipeline ที่ทดสอบร่วมกับระบบ

เพื่อไม่ให้ไฟล์ภาพขนาดหลาย MB ถูกส่งขึ้นฟอร์มโดยไม่จำเป็น ให้เก็บ original แยกจาก processed เสมอ

โครงสร้างตัวอย่าง:

```text
data/
├── images/
│   ├── inbox/
│   ├── set-001/
│   ├── set-002/
│   └── ...
└── processed/
    └── set-001/
```

กฎที่ใช้ในการทดลอง:

- 1 set = 5 ภาพพอดี
- 4 หรือ 6 ภาพ = NOT READY
- ไม่ยืมภาพจาก set ข้างเคียง
- original ไม่ลบ ไม่เขียนทับ
- bot ใช้ processed copy สำหรับ upload

ImageMagick 6 ใช้ `convert`:

```bash
for f in ~/lava-bot/data/images/set-001/*.jpg; do
    convert "$f" \
      -auto-orient \
      -resize '1600x1600>' \
      -strip \
      -quality 75 \
      ~/lava-bot/data/processed/set-001/"$(basename "$f")"
done
```

จากการทดลอง ภาพต้นฉบับขนาดประมาณ 2–5 MB ลดลงเหลือเฉลี่ยประมาณ 500 KB/ภาพ โดยยังคงต้นฉบับไว้

`-strip` จะตัด metadata เช่น EXIF/GPS ออกจากสำเนา ซึ่งช่วยลดข้อมูลส่วนเกิน แต่ต้องตัดสินใจตามวัตถุประสงค์ของหลักฐานจริงเสมอว่าจำเป็นต้องรักษา metadata ใดไว้หรือไม่

---

## 16. Guardrails สำหรับ Automation

ก่อนอนุญาต Submit จริง ควรมีอย่างน้อย:

```text
URL VALID       ?
AUTH PASS       ?
FORM KNOWN      ?
SCHEMA MATCH    ?
IMAGE SET = 5   ?
PROCESSED = 5   ?
JOB NOT SENT    ?
USER/WORK RULE  ?
--------------------------------
ALL PASS → SUBMIT ONCE
ANY FAIL → STOP
```

และใช้กฎ:

> **ONE JOB → ONE SUBMIT → ONE PASS**

ต้องมี duplicate lock ไม่ให้ retry แบบไม่รู้สถานะกลายเป็นส่งซ้ำ

---

## 17. สิ่งที่ยังไม่เสร็จ

ณ จุดบันทึกนี้ architecture ต่อไปนี้พิสูจน์แล้ว:

```text
Xvfb :99        PASS
x11vnc          PASS
localhost-only  PASS
SSH tunnel      PASS
TigerVNC        PASS
Chromium GUI    PASS (prototype with --no-sandbox)
Google page     PASS
Real Auth       HOLD
Form reading    NOT TESTED
Upload          NOT TESTED
Submit          LOCKED
```

งานถัดไปคือทำ Chromium sandbox ให้เหมาะสมกับ Ubuntu/AppArmor ก่อนใช้ Google credential จริง จากนั้นจึงทดสอบ Authentication และ form discovery ทีละขั้น

---

## 18. Lessons Learned

1. **Big Picture ก่อน Command** — รู้ก่อนว่ากำลังเชื่อมอะไรกับอะไร
2. **หนึ่งปัญหาต่อหนึ่งรอบ** — อย่าแก้ X, VNC, Browser และ Auth พร้อมกัน
3. **ตรวจ process/socket แทนการเดา** — `ps`, `ss`, `which` ให้ข้อเท็จจริงเร็วมาก
4. **Foreground ก่อน Background** — โปรแกรมไม่ขึ้น ให้เอา `&` ออกเพื่อเห็น stderr
5. **Warning ไม่เท่ากับ Failure** — อ่านบรรทัดสุดท้ายและตรวจ process จริง
6. **จอดำก็เป็น PASS ได้** — ถ้า milestone คือพิสูจน์ว่า remote display เชื่อมถึง
7. **Workaround ไม่เท่ากับ Production Fix** — `--no-sandbox` พิสูจน์สาเหตุได้ แต่ไม่ควรรีบใช้กับ credential จริง
8. **Human-in-the-loop เป็น feature ไม่ใช่ข้อเสีย** — ให้คน Authentication แล้วให้เครื่องทำงานซ้ำต่อ
9. **ต้นฉบับต้องอยู่รอด** — image processing ทำกับ copy
10. **Bot ต้องหยุดเป็น** — ความสามารถในการ STOP เมื่อเงื่อนไขไม่ตรง สำคัญพอ ๆ กับความสามารถในการ Submit

---

## 19. แนวคิดสรุป

ระบบ automation ที่ดีไม่จำเป็นต้องทำทุกอย่างแทนคน จุดที่คุ้มค่าคือให้คนทำเฉพาะส่วนที่ต้องใช้สิทธิ์ การตัดสินใจ หรือความรับผิดชอบ แล้วส่งงานซ้ำที่ deterministic ให้เครื่องทำ

```text
HUMAN
  │ authentication / decision
  ▼
LAVA-B
  │ repetitive deterministic work
  ▼
VERIFY
  │
  ├── PASS → finish
  └── FAIL → stop + log
```

เป้าหมายไม่ใช่ “ทำให้ดูไฮเทค” แต่คือ **ลดงานซ้ำโดยไม่ลดความถูกต้องและความปลอดภัย**

---

### หมายเหตุด้านความปลอดภัย

เอกสารนี้เป็นบันทึกการทดลอง ไม่ใช่ security hardening guide สำหรับ production โดยสมบูรณ์ โดยเฉพาะ `--no-sandbox` มีไว้บันทึกการวินิจฉัยที่เกิดขึ้นจริงเท่านั้น ไม่ควรนำไปใช้กับ credential จริงโดยอัตโนมัติ ควรแก้ sandbox/AppArmor ให้ถูกต้องก่อนใช้งานจริง
