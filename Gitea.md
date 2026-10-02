# คู่มือติดตั้ง Gitea บน Ubuntu VPS สำหรับทีมที่ใช้ Windows

ปรับปรุงและตรวจเทียบเอกสารทางการ: 2 ตุลาคม 2026

คู่มือนี้ติดตั้ง Gitea แบบ binary + systemd พร้อม SQLite, Git LFS, Nginx และ HTTPS สำหรับทีมขนาดเล็ก ใช้ Windows ทำงานกับสำเนาโปรเจกต์ในเครื่อง แล้ว push/pull ไปยัง VPS ไม่ต้องติดตั้ง Unreal Editor บน VPS

## 1. ขอบเขตและค่าที่ต้องแทน

ใช้กับ **VPS ใหม่** ที่เป็น Ubuntu Server LTS และ CPU `x86_64` (ไฟล์ `linux-amd64`) ตัวอย่างการติดตั้งในเซสชันต้นทางใช้ Ubuntu 26.04 LTS และ Gitea 28.0.0 หากใช้ ARM64 ต้องเปลี่ยน binary และไฟล์ลายเซ็นให้ตรงสถาปัตยกรรม

คำสั่ง Linux ในคู่มือนี้รันด้วย `root` ส่วนคำสั่งที่ระบุ PowerShell รันบน Windows หากใช้บัญชี sudo ให้เข้าสู่ root shell ด้วย `sudo -i` ก่อนทำขั้นติดตั้ง อย่ารันคำสั่ง Linux ใน PowerShell โดยตรง

| ตัวอย่าง / placeholder | แทนด้วย |
| --- | --- |
| `203.0.113.10` | Public IPv4 ของ VPS (เลขนี้เป็น IP สำหรับเอกสาร ใช้งานจริงไม่ได้) |
| `git.example.com` | โดเมนหรือ subdomain ของ Gitea เช่น `git.โดเมนของทีม` |
| `example.com` | โดเมนหลักของทีม |
| `Team Gitea` | ชื่อเว็บ / ชื่อทีม |
| `team-admin` | ชื่อบัญชีผู้ดูแล Gitea |
| `YOUR_ORG` / `YOUR_REPO` | ชื่อ Organization และ repository ที่สร้างจริง |

เปลี่ยนโดเมนและ IP ตัวอย่างก่อนรันคำสั่ง อย่าเปลี่ยนโดเมนของแหล่งดาวน์โหลด เช่น `dl.gitea.com` หรือ `keys.openpgp.org` ค่า `localhost` ในขั้นทดสอบต้องคงไว้จนถึงขั้นเปลี่ยน URL

ต้องมีสิทธิ์จัดการ DNS ของโดเมน และเข้าถึงพอร์ต 80/443 จากอินเทอร์เน็ตเพื่อใช้วิธีออกใบรับรองในคู่มือนี้ พอร์ต SSH ตัวอย่างคือ 22 หากใช้พอร์ตอื่นต้องปรับคำสั่ง SSH และกฎ firewall ตามจริง

คู่มือนี้ **ยังไม่ครอบคลุมการสร้าง SSH key, ปิด root login หรือกำหนด fail2ban** ให้ทำเป็นขั้นดูแล SSH แยกต่างหากหลังทดสอบการเข้าถึง วิธีนี้ไม่ควรถูกตีความว่าระบบได้รับการตรวจความปลอดภัยครบแล้ว

> สำหรับเซิร์ฟเวอร์ที่มี Gitea หรือเว็บไซต์เดิมอยู่แล้ว อย่ารันขั้นสร้างไฟล์ซ้ำ: `cat >` จะเขียนทับไฟล์เดิม ต้องตรวจและสำรองก่อน ห้ามนำคู่มือติดตั้งใหม่ไปใช้แทนขั้นอัปเกรด

## 2. เชื่อมต่อและตรวจเครื่อง

**Windows PowerShell:**

```powershell
ssh root@203.0.113.10
```

