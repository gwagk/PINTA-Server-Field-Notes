# คู่มือดาวน์โหลด Ubuntu Server ISO และสร้าง USB Boot บน Ubuntu

> คู่มือภาคสนามจากประสบการณ์จริงในการเตรียม **PINTA Server** ด้วย Ubuntu Server 24.04.4 LTS

**เครดิต:** ลุงอั้นตี้ × น้อนแชตจี้ (GPT-5.6 Sol) × Penguin 🐧 × PINTA 🎨  
**แนวคิด:** ประสบการณ์ที่ทำแล้วต้องถูกจัดเป็นระบบ เพื่อส่งต่อให้น้อง ๆ รุ่นหลังได้ใช้ต่อ

---

## 1. ขอบเขตของคู่มือนี้

คู่มือนี้ครอบคลุมขั้นตอนที่ทดลองจริงแล้ว ได้แก่

1. ดาวน์โหลด Ubuntu Server ISO จากแหล่งทางการ
2. ตรวจขนาดไฟล์ ISO
3. ตรวจ SHA-256 checksum ก่อนใช้งาน
4. ตรวจว่า USB มีข้อมูลอะไรอยู่ก่อนล้าง
5. Mount พาร์ทิชันแบบ read-only เพื่อดูข้อมูลอย่างปลอดภัย
6. Unmount USB ก่อนเขียน image
7. ใช้ **Disk Image Writer / GNOME Disks** เขียน ISO ลง USB โดยไม่ต้องติดตั้ง Rufus, Etcher หรือ Ventoy
8. ตรวจจุดเสี่ยงสำคัญก่อนกด Restore
9. รอการเขียนจนเสร็จและเตรียม USB สำหรับ Boot

> **หมายเหตุ:** ขั้นตอนติดตั้ง Ubuntu Server บนเครื่อง PINTA Server จะจัดทำต่อเป็นภาคถัดไป หลังจากทดลองจริงครบทุกหน้าจอ เพื่อไม่บันทึกขั้นตอนจากการเดาหรือความจำเพียงอย่างเดียว

---

## 2. อุปกรณ์และเงื่อนไข

- เครื่อง Ubuntu Desktop ที่ใช้งานได้ปกติ
- อินเทอร์เน็ต
- USB Flash Drive อย่างน้อย 4 GB (แนะนำ 8 GB ขึ้นไป)
- ไฟล์ Ubuntu Server ISO
- สิทธิ์ `sudo`
- ต้องยอมรับว่า **ข้อมูลเดิมใน USB จะถูกลบทั้งหมด** เมื่อเขียน ISO

---

## 3. ดาวน์โหลด Ubuntu Server ISO

เวอร์ชันที่ใช้ในคู่มือนี้:

```text
Ubuntu Server 24.04.4 LTS (Noble Numbat)
Architecture: AMD64
File: ubuntu-24.04.4-live-server-amd64.iso
```

แหล่งทางการ:

```text
https://releases.ubuntu.com/24.04.4/
```

ดาวน์โหลดด้วยเว็บเบราว์เซอร์ได้ตามปกติ หรือใช้ Terminal:

```bash
mkdir -p "$HOME/Downloads/PINTA-SERVER"

wget -c \
  -O "$HOME/Downloads/PINTA-SERVER/ubuntu-24.04.4-live-server-amd64.iso" \
  https://releases.ubuntu.com/24.04.4/ubuntu-24.04.4-live-server-amd64.iso
```

`-c` หมายถึง continue หากการดาวน์โหลดขาดช่วง สามารถทำต่อจากเดิมได้ในหลายกรณี

### ตัวอย่าง: นัดดาวน์โหลดตอนกลางคืนด้วย systemd

หากใช้อินเทอร์เน็ตจำกัดและต้องการย้าย Digital Labor ไปกลางคืน สามารถตั้ง one-shot timer ได้ เช่น 23:00:

```bash
mkdir -p "$HOME/Downloads/PINTA-SERVER"

systemd-run --user \
  --unit=pinta-ubuntu-iso-download \
  --on-calendar="2026-09-08 23:00:00" \
  /usr/bin/wget -c \
  -O "$HOME/Downloads/PINTA-SERVER/ubuntu-24.04.4-live-server-amd64.iso" \
  https://releases.ubuntu.com/24.04.4/ubuntu-24.04.4-live-server-amd64.iso
```

