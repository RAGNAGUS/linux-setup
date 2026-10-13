# ล็อกอินผู้ดูแลด้วย SSH key และปิดล็อกอินด้วยรหัสผ่าน

ใช้กับ Linux server ทุกเครื่องของบริษัท (Ubuntu) ผู้ดูแลเข้าเครื่องจาก Windows ด้วย SSH key แทนรหัสผ่าน แล้วปิดการล็อกอินด้วยรหัสผ่านทาง SSH เพื่อกันบอทเดารหัส

**พอร์ต SSH (22) ยังเปิดเหมือนเดิม:** สิ่งที่ปิดคือวิธีล็อกอินด้วยรหัสผ่าน ไม่ใช่พอร์ต
- **ห้ามปิดพอร์ต SSH:** จะเข้าเครื่องไม่ได้
- **บนเครื่อง Gitea:** Git ผ่าน SSH ใช้พอร์ตนี้ด้วย (ดู `Gitea.md` ข้อ 15)

| | ก่อน | หลัง |
| --- | --- | --- |
| พอร์ต 22 | เปิด | เปิด |
| ล็อกอิน SSH ด้วยรหัสผ่าน | ใช้ได้ ใครก็เดาได้ | ปิด |
| ล็อกอิน SSH ด้วย key | - | ใช้ได้เฉพาะคนที่มีกุญแจลับ |
| ล็อกอินผ่าน Console บนเว็บของผู้ให้บริการ | ใช้รหัสผ่าน root | **ยังใช้รหัสผ่าน root ได้** (ทางสำรองเมื่อ SSH พัง) |

## ต้องทำทั้งหมดไหม

| สถานการณ์ | ควรทำ |
| --- | --- |
| ทุกเครื่อง | **ข้อ 6 fail2ban** ทำได้ทันทีโดยไม่ต้องทำข้อ 1–5 ไม่มีความเสี่ยงล็อกตัวเองออก |
| ผู้ดูแลคนเดียว ใช้รหัสผ่าน root สุ่มยาว (20 ตัวขึ้นไป) ไม่ซ้ำที่อื่น เก็บใน password manager | ข้อ 1–4 (ใช้ key) **ไม่บังคับ** ทำเพื่อความสะดวกได้ และไม่ต้องทำข้อ 5 ก็ได้ |
| ผู้ดูแลหลายคน หรือเครื่องเก็บข้อมูลสำคัญของหลายฝ่าย | ทำข้อ 1–5 ทุกคนใช้ key ของตัวเอง ไม่ต้องแชร์รหัสผ่าน root |
| รหัสผ่าน root สั้น เดาได้ หรือเคยใช้ที่อื่น | เปลี่ยนรหัสผ่านทันที แล้วทำข้อ 1–5 |

**เหตุผล:**
- รหัสผ่านสุ่มยาวมากเดาไม่ได้ในทางปฏิบัติ
- ความเสี่ยงที่เหลือของรหัสผ่านคือ**รหัสหลุด** เช่น เครื่องติดโปรแกรมดักคีย์บอร์ด หรือเผลอวางผิดที่ ส่วนกุญแจลับของ SSH key ไม่เคยถูกพิมพ์หรือส่งออกจากเครื่อง
- **key หายไม่ทำให้เข้าเครื่องไม่ได้:** Console บนเว็บของผู้ให้บริการยังใช้รหัสผ่าน root ได้เสมอ (ข้อ 0) เข้าทางนั้นแล้ววาง key ใหม่

| ตัวอย่าง / placeholder | แทนด้วย |
| --- | --- |
| `server.example.com` | โดเมนหรือ IP ของเครื่อง |
| `myserver` | ชื่อย่อที่จะพิมพ์แทน เช่น `gitvps` |
| `admin_ชื่อ` | ชื่อไฟล์ key ของผู้ดูแลแต่ละคน เช่น `admin_somchai` |

**ห้ามข้ามลำดับ:** ข้อ 5 (ปิดรหัสผ่าน) ทำได้ก็ต่อเมื่อข้อ 3 ผ่านแล้วเท่านั้น และระหว่างทำข้อ 5 ต้อง**เปิดหน้าต่าง SSH เดิมค้างไว้เสมอ** จนทดสอบหน้าต่างใหม่ผ่าน

## 0. เตรียมทางสำรองก่อน

1. **ลองเข้า Console บนเว็บของผู้ให้บริการ VPS:** มักชื่อ VNC / Console / Emergency Console
   - ต้องล็อกอินเป็น root ด้วยรหัสผ่านได้จริง
   - ใช้กู้คืนถ้าเข้า SSH ไม่ได้
