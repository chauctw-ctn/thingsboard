ThingsBoard — Hướng Dẫn Đầy Đủ
> Phiên bản: v4.3.1.2 | Windows (dev) → WSL2 (build) → Docker Linux (production)
---
Mục Lục
Phase 1 — Phát triển & Test trên Windows
Phase 2 — Build Docker Image trong WSL2
Phase 3 — Deploy Production trên Linux
Phase 4 — Cập nhật UI (lặp lại mỗi lần sửa)
Xử lý lỗi thường gặp
---
Phase 1 — Phát triển & Test trên Windows
> Toàn bộ Phase 1 chạy trong **PowerShell với quyền Administrator**.
Bước 1 — Gỡ Java cũ & làm sạch biến môi trường
```powershell
winget list | findstr /i "java temurin jdk"
winget uninstall EclipseAdoptium.Temurin.25.JDK
winget uninstall EclipseAdoptium.Temurin.17.JDK

[Environment]::SetEnvironmentVariable("JAVA_HOME", $null, "Machine")
$path = [Environment]::GetEnvironmentVariable("Path","Machine")
$path = ($path -split ';' | Where-Object { $_ -notmatch 'Java|jdk|Adoptium' }) -join ';'
[Environment]::SetEnvironmentVariable("Path",$path,"Machine")
```
Bước 2 — Cài Java 17 (Temurin)
```powershell
winget install EclipseAdoptium.Temurin.17.JDK
setx JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot" /M
setx Path "C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot\bin;%Path%" /M
```
> **Mở PowerShell mới** rồi kiểm tra: `java -version`
Bước 3 — Cài Maven
```powershell
choco install maven -y
setx Path "C:\ProgramData\chocolatey\lib\maven\apache-maven-3.9.16\bin;%Path%" /M

$env:JAVA_HOME="C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot"
$env:Path="C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot\bin;C:\ProgramData\chocolatey\lib\maven\apache-maven-3.9.16\bin;$env:Path"
mvn -version
```
Bước 4 — Tắt Mosquitto ⚠️ (tránh xung đột cổng 1883)
> ThingsBoard có MQTT broker tích hợp — Mosquitto phải tắt trước.
```powershell
Get-NetTCPConnection -LocalPort 1883 -ErrorAction SilentlyContinue | Select-Object LocalPort,State,OwningProcess
Get-Process -Id <PID>
Stop-Service -Name mosquitto -Force -ErrorAction SilentlyContinue
Set-Service -Name mosquitto -StartupType Disabled -ErrorAction SilentlyContinue

# Xác nhận cổng trống (phải rỗng)
Get-NetTCPConnection -LocalPort 1883 -ErrorAction SilentlyContinue
```
Bước 5 — Kiểm tra toàn bộ cổng ThingsBoard cần dùng
> ⚠️ KHÔNG dùng `netstat` trong PowerShell — dùng `Get-NetTCPConnection` thay thế.
```powershell
Get-NetTCPConnection | Where-Object { $_.LocalPort -in @(8080,1883,8883,8083) } | Format-Table LocalPort,State,OwningProcess
Stop-Process -Id <PID> -Force
```
Bước 6 — Tải source ThingsBoard
```powershell
cd C:\Users\nnchau
git clone https://github.com/thingsboard/thingsboard.git
cd C:\Users\nnchau\thingsboard
git fetch --tags
git switch --detach v4.3.1.1
git describe --tags
```
Bước 7 — Cài PostgreSQL 16
```powershell
winget install PostgreSQL.PostgreSQL.16
```
Bước 8 — Tạo database thingsboard
```powershell
$env:PAGER="cat"
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" -U postgres -c "CREATE DATABASE thingsboard;"

# Reset sạch nếu cần
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" -U postgres -c "DROP DATABASE thingsboard;"
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" -U postgres -c "CREATE DATABASE thingsboard;"
```
Bước 9 — Build backend ThingsBoard
> Mất 10–30 phút, cần kết nối Internet.
```powershell
cd C:\Users\nnchau\thingsboard\application
$env:DATABASE_TS_TYPE="sql"
$env:SPRING_DATASOURCE_URL="jdbc:postgresql://localhost:5432/thingsboard"
$env:SPRING_DATASOURCE_USERNAME="postgres"
$env:SPRING_DATASOURCE_PASSWORD="postgres"
$env:TB_QUEUE_TYPE="in-memory"
$env:TB_SERVICE_ID="tb-node-0"
mvn clean install -DskipTests
```
Bước 10 — Chuẩn bị thư mục SQL schema
```powershell
cd C:\Users\nnchau\thingsboard\application
mkdir target\windows\data\sql -Force
Copy-Item "C:\Users\nnchau\thingsboard\dao\src\main\resources\sql\*" `
  "C:\Users\nnchau\thingsboard\application\target\windows\data\sql\" `
  -Recurse -Force
