# PINTA Field Notes — จาก `systemctl: command not found` สู่ Self-Care Server

**วันที่ทดลองจริง:** 15 กันยายน 2569 (2026-09-15)  
**เครื่อง:** PINTA Server / Ubuntu Server  
**เครดิต:** ลุงอั้นตี้ × น้อนแชตจี้ (GPT-5.6 Sol) × Penguin 🐧 × PINTA 🎨

> หลักของบันทึกนี้คือ **ไม่เดา — แยกอาการออกจากสาเหตุ — ตรวจจากชั้นล่างขึ้นชั้นบน — ซ่อมเท่าที่จำเป็น แล้วเฝ้าดูแนวโน้ม**

---

## 1. จุดเริ่มต้น: ทำไม `systemctl` ถึงกลายเป็น `command not found`

อาการแรกคือคำสั่งสำคัญ เช่น `systemctl`, `ls`, `sudo` เริ่มเรียกไม่ได้ จนดูคล้ายพิมพ์คำสั่งผิดหรือ PATH มีปัญหา

สิ่งที่เรียนรู้สำคัญคือ **อย่าหยุดการวินิจฉัยที่ typo** หาก external commands หลายตัวหายพร้อมกัน โดยเฉพาะเมื่อ shell built-in เช่น `echo` หรือ `type` ยังทำงานได้ ต้องสงสัยชั้น storage/filesystem ด้วย

เหตุการณ์นี้ผู้ช่วยวินิจฉัยช่วงแรกผิดทาง โดยไปโฟกัสว่าตัว `l` อาจถูกพิมพ์เป็นเลข `1` มากเกินไป ภายหลังหลักฐานจาก kernel ชี้ชัดว่าเป็นปัญหา I/O จริง

อาการที่พบภายหลัง ได้แก่:

```text
device offline error, dev sda
Buffer I/O error on dev sda2
JBD2: I/O error
EXT4-fs error
```

เมื่อ root filesystem หรือ storage path มีปัญหา โปรแกรมที่อยู่บนดิสก์ เช่น `/usr/bin/systemctl` อาจอ่านไม่ได้ แม้ shell process ที่โหลดอยู่ใน RAM จะยังมีชีวิตและ shell built-ins ยังตอบสนองได้

### ความเข้าใจที่ต้องแก้ให้แม่น

นี่ **ไม่ใช่ Ubuntu มีโหมดพิเศษที่ย้ายระบบทั้งหมดไปรันบน RAM อัตโนมัติ** แต่เป็นผลจาก process/code/data บางส่วนที่ถูกโหลดและ cache อยู่ใน RAM แล้ว จึงยังทำงานต่อได้ชั่วคราว ขณะที่การอ่าน executable หรือข้อมูลใหม่จากดิสก์ล้มเหลว

ดังนั้นอาการ “shell ยังไม่ตาย แต่คำสั่งข้างนอกหาย” เป็นเบาะแสสำคัญ ไม่ใช่หลักฐานว่าดิสก์ยังปกติ

---

## 2. Root cause: SATA path ไม่เสถียร

หลังตรวจหน้างานพบว่าสาย SATA หลวม เมื่อถอดเสียบใหม่และ reboot ระบบกลับมาอ่าน SSD ได้ตามปกติ

SSD ที่ตรวจพบ:

```text
KINGSTON SMS200S3120G
120 GB
```

SMART overall health:

```text
PASSED
```

ไม่พบ retired block, program fail, erase fail หรือ reported uncorrectable error ที่ชี้ตรงไปยัง NAND เสียในเวลาที่ตรวจ

แต่พบค่าที่สำคัญมาก:

```text
Number of Interface CRC Errors = 29317
```

CRC error ชนิดนี้เป็น **ข้อผิดพลาดการสื่อสารบนเส้นทาง SATA** จึงต้องพิจารณาสาย, connector, port, controller และสัญญาณ ไม่ควรรีบสรุปว่า NAND/SSD เสีย

หลังเปลี่ยนสาย SATA ใหม่ เราปักค่า **29317 เป็น baseline** แล้วติดตาม `delta` แทนการตกใจกับเลขสะสมในอดีต ถ้าค่ายังคง 29317 แปลว่าไม่มี CRC ใหม่เกิดขึ้นในช่วงที่เฝ้าดู