2. **เก็บรหัสผ่าน root ไว้ใน password manager:** หลังทำเสร็จจะไม่ได้ใช้ทาง SSH แล้ว แต่ Console ยังต้องใช้
3. **รู้ IP ของ Windows ที่ใช้อยู่:** ใช้ปลดแบนถ้า fail2ban แบนตัวเอง ดูได้จากเว็บ เช่น https://ifconfig.me

## 1. สร้าง key ของผู้ดูแลบน Windows

key ของผู้ดูแลเข้าเครื่องเป็น root ได้ จึง**ต้องตั้ง passphrase** และเก็บใน `$HOME\.ssh` ของเจ้าของ (ไม่ใช่โฟลเดอร์ของโปรเจกต์) ผู้ดูแลแต่ละคนมี key ของตัวเอง ห้ามใช้ร่วมกัน

**Windows PowerShell:**

```powershell
cd $HOME\.ssh
```

ถ้าขึ้นว่าไม่มีโฟลเดอร์:

```powershell
mkdir $HOME\.ssh
```

```powershell
ssh-keygen -t ed25519 -C "ชื่อ-ชื่อเครื่อง" -f admin_ชื่อ
```

- `Enter passphrase`: ตั้งรหัสสำหรับ key (คนละอันกับรหัส root ของเครื่อง) ตอนพิมพ์จะไม่เห็นตัวอักษร ซึ่งเป็นเรื่องปกติ
- `Enter same passphrase again`: พิมพ์ซ้ำ
- ถ้าถาม `Overwrite (y/n)?` ตอบ `n` แล้วใช้ชื่อไฟล์อื่น

ได้ `admin_ชื่อ` (กุญแจลับ ห้ามส่งให้ใคร) และ `admin_ชื่อ.pub` (เปิดเผยได้) เก็บ passphrase ไว้ใน password manager

## 2. ใส่ public key ให้ root บนเครื่อง

ใช้รหัสผ่าน root เป็นครั้งสุดท้ายเพื่อวาง key **Windows PowerShell** (ในโฟลเดอร์ `$HOME\.ssh`):

```powershell
type admin_ชื่อ.pub | ssh root@server.example.com "umask 077; mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

- จะถามรหัสผ่าน root ตามปกติ ใส่แล้วกลับมาที่ PowerShell โดยไม่มีข้อความอะไร ถือว่าสำเร็จ
- คำสั่งนี้**ต่อท้าย**ไฟล์ ไม่ลบ key เดิมที่มีอยู่
- key นี้ไปอยู่ที่ `/root/.ssh/authorized_keys` ของ root คนละไฟล์กับ key ของ Gitea (`/home/git/.ssh/authorized_keys`)

ตรวจบน **VPS** (หน้าต่าง SSH ที่เข้าเป็น root):

```bash
ls -ld /root/.ssh
ls -l /root/.ssh/authorized_keys
tail -n 1 /root/.ssh/authorized_keys
```

- `/root/.ssh` ต้องเป็น `drwx------`
- `authorized_keys` ต้องเป็น `-rw-------`
- บรรทัดสุดท้ายต้องลงท้ายด้วยคำอธิบายที่ตั้งตอนสร้าง key

## 3. ทดสอบเข้าด้วย key

**Windows PowerShell หน้าต่างใหม่** (หน้าต่างเดิมเปิดค้างไว้):

```powershell
ssh -i $HOME\.ssh\admin_ชื่อ root@server.example.com
```

- **สำเร็จ:** ถาม `Enter passphrase for key ...` (passphrase ของ key) แล้วเข้าเครื่องได้
- **ยังไม่สำเร็จ:** ถาม `root@server.example.com's password:` แปลว่า key ยังใช้ไม่ได้ ให้กด Ctrl+C แล้วตรวจข้อ 2 ใหม่ อย่าทำข้อ 5

## 4. ตั้งชื่อย่อให้พิมพ์ง่าย

เปิดไฟล์ config ของ SSH บน Windows (สร้างใหม่ถ้ายังไม่มี):

```powershell
notepad $HOME\.ssh\config
```

เพิ่มท้ายไฟล์:

```text
Host myserver
    HostName server.example.com
    User root
    IdentityFile ~/.ssh/admin_ชื่อ
    IdentitiesOnly yes
```

- บันทึกแล้วปิด Notepad (ถ้า Notepad เติม `.txt` ให้ตอนบันทึก ต้องเปลี่ยนชื่อไฟล์กลับเป็น `config` ไม่มีนามสกุล)
- ต่อไปเข้าเครื่องด้วย:

  ```powershell
  ssh myserver
  ```