> เปลี่ยนวันและเวลาให้ตรงกับวันที่ใช้งานจริง

ตรวจ log หลังงานทำเสร็จ:

```bash
journalctl --user -u pinta-ubuntu-iso-download.service
```

---

## 4. ตรวจว่า ISO ดาวน์โหลดมาครบหรือไม่

ตรวจไฟล์และขนาด:

```bash
ls -lh ~/Downloads/PINTA-SERVER/
```

ตัวอย่างผลที่ได้:

```text
total 3.2G
-rw-rw-r-- 1 user user 3.2G ... ubuntu-24.04.4-live-server-amd64.iso
```

ขนาดประมาณ 3.2 GB เป็นเพียงการตรวจเบื้องต้น **ยังไม่พอ** ต้องตรวจ checksum ต่อ

---

## 5. ตรวจ SHA-256 checksum

คำนวณ SHA-256 ของไฟล์ที่ดาวน์โหลด:

```bash
sha256sum ~/Downloads/PINTA-SERVER/ubuntu-24.04.4-live-server-amd64.iso
```

ค่าที่ตรวจได้จากไฟล์จริงในคู่มือนี้:

```text
e907d92eeec9df64163a7e454cbc8d7755e8ddc7ed42f99dbc80c40f1a138433
```

ค่าทางการของ Ubuntu สำหรับไฟล์นี้:

```text
e907d92eeec9df64163a7e454cbc8d7755e8ddc7ed42f99dbc80c40f1a138433
```

ตรวจจาก:

```text
https://releases.ubuntu.com/24.04.4/SHA256SUMS
```

หากค่า **ตรงกันทุกตัว** จึงถือว่าไฟล์ผ่านการตรวจความสมบูรณ์สำหรับขั้นตอนนี้

> ห้ามใช้ checksum ที่คัดลอกจากคู่มือนี้กับ Ubuntu เวอร์ชันอื่น ต้องเปิด `SHA256SUMS` ของเวอร์ชันที่ดาวน์โหลดจริงทุกครั้ง

---

## 6. เสียบ USB แล้วระบุตัวอุปกรณ์ให้ถูกก่อน