ครั้งแรกให้ตรวจ host key fingerprint เทียบกับช่องทางที่เชื่อถือได้ของผู้ให้บริการก่อนยอมรับ เมื่อเข้า VPS แล้วตรวจ:

```bash
cat /etc/os-release
uname -m
df -h /
free -h
```

คาดว่า `uname -m` เป็น `x86_64` และพื้นที่ว่างพอสำหรับ repo, LFS และประวัติ Asset ถ้ามีดิสก์ข้อมูลแยก ต้อง mount และวางแผน path ก่อนติดตั้ง ค่า path ในคู่มือนี้ใช้ดิสก์ที่ mount เป็น `/`

ติดตั้งเครื่องมือ:

```bash
apt update
apt install -y git curl wget ca-certificates gnupg nano
git --version
```

หาก apt รอ lock จาก `unattended-upgrades` ให้รอจนจบ อย่าลบไฟล์ lock หรือ kill กระบวนการอัปเดต อัปเดตแพ็กเกจที่ค้างตามขั้นดูแลระบบท้ายคู่มือก่อนส่งมอบใช้งานจริง

## 3. สร้างบัญชีบริการและโฟลเดอร์

บัญชี Linux `git` ใช้รัน Gitea สมาชิกทีมแต่ละคนใช้บัญชีใน Gitea ไม่ต้องสร้าง Linux user รายคนสำหรับ HTTPS push/pull

```bash
adduser --system --shell /bin/bash --gecos 'Git Version Control' --group --disabled-password --home /home/git git

install -d -o git -g git -m 750 /var/lib/gitea
install -d -o git -g git -m 750 /var/lib/gitea/custom /var/lib/gitea/data /var/lib/gitea/log
install -d -o root -g git -m 770 /etc/gitea

id git
ls -ld /home/git /var/lib/gitea /etc/gitea
```

หากบัญชี `git` มีอยู่แล้ว ให้ตรวจที่มาและสิทธิ์ก่อนใช้ ห้ามเปลี่ยนบัญชีของบริการอื่นโดยไม่ตรวจ `/etc/gitea` เขียนได้โดยกลุ่ม `git` ชั่วคราวเพื่อให้ web installer บันทึก config หลังติดตั้งจะจำกัดสิทธิ์

## 4. ดาวน์โหลด ตรวจลายเซ็น และติดตั้ง binary