- `Host` ต้องเป็นชื่อย่อที่ไม่ซ้ำกับรายการอื่นในไฟล์ เช่น ถ้ามีรายการ `Host git.example.com` สำหรับ Git อยู่แล้ว (Gitea.md ข้อ 15.6) ให้ตั้งชื่อย่อของผู้ดูแลเป็นอย่างอื่น เพราะรายการนั้นใช้ user `git`

**ไม่อยากพิมพ์ passphrase ทุกครั้ง (ไม่บังคับ):** ใช้ ssh-agent ของ Windows จำ key ไว้จนปิดเครื่อง เปิด **PowerShell แบบ Run as administrator** ครั้งเดียว:

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

แล้วใน PowerShell ปกติ:

```powershell
ssh-add $HOME\.ssh\admin_ชื่อ
```

## 5. ปิดล็อกอินด้วยรหัสผ่าน

**ก่อนเริ่ม:** ข้อ 3 ต้องผ่าน และเปิดหน้าต่าง SSH ที่เข้าเป็น root ค้างไว้ 1 หน้าต่าง **ห้ามปิดจนจบข้อนี้**

### 5.1 ดูค่าปัจจุบัน (VPS)

```bash
sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication)'
grep -rn -E '^\s*(PasswordAuthentication|KbdInteractiveAuthentication|PermitRootLogin)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

คำสั่งแรกแสดงค่าที่ใช้จริง คำสั่งที่สองแสดงว่าค่ามาจากไฟล์ไหน Ubuntu บนบาง VPS มีไฟล์ใน `/etc/ssh/sshd_config.d/` (เช่น `50-cloud-init.conf`) ที่เปิด `PasswordAuthentication yes` ไว้ ถ้าแก้แค่ `sshd_config` ไฟล์นั้นจะทำให้รหัสผ่านยังใช้ได้

### 5.2 สร้างไฟล์ตั้งค่าของเรา (VPS)

ใช้ไฟล์แยกชื่อขึ้นต้น `00-` ให้ถูกอ่านก่อนไฟล์อื่นในโฟลเดอร์ (SSH ใช้ค่าที่อ่านเจอก่อน) ไม่ต้องแก้ไฟล์ของระบบ:

```bash
cat > /etc/ssh/sshd_config.d/00-key-only.conf <<'EOF'
# ผู้ดูแลเข้าด้วย SSH key เท่านั้น (ดู SSHKeyLogin.md)
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
EOF
```

`PermitRootLogin prohibit-password` หมายถึง root เข้าได้ด้วย key เท่านั้น

ตรวจไวยากรณ์ (ไม่มีข้อความ = ผ่าน):

```bash
sshd -t
```

ถ้ามี error **อย่าทำต่อ** ให้ลบไฟล์ที่เพิ่งสร้างแล้วตรวจใหม่:

```bash
rm /etc/ssh/sshd_config.d/00-key-only.conf
```

### 5.3 ใช้ค่าใหม่ และตรวจผล (VPS)

```bash
systemctl reload ssh
sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin)'
```

ต้องได้:

```text
passwordauthentication no
kbdinteractiveauthentication no
permitrootlogin without-password
```

(`without-password` คือชื่อเดิมของ `prohibit-password` ความหมายเดียวกัน) การ reload ไม่ตัดหน้าต่างที่เปิดอยู่

### 5.4 ทดสอบจาก Windows หน้าต่างใหม่

เข้าด้วย key ต้องได้:

```powershell
ssh myserver
```

ลองบังคับใช้รหัสผ่าน ต้อง**ถูกปฏิเสธ**:

```powershell
ssh -o PubkeyAuthentication=no root@server.example.com
```

ผลที่ถูกต้องคือ `Permission denied (publickey).` โดยไม่ถามรหัสผ่าน

**ถ้าเข้าด้วย key ไม่ได้:** กลับไปหน้าต่างเดิมที่เปิดค้างไว้ แล้วคืนค่าเดิม:

```bash
rm /etc/ssh/sshd_config.d/00-key-only.conf && systemctl reload ssh
```

ถ้าไม่มีหน้าต่างเดิมเหลือ ให้เข้าทาง Console ของผู้ให้บริการ (ข้อ 0) แล้วรันคำสั่งเดียวกัน

**เครื่อง Gitea:** ทดสอบ Git ผ่าน SSH ซ้ำ (Gitea.md ข้อ 15.5) ต้องยังผ่าน เพราะ user `git` ใช้ key อยู่แล้ว

## 6. ติดตั้ง fail2ban

แบน IP ที่พยายามล็อกอินผิดซ้ำ ๆ ชั่วคราว ลดภาระและ log ขยะ ทำได้เลยโดยไม่ต้องทำข้อ 1–5 ก่อน

```bash
apt update
apt install -y fail2ban
systemctl enable --now fail2ban
systemctl status fail2ban --no-pager
```

ต้องเห็น `active (running)` แล้วดูว่าป้องกัน SSH อยู่:

```bash
fail2ban-client status
fail2ban-client status sshd
```

- คำสั่งแรกต้องมี `sshd` ใน `Jail list` (Ubuntu เปิด jail นี้ให้เป็นค่าเริ่มต้น)
- คำสั่งที่สองแสดงจำนวนครั้งที่ล็อกอินผิด (`Total failed`) และ IP ที่ถูกแบนอยู่ (`Banned IP list`) เครื่องที่เปิดพอร์ต 22 สู่อินเทอร์เน็ตมักเห็นตัวเลขขึ้นภายในไม่กี่นาที

**ค่าเริ่มต้นของ Ubuntu:** ผิด 5 ครั้งภายใน 10 นาที แบน 10 นาที ถ้าต้องการแบนนานขึ้น สร้างไฟล์ของเราเอง (อย่าแก้ `jail.conf` เพราะถูกเขียนทับตอนอัปเดต):

```bash
cat > /etc/fail2ban/jail.local <<'EOF'
[sshd]
enabled  = true
maxretry = 5
findtime = 10m
bantime  = 1h
EOF
systemctl restart fail2ban
fail2ban-client status sshd
```

**ถ้า `sshd` ไม่อยู่ใน Jail list หรือ fail2ban เปิดไม่ขึ้น:** ดูสาเหตุ

```bash
journalctl -u fail2ban -n 30 --no-pager
```

ถ้าเห็นว่าหาไฟล์ log ของ SSH ไม่เจอ (เช่น `/var/log/auth.log`) ให้อ่าน log จาก systemd แทน โดยเพิ่มบรรทัด `backend = systemd` ใต้ `[sshd]` ใน `jail.local` แล้ว `systemctl restart fail2ban`

ถ้าแบนตัวเองจนเข้าไม่ได้ ให้เข้าทาง Console หรือจากเครื่องอื่น แล้วปลดแบน (แทน IP ของคุณ):

```bash
fail2ban-client set sshd unbanip 203.0.113.50
```

## 7. เพิ่มผู้ดูแล / เครื่องใหม่ และยกเลิก

- **เพิ่มคนหรือเครื่องใหม่:** ทำข้อ 1 บนเครื่องใหม่ ส่งไฟล์ `.pub` ให้คนที่เข้าเครื่องได้อยู่แล้ว แล้วต่อท้ายลง `/root/.ssh/authorized_keys` (หลังข้อ 5 ใช้รหัสผ่านวาง key แบบข้อ 2 ไม่ได้แล้ว):

  ```bash
  nano /root/.ssh/authorized_keys
  ```

  วาง public key เป็นบรรทัดใหม่ท้ายไฟล์ บันทึก แล้วให้เจ้าของ key ทดสอบข้อ 3

- **คนออก หรือเครื่องหาย:** ลบบรรทัดของ key นั้นใน `/root/.ssh/authorized_keys` ทันที ดูได้จากคำอธิบายท้ายบรรทัด
- **ห้ามแก้ `/home/git/.ssh/authorized_keys`:** เป็นของ Gitea จัดการผ่านหน้าเว็บ Gitea เท่านั้น

## ข้อจำกัด

- **ยังใช้ root เข้าเครื่องโดยตรง:** ขั้นถัดไปที่ปลอดภัยกว่าคือสร้าง user ผู้ดูแลที่ใช้ `sudo` แล้วตั้ง `PermitRootLogin no` แต่ต้องวางแผนแยก เพราะคู่มืออื่นในโฟลเดอร์นี้ใช้ root
- **ไม่ได้แทนการอัปเดตระบบ, firewall และ backup:** ดู `Gitea.md` ข้อ 12 และ 14

ข้อ 6 (fail2ban รวม `jail.local` แบน 1 ชั่วโมง) ทดสอบกับเครื่องจริงแล้ว (10 ตุลาคม 2026): fail2ban อ่าน log ของ SSH จาก systemd journal ได้เองโดยไม่ต้องตั้ง `backend` และภายในไม่กี่นาทีพบการล็อกอินผิดจากบอท 48 ครั้ง แบนไป 3 IP ส่วนข้อ 1–5 เขียนตามเอกสาร OpenSSH และ Ubuntu ยังไม่ได้ทดสอบ