```
Bước 11 — Khởi tạo bảng SQL (install mode)
> Phải thấy dòng: **ThingsBoard installed successfully**
```powershell
$env:JAVA_HOME="C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot"
$env:Path="C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot\bin;$env:Path"

$JAR = Get-ChildItem "C:\Users\nnchau\thingsboard\application\target\*boot*.jar" | Select-Object -First 1
Write-Host "JAR: $($JAR.Name)"

java -jar $JAR.FullName `
  --install.data_dir="C:\Users\nnchau\thingsboard\application\target\windows\data" `
  --install.load_demo=true `
  --spring.jpa.hibernate.ddl-auto=none `
  --install.upgrade=false `
  --spring.datasource.url=jdbc:postgresql://localhost:5432/thingsboard `
  --spring.datasource.username=postgres `
  --spring.datasource.password=postgres `
  2>&1 | Tee-Object -FilePath tb_install.log

notepad tb_install.log
```
Bước 12 — Kiểm tra bảng đã tạo
> Phải thấy 30+ bảng. Nếu rỗng → xem file `tb_install.log`.
```powershell
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" -U postgres -d thingsboard -P pager=off -c "\dt"
```
Bước 13 — Chạy ThingsBoard trên Windows (test local)
```powershell
cd C:\Users\nnchau\thingsboard\application
$env:DATABASE_TS_TYPE="sql"
$env:SPRING_DATASOURCE_URL="jdbc:postgresql://localhost:5432/thingsboard"
$env:SPRING_DATASOURCE_USERNAME="postgres"
$env:SPRING_DATASOURCE_PASSWORD="postgres"
$env:TB_QUEUE_TYPE="in-memory"
$env:TB_SERVICE_ID="tb-node-0"
$JAR = Get-ChildItem "target\*boot*.jar" | Select-Object -First 1
java -jar $JAR.FullName
```
> Truy cập `http://localhost:8080` — đăng nhập: `sysadmin@thingsboard.org` / `sysadmin`
Bước 14 — Sửa & test UI (ui-ngx)
```powershell
# Terminal 1 — Backend (bước 13)





# Terminal 2 — UI dev server
cd C:\Users\nnchau\thingsboard\ui-ngx
npm install --legacy-peer-deps
npm start
# Xem UI tại http://localhost:4200
```
Source UI nằm tại: `C:\Users\nnchau\thingsboard\ui-ngx\src`
---
Phase 2 — Build Docker Image trong WSL2
> Chạy trong **WSL2 (Ubuntu)**. Mở bằng lệnh `wsl` trong PowerShell.
Bước 15 — Dọn sạch ThingsBoard cũ trong WSL2 (nếu có)
> ⚠️ Chạy bước này nếu đã từng cài ThingsBoard trực tiếp trong WSL2 — tránh xung đột cổng.
```bash
# Tìm và kill tiến trình ThingsBoard đang chạy
sudo lsof -i :1883 -i :8080 -i :8083 2>/dev/null
sudo kill -9 $(sudo lsof -t -i :1883 2>/dev/null) 2>/dev/null
sudo kill -9 $(sudo lsof -t -i :8080 2>/dev/null) 2>/dev/null

# Gỡ package thingsboard nếu đã cài bằng dpkg
sudo dpkg -r thingsboard 2>/dev/null
sudo dpkg --purge thingsboard 2>/dev/null

# Xóa thư mục cũ
sudo rm -rf /usr/share/thingsboard
sudo rm -rf /var/log/thingsboard
sudo rm -rf /etc/thingsboard

# Xác nhận cổng sạch
sudo lsof -i :1883 -i :8080 2>/dev/null || echo "Cổng trống OK"
```
Bước 16 — Cài môi trường build trong WSL2 (1 lần duy nhất)
```bash
sudo apt update

# Java 17
sudo apt install -y openjdk-17-jdk
java -version

# Maven
sudo apt install -y maven
mvn -version

# Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v && npm -v

# Docker CLI
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# Công cụ build .deb
sudo apt install -y dpkg-dev fakeroot
```
Bước 17 — Đăng nhập Docker Hub & xác nhận username
```bash
docker login

# Xác nhận username: chauctw thực tế (QUAN TRỌNG — có thể khác tên Windows)
docker info | grep Username
```
> ⚠️ Ghi nhớ username này. Dùng cho tất cả lệnh `docker build`, `docker tag`, `docker push`.
> Ví dụ: Windows user = `nnchau` nhưng Docker Hub username = `chauctw`.
```bash
# Tạo repo trên Docker Hub (1 lần duy nhất)
# Vào https://hub.docker.com → Create Repository
# Tên: thingsboard-custom | Visibility: Private → Create
```
Bước 18 — Build .deb trong WSL2
```bash
cd /mnt/c/Users/nnchau/thingsboard

# Lần đầu — build toàn bộ (10-30 phút)
mvn clean install -DskipTests -Pdeb

# Xác nhận file .deb (tên thực tế: thingsboard.deb)
ls -lh application/target/*.deb
```
Bước 19 — Chuẩn bị thư mục build Docker
> ⚠️ KHÔNG build Docker từ thư mục source lớn — dùng thư mục riêng `~/tb-build`.
```bash
mkdir -p ~/tb-build

# Copy .deb vào cùng cấp với Dockerfile (QUAN TRỌNG)
cp /mnt/c/Users/nnchau/thingsboard/application/target/thingsboard.deb ~/tb-build/

ls -lh ~/tb-build/
# Phải thấy: thingsboard.deb (~339MB)
```
Bước 20 — Tạo Dockerfile trong ~/tb-build
> ⚠️ CMD phải dùng `java -jar thingsboard.jar` — KHÔNG dùng `thingsboard.sh` (không tồn tại trong .deb).
> ⚠️ KHÔNG dùng heredoc (`<< EOF`) — dùng python3 để tránh lỗi CRLF/indent.
```bash
python3 -c "
import os
content = '''FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y \\\\
    openjdk-17-jre-headless \\\\
    postgresql-client \\\\
    wget curl && \\\\
    rm -rf /var/lib/apt/lists/*

COPY thingsboard.deb /tmp/thingsboard.deb
RUN dpkg -i /tmp/thingsboard.deb && rm /tmp/thingsboard.deb

EXPOSE 8080 1883 8883 8083
CMD [\"java\", \"-jar\", \"/usr/share/thingsboard/bin/thingsboard.jar\"]
'''
path = os.path.expanduser('~/tb-build/Dockerfile.custom')
with open(path, 'w') as f:
    f.write(content)
print('Saved to:', path)
"

cat ~/tb-build/Dockerfile.custom
```
Bước 21 — Kiểm tra thư mục trước khi build
```bash
ls -lh ~/tb-build/
# Phải thấy đúng 2 file:
#   Dockerfile.custom
#   thingsboard.deb  (~339MB)
```
> ⚠️ Nếu `transferring context: 2B` → file `.deb` chưa đúng vị trí.
Bước 22 — Build Docker image
> ⚠️ Thay `chauctw` bằng Docker Hub username thực tế (`docker info | grep Username`).
```bash
cd ~/tb-build

docker build -f Dockerfile.custom \
  -t chauctw/thingsboard-custom:latest \
  -t chauctw/thingsboard-custom:v4.3.1-ui-$(date +%Y%m%d) \
  .
```
`load build context` phải thấy ~339MB. Build mất 5–10 phút.
Bước 23 — Push lên Docker Hub
```bash
docker push chauctw/thingsboard-custom:latest
docker push chauctw/thingsboard-custom:v4.3.1-ui-$(date +%Y%m%d)
```
---
Phase 3 — Deploy Production trên Linux
> Chạy trên **máy Linux production** qua SSH. Thực hiện **1 lần duy nhất**.
Bước 24 — Cài Docker Engine trên Linux
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