---

## 3. SATA 6 Gb/s แต่ negotiate ได้ 1.5 Gb/s

สายใหม่ระบุ `Serial ATA 6G — 26AWG` และ SSD ประกาศรองรับ SATA 3.0 สูงสุด 6.0 Gb/s แต่ Linux รายงาน link ปัจจุบันเพียง:

```text
SATA 3.0, 6.0 Gb/s (current: 1.5 Gb/s)
ata3: SATA link up 1.5 Gbps
```

ทดลองเปลี่ยนสายและย้าย motherboard port แล้ว link ยังอยู่ 1.5 Gb/s ดังนั้นสาย/port อย่างเดียวไม่สามารถอธิบายอาการทั้งหมดได้ อาจเกี่ยวกับ negotiation, firmware/BIOS, controller หรืออุปกรณ์อื่นใน SATA path ซึ่งยัง **ไม่สรุปสาเหตุ**

สำหรับ PINTA ซึ่งเป็น lightweight automation server เราเลือก **stability before speed**: ยังไม่ไล่แก้ BIOS ในวันที่เพิ่งกู้ระบบกลับมา แต่เฝ้าดู CRC และ I/O errors ก่อน

> 6 Gb/s ที่เขียนบนสายคือระดับมาตรฐาน/ความสามารถ ไม่ใช่คำรับประกันว่า link จะ negotiate ที่ 6 Gb/s ทุกระบบ

---

## 4. Tailscale: เป้าหมายคือ reboot แล้วกลับมาเอง

ปัญหาที่กังวลคือ remote server หาก reboot แล้วต้องเปิด browser เพื่อ authenticate ใหม่ทุกครั้ง จะไม่เหมาะกับ headless server

แนวคิดที่ใช้คือ **ทำให้ Tailscale เป็น persistent system service และตรวจผลด้วย reboot จริง** ไม่ใช่เพียงดูว่าเชื่อมต่อได้ใน session ปัจจุบัน

ตรวจ service:

```bash
systemctl status tailscaled --no-pager
```

สิ่งที่ต้องการคือ service อยู่ในสถานะ enabled/active และหลัง reboot เครื่องกลับมาอยู่ใน tailnet เดิมโดยไม่ต้อง login ผ่านเว็บใหม่

ตรวจจากเครื่อง remote:

```bash
tailscale status | grep pinta
```

และทดสอบ SSH:

```bash
ssh <user>@<TAILSCALE-IP>
```

ในการทดสอบจริง PINTA reboot แล้ว Tailscale กลับมาเอง และ SSH ผ่าน Tailscale ใช้งานได้โดยไม่ต้อง web-auth ซ้ำ จึงถือว่าคุณสมบัติ persistence ผ่านการทดสอบในรอบนี้

> อย่าตั้งสมมติฐานว่า Tailscale “ต้อง auth ทุก reboot” หรือ “ไม่มีวัน auth ใหม่” แบบตายตัว ต้องแยก service persistence ออกจากกรณี key expiry, node removal, policy หรือ credential state ซึ่งอาจทำให้ต้อง authenticate ใหม่ในอนาคต

---

## 5. PINTA Self-Care v1

เราออกแบบให้เครื่อง **ตรวจตัวเองก่อน แต่ไม่ซ่อมตัวเองแบบเสี่ยง**

สถานะมี 3 ระดับ:

```text
🟢 ALIVE    = ทำงานปกติ
🟡 OBSERVE  = ยังทำงานได้ แต่มีสิ่งผิดปกติให้เฝ้าดู
🔴 CPR      = ต้องให้คนเข้าตรวจ
```

### โครงสร้างไฟล์

```text
/opt/pinta/
├── pinta-health.sh
├── pinta-maintenance.sh
├── logs/
│   ├── health.log
│   └── maintenance.log
└── state/
    └── crc-baseline

/etc/systemd/system/
├── pinta-health.service
├── pinta-health.timer
├── pinta-maintenance.service
└── pinta-maintenance.timer
```

สิทธิ์ script:

```bash
sudo chmod 750 /opt/pinta/pinta-health.sh
sudo chmod 750 /opt/pinta/pinta-maintenance.sh
```

ความหมายโดยย่อ: owner อ่าน/เขียน/execute ได้, group อ่าน/execute ได้, other ไม่มีสิทธิ์