ตัวอย่างตรึงเวอร์ชัน `28.0.0` เพื่อให้ทำซ้ำได้ ไม่ใช่คำยืนยันว่าเป็นเวอร์ชันล่าสุดตลอดไป ก่อนติดตั้งใหม่ให้ตรวจ stable release และข้อกำหนดจาก [หน้าดาวน์โหลดทางการ](https://dl.gitea.com/gitea/) และ [คู่มือติดตั้ง binary](https://docs.gitea.com/installation/install-from-binary/)

ตัวแปรด้านล่างอยู่ใน shell ของ VPS ปัจจุบัน หากเปิด SSH ใหม่ให้กำหนดตัวแปรอีกครั้ง

```bash
GITEA_VERSION='28.0.0'
GITEA_ARCH='linux-amd64'
GITEA_FILE="gitea-${GITEA_VERSION}-${GITEA_ARCH}"

mkdir -p /root/gitea-install
cd /root/gitea-install
curl -fLO "https://dl.gitea.com/gitea/${GITEA_VERSION}/${GITEA_FILE}"
curl -fLO "https://dl.gitea.com/gitea/${GITEA_VERSION}/${GITEA_FILE}.asc"
ls -lh "$GITEA_FILE" "${GITEA_FILE}.asc"
```

ถ้าดาวน์โหลดใดล้มเหลว ให้หยุดและตรวจ URL/เวอร์ชัน อย่าติดตั้งไฟล์เก่าที่อาจค้างในโฟลเดอร์

รับ signing key โดยตรวจ fingerprint เทียบ [คู่มือ Gitea](https://docs.gitea.com/installation/install-from-binary/#gpg) ก่อนใช้งาน (key อาจเปลี่ยนในอนาคต):

```bash
gpg --keyserver hkps://keys.openpgp.org --recv-keys 7C9E68152594688862D62AF62D9AE806EC1592E2
gpg --fingerprint 7C9E68152594688862D62AF62D9AE806EC1592E2
```

ตรวจลายเซ็นแล้วติดตั้ง **เฉพาะเมื่อตรวจผ่าน** ด้วย `&&`:

```bash
gpg --verify "${GITEA_FILE}.asc" "$GITEA_FILE" && install -o root -g root -m 755 "$GITEA_FILE" /usr/local/bin/gitea
/usr/local/bin/gitea --version
```

ต้องเห็น `Good signature` จาก key ที่ fingerprint ตรงกับทางการ หากขึ้น `BAD signature`, `No public key` หรือโหลด key ไม่ได้ ให้หยุด คำเตือน `not certified with a trusted signature` หมายถึงยังไม่ได้รับรองความเชื่อถือของ key ใน keyring ของเครื่อง ไม่ใช่ลายเซ็นไฟล์ผิด และไม่ใช่เหตุผลให้ข้ามการตรวจ fingerprint

## 5. สร้าง config สำหรับติดตั้งผ่าน SSH tunnel

สำหรับการติดตั้งใหม่เท่านั้น คัดลอกทั้งบล็อกรวม `EOF` บรรทัดสุดท้าย:

```bash
cat > /etc/gitea/app.ini <<'EOF'
APP_NAME = Team Gitea
RUN_USER = git
WORK_PATH = /var/lib/gitea
RUN_MODE = prod

[server]
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 3000
DOMAIN = localhost
ROOT_URL = http://localhost:3000/
LFS_START_SERVER = true

[lfs]
PATH = /var/lib/gitea/data/lfs

[security]
INSTALL_LOCK = false

[service]
DISABLE_REGISTRATION = true
REQUIRE_SIGNIN_VIEW = true
EOF

chown git:git /etc/gitea/app.ini
chmod 640 /etc/gitea/app.ini
ls -l /etc/gitea/app.ini
```

เว็บฟังเฉพาะ loopback `127.0.0.1` ไม่เปิดพอร์ต 3000 สู่ภายนอก `INSTALL_LOCK = false` ใช้เฉพาะก่อนติดตั้ง ส่วน Git LFS เปิดที่เซิร์ฟเวอร์แล้ว แต่ฝั่ง Windows ยังต้องกำหนด `.gitattributes` ก่อน commit Asset

## 6. สร้าง systemd service

```bash
cat > /etc/systemd/system/gitea.service <<'EOF'
[Unit]
Description=Gitea
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now gitea
systemctl status gitea --no-pager -l
```

ต้องเห็น `active (running)` และเว็บฟังที่ `127.0.0.1:3000` ถ้าเปิดไม่สำเร็จ:

```bash
journalctl -u gitea -n 50 --no-pager
```

แก้สาเหตุให้สำเร็จก่อนทำขั้นต่อไป ไม่ควรให้ restart loop ทำงานค้างโดยไม่แก้ไข

## 7. เปิด web installer จาก Windows

เปิด PowerShell **อีกหน้าต่างบน Windows** และคง SSH เดิมไว้:

```powershell
ssh -N -o ExitOnForwardFailure=yes -L 127.0.0.1:3000:127.0.0.1:3000 root@203.0.113.10
```

ปล่อยหน้าต่าง tunnel เปิดอยู่ แม้ไม่มีข้อความก็เป็นปกติ แล้วเปิด `http://localhost:3000/` หากพอร์ต 3000 บน Windows ถูกใช้อยู่ ให้หยุดโปรแกรมที่ใช้อย่างเหมาะสม หรือปรับพอร์ต local และ URL ทดสอบให้ตรงกัน

ตั้งค่า installer (ชื่อช่องอาจต่างตามเวอร์ชัน):

| ช่อง | ค่า |
| --- | --- |
| Database Type | SQLite3 |
| Database Path | `/var/lib/gitea/data/gitea.db` |
| Site Title | ชื่อทีม เช่น `Team Gitea` |
| Data Path | `/var/lib/gitea/data` |
| Run As Username | `git` |
| Repository Root Path ถ้ามี | `/var/lib/gitea/data/gitea-repositories` |
| SSH Server Port | `22` หาก SSH ของ VPS ใช้พอร์ตนี้ |
| HTTP Server Port | `3000` |
| Domain ถ้ามี | `localhost` |
| Gitea Website URL / Base URL | `http://localhost:3000/` |

เปิด **Only administrators can create user accounts** และ **Require sign-in to view pages** เปิด update checker ได้ (เป็นการแจ้งเตือน ไม่ได้อัปเดต binary ให้อัตโนมัติ) ไม่จำเป็นต้องเปิด CAPTCHA เมื่อปิดสมัครสมาชิกเอง

เปิด **Administrator Account Settings** กรอก username เช่น `team-admin`, อีเมลจริง และรหัสผ่านที่ไม่ซ้ำกับ root แล้วกด **Install Gitea** ต้องเข้า Dashboard ได้

เว้น SMTP ไว้ก่อนได้ แต่จะยังส่งอีเมลแจ้งเตือนและลิงก์รีเซ็ตรหัสไม่ได้ บัญชีผู้ดูแล Gitea แยกจาก root และ Linux user `git`

หลังติดตั้ง ห้ามแชร์ `app.ini` ทั้งไฟล์ เพราะมี secret ให้ตรวจเฉพาะค่าที่จำเป็น:

```bash
grep -E '^[[:space:]]*(INSTALL_LOCK|DISABLE_REGISTRATION|REQUIRE_SIGNIN_VIEW)[[:space:]]*=' /etc/gitea/app.ini
```

ต้องเป็น `INSTALL_LOCK = true`, `DISABLE_REGISTRATION = true` และ `REQUIRE_SIGNIN_VIEW = true` ตรวจว่าค่าแต่ละตัวอยู่ใต้ section ที่ถูกต้อง หาก installer เปลี่ยนค่า ให้แก้ค่านั้นใน section เดิมและ restart Gitea ไม่ต้องสร้าง config ทั้งไฟล์ใหม่

จำกัดสิทธิ์หลังติดตั้ง:

```bash
chown root:git /etc/gitea/app.ini
chmod 750 /etc/gitea
chmod 640 /etc/gitea/app.ini
ls -ld /etc/gitea
ls -l /etc/gitea/app.ini
```

## 8. ตั้ง DNS ของโดเมน

เพิ่ม record ที่ผู้ให้บริการ DNS ซึ่งเป็น authoritative nameserver ของโดเมน (อาจต่างจากผู้จดทะเบียนโดเมน) หากใช้ Squarespace เข้า Domains → เลือกโดเมน → DNS / DNS Settings → Custom Records → Add Record

| ช่อง | ค่าตัวอย่าง |
| --- | --- |
| Host / Name | `git` สำหรับ `git.example.com` |
| Type | `A` |
| Data / Value | `203.0.113.10` → ต้องแทนด้วย Public IPv4 จริง |
| TTL | 1 hour ถ้ามี หรือค่าเริ่มต้น |
| Priority | ไม่ใช้สำหรับ A record / เว้นว่าง |

ไม่ใส่ `http://`, path หรือ `:3000` ในค่า IP หากมี A/CNAME ของ host `git` อยู่แล้ว ให้ตรวจและแก้รายการเดิมอย่างเหมาะสม อย่าเพิ่มรายการขัดกัน ไม่ต้องแก้ `@`, `www` หรือ MX ของอีเมล

หากมี AAAA ของ `git` ต้องชี้ IPv6 ที่ VPS ใช้งานได้และเปิดเว็บ/ตรวจใบรับรองได้จริง มิฉะนั้นเอา AAAA ที่ผิดออกหลังตรวจว่ารายการนั้นไม่ใช่บริการอื่น หากตั้ง CAA ให้แน่ใจว่าอนุญาต Let's Encrypt (`letsencrypt.org`)

**Windows PowerShell:**

```powershell
nslookup git.example.com
```

ต้องตอบ IP ของ VPS จริง รอ DNS propagation หากยังตอบค่าเก่า ไม่ใช้ IP เฉพาะเครื่องใดเป็นค่าคาดหวังของทุกทีม

## 9. ตั้ง Nginx reverse proxy

**VPS:**

```bash
apt install -y nginx
systemctl enable --now nginx
```

สร้างไฟล์สำหรับเซิร์ฟเวอร์ใหม่ โดยแทน `git.example.com` ก่อนวาง:

```bash
cat > /etc/nginx/sites-available/gitea <<'EOF'
server {
    listen 80;
    server_name git.example.com;

    client_max_body_size 10G;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_request_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
EOF

ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/gitea
nginx -t
```

`10G` เป็นตัวอย่างเพดาน **ต่อ HTTP request** ไม่ใช่โควตาพื้นที่ repo หรือข้อจำกัด LFS ทั้งระบบ ปรับตามขนาด Asset ที่ทีมต้องส่งจริง หากใช้ `0` จะปิดเพดานนี้ ซึ่งเพิ่มความเสี่ยงจากการอัปโหลดขนาดใหญ่และการใช้ทรัพยากรจนเต็ม

ถ้า symlink มีอยู่แล้ว อย่าลบทันที ให้ตรวจว่าอ้างไฟล์ที่ต้องการ ไม่มี `server_name` เดียวกันซ้ำใน config อื่น เมื่อ `nginx -t` ผ่านเท่านั้นจึงรัน:

```bash
systemctl reload nginx
curl -I -H 'Host: git.example.com' http://127.0.0.1
```

ต้องได้การตอบจากเว็บที่คาดไว้ เช่น 200 หรือ redirect ที่เหมาะสม หากเป็น 502 ให้ตรวจ Gitea และ `journalctl` การตอบ 200 เพียงอย่างเดียวไม่ได้พิสูจน์ว่าเป็น Gitea หากมีเว็บอื่นบนเซิร์ฟเวอร์ ให้ตรวจเนื้อหาด้วย

ตรวจ firewall ของ VPS และของผู้ให้บริการ: อนุญาต TCP 80/443 พร้อมคงพอร์ต SSH จริงไว้ ไม่เปิดพอร์ต 3000 สู่ภายนอก หากใช้ UFW ให้ตรวจ `ufw status verbose` ก่อนแก้ และอย่าเปิด firewall โดยยังไม่อนุญาต SSH หรือไม่มีช่องทาง console กู้คืนจากผู้ให้บริการ

**Windows PowerShell:**

```powershell
Test-NetConnection git.example.com -Port 80
```

ต้องได้ `TcpTestSucceeded : True` ยังไม่ล็อกอินหรือส่ง credentials ผ่าน HTTP ในช่วงนี้

## 10. ออกใบรับรอง HTTPS

**VPS:**

```bash
apt install -y certbot python3-certbot-nginx
certbot --version
cp -a /etc/nginx /root/nginx-before-https
certbot --nginx -d git.example.com --redirect
```

โฟลเดอร์ `/root/nginx-before-https` เป็นสำเนาก่อนให้ Certbot แก้ไข สำหรับการรันครั้งแรกเท่านั้น หากมีสำเนาชื่อนี้แล้วให้ใช้ชื่อใหม่เพื่อไม่ให้สำเนาปะปน ใส่อีเมลจริง อ่านและยอมรับเงื่อนไขหากตกลง การรับข่าวสารเป็นตัวเลือก ต้องเห็นข้อความออกและติดตั้งใบรับรองสำเร็จ Certbot จะปรับ Nginx ให้รับ HTTPS และ redirect HTTP

หากล้มเหลว ให้ตรวจ DNS A/AAAA, CAA, พอร์ต 80 และ config Nginx ก่อนลองใหม่ ไม่ retry ถี่ ๆ เพราะมี rate limit ของการออกใบรับรอง

## 11. เปลี่ยน URL ของ Gitea เป็นโดเมนจริง

สำรอง config ภายในโฟลเดอร์ root และจำกัดสิทธิ์ (ไฟล์มี secret):

```bash
install -o root -g root -m 600 /etc/gitea/app.ini /root/gitea-app.ini.before-domain
nano /etc/gitea/app.ini
```

แก้ **บรรทัดเดิม** ใน `[server]` ให้เป็นค่าด้านล่าง เพิ่มเฉพาะ `SSH_DOMAIN` ถ้าไม่มี อย่าสร้าง `[server]` ซ้ำ และอย่าวางชื่อ key ซ้ำในค่าด้านขวา:

```ini
DOMAIN = git.example.com
SSH_DOMAIN = git.example.com
ROOT_URL = https://git.example.com/
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 3000
```

URL ต้องลงท้าย `/` ถูก: `ROOT_URL = https://git.example.com/` ผิด: `ROOT_URL = ROOT_URL = https://git.example.com/`

HTTPS จบที่ Nginx จึงให้ Gitea ภายในใช้ HTTP ต่อไป ไม่เปลี่ยน Gitea ให้ฟัง 443 หรือ `0.0.0.0` ค่า `SSH_DOMAIN` กำหนด URL clone ไม่ได้เปลี่ยน DNS หรือเปิด SSH server ให้เอง

บันทึก nano ด้วย Ctrl+O → Enter แล้วออก Ctrl+X จากนั้น:

```bash
systemctl restart gitea
systemctl status gitea --no-pager -l
nginx -t
curl -I https://git.example.com/
```

ถ้า Gitea เป็น `failed` หรือ `activating (auto-restart)` ให้ดู `journalctl -u gitea -n 50 --no-pager` ก่อนแก้เพิ่ม ทดสอบ HTTPS โดยไม่ใช้ `curl -k` เพื่อไม่ข้ามการตรวจใบรับรอง

เปิด `https://git.example.com/` บน Windows ตรวจว่าไม่มีคำเตือนใบรับรองและล็อกอินได้ จากนั้นปิด SSH tunnel ได้ ไม่ต้องใช้ tunnel สำหรับเว็บอีก

## 12. ตรวจต่ออายุ HTTPS และอัปเดต Ubuntu

### HTTPS อัตโนมัติ

```bash
systemctl status certbot.timer --no-pager
certbot renew --dry-run
```

คาดว่า timer เป็น `active (waiting)` และการจำลองต่ออายุสำเร็จทุกใบรับรอง หาก timer ไม่เปิด ให้ตรวจว่าใช้ Certbot จาก apt ตามคู่มือนี้แล้วรัน `systemctl enable --now certbot.timer` Timer ตรวจเป็นระยะและต่ออายุเมื่อถึงเกณฑ์ ไม่ได้ออกใบรับรองใหม่ทุกครั้ง

ต้องคง DNS และพอร์ต 80 ให้รองรับ HTTP challenge รวมถึงพอร์ต 443 สำหรับเว็บ ทดสอบ dry-run หลังเปลี่ยน DNS/firewall และตรวจความล้มเหลวของบริการต่ออายุเป็นระยะ

### อัปเดตอัตโนมัติ

```bash
systemctl status apt-daily-upgrade.timer --no-pager
cat /etc/apt/apt.conf.d/20auto-upgrades
apt-config dump | grep -E 'APT::Periodic|Unattended-Upgrade::(Allowed-Origins|Origins-Pattern|Automatic-Reboot)'
```

สองค่า periodic ที่ต้องเปิดคือ:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

การเปิด timer และ periodic ไม่ได้ยืนยันว่าทุกแพ็กเกจอัปเดตอัตโนมัติ ต้องดู allowed origins และ log ด้วย Ubuntu มักเปิด security updates เป็นค่าเริ่มต้น หากยังไม่ได้ติดตั้ง/เปิด ให้ใช้:

```bash
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

เลือกเปิดใช้งาน แล้วตรวจค่าข้างต้นอีกครั้ง ตรวจการทำงานได้ด้วย:

```bash
unattended-upgrade --dry-run --debug
tail -n 30 /var/log/unattended-upgrades/unattended-upgrades.log
```

ไม่เปิด automatic reboot โดยไม่วางแผน downtime และช่องทางกู้คืน Gitea binary ใน `/usr/local/bin` **ไม่อัปเดตผ่าน apt** ต้องติดตาม stable/security releases และใช้ขั้นอัปเกรดของ Gitea พร้อม backup ก่อน

### อัปเดตด้วยตนเองและรีบูตเมื่อจำเป็น

ทำช่วงที่ทีมไม่ push/pull สำรองข้อมูลก่อนอัปเดตสำคัญ:

```bash
apt update
apt list --upgradable
apt upgrade
```

หากรายการเปิดผ่าน pager กด `q` เพื่อออก อ่านรายการแล้วตอบ Y เมื่อต้องการอัปเดต หากถาม config โดยเฉพาะ SSH/network ให้ตรวจ diff และคงค่าที่ใช้งานได้จนประเมินแล้ว อย่าเลือกเขียนทับโดยไม่อ่าน

เมื่อ apt จบและกลับมาที่ prompt ตรวจ:

```bash
systemctl is-active gitea nginx
if [ -f /var/run/reboot-required ]; then
    cat /var/run/reboot-required
    cat /var/run/reboot-required.pkgs 2>/dev/null
fi
uname -r
```

หากมีข้อความแจ้ง kernel ใหม่หรือ reboot-required ให้นัด downtime ตรวจว่ามี provider console ใช้กู้คืนได้ แล้วค่อยรัน **แยกต่างหาก**:

```bash
reboot
```

SSH จะตัด เว็บหยุดชั่วคราว เมื่อเครื่องกลับมา **Windows PowerShell:**

```powershell
ssh root@203.0.113.10
```

ตรวจบน VPS:

```bash
uname -r
systemctl is-active gitea nginx
systemctl status certbot.timer --no-pager
```

ไม่ยึดเลข kernel ตายตัวสำหรับทุกเครื่อง ให้เทียบกับ kernel ที่ติดตั้งล่าสุดและข้อความหลังอัปเดต แล้วทดสอบเว็บและ push/pull อีกครั้ง

## 13. ส่งมอบให้ทีมและตรวจความพร้อม

สร้าง Organization และ **private repository** แยกบัญชีสมาชิกแต่ละคน ไม่แชร์บัญชี admin ให้สิทธิ์ผ่าน Teams เท่าที่จำเป็น เปิด 2FA และเก็บ recovery codes หากมี local repo อยู่แล้ว ให้สร้าง repo ปลายทางแบบว่าง ไม่ initialize README ซ้ำ

GitHub Desktop ใช้ commit/push/pull กับ repo นี้ได้ แต่ปุ่ม Publish repository มุ่งไป GitHub ให้สร้าง repo บนเว็บ Gitea ก่อนแล้วตั้ง remote ใช้ URL ที่คัดลอกจาก repo จริง

**ใน terminal ของ local repo บน Windows** (ไม่ใช่ SSH ของ VPS):

```bash
git remote -v
```

ถ้าไม่มี origin:

```bash
git remote add origin https://git.example.com/YOUR_ORG/YOUR_REPO.git
```

ถ้ามี origin และตั้งใจเปลี่ยนปลายทาง:

```bash
git remote set-url origin https://git.example.com/YOUR_ORG/YOUR_REPO.git
```

ตรวจ commit และชื่อ branch จริงก่อน push (ตัวอย่างใช้ `main`):

```bash
git status
git branch --show-current
git push -u origin main
```

ใช้บัญชี Gitea สำหรับ credentials ไม่ใช้รหัส root หากเปิด 2FA ให้ใช้วิธี authentication ที่ Gitea รองรับ เช่น access token สำหรับ HTTPS โดยจำกัดสิทธิ์ตามงาน ไม่ฝัง token ลง URL ไม่แชร์ token สมาชิกแต่ละคนมี credentials ของตัวเอง

สำหรับ Unreal ให้ติดตั้ง Git LFS บน Windows ทุกเครื่อง แล้วทำ **ก่อน add/commit Asset ครั้งแรก**:

```bash
git lfs install
git lfs track "*.uasset"
git lfs track "*.umap"
git add .gitattributes
```

เพิ่ม .gitignore ที่เหมาะกับโปรเจกต์ เช่น `Intermediate/`, `Saved/`, `DerivedDataCache/` ตรวจ plugin และ binary ที่ทีมจำเป็นต้องเก็บก่อน ignore ทั้งหมด `git lfs track` ไม่ย้ายประวัติเดิมเข้า LFS และอย่า rewrite ประวัติ repo ที่ทีมใช้งานร่วมกันโดยไม่ได้วางแผน

ทดสอบด้วยบัญชีสมาชิกจริง: push/pull, clone ลงโฟลเดอร์ใหม่, `git lfs pull`, ตรวจว่า Asset เปิดใน Unreal ได้ พร้อมทดสอบสิทธิ์ปฏิเสธ repo ที่ไม่ได้รับอนุญาต Branch protection และ file locking สำหรับ Asset ต้องวางแผนแยกจากการติดตั้งเซิร์ฟเวอร์

ก่อนใช้เป็นแหล่งเก็บงานหลัก ต้องมี backup **นอก VPS** ที่รวม repo, LFS, SQLite, config และข้อมูลจำเป็นอื่น พร้อมทดสอบกู้คืน ใช้ขั้น backup ที่ทำให้ข้อมูลสอดคล้องกัน ไม่คัดลอกฐานข้อมูลที่กำลังเขียนแบบสุ่ม `git clone` อย่างเดียวไม่ใช่ backup ของ Gitea ทั้งระบบหรือ LFS ทุกเวอร์ชัน

ยังต้องทำขั้น SSH key/firewall/fail2ban ตามนโยบายทีม อย่าปิด password/root login จนทดสอบบัญชีและ key ที่จะใช้แทนผ่านใน SSH อีกหน้าต่างแล้ว เก็บช่องทาง provider console ไว้สำหรับกู้คืน

## แหล่งอ้างอิง

- [Gitea: ติดตั้ง binary และตรวจลายเซ็น](https://docs.gitea.com/installation/install-from-binary/)
- [Gitea: systemd service](https://docs.gitea.com/installation/linux-service/)
- [Gitea: reverse proxies](https://docs.gitea.com/administration/reverse-proxies/)
- [Gitea: Git LFS](https://docs.gitea.com/administration/git-lfs-setup/)
- [Gitea: backup และ restore](https://docs.gitea.com/administration/backup-and-restore/)
- [Ubuntu: automatic updates](https://documentation.ubuntu.com/server/how-to/software/automatic-updates/)
- [Certbot: คำสั่งและการต่ออายุ](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Squarespace: แก้ไข DNS records](https://support.squarespace.com/hc/en-us/articles/360002101888-Edit-your-domain-s-DNS-records)
- [GitHub Desktop: เพิ่ม repo รวมถึง repo นอก GitHub](https://docs.github.com/en/desktop/adding-and-cloning-repositories)

คู่มือนี้ตรวจคำสั่งและลำดับกับเอกสารทางการและผลการติดตั้งในเซสชันต้นทาง ไม่ได้ทดสอบติดตั้งใหม่ครบทุกขั้นบน VPS ของทุกทีม ต้องตรวจสถาปัตยกรรม เวอร์ชัน พอร์ต และ policy จริงก่อนใช้งาน