docker --version
docker compose version
```
Bước 25 — Kiểm tra cổng trống trên Linux trước khi deploy
> ⚠️ Đảm bảo không có tiến trình nào chiếm cổng 8080, 1883, 8883, 8083.
```bash
sudo lsof -i :1883 -i :8080 -i :8083 -i :8883 2>/dev/null

# Nếu có tiến trình → kill (thay <PID>)
sudo kill -9 <PID>

# Nếu là ThingsBoard chạy trực tiếp (java process)
sudo kill -9 $(sudo lsof -t -i :1883 2>/dev/null) 2>/dev/null
```
Bước 26 — Tạo cấu trúc thư mục & docker-compose.yml
```bash
sudo mkdir -p /opt/thingsboard/{data,logs,postgres}
sudo chown -R $USER:$USER /opt/thingsboard
```
```bash
# Tạo docker-compose.yml bằng python3 (tránh lỗi CRLF)
python3 -c "
content = '''services:
  postgres:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_DB: thingsboard
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - /opt/thingsboard/postgres:/var/lib/postgresql/data
    networks:
      - tb-net

  thingsboard:
    image: chauctw/thingsboard-custom:latest
    restart: always
    depends_on:
      - postgres
    ports:
      - \"8080:8080\"
      - \"1883:1883\"
      - \"8883:8883\"
      - \"8083:8083\"
    environment:
      TB_QUEUE_TYPE: in-memory
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/thingsboard
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
    volumes:
      - /opt/thingsboard/data:/data
      - /opt/thingsboard/logs:/var/log/thingsboard
    networks:
      - tb-net

networks:
  tb-net:
    driver: bridge
'''
with open('/opt/thingsboard/docker-compose.yml', 'w') as f:
    f.write(content)
