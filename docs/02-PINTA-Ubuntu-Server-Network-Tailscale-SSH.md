# PINTA Server Field Notes — Part 2

## Fresh Ubuntu Server → Network → Tailscale → Secure Remote Administration

> บันทึกจากการติดตั้งและแก้ปัญหาจริงบนเครื่อง PINTA Server (Ubuntu Server 24.04.4 LTS) โดยเน้นระบบเล็ก ลีน ตรวจสอบได้ และไม่เปิด SSH สู่ Public Internet โดยไม่จำเป็น

**Field experience / workflow:** ลุงอั้นตี้  
**AI collaborator / documentation:** น้อนแชตจี้ — GPT-5.6 Sol  
**Project:** PINTA Server Field Notes  
**With:** Ubuntu × GitHub × Tailscale × Penguin 🐧 × PINTA 🎨 × ปู่ Celeron

---

## 1. เป้าหมายของ Part 2

หลังจาก Part 1 สร้าง USB Boot Ubuntu Server สำเร็จ เป้าหมายของบทนี้คือทำให้เครื่องเก่ากลายเป็น headless server ที่:

- ต่อ LAN และรับ IP ด้วย DHCP
- ตั้งค่า Network ผ่าน Netplan ได้อย่างถูกต้อง
- ออก Internet และ resolve DNS ได้
- มี OpenSSH Server
- เชื่อม Tailscale
- SSH จากเครื่องอื่นผ่าน Tailnet ได้
- ไม่ต้อง Port Forward Router
- ไม่ต้องเปิด SSH สู่ Public Internet
- รับ security updates อัตโนมัติ
- พร้อมเปิดใช้งานระยะยาวแบบ 24/7 หลังผ่าน reboot test

---

## 2. หลักการก่อนแตะ Network

อย่าเดาชื่อ interface และอย่าก๊อปชื่อจากเครื่องอื่น เพราะชื่อจริงอาจเป็น `eno1`, `enp2s0`, `enp3s0` หรือแบบอื่น

ดูของจริงก่อน:

```bash
ip link
```

หรือ:

```bash
ip -br link
```

ดู IP ปัจจุบัน:

```bash
ip -br addr
```

ดู route:

```bash
ip route
```

> กฎ: **เห็น Interface ก่อนแก้ Interface**

---

## 3. Netplan อยู่ตรงไหน

Ubuntu Server ใช้ Netplan สำหรับกำหนด network configuration โดยไฟล์ YAML อยู่ใน:

```text
/etc/netplan/
```

ดูชื่อไฟล์จริงก่อน:

```bash
ls -l /etc/netplan/
```

ชื่ออาจเป็นเช่น:

```text
00-installer-config.yaml
01-netcfg.yaml
50-cloud-init.yaml
```

ชื่อ `01-xxxx.yaml` ในตัวอย่างไม่ใช่ชื่อบังคับ ต้องใช้ไฟล์ที่มีอยู่จริงหรือสร้างไฟล์ใหม่โดยเข้าใจลำดับ configuration ก่อน

ดู configuration ปัจจุบัน:

```bash
sudo netplan get
```

---

## 4. DHCP แบบง่ายที่สุด

ตัวอย่างเท่านั้น — เปลี่ยน `<interface>` เป็นชื่อ NIC จริงจาก `ip link`

```yaml
network:
  version: 2
  ethernets:
    <interface>:
      dhcp4: true
```

ตัวอย่าง ถ้า NIC จริงคือ `enp2s0`:

```yaml
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: true
```

### YAML ต้องเคาะอย่างไร

ใช้ **spaces เท่านั้น ห้าม Tab**

มองเป็นชั้น:

```text
network:                 ← 0 spaces
  version: 2             ← 2 spaces
  ethernets:             ← 2 spaces
    enp2s0:              ← 4 spaces
      dhcp4: true        ← 6 spaces
```

ถ้า indentation ผิด Netplan อาจ parse ไม่ผ่าน

---

## 5. เครื่อง Minimal ไม่มี nano/pico ทำอย่างไร

ก่อนติดตั้ง editor เพิ่ม ให้ดูว่ามีอะไรอยู่แล้ว:

```bash
command -v nano
command -v vi
command -v vim
```

Ubuntu Server minimal มักยังมี `vi` หรือ editor พื้นฐานบางตัวให้ใช้ได้ แต่ถ้าต้องการ `nano` ซึ่งเหมาะกับผู้เริ่มต้น:

```bash
sudo apt update
sudo apt install -y nano
```

เปิดไฟล์ เช่น:

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

ใน nano:

```text
Ctrl+O   Save
Enter    ยืนยันชื่อไฟล์
Ctrl+X   Exit
```

`pico` ไม่จำเป็นสำหรับ PINTA หากมี `nano` แล้ว — ลด package ที่ซ้ำหน้าที่กัน

---

## 6. ตรวจ YAML ก่อน Apply

หลังแก้ไฟล์ อย่ารีบ `apply` โดยไม่ตรวจ

```bash
sudo netplan generate
```

`generate` แปลง YAML เป็น backend configuration และช่วยให้พบ parsing/validation error โดยยังไม่เปลี่ยน network ที่กำลังใช้งาน

จากนั้น ถ้าแก้ network ผ่าน remote session แนะนำ:

```bash
sudo netplan try
```

Netplan จะทดลอง configuration และ rollback หากไม่ยืนยันภายในเวลาที่กำหนด เหมาะมากสำหรับลดโอกาสล็อกตัวเองออกจาก server

เมื่ออยู่หน้าเครื่องและ configuration ผ่านแล้ว สามารถใช้:

```bash
sudo netplan apply
```

ตรวจผล:

```bash
ip -br addr
ip route
sudo netplan status
```

---

## 7. ทดสอบ Network ให้ครบ 3 ชั้น

### ชั้น 1 — มี IP หรือยัง

```bash
ip -br addr
```

### ชั้น 2 — ออก network/IP ได้หรือไม่

```bash
ping -c 4 1.1.1.1
```

### ชั้น 3 — DNS ทำงานหรือไม่

```bash
ping -c 4 ubuntu.com
```

ถ้า ping IP ได้ แต่ชื่อ domain ไม่ได้ ให้สงสัย DNS ก่อน ไม่ควรสุ่มแก้ทั้ง network stack

ดู DNS:

```bash
resolvectl status
```

---

## 8. Package พื้นฐาน — ลงเท่าที่จำเป็น

อัปเดตรายการ package:

```bash
sudo apt update
```

เครื่อง PINTA ใช้แนวคิด **Minimal first** จึงไม่ติดตั้ง Desktop, Docker, nginx, Apache หรือ package อื่นเพียงเพราะ “server มักมี”

Editor ที่เลือกใช้:

```bash
sudo apt install -y nano
```

OpenSSH Server:

```bash
sudo apt install -y openssh-server
```

เปิดและให้เริ่มพร้อมระบบ:

```bash
sudo systemctl enable --now ssh
```

ตรวจ:

```bash
sudo systemctl status ssh --no-pager
```

ตรวจว่ามี process ฟัง TCP port 22:

```bash
sudo ss -lntp | grep ':22'
```

> SSH ปกติใช้ TCP port 22. ไม่ต้องเปิด “secure port” เพิ่มอีกพอร์ตหนึ่ง

---

## 9. ติดตั้ง Tailscale

ใช้วิธีจากเอกสาร Tailscale สำหรับ Linux:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

จากนั้นเชื่อมเครื่องเข้า Tailnet:

```bash
sudo tailscale up
```

ระบบจะแสดง URL ให้เปิดใน browser เพื่อยืนยันตัวตน

ตรวจสถานะ:

```bash
tailscale status
```

ดู Tailscale IPv4 ของเครื่อง:

```bash
tailscale ip -4
```

> จด IP จาก **เครื่องเป้าหมายจริง** อย่าจำจากเครื่องอื่น

---

## 10. Tailscale SSH กับ OpenSSH-over-Tailscale เป็นคนละแนวคิด

### A. OpenSSH วิ่งผ่าน Tailscale network

OpenSSH Server ฟัง port 22 แต่เราเชื่อมไปยัง Tailscale IP ของ server:

```bash
ssh <username>@<tailscale-ip>
```

ตัวอย่างรูปแบบ:

```bash
ssh gwagk@100.x.x.x
```

### B. Tailscale SSH

Tailscale มี SSH feature ของตัวเอง เปิดได้ด้วย:

```bash
tailscale set --ssh
```

การเชื่อมต่ออาจมี browser re-authentication ตาม policy ของ Tailnet เช่นข้อความ:

```text
Tailscale SSH requires an additional check
To authenticate, visit: https://login.tailscale.com/...
```

นี่ไม่ใช่ error — เป็น authentication check เพิ่มเติม

PINTA ที่ทดลองจริงพบ flow แบบนี้และยืนยันผ่าน browser สำเร็จ

> สำหรับ production ควรเลือก policy ให้ชัดว่าจะใช้ Tailscale SSH หรือ OpenSSH-over-Tailscale เป็นมาตรฐานหลัก ไม่ควรเปิดหลายทางโดยไม่มีเหตุผล

---

## 11. เชื่อมจากเครื่อง Admin

เครื่อง Admin ต้องติดตั้ง Tailscale และอยู่ Tailnet เดียวกัน

ตรวจ:

```bash
tailscale status
```

จากนั้น:

```bash
ssh <username>@<PINTA-TAILSCALE-IP>
```

ครั้งแรก OpenSSH อาจถาม host fingerprint:

```text
The authenticity of host ... can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

ก่อนตอบ `yes` ควรตรวจว่ากำลังเชื่อม IP/hostname ของเครื่องที่ตั้งใจจริง

เมื่อยืนยันแล้ว host key จะถูกบันทึกใน `known_hosts`

---

## 12. บทเรียนจริง: SSH ค้าง ไม่ได้แปลว่าต้องเปิด Port

ในการติดตั้ง PINTA ครั้งแรก SSH จากเครื่อง Admin ค้าง ไม่มี acknowledgement

สิ่งที่ตรวจตามลำดับคือ:

1. OpenSSH Server ติดตั้งหรือยัง
2. `ssh.service` active หรือไม่
3. TCP 22 LISTEN หรือไม่
4. Firewall มีหรือไม่
5. Tailscale online หรือไม่
6. **Tailscale IP ที่พิมพ์ถูกเครื่องหรือไม่**

สาเหตุจริงครั้งนั้นคือ **พิมพ์ Tailscale IP ผิด**

เมื่อใช้ IP ของ PINTA ถูกต้อง SSH ทำงานทันที

> **ก่อนแก้ Firewall หรือเปิด Port เพิ่ม ให้ยืนยัน Target IP ก่อน**

นี่เป็นเหตุผลว่าทำไม troubleshooting ควรเดินจากข้อเท็จจริง ไม่ใช่สุ่มเปิด service/port

---

## 13. Firewall — อย่าเปิดกว้างเพียงเพื่อให้มันใช้ได้

ในการทดลองจริง PINTA minimal ตอบว่า:

```text
ufw: command not found
```

ดังนั้น UFW ไม่ใช่สาเหตุที่บล็อก SSH ในตอนนั้น

ถ้าต้องการติดตั้ง UFW ใน baseline ภายหลัง:

```bash
sudo apt update
sudo apt install -y ufw
```

**อย่าเพิ่ง `ufw enable` ก่อนสร้าง rule ที่จำเป็น โดยเฉพาะเมื่อทำงานผ่าน remote SSH**

สำหรับแนวทางที่ให้ SSH เข้าทาง Tailscale interface เท่านั้น สามารถกำหนด rule ก่อนเปิด firewall เช่น:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

ตรวจ:

```bash
sudo ufw status verbose
```

เมื่อแน่ใจว่ามีช่องทาง recovery และ rule ถูกต้องแล้วจึงพิจารณา:

```bash
sudo ufw enable
```

> ไม่จำเป็นต้อง `ufw allow 22` จากทุก interface หากสถาปัตยกรรมกำหนดให้ remote administration ผ่าน Tailnet เท่านั้น

> หมายเหตุ: หากเลือกใช้ Tailscale SSH โดยตรง ให้ทบทวน firewall และ Tailscale access policy ให้สอดคล้องกับวิธีนั้น ไม่ควรก๊อป rule นี้โดยไม่เข้าใจเส้นทาง traffic จริง

---

## 14. ไม่ต้อง Port Forward Router

เมื่อ PINTA และเครื่อง Admin อยู่ Tailnet เดียวกัน การ remote administration ไม่จำเป็นต้อง:

- Forward TCP/22 จาก Internet มาที่ PINTA
- มี Public Static IP
- เปิด web admin สู่ Internet
- เปิด router inbound เพิ่มเพียงเพื่อ SSH

แนวทางของโครงการคือ:

```text
Admin device
    ↓