### Health script ตรวจอะไร

รอบทดสอบจริงรายงาน:

```text
🟢 PINTA ALIVE
Root Disk: 11%
RAM: 2%
SMART: PASSED
SATA CRC: 29317 (delta 0)
Kernel Storage Errors: 0
Tailscale: OK
SSH: OK
Failed Services: 0
Assessment: ไปต่อได้ ไม่ต้อง CPR
```

ตัวตรวจประกอบด้วย root disk usage, RAM usage, SMART health, SATA CRC + delta, kernel storage errors, Tailscale, SSH และ failed systemd services

จุดที่พบระหว่างพัฒนาคือ parser ของ `smartctl -x` หยิบ field ผิดจนรายงาน:

```text
SATA CRC: --- (delta ?)
```

จึงแก้ให้ดึงตัวเลขจริง `29317` และทดสอบใหม่จนได้ `delta 0` นี่เป็นตัวอย่างว่าต้อง **ทดสอบ parser กับ output จริงของเครื่อง** ไม่ควรเชื่อ script เพียงเพราะ syntax ผ่าน

### Maintenance script ทำอะไร

v1 ตั้งใจให้ conservative:

```text
apt-get update
แสดง package ที่มี update
apt-get autoclean
ลด journal เก่ากว่า 30 วัน
fstrim -av
เรียก health check ซ้ำ
```

สิ่งที่ **ไม่ให้ทำอัตโนมัติ** ใน v1:

```text
apt upgrade
apt autoremove แบบก้าวร้าว
reboot
fsck
remount/repair filesystem
SMART repair
```

เหตุผลคือ automation ที่ดูแล production/headless machine ควร “สังเกตและรายงาน” ก่อน “แก้ไข” โดยเฉพาะการกระทำที่อาจทำให้เครื่อง remote กลับมาไม่ได้

---

## 6. ให้ systemd อ่าน shell script และทำงานตามเวลา

เราใช้ `systemd service + timer` แทน cron เพื่อให้ตรวจสถานะและ log ได้เป็นระบบ

หลังสร้าง unit files:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now pinta-health.timer
sudo systemctl enable --now pinta-maintenance.timer
```

ตารางที่ทดสอบจริง:

```text
Wed 2026-09-16 08:03:54 +07  pinta-health.timer
Sun 2026-09-20 09:03:05 +07  pinta-maintenance.timer
```

Health ตรวจทุกวันประมาณ 08:00 และ Maintenance ทุกวันอาทิตย์ประมาณ 09:00 มี randomized delay เล็กน้อยโดยตั้งใจ

Timezone ถูกตั้งเป็น:

```text
Asia/Bangkok (+07, +0700)
System clock synchronized: yes
NTP service: active
RTC in local TZ: no
```

การให้ RTC เก็บ UTC (`RTC in local TZ: no`) เป็นรูปแบบปกติและเหมาะกับ Linux server

ดู timers:

```bash
systemctl list-timers --all | grep pinta
```

ดู health log:

```bash
sudo tail -20 /opt/pinta/logs/health.log
```

---

## 7. Decision tree ภาคสนาม

```text
                 ┌────────────────────────────┐
                 │ คำสั่งหลายตัว command not │
                 │ found / เครื่องเริ่มแปลก   │
                 └─────────────┬──────────────┘
                               │
                    ┌──────────▼──────────┐
                    │ shell built-in ยังได้? │
                    │ echo / type ฯลฯ        │
                    └──────┬──────────┬────┘
                           │ใช่       │ไม่ใช่
                           ▼           ▼
               ┌────────────────┐   ตรวจ power/
               │ อย่าโทษ PATH   │   console/network
               │ อย่างเดียว      │
               └───────┬────────┘
                       ▼
          ┌───────────────────────────┐
          │ ตรวจ kernel/storage error │
          │ I/O / EXT4 / JBD2 / sda   │
          └────────────┬──────────────┘
                       │พบ
                       ▼
          ┌───────────────────────────┐
          │ หยุดงานเขียนที่ไม่จำเป็น │
          │ ตรวจ SATA/power/connector │
          └────────────┬──────────────┘
                       ▼
          ┌───────────────────────────┐
          │ กู้ physical path → reboot │
          │ → SMART → CRC baseline    │
          └────────────┬──────────────┘
                       ▼
          ┌───────────────────────────┐
          │ CRC เพิ่ม / I/O error ใหม่?│
          └────────┬───────────┬──────┘
                   │ใช่        │ไม่
                   ▼            ▼
             🔴 CPR/ตรวจต่อ   🟢 Burn-in
                               + Self-Care