print('OK')
"
```
Bước 27 — Khởi động PostgreSQL
```bash
cd /opt/thingsboard
docker login
docker pull chauctw/thingsboard-custom:latest

docker compose up postgres -d
sleep 15

# Xác nhận postgres đang chạy
docker compose ps
```
Bước 28 — Khởi tạo database (chỉ chạy 1 lần duy nhất)
> ⚠️ File `.sh` trong `.deb` có thể có line ending CRLF (Windows) — phải fix trước khi chạy.
> ⚠️ Cần tạo user `thingsboard` và thư mục log trước khi chạy install script.
```bash
docker run --rm --network thingsboard_tb-net \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/thingsboard \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=postgres \
  --user root \
  chauctw/thingsboard-custom:latest \
  bash -c "
    useradd -r thingsboard 2>/dev/null
    mkdir -p /var/log/thingsboard
    chown thingsboard /var/log/thingsboard
    find /usr/share/thingsboard/conf -type f | xargs sed -i 's/\r//'
    find /usr/share/thingsboard/bin -type f -name '*.sh' | xargs sed -i 's/\r//'
    find /usr/share/thingsboard/bin -type f -name '*.conf' | xargs sed -i 's/\r//'
    /usr/share/thingsboard/bin/install/install.sh --loadDemo
  "