Tailscale / Tailnet
    ↓
PINTA Server
```

Server ยังคงออก Internet ตามปกติ แต่ไม่ต้องนำ SSH ไปวางเป็นประตูสาธารณะ

---

## 15. Automatic Security Updates

Ubuntu ใช้ `unattended-upgrades` สำหรับติดตั้ง security updates อัตโนมัติ

ติดตั้ง/ยืนยันว่ามี package:

```bash
sudo apt update
sudo apt install -y unattended-upgrades
```

เปิด service:

```bash
sudo systemctl enable --now unattended-upgrades
```

ตรวจ:

```bash
systemctl status unattended-upgrades --no-pager
```

ดู APT timers:

```bash
systemctl list-timers 'apt-*'
```

Ubuntu ตั้ง `unattended-upgrades` ให้ทำงานรายวันโดยค่าเริ่มต้น และ systemd timers มี randomized delay เพื่อไม่ให้ทุกเครื่องยิง repository พร้อมกัน

ไฟล์สำคัญ:

```text
/etc/apt/apt.conf.d/20auto-upgrades
/etc/apt/apt.conf.d/50unattended-upgrades
/var/log/unattended-upgrades/
```

ค่ารายวันทั่วไปใน `20auto-upgrades`:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

เลข `1` หมายถึงทุก 1 วัน

### PINTA policy ระยะแรก

- Security updates: อัตโนมัติรายวัน
- Automatic reboot: **ยังไม่เปิด**
- Reboot ที่จำเป็น: วางช่วง maintenance หลังรู้ตาราง Bot แล้ว

เหตุผลคือ server จะมี scheduled automation ในอนาคต การ reboot อัตโนมัติโดยยังไม่กำหนด maintenance window อาจชนงาน

---

## 16. Server 24/7 — Acceptance Test ก่อนปล่อยยาว

PINTA สามารถออกแบบให้เปิด 24/7 ได้ แต่ก่อนถือว่า baseline เสร็จควรทดสอบ:

```text
[ ] Network DHCP ขึ้นเองหลัง boot
[ ] Internet/DNS กลับมาเอง
[ ] ssh.service active หลัง boot
[ ] tailscaled active หลัง boot
[ ] Tailscale IP/hostname reachable
[ ] SSH จากเครื่อง Admin เข้าได้โดยไม่เดินไปแตะ server
[ ] unattended-upgrades ทำงาน
[ ] ไม่มี service ที่ไม่จำเป็นเปิดทิ้ง
[ ] เครื่องไม่ suspend/sleep เอง
[ ] อุณหภูมิ/พัดลม/ดิสก์ของเครื่องเก่าอยู่ในสภาพเหมาะสม
```

ตรวจ service สำคัญ:

```bash
systemctl is-enabled ssh
systemctl is-active ssh
systemctl is-enabled tailscaled
systemctl is-active tailscaled
```

จากนั้นทำ **Reboot Acceptance Test**:

```bash
sudo reboot
```

รอเครื่องกลับมา แล้วจากเครื่อง Admin:

```bash
ssh <username>@<PINTA-TAILSCALE-IP>
```

ถ้ากลับเข้าได้โดยไม่ต้องไปแตะ PINTA ถือว่า Remote Administration baseline ผ่าน

---

## 17. เรื่อง Sleep / Suspend

Server ที่จะเปิด 24/7 ไม่ควร suspend เอง แต่ **อย่าเดาว่าเครื่องนี้ตั้งค่าอะไรอยู่**

ตรวจ targets ก่อน:

```bash
systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target --no-pager
```

หากภายหลังพบว่า server suspend เองจริง จึงค่อยกำหนด policy ปิด suspend ให้เหมาะสม และบันทึกสิ่งที่เปลี่ยน

สำหรับ BIOS/UEFI หากต้องการให้ server กลับมาเองหลังไฟดับ ให้มองหาตัวเลือกประเภท:

```text
Restore on AC Power Loss
AC Back
After Power Failure
Power On after AC Loss
```

ชื่อจริงขึ้นกับเมนบอร์ด จึงต้องดู BIOS ของเครื่องจริง ไม่ควรเดาชื่อเมนู

---

## 18. Troubleshooting Cheat Sheet

### ดูชื่อเครื่อง

```bash
hostname
hostnamectl
```

### ดู NIC/IP

```bash
ip -br link
ip -br addr
```

### ดู route

```bash
ip route
```

### ดู DNS

```bash
resolvectl status
```

### ดู Netplan

```bash
sudo netplan get
sudo netplan status
```

### ตรวจ SSH

```bash
sudo systemctl status ssh --no-pager
sudo ss -lntp | grep ':22'
```

### ตรวจ Tailscale

```bash
tailscale status
tailscale ip -4
systemctl status tailscaled --no-pager
```

### SSH แบบ verbose เมื่อมีปัญหา

```bash
ssh -v <username>@<tailscale-ip>
```

### ดู update timers

```bash
systemctl list-timers 'apt-*'
```

### ดู unattended-upgrades log

```bash
sudo tail -n 100 /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## 19. สิ่งที่ยังไม่ทำใน Part 2