หลังเสียบ USB ใช้คำสั่ง:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,LABEL,MOUNTPOINTS
```

ตัวอย่างจากการทดลองจริง:

```text
sdb   28.7G Cruze usb    iso966 Linux Mint 22.2 Cinnamon 64-bit
├─sdb1 2.8G              iso966 Linux Mint 22.2 Cinnamon 64-bit /run/media/...
├─sdb2   5M              vfat
└─sdb3 25.8G             ext4   writable
```

ในกรณีนี้ USB คือ **`/dev/sdb`**

> **กฎสำคัญ: เห็น Device ก่อนเขียน Device**  
> ห้ามเดาว่า `/dev/sdb` จะเป็น USB ทุกเครื่อง เพราะลำดับอุปกรณ์อาจเปลี่ยนได้

---

## 7. ตรวจข้อมูลเดิมใน USB ก่อนล้าง

ถ้ามีพาร์ทิชันที่ยังไม่ mount และต้องการดูว่ามีไฟล์สำคัญหรือไม่ ให้ mount แบบ read-only:

```bash
sudo mkdir -p /mnt/usbcheck
sudo mount -o ro /dev/sdb3 /mnt/usbcheck
```

ดูรายการไฟล์:

```bash
ls -lah /mnt/usbcheck
```

หรือดูไฟล์ไม่เกิน 2 ระดับ:

```bash
find /mnt/usbcheck -maxdepth 2 -type f | head -100
```

ตัวอย่างที่พบในการทดลอง:

```text
install-logs-2026-07-13.0
install-logs-2026-07-13.1
install-logs-2026-07-13.2
lost+found
```

ไม่พบไฟล์ส่วนตัว จึงตัดสินใจนำ USB ไปเขียน Ubuntu Server ใหม่

> หากพบไฟล์สำคัญ ให้หยุดและสำรองข้อมูลก่อนทันที

---

## 8. Unmount USB ก่อนเขียน ISO

Unmount พาร์ทิชันที่เราเปิดตรวจ:

```bash
sudo umount /mnt/usbcheck
```

Unmount พาร์ทิชันที่ Desktop mount ให้อัตโนมัติ เช่น:

```bash
sudo umount /dev/sdb1
```

หากขึ้นว่า `not mounted` ให้ตรวจสถานะด้วย `lsblk` อีกครั้ง ไม่ต้องฝืน

---

## 9. ระวังกับดัก: `writable` ไม่ใช่ไฟล์ ISO

![writable ไม่ใช่ ISO](images/01-wrong-writable-selection.png)

ภาพนี้แสดง sidebar ของ Files ซึ่งมี `writable` เป็นพาร์ทิชันของ USB เก่า ไม่ใช่ไฟล์ Ubuntu ISO

ไฟล์ที่ต้องเลือกจริงอยู่ที่:

```text
Downloads → PINTA-SERVER → ubuntu-24.04.4-live-server-amd64.iso
```

---

## 10. เปิด ISO ด้วย Disk Image Writer

คลิกขวาที่ ISO แล้วเลือก **Open With...**

![Open With](images/02-open-with.png)

จากนั้นเลือก **Disk Image Writer** ไม่ใช่ Disk Image Mounter

![Disk Image Writer](images/03-disk-image-writer.png)

ความแตกต่าง:

- **Disk Image Mounter** = เปิด ISO เพื่อดูไฟล์ภายใน
- **Disk Image Writer** = เขียน image ลง USB เพื่อสร้าง boot media

---

## 11. เลือก Destination ให้ถูกตัว

หน้าต่าง Restore Disk Image จะแสดงอุปกรณ์ปลายทาง

![Select destination](images/04-select-destination.png)

ตัวอย่างจากการทดลอง:

```text
256 GB Disk — SATA SSD (/dev/sda)        ← ห้ามเลือก
CD/DVD Drive (/dev/sr0)                  ← ไม่ใช่เป้าหมาย
31 GB Thumb Drive — SanDisk Cruzer Blade (/dev/sdb) ← เลือกอันนี้
```

> หากเลือก SSD ระบบผิด เครื่องหลักอาจถูกเขียนทับและข้อมูลเสียหายทันที

ก่อนกด Restore ให้ตรวจ 3 อย่างพร้อมกัน:

1. ชื่อ USB
2. ความจุ USB
3. device path เช่น `/dev/sdb`

---

## 12. ยืนยันก่อน Start Restoring

![Confirm restore](images/05-confirm-restore.png)

หน้าต่างอาจแจ้งว่า image มีขนาดเล็กกว่าอุปกรณ์ปลายทาง เช่น:

```text
The disk image is 27 GB smaller than the target device
```

ข้อความนี้เป็นเพียงข้อมูลว่า ISO 3.4 GB ถูกเขียนลง USB 31 GB ไม่ใช่ error

ตรวจอีกครั้ง:

```text
Image: ubuntu-24.04.4-live-server-amd64.iso
Destination: SanDisk Cruzer Blade (/dev/sdb)
```

จากนั้นจึงกด **Start Restoring...**

> ขั้นตอนนี้จะเขียนทับ partition table และข้อมูลเดิมใน USB

---

## 13. ระหว่างเขียน USB

![Writing progress](images/06-writing-progress.png)

ระหว่าง Restore:

- ห้ามถอด USB
- ห้ามปิดเครื่อง
- ห้ามกดยกเลิกโดยไม่จำเป็น
- รอ progress ถึง 100%
- หลังจบให้รอโปรแกรม sync ข้อมูลลงอุปกรณ์ให้เสร็จ

ระหว่างเขียน อาจยังเห็นชื่อ/พาร์ทิชันของ Linux เดิมอยู่ชั่วคราวใน UI เพราะข้อมูลกำลังถูกแทนที่ทีละส่วน ซึ่งไม่ใช่ข้อผิดพลาด

---

## 14. หลังเขียนเสร็จ

เมื่อ Restore เสร็จ:

1. รอจนโปรแกรมไม่มีงานเขียนค้าง
2. Eject/Power Off USB อย่างปลอดภัยจาก GNOME Disks หรือ Files
3. ถอด USB
4. นำ USB ไปเสียบเครื่องที่จะติดตั้ง Ubuntu Server
5. เข้า Boot Menu/UEFI/BIOS แล้วเลือก Boot จาก USB

> ปุ่มเข้า Boot Menu แตกต่างกันตามเมนบอร์ด เช่น F8, F10, F11, F12, Esc หรืออื่น ๆ ให้ดูคู่มือเครื่องจริง

---

## 15. Checklist ก่อนนำ USB ไป Boot

```text
[ ] ISO ดาวน์โหลดจาก releases.ubuntu.com
[ ] ชื่อไฟล์ถูกเวอร์ชัน
[ ] SHA-256 ตรงกับ SHA256SUMS ทางการ
[ ] ตรวจข้อมูลเดิมใน USB แล้ว
[ ] สำรองไฟล์ที่ต้องเก็บแล้ว
[ ] ระบุ USB จาก lsblk ได้แน่นอน
[ ] Restore ลง USB ทั้งลูก ไม่ใช่ SSD ระบบ
[ ] Restore เสร็จ 100%
[ ] Eject USB อย่างปลอดภัย
```

---

## 16. แนวคิดที่ได้จากงานนี้

### 16.1 อย่าแก้ปัญหาด้วยการลงโปรแกรมเพิ่มเสมอไป
Ubuntu Desktop มี Disk Image Writer/GNOME Disks อยู่แล้ว จึงไม่จำเป็นต้องติดตั้ง Rufus, Balena Etcher หรือ Ventoy เพียงเพื่อสร้าง USB installer ในกรณีนี้

### 16.2 ตรวจสอบก่อนทำลาย
คำสั่ง `lsblk`, การ mount แบบ `ro` และการตรวจ SHA-256 ช่วยเปลี่ยนงานจาก “ลองดู” ให้เป็นขั้นตอนที่ตรวจสอบย้อนกลับได้

### 16.3 Human × AI × Digital Labor
มนุษย์ตัดสินใจเรื่องเป้าหมาย ความเสี่ยง และข้อมูลที่ต้องเก็บ ส่วนเครื่องทำงานซ้ำที่ชัดเจน เช่น ดาวน์โหลด คำนวณ checksum และเขียน image

### 16.4 ประสบการณ์ต้องจัดเก็บเป็นองค์ความรู้
งานที่ทำสำเร็จหนึ่งครั้งควรถูกจัดเป็นขั้นตอน เพื่อให้คนรุ่นถัดไปทำซ้ำได้ง่ายกว่าเดิมและไม่ต้องเสียเวลาพบหลุมเดิมอีกครั้ง

---

## 17. คำเตือนสำคัญ

คำสั่งและภาพในคู่มือนี้มาจากเครื่องทดลองจริง แต่ชื่อ device เช่น `/dev/sda` หรือ `/dev/sdb` **ไม่ใช่ค่าคงที่** ของทุกเครื่อง

ก่อนคำสั่งที่มีผลลบ/เขียนข้อมูล ควรตรวจอุปกรณ์จริงด้วย `lsblk` ทุกครั้ง

คู่มือนี้ตั้งใจใช้วิธีที่ตรวจสอบได้และหลีกเลี่ยงคำสั่ง destructive อย่าง `dd` ในขั้นตอนทั่วไป เพื่อให้น้องใหม่เห็นปลายทางผ่าน GUI ชัดเจนก่อนกดยืนยัน

---

## 18. แหล่งอ้างอิงทางการ

- Ubuntu 24.04.4 LTS Release: <https://releases.ubuntu.com/24.04.4/>
- SHA256SUMS: <https://releases.ubuntu.com/24.04.4/SHA256SUMS>

---

## Credits

**Field experience / workflow:** ลุงอั้นตี้  
**AI collaborator / documentation:** น้อนแชตจี้ — GPT-5.6 Sol  
**Mascots & spirit:** Penguin 🐧 × PINTA 🎨  

> **Human × AI Collaborative Work — PINTA Server Field Notes**  
> ทำจริง → ตรวจจริง → บันทึกจริง → ส่งต่อได้จริง