```
> Phải thấy: **ThingsBoard installed successfully!**
Bước 29 — Khởi động toàn bộ stack
```bash
cd /opt/thingsboard
docker compose up -d
docker compose logs -f thingsboard
```
> Chờ dòng: **Started ThingsboardServerApplication**
> Truy cập `http://localhost:8080` (WSL2) hoặc `http://<IP-Linux>:8080` (production)
> Đăng nhập: `sysadmin@thingsboard.org` / `sysadmin`
---
Phase 4 — Cập nhật UI (lặp lại mỗi lần sửa)
Bước A — Sửa & test UI trên Windows
```powershell
# Terminal 1 — Backend
cd C:\Users\nnchau\thingsboard\application
$JAR = Get-ChildItem "target\*boot*.jar" | Select-Object -First 1
java -jar $JAR.FullName

# Terminal 2 — UI dev server
cd C:\Users\nnchau\thingsboard\ui-ngx
npm start
# Kiểm tra tại http://localhost:4200
```
Bước B — Build .deb & image mới trong WSL2
```bash
# Build nhanh chỉ phần UI
cd /mnt/c/Users/nnchau/thingsboard
mvn install -DskipTests -Pdeb -pl ui-ngx,application --also-make

# Copy .deb mới vào thư mục build
cp application/target/thingsboard.deb ~/tb-build/thingsboard.deb

# Build & push image mới
cd ~/tb-build
docker build -f Dockerfile.custom \
  -t chauctw/thingsboard-custom:latest \
  -t chauctw/thingsboard-custom:v4.3.1-ui-$(date +%Y%m%d) \
  .

docker push chauctw/thingsboard-custom:latest
docker push chauctw/thingsboard-custom:v4.3.1-ui-$(date +%Y%m%d)
```
Bước C — Cập nhật trên Linux (~30 giây downtime)
```bash
cd /opt/thingsboard
docker compose pull thingsboard
docker compose up -d thingsboard
docker compose logs -f thingsboard
```
> Xoá cache trình duyệt nếu UI chưa thay đổi: `Ctrl+Shift+R`
Rollback nếu có sự cố
```bash
docker compose stop thingsboard
docker tag chauctw/thingsboard-custom:v4.3.1-ui-20260525 \
  chauctw/thingsboard-custom:latest
docker compose up -d thingsboard
```
---
Xử lý lỗi thường gặp
Lỗi: Cổng 1883/8080 bị chiếm bởi ThingsBoard chạy trực tiếp trong WSL2
```bash
# Tìm tiến trình
sudo lsof -i :1883
sudo lsof -i :8080

# Kill theo PID
sudo kill -9 <PID>

# Hoặc kill tất cả cùng lúc
sudo kill -9 $(sudo lsof -t -i :1883 2>/dev/null) 2>/dev/null
sudo kill -9 $(sudo lsof -t -i :8080 2>/dev/null) 2>/dev/null
```
Lỗi: Cổng 1883 bị Mosquitto chiếm (Windows)
```powershell
Get-NetTCPConnection -LocalPort 1883 -ErrorAction SilentlyContinue | Select-Object LocalPort,State,OwningProcess
Stop-Service -Name mosquitto -Force
Set-Service -Name mosquitto -StartupType Disabled
```
Lỗi: `netstat` không nhận ra trong PowerShell
```powershell
Get-NetTCPConnection | Where-Object { $_.LocalPort -in @(8080,1883,8083) } | Format-Table LocalPort,State,OwningProcess
```
Lỗi: `push access denied` hoặc `tag does not exist`
```bash
# Kiểm tra username thực tế
docker info | grep Username

# Tag lại với username đúng
docker tag nnchau/thingsboard-custom:latest chauctw/thingsboard-custom:latest
docker push chauctw/thingsboard-custom:latest
```
Lỗi: `repository name must be lowercase`
```bash
# Sai:  YOUR_DOCKERHUB_USERNAME/thingsboard-custom
# Đúng: chauctw/thingsboard-custom
```
Lỗi: `docker build` — `requires 1 argument`
```bash
# Thiếu dấu . ở cuối — bắt buộc phải có
docker build -f Dockerfile.custom -t chauctw/thingsboard-custom:latest .
```
Lỗi: `transferring context: 2B` — Docker không thấy file .deb
```bash
# Giải pháp: dùng thư mục build riêng ~/tb-build
mkdir -p ~/tb-build
cp /mnt/c/Users/nnchau/thingsboard/application/target/thingsboard.deb ~/tb-build/
# Dockerfile và thingsboard.deb phải cùng cấp
ls -lh ~/tb-build/
```
Lỗi: `lstat /application/target: no such file or directory`
```bash
# Dockerfile phải dùng tên file đơn giản (cùng cấp)
grep COPY ~/tb-build/Dockerfile.custom
# Phải thấy: COPY thingsboard.deb /tmp/thingsboard.deb
# KHÔNG phải: COPY application/target/thingsboard-*.deb ...
```
Lỗi: `stat /usr/share/thingsboard/bin/thingsboard.sh: no such file or directory`
```bash
# .deb không có thingsboard.sh — dùng java -jar thay thế
# CMD đúng trong Dockerfile:
CMD ["java", "-jar", "/usr/share/thingsboard/bin/thingsboard.jar"]
```
Lỗi: `^M: bad interpreter` hoặc CRLF trong .sh/.conf
```bash
# Fix toàn bộ file bị CRLF trong container
find /usr/share/thingsboard/conf -type f | xargs sed -i 's/\r//'
find /usr/share/thingsboard/bin -type f -name '*.sh' | xargs sed -i 's/\r//'
find /usr/share/thingsboard/bin -type f -name '*.conf' | xargs sed -i 's/\r//'
```
Lỗi: `PermissionError` khi tạo file trong WSL2
```bash
# Dùng os.path.expanduser('~') thay vì hardcode /root
python3 -c "
import os
path = os.path.expanduser('~/ten-file')
with open(path, 'w') as f:
    f.write('nội dung')
print('Saved to:', path)
"
```
Lỗi: Không tạo được file bằng heredoc trong WSL2
```bash
# Dùng python3 thay vì heredoc — tránh lỗi indent/CRLF
python3 -c "
import os
content = 'nội dung file'
path = os.path.expanduser('~/ten-file')
with open(path, 'w') as f:
    f.write(content)
print('OK')
"
```
Lỗi: ThingsBoard install schema thất bại
```powershell
# Ghi log ra file để xem lỗi chi tiết
java -jar $JAR.FullName [args] 2>&1 | Tee-Object -FilePath tb_install.log
notepad tb_install.log
```
---
Cấu trúc thư mục build Docker (chuẩn)
```
~/tb-build/
├── Dockerfile.custom      ← CMD dùng java -jar thingsboard.jar
└── thingsboard.deb        ← File .deb (~339MB), cùng cấp với Dockerfile
```
> ⚠️ KHÔNG để `.deb` trong subfolder — phải cùng cấp với Dockerfile.
---
Thông tin tài khoản mặc định
Tài khoản	Email	Mật khẩu
Sysadmin	sysadmin@thingsboard.org	sysadmin
Tenant Admin	tenant@thingsboard.org	tenant
Customer	customer@thingsboard.org	customer
Cổng dịch vụ
Cổng	Dịch vụ
8080	HTTP (giao diện web & REST API)
1883	MQTT
8883	MQTT over SSL
8083	MQTT over WebSocket
---
Cập nhật: 26/05/2026 — ThingsBoard v4.3.1.2 — Windows 11 + WSL2 Ubuntu + Docker