เพื่อรักษาหลัก **ทำจริง → ตรวจจริง → บันทึกจริง** เรายังไม่ถือว่าสิ่งต่อไปนี้เสร็จจนกว่าจะทดลองกับ PINTA จริง:

- Firewall production policy ขั้นสุดท้าย
- SSH key-only policy
- ปิด password authentication
- BIOS auto power-on after outage
- Monitoring/health check
- Bot Runner / Playwright
- Google Form automation

สิ่งเหล่านี้จะถูกเพิ่มเมื่อผ่านการทดลองจริง ไม่เขียนให้ดูครบโดยอาศัยการเดา

---

## 20. Milestone ที่ผ่านแล้ว

```text
Ubuntu Server installed             ✅
LAN / DHCP working                  ✅
Internet working                    ✅
OpenSSH Server installed            ✅
Tailscale connected                 ✅
Remote SSH via Tailnet              ✅
Browser re-auth observed            ✅
First remote login successful       ✅
Automatic security updates enabled  ✅
Reboot persistence test             ⏳
Firewall final policy               ⏳
Bot runtime                         ⏳
```

### Remote Administration Milestone #1

> **JTECH → Tailscale → PINTA Server → SSH SUCCESS**

ปู่ Celeron กลับมารับราชการอีกครั้ง 🐧

---

## 21. Official references

- Ubuntu Server — Automatic updates: https://ubuntu.com/server/docs/how-to/software/automatic-updates/
- Ubuntu Server — Security suggestions: https://ubuntu.com/server/docs/explanation/security/security_suggestions/
- Ubuntu Netplan manpage: https://manpages.ubuntu.com/manpages/noble/man8/netplan.8.html
- Tailscale — Install on Linux: https://tailscale.com/docs/install/linux
- Tailscale SSH: https://tailscale.com/docs/features/tailscale-ssh
- Tailscale — UFW on Ubuntu: https://tailscale.com/docs/how-to/secure-ubuntu-server-with-ufw

---

## Credits

**Field experience / workflow:** ลุงอั้นตี้  
**AI collaborator / documentation:** น้อนแชตจี้ — GPT-5.6 Sol  
**Platform & tools:** Ubuntu × GitHub × Tailscale  
**Mascots & spirit:** Penguin 🐧 × PINTA 🎨  
**Hardware comeback:** Celeron

> **Human × AI Collaborative Work — PINTA Server Field Notes**  
> ทำจริง → ตรวจจริง → บันทึกจริง → ส่งต่อได้จริง