```

---

## 8. บทเรียนเชิงระบบ

1. **อาการไม่ใช่สาเหตุ** — `command not found` อาจเกิดจาก storage I/O ไม่ใช่การพิมพ์ผิด
2. **shell ที่ยังตอบ ไม่ได้แปลว่าดิสก์ยังดี** — process และข้อมูลบางส่วนอาจยังอยู่ใน RAM/cache
3. **SMART PASSED ไม่ได้ปิดคดีทั้งหมด** — ต้องดู interface CRC และ kernel log ร่วมกัน
4. **ดู delta ดีกว่าดูเลขสะสม** — CRC 29317 เป็นประวัติสะสม; สิ่งสำคัญหลังซ่อมคือเพิ่มหรือไม่
5. **Stability before speed** — server bot ที่วิ่งสัปดาห์ละครั้งไม่จำเป็นต้องเสี่ยง BIOS เพื่อไล่ SATA 6 Gb/s ในวันที่เพิ่งกู้ระบบ
6. **Remote persistence ต้องพิสูจน์ด้วย reboot** — service ที่ active ตอนนี้ยังไม่พอ
7. **Automation ต้อง fail safe** — v1 ตรวจและเตือน ไม่ reboot/fsck/upgrade เอง
8. **ก่อนติดตั้ง script ให้ syntax-check** — `bash -n script.sh` และควรสร้างไฟล์แบบ atomic เพื่อลดความเสียหายหาก SSH หลุดกลางการ paste
9. **เวลาของ server เป็น dependency** — ตั้ง timezone/NTP ก่อนเปิด timer
10. **เก็บ field notes จากของจริง** — command, error, baseline และสิ่งที่ทดลองแล้วช่วยลดการเดาซ้ำในครั้งหน้า

---

## 9. งานต่อ: Notification / Email

Self-Care v1 เขียน log ได้แล้ว แต่ยังไม่มี outbound notification ปัจจุบันต้องเปิดดูเอง:

```bash
sudo tail -20 /opt/pinta/logs/health.log
```

เป้าหมาย v2 คือส่งข้อความ/อีเมลโดยใช้หลัก **alert fatigue control**:

- 🟢 ไม่ต้องส่งทุกวัน; อาจรวมเป็น weekly summary
- 🟡 ส่งเมื่อพบ anomaly ที่ควรติดตาม
- 🔴 ส่งทันทีเมื่อ SMART fail, CRC เพิ่มผิดปกติ, storage error, Tailscale down หรือเงื่อนไขวิกฤต
- หลีกเลี่ยงการแนบ log ทั้งก้อนโดยไม่จำเป็น; ส่ง summary และเฉพาะช่วง log ที่เกี่ยวข้อง

ยังไม่กำหนด SMTP/provider หรือ credential ในเอกสารนี้ เพราะควรออกแบบเรื่อง secret storage และช่องทางแจ้งเตือนก่อน ไม่ควรฝัง password/token ลง shell script หรือ Git repository

---

## สรุป

เช้าวันนี้เริ่มจากอาการที่ดูเหมือนคำสั่ง Linux พัง แต่สุดท้ายพาเราไล่ลงไปถึง **physical SATA path**, ตรวจ SMART/CRC, พิสูจน์ Tailscale persistence, ตั้ง timezone และสร้าง **PINTA Self-Care v1** ที่ตรวจชีพตัวเองด้วย systemd timer

สิ่งที่มีค่าที่สุดไม่ใช่การทำให้ `systemctl` กลับมาเพียงอย่างเดียว แต่คือการเปลี่ยนเหตุขัดข้องหนึ่งครั้งให้กลายเป็น **วิธีคิดและระบบเฝ้าระวังที่ใช้ซ้ำได้**

> **เห็นอาการ → เก็บหลักฐาน → แยกชั้น → หา root cause → แก้ให้น้อยที่สุด → วัด baseline → เฝ้าดูแนวโน้ม → ค่อย automate**