# ThingsBoard v4.3.1.2 — Làm sạch và khởi tạo SQL trên Windows
> Áp dụng cho:
>
> - Windows 11
> - PostgreSQL 16
> - Java 17
> - ThingsBoard v4.3.1.2

---

# ThingsBoard v4.3.1.2 — Làm sạch và khởi tạo SQL trên Windows

> Áp dụng cho:
>
> * Windows 11
> * PostgreSQL 16
> * Java 17
> * ThingsBoard v4.3.1.2

---

# Bước 1 — Dừng ThingsBoard nếu đang chạy

Mở PowerShell Administrator:

```powershell
Get-Process java -ErrorAction SilentlyContinue
```

Nếu có tiến trình Java:

```powershell
Stop-Process -Name java -Force
```

---

# Bước 2 — Kiểm tra PostgreSQL đang chạy

```powershell
Get-Service postgresql*
```

Phải thấy:

```text
Running
```

Nếu chưa chạy:

```powershell
Start-Service postgresql-x64-16
```

---

# Bước 3 — Xóa sạch database cũ

Terminate toàn bộ connection:

```powershell
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" `
  -U postgres `
  -d postgres `
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='thingsboard';"
```

Xóa database:

```powershell
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" `
  -U postgres `
  -d postgres `
  -c "DROP DATABASE IF EXISTS thingsboard;"
```

Tạo lại database:

```powershell
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" `
  -U postgres `
  -d postgres `
  -c "CREATE DATABASE thingsboard;"
```

---

# Bước 4 — Chuẩn bị SQL schema

```powershell
cd C:\Users\nnchau\thingsboard\application
```

Tạo thư mục:

```powershell
mkdir target\windows\data\sql -Force
```

Copy SQL schema:

```powershell
Copy-Item `
  "C:\Users\nnchau\thingsboard\dao\src\main\resources\sql\*" `
  "C:\Users\nnchau\thingsboard\application\target\windows\data\sql\" `
  -Recurse -Force
```

---

# Bước 5 — Copy JSON demo data

```powershell
Copy-Item `
  "C:\Users\nnchau\thingsboard\application\src\main\data\json" `
  "C:\Users\nnchau\thingsboard\application\target\windows\data\" `
  -Recurse -Force
```

---

# Bước 6 — Tạo thư mục lib cho install.bat

```powershell
cd C:\Users\nnchau\thingsboard\application\target\windows
```

```powershell
mkdir lib -Force
```

Copy boot jar:

```powershell
Copy-Item `
  "C:\Users\nnchau\thingsboard\application\target\thingsboard-4.3.1.2-boot.jar" `
  ".\lib\thingsboard.jar" `
  -Force
```

---

# Bước 7 — Khôi phục PATH Windows nếu mất cmd.exe

Nếu `cmd.exe` không chạy được:

```powershell
$env:Path="C:\Windows\System32;$env:Path"
```

Kiểm tra:

```powershell
cmd.exe /c echo OK
```

Phải hiện:

```text
OK
```

---

# Bước 8 — Set Java 17

```powershell
$env:JAVA_HOME="C:\Program Files\Eclipse Adoptium\jdk-17.0.19.10-hotspot"
$env:Path="$env:JAVA_HOME\bin;$env:Path"
```

Kiểm tra:

```powershell
java -version
```

Phải là Java 17.

---

# Bước 9 — Chạy install SQL schema

```powershell
cd C:\Users\nnchau\thingsboard\application\target\windows
```

```powershell
cmd.exe /c install.bat --loadDemo
```

---

# Bước 10 — Kết quả đúng

Phải xuất hiện:

```text
Installing thingsboard ...
```

sau đó:

```text
ThingsBoard installed successfully!
```

---

# Bước 11 — Kiểm tra bảng SQL

```powershell
& "C:\Program Files\PostgreSQL\16\bin\psql.exe" `
  -U postgres `
  -d thingsboard `
  -P pager=off `
  -c "\dt"
```

Phải thấy các bảng:

```text
queue
device
asset
tenant
rule_chain
tb_user
```

---

# Bước 12 — Chạy ThingsBoard

```powershell
cd C:\Users\nnchau\thingsboard\application
```

```powershell
$JAR = Get-ChildItem "target\*boot*.jar" | Select-Object -First 1
```

```powershell
java -jar $JAR.FullName
```

---

# Bước 13 — Truy cập web

Mở trình duyệt:

```text
http://localhost:8080
```

## Tài khoản mặc định

| Role     | User                                                        | Password |
| -------- | ----------------------------------------------------------- | -------- |
| Sysadmin | [sysadmin@thingsboard.org](mailto:sysadmin@thingsboard.org) | sysadmin |
| Tenant   | [tenant@thingsboard.org](mailto:tenant@thingsboard.org)     | tenant   |
| Customer | [customer@thingsboard.org](mailto:customer@thingsboard.org) | customer |

---







# Kế hoạch Custom Giao diện ThingsBoard

## Context

Cần custom giao diện ThingsBoard cho hệ thống giám sát nhà máy và mạng lưới cấp nước. ThingsBoard không có tính năng white-label built-in, customization được thực hiện qua source code.

---

## Các điểm custom chính

### 1. Logo & Branding

| File | Mô tả |
|------|-------|
| [ui-ngx/src/assets/logo_title_white.svg](ui-ngx/src/assets/) | Logo chính với title |
| [ui-ngx/src/assets/logo_white.svg](ui-ngx/src/assets/) | Logo đơn giản |
| [ui-ngx/src/app/shared/components/logo.component.ts](ui-ngx/src/app/shared/components/logo.component.ts) | Logo component |

**Hành động:**
- Replace logo SVG files trong `assets/`
- Kiểm tra logo size (tránh file quá lớn > 300KB)

### 2. Màu sắc Theme

| File | Mô tả |
|------|-------|
| [ui-ngx/src/scss/constants.scss](ui-ngx/src/scss/constants.scss) | Color variables |
| [ui-ngx/src/theme.scss](ui-ngx/src/theme.scss) | Angular Material theme |

**Variables chính:**
```scss
$tb-primary-color: #305680;      // Xanh đậm
$tb-secondary-color: #527dad;    // Xanh nhạt
$tb-hue3-color: #a7c1de;         // Xanh rất nhạt
$tb-dark-primary-color: #9fa8da; // Xanh dark mode
```

**Hành động:**
- Thay đổi `$tb-primary-color` theo brand mới
- Test cả light và dark mode

### 3. Title & Metadata

| File | Mô tả |
|------|-------|
| [ui-ngx/src/index.html](ui-ngx/src/index.html) | Page title, favicon |
| [ui-ngx/src/environments/environment.prod.ts](ui-ngx/src/environments/environment.prod.ts) | `appTitle` |

**Current:**
- Title: `Hệ thống giám sát nhà máy và mạng lưới cấp nước`
- Favicon: `thingsboard.ico`

**Hành động:**
- Update `<title>` tag trong `index.html`
- Replace favicon `.ico` file
- Update `appTitle` trong environment

### 4. Login Page

| File | Mô tả |
|------|-------|
| [ui-ngx/src/app/modules/home/pages/login/](ui-ngx/src/app/modules/home/pages/login/) | Login components |

**Hành động:**
- Custom logo trên login page
- Kiểm tra link logo (hiện link tới thingsboard.io)

---

## Thứ tự thực hiện

1. **Logo** - Thay thế SVG files trong `assets/`
2. **Favicon** - Thay thế `thingsboard.ico`
3. **Title** - Update `index.html` và `environment.ts`
4. **Colors** - Modify `constants.scss` cho brand colors
5. **Build & Test** - Chạy `yarn start` để kiểm tra

---

## Build và Deployment

**Development:**
```bash
cd ui-ngx
yarn install
yarn start
```

**Production Docker:**
```bash
cd ui-ngx
yarn build
# Hoặc dùng docker build với Dockerfile.custom
```

---

## Verification

1. `yarn start` - Kiểm tra local dev server
2. Verify logo hiển thị đúng trên:
   - Login page
   - Sidebar
   - Loading screen
3. Check colors đúng trên cả light/dark mode
4. Verify page title trong browser tab




# Kế hoạch Custom ThingsBoard

## 1. Custom Giao diện UI

### 1.1 Logo & Branding

| File | Mô tả |
|------|-------|
| [ui-ngx/src/assets/logo_title_white.svg](ui-ngx/src/assets/) | Logo chính |
| [ui-ngx/src/app/shared/components/logo.component.ts](ui-ngx/src/app/shared/components/logo.component.ts) | Logo component |

### 1.2 Màu sắc Theme

| File | Mô tả |
|------|-------|
| [ui-ngx/src/scss/constants.scss](ui-ngx/src/scss/constants.scss) | Color variables |
| [ui-ngx/src/theme.scss](ui-ngx/src/theme.scss) | Angular Material theme |

### 1.3 Title & Metadata

| File | Mô tả |
|------|-------|
| [ui-ngx/src/index.html](ui-ngx/src/index.html) | Page title, favicon |
| [ui-ngx/src/environments/environment.prod.ts](ui-ngx/src/environments/environment.prod.ts) | `appTitle` |

---

## 2. Custom Widget Library

### 2.1 Kiến trúc Widget

**Angular Components:** `ui-ngx/src/app/modules/home/components/widget/lib/`

| Thư mục | Widget Type |
|---------|------------|
| `alarm/` | Alarms table |
| `button/` | Action buttons |
| `cards/` | Value/label cards |
| `chart/` | Charts (echarts) |
| `entity/` | Entities table |
| `gauge/` | Gauges |
| `maps/` | Map widgets |
| `rpc/` | RPC controls |
| `scada/` | SCADA symbols |
| `html/` | HTML containers |

**Server-side entities:**
- `WidgetType.java` - Widget type definition
- `WidgetsBundle.java` - Widget bundle

**UI Models:** `ui-ngx/src/app/shared/models/widget.models.ts`

### 2.2 Cách thêm Custom Widget

**Cách 1: Qua Widget Editor (UI)**
1. Vào Widgets Library
2. Tạo Widget Bundle mới
3. Dùng Widget Editor để:
   - Viết HTML template
   - Viết CSS styles
   - Implement controller JavaScript
   - Configure settings forms

**Cách 2: Code (Angular Module)**
1. Tạo component trong `lib/`
2. Register trong `widget-settings.module.ts`
3. Thêm vào `widget-components.module.ts`

**Key Files:**
- [ui-ngx/src/app/modules/home/pages/widget/widget-editor.component.ts](ui-ngx/src/app/modules/home/pages/widget/widget-editor.component.ts) - Widget editor
- [ui-ngx/src/app/modules/home/components/widget/widget-component.service.ts](ui-ngx/src/app/modules/home/components/widget/widget-component.service.ts) - Dynamic loading

### 2.3 Widget Models

```typescript
interface WidgetTypeDescriptor {
  type: widgetType;           // timeseries, latest, rpc, alarm, static
  resources: Array<WidgetResource>;
  templateHtml: string;
  templateCss: string;
  controllerScript: TbFunction;
  settingsForm?: FormProperty[];
  dataKeySettingsForm?: FormProperty[];
  defaultConfig: string;
  sizeX: number;
  sizeY: number;
}
```

---

## Thứ tự thực hiện

### Phase 1: UI Customization
1. Logo - Replace SVG files
2. Favicon - Replace .ico
3. Title - Update index.html, environment.ts
4. Colors - Modify constants.scss

### Phase 2: Custom Widget
1. Xác định widget type cần custom
2. Thiết kế widget theo business requirement
3. Implement bằng Widget Editor hoặc Angular code
4. Test trên dashboard

---

## Verification

1. `yarn start` - Dev server
2. Verify logo trên login, sidebar, loading screen
3. Test widgets trên dashboard
4. Check cả light/dark mode






cd ~/tb-build
docker build -f Dockerfile.custom \
  -t chauctw/thingsboard-custom:latest \
  -t chauctw/thingsboard-custom:v4.3.1.2-$(date +%Y%m%d) \
  .


git clone https://github.com/chauctw-ctn/thingsboard-fork.git thingsboard
cd thingsboard
git fetch --tags
git switch --detach v4.3.1.2
git describe --tags  