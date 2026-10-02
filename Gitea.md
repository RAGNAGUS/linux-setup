อัพเดตและติดตั้ง Git

```
apt update
```

```
apt install -y git curl wget ca-certificates gnupg
```

สร้างบัญชีและโฟลเดอร์สำหรับ Gitea

`adduser --system --shell /bin/bash --gecos 'Git Version Control' --group --disabled-password --home /home/git git`

สร้างโฟลเดอร์และกำหนดสิทธิ์

```
install -d -o git -g git -m 750 /var/lib/gitea
install -d -o git -g git -m 750 /var/lib/gitea/custom /var/lib/gitea/data /var/lib/gitea/log
install -d -o root -g git -m 770 /etc/gitea
```

 **ดาวน์โหลดไฟล์ Gitea** เท่านั้น รันทีละบรรทัดใน SSH เดิม แล้วดาวน์โหลดไฟล์ลายเซ็นสำหรับตรวจสอบ จากนั้นตรวจว่ามีทั้งสองไฟล์:

```
mkdir -p /root/gitea-install
cd /root/gitea-install
curl -fLO https://dl.gitea.com/gitea/28.0.0/gitea-28.0.0-linux-amd64
curl -fLO https://dl.gitea.com/gitea/28.0.0/gitea-28.0.0-linux-amd64.asc
ls -lh /root/gitea-install
```

รันคำสั่งนี้เพื่อรับ public key ของ Gitea

```
gpg --keyserver hkps://keys.openpgp.org --recv-keys 7C9E68152594688862D62AF62D9AE806EC1592E2
gpg --verify gitea-28.0.0-linux-amd64.asc gitea-28.0.0-linux-amd64
```

ติดตั้งไฟล์ไปที่ `/usr/local/bin/gitea` จากนั้นตรวจเวอร์ชัน

```
install -o root -g root -m 755 /root/gitea-install/gitea-28.0.0-linux-amd64 /usr/local/bin/gitea
/usr/local/bin/gitea --version
```

คัดลอกทั้งบล็อกนี้ลงใน SSH รวมถึงบรรทัด `EOF` สุดท้าย

```
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
EOF
```

จากนั้นกำหนดสิทธิ์ไฟล์ และตรวจผล

```
chown git:git /etc/gitea/app.ini
chmod 640 /etc/gitea/app.ini

ls -l /etc/gitea/app.ini
cat /etc/gitea/app.ini
```

ต่อไปสร้างบริการให้ Gitea เปิดอัตโนมัติหลังรีบูต คัดลอกทั้งบล็อกนี้ลงใน SSH รวมบรรทัด `EOF`

```
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
```

จากนั้นรันทีละบรรทัด สุดท้ายแล้วควรเห็น **`active (running)`**

```
systemctl daemon-reload
systemctl enable --now gitea
systemctl status gitea --no-pager -l
```

เปิด **PowerShell หน้าต่างใหม่บน Windows** โดยเปิด SSH เดิมทิ้งไว้ รันคำสั่งนี้ โดยแทน `YOUR_VPS_IP` ด้วย IP ของ VPS

```
ssh -N -o ExitOnForwardFailure=yes -L 3000:127.0.0.1:3000 root@YOUR_VPS_IP
```

ใส่รหัส root หากระบบถาม หลังจากนั้นหน้าต่างจะเงียบและค้างอยู่ **เป็นปกติ ให้เปิดไว้**

เปิดเบราว์เซอร์บน Windows เข้า:

```
http://localhost:3000/
```

**หน้า setup เปิดได้แล้ว** เริ่มเฉพาะส่วน **Database Settings** ก่อน

เปลี่ยน **Database Type** จาก `MySQL` เป็น **`SQLite3`**

ช่อง **Database Path** ที่ปรากฏ ให้ตั้งเป็น 

```
/var/lib/gitea/data/gitea.db
```

ต่อไปส่วน **General Settings** ให้ใช้ค่าตามนี้

| ช่อง             | ค่า                        |
| --------------- | ------------------------- |
| Site Title      | `Team Gitea` หรือชื่อทีมของคุณ |
| Data Path       | `/var/lib/gitea/data`     |
| Run As Username | `git` — คงเดิม             |

ต่อไปเลื่อนลงไปที่ช่อง **พอร์ตและ URL** ครับ ตั้งตามนี้สำหรับทดสอบผ่าน SSH tunnel:

| ช่อง                        | ค่า                       |
| -------------------------- | ------------------------ |
| SSH Server Port            | `22`                     |
| HTTP Server Port           | `3000`                   |
| Server Domain / Domain ถ้ามี | `localhost`              |
| Gitea Base URL / Base URL  | `http://localhost:3000/` |

`localhost` ใช้ชั่วคราวสำหรับการตั้งค่าครั้งนี้ ก่อนให้ทีมเชื่อมต่อเราจะเปลี่ยนเป็นที่อยู่จริงและตั้ง HTTPS ครับ

จากนั้นเลื่อนลงไปส่วน **Administrator Account Settings** แล้วเปิดส่วนนั้น **ยังไม่ต้องกด Install** ครับ

**พอร์ตและ URL ในภาพถูกต้องสำหรับขั้นทดสอบครับ**

สำหรับเว็บที่ใช้เฉพาะทีม ให้ตั้งช่องเลือกดังนี้:

- ✅ **Only administrators can create user accounts** — คงไว้ คุณจะสร้างบัญชีให้สมาชิกทีม
- ✅ **Require sign-in to view pages** — ติ๊กเพิ่ม เพื่อให้ล็อกอินก่อนดูเว็บ
- ✅ **Enable update checker** — คงไว้
- **Enable registration CAPTCHA** — ปล่อยว่างได้ เพราะปิดสมัครสมาชิกเองแล้ว

จากนั้นคลิก **Administrator Account Settings** แล้วกรอก:

| ช่อง                    | ใส่อะไร                           |
| ---------------------- | -------------------------------- |
| Administrator Username | ชื่อบัญชีของคุณ เช่น `casno`           |
| Password               | รหัสใหม่สำหรับ Gitea แยกจากรหัส root |
| Confirm Password       | รหัสเดียวกันอีกครั้ง                   |
| Email                  | อีเมลของคุณ                        |

บัญชีนี้จะเป็นผู้ดูแล Gitea ส่วน **Email Settings** ยังเว้นไว้ก่อนได้ครับ

กรอกครบแล้วกด **Install Gitea** ได้เลย เปิดหน้าต่าง SSH tunnel ค้างไว้ระหว่างติดตั้ง แล้วบอกผมว่าเข้าหน้า Dashboard ได้หรือมี error ครับ

ขั้นถัดไป **ล็อกสิทธิ์ไฟล์ตั้งค่าหลังติดตั้ง** กลับไปหน้าต่าง SSH ของ VPS แล้วรัน:

```
chown root:git /etc/gitea/app.ini
chmod 750 /etc/gitea
chmod 640 /etc/gitea/app.ini
```

จากนั้นตรวจผล

```
ls -ld /etc/gitea
ls -l /etc/gitea/app.ini
```

เพิ่ม **A record** ให้ `git.mendoka.com` ชี้ไปที่ **Public IPv4 ของ VPS** ครับ ทำเฉพาะ DNS ก่อน

| ช่อง               | ค่า                                         |
| ----------------- | ------------------------------------------ |
| Host / Name       | `git`                                      |
| Type              | `A`                                        |
| Priority          | เว้นว่าง                                     |
| TTL               | `1 hour` ถ้ามี หรือใช้ค่าเดิม                    |
| Data / IP Address | **Public IPv4 ของ VPS จากหน้า InterServer** |

บันทึกแล้วเปิด **PowerShell บน Windows** รัน

```
nslookup git.mendoka.com
```

**DNS ตอบกลับว่า `git.mendoka.com` ชี้ไปที่ `162.35.103.239` แล้วครับ** ถ้าตรงกับ Public IP ของ VPS ใน InterServer ก็พร้อมทำขั้นต่อไป

ตอนนี้ติดตั้ง **Nginx** เพื่อรับการเชื่อมต่อจากโดเมน แล้วส่งต่อไปยัง Gitea ครับ กลับไปหน้าต่าง **SSH ของ VPS** แล้วรัน

```
apt install -y nginx
systemctl status nginx --no-pager -l
```

ใน **SSH ของ VPS** คัดลอกทั้งบล็อกนี้ รวมบรรทัด `EOF`:

```
cat > /etc/nginx/sites-available/gitea <<'EOF'
server {
    listen 80;
    server_name git.mendoka.com;

    client_max_body_size 0;

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
```

ค่า `client_max_body_size 0` ให้ Nginx ไม่จำกัดขนาดไฟล์อัปโหลด เพื่อรองรับ Asset ขนาดใหญ่ครับ

จากนั้นเปิดใช้งาน config และตรวจความถูกต้อง:

```
ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/gitea
nginx -t
```

ต่อไปโหลดค่าใหม่ให้ Nginx

```
systemctl reload nginx
```

จากนั้นทดสอบบน VPS ว่า Nginx ส่งต่อถึง Gitea ได้

```
curl -I -H 'Host: git.mendoka.com' http://127.0.0.1
```

เปิด **PowerShell บน Windows** แล้วรัน

```
Test-NetConnection git.mendoka.com -Port 80
```

ส่งผลลัพธ์มาให้ผมครับ เราต้องการเห็น

```
TcpTestSucceeded : True
```

**พอร์ต 80 เข้าถึงจาก Windows ได้แล้วครับ** ต่อไปติดตั้ง **Certbot** เพื่อออกใบรับรอง HTTPS ให้ `git.mendoka.com` กลับไปหน้าต่าง **SSH ของ VPS** แล้วรัน

```
apt install -y certbot python3-certbot-nginx
```

เมื่อติดตั้งเสร็จ ตรวจด้วย

```
certbot --version
```

**Certbot ติดตั้งแล้วครับ** ต่อไปออกใบรับรอง HTTPS รันใน SSH ของ VPS:

```
certbot --nginx -d git.mendoka.com --redirect
```

ระหว่างทำ ระบบอาจถาม:

- **Email:** ใส่อีเมลของคุณ
- **ยอมรับเงื่อนไข:** อ่านแล้วตอบ `Y` หากยอมรับ
- **รับข่าวสาร/แชร์อีเมล:** ตอบ `N` ได้ เป็นตัวเลือก

คำสั่งนี้จะตั้งใบรับรองให้ Nginx และเปลี่ยน HTTP ให้ไป HTTPS อัตโนมัติ ถ้าสำเร็จจะเห็นข้อความประมาณ:

```
Successfully received certificate.
Successfully deployed certificate ...
```

**HTTPS เปิดสำเร็จแล้วครับ** ต่อไปเปลี่ยน URL ใน Gitea ให้เป็นโดเมนจริง ใน **SSH ของ VPS** สำรอง config ก่อน:

```
cp -p /etc/gitea/app.ini /etc/gitea/app.ini.before-domain
```

เปิดไฟล์:

```
nano /etc/gitea/app.ini
```

ในส่วน **`[server]`** แก้ค่าต่อไปนี้ หากไม่มี `SSH_DOMAIN` ให้เพิ่มในส่วนเดียวกัน:

```
DOMAIN = git.mendoka.com
SSH_DOMAIN = git.mendoka.com
ROOT_URL = https://git.mendoka.com/
```

คงสองค่านี้ไว้ตามเดิม เพราะ Nginx จะส่งต่อเข้ามาภายใน VPS:

```
HTTP_ADDR = 127.0.0.1
HTTP_PORT = 3000
```

บันทึกด้วย **Ctrl+O → Enter** แล้วออกด้วย **Ctrl+X  **จากนั้นรัน:

```
systemctl restart gitea
systemctl status gitea --no-pager -l
```

ขั้นต่อไป บน **Windows** เปิด

```
https://git.mendoka.com
```

ตอนนี้ Git ก็พร้อมใช้แล้ว ต่อไปเริ่มจาก **ตรวจต่ออายุ HTTPS อัตโนมัติ** ก่อน ใน SSH ของ VPS รัน

```
systemctl status certbot.timer --no-pager
```

แล้วทดสอบกระบวนการต่ออายุ:

```
certbot renew --dry-run
```

หากผ่านควรเห็นข้อความประมาณ

```
Congratulations, all simulated renewals succeeded
```

ต่อไป **ตรวจระบบอัปเดตอัตโนมัติของ Ubuntu** ก่อนเปลี่ยนค่าใด ๆ รันทีละคำสั่งใน SSH:

```
systemctl status apt-daily-upgrade.timer --no-pager
cat /etc/apt/apt.conf.d/20auto-upgrades
apt list --upgradable
apt update
apt upgrade
reboot
```

```
ssh root@git.mendoka.com
```

```
uname -r
systemctl is-active gitea nginx
```

ควรเห็น kernel `7.0.0-38-generic` และ `active` สองบรรทัด จากนั้นลองเปิด https://git.mendoka.com