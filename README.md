# 🚀 BT4 – Tự Động Đăng Bài WordPress Với N8N

> **Môn:** Phát triển ứng dụng với mã nguồn mở – TEE0421  
> **Lớp:** 58KTPM  
> **Sinh viên:** Như Khiêm  
> **Deadline:** 23h59 ngày 25/05/2026

[![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white)](https://wordpress.nhukhiem.id.vn)
[![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.nhukhiem.id.vn)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![DeepSeek](https://img.shields.io/badge/DeepSeek_AI-4D6BFE?style=for-the-badge&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PC9zdmc+&logoColor=white)](#)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://cloudflare.com)

---

## 📋 Mục Lục

- [Giới thiệu](#-giới-thiệu)
- [Kiến trúc hệ thống](#-kiến-trúc-hệ-thống)
- [Yêu cầu hệ thống](#-yêu-cầu-hệ-thống)
- [Cấu trúc thư mục](#-cấu-trúc-thư-mục)
- [Hướng dẫn triển khai](#-hướng-dẫn-triển-khai)
  - [Bước 1 – Clone & cấu hình môi trường](#bước-1--clone--cấu-hình-môi-trường)
  - [Bước 2 – Chạy Docker Compose](#bước-2--chạy-docker-compose)
  - [Bước 3 – Cấu hình Cloudflare Tunnel](#bước-3--cấu-hình-cloudflare-tunnel)
  - [Bước 4 – Cài đặt WordPress](#bước-4--cài-đặt-wordpress)
- [Cấu hình N8N](#-cấu-hình-n8n)
  - [Bước 1 – Tạo tài khoản & Activate License](#bước-1--tạo-tài-khoản--activate-license)
  - [Bước 2 – Tạo Telegram Bot](#bước-2--tạo-telegram-bot)
  - [Bước 3 – Lấy DeepSeek API Key](#bước-3--lấy-deepseek-api-key)
  - [Bước 4 – Tạo WordPress Application Password](#bước-4--tạo-wordpress-application-password)
  - [Bước 5 – Build Workflow](#bước-5--build-workflow)
- [Kết quả đạt được](#-kết-quả-đạt-được)
- [Nhận xét](#-nhận-xét)

---

## 📖 Giới Thiệu

Bài tập 4 mở rộng từ BT3 bằng cách **bổ sung service N8N** vào hệ thống Docker Compose, sau đó xây dựng một **workflow tự động hoàn chỉnh**:

> 💬 Nhắn tin với Telegram Bot → 🤖 DeepSeek AI sinh nội dung HTML → ⚙️ N8N xử lý → 📝 Bài viết tự động được đăng lên WordPress

**5 service hoạt động song song:**

| Service | Image | Vai trò |
|---|---|---|
| `mariadb` | `mariadb:latest` | Cơ sở dữ liệu |
| `phpmyadmin` | `phpmyadmin:latest` | Quản trị CSDL qua giao diện web |
| `wordpress` | `wordpress:latest` | CMS – trang web chính |
| `cloudflared` | `cloudflare/cloudflared:latest` | Tunnel public ra Internet |
| `n8n` | `n8nio/n8n:latest` | Automation workflow engine |

---

## 🏗️ Kiến Trúc Hệ Thống

### Sơ đồ hạ tầng Docker

```mermaid
graph TB
    subgraph Internet
        USER["👤 Người dùng"]
        TGBOT["📱 Telegram Bot"]
        DS["🤖 DeepSeek AI"]
        CF["☁️ Cloudflare DNS"]
    end

    subgraph "Ubuntu Server"
        subgraph "Docker Network: wordpress_network"
            CFD["cloudflared\nTunnel Agent"]
            WP["wordpress:latest\n:8080"]
            PMA["phpmyadmin:latest\n:8083"]
            N8N["n8nio/n8n:latest\n:5678"]
            DB["mariadb:10.11\n:3306"]
        end

        subgraph "Volumes (Persistent)"
            V1[("./volumes/mariadb")]
            V2[("./volumes/wordpress")]
            V3[("./volumes/phpmyadmin")]
            V4[("./volumes/n8n")]
        end
    end

    USER -->|"https://wordpress.nhukhiem.id.vn"| CF
    USER -->|"https://phpmyadmin.nhukhiem.id.vn"| CF
    USER -->|"https://n8n.nhukhiem.id.vn"| CF

    CF -->|Route| CFD
    CFD -->|":8080"| WP
    CFD -->|":8083"| PMA
    CFD -->|":5678"| N8N

    WP --> DB
    PMA --> DB
    N8N --> WP

    DB --- V1
    WP --- V2
    PMA --- V3
    N8N --- V4
```

### Sơ đồ N8N Workflow

```mermaid
flowchart LR
    A["📱 Telegram\nTrigger\nOnMessage"] -->|"message.text"| B["🤖 DeepSeek AI\nMessage a Model\nPrompt → JSON"]
    B -->|"json.text"| C["⚙️ Code Node\nJavaScript\nParse JSON"]
    C -->|"title + content"| D["📝 WordPress\nCreate a Post\nStatus: Publish"]

    style A fill:#229ED9,color:#fff
    style B fill:#4D6BFE,color:#fff
    style C fill:#FF6D00,color:#fff
    style D fill:#21759B,color:#fff
```

### Luồng dữ liệu chi tiết

```mermaid
sequenceDiagram
    actor U as 👤 Người dùng
    participant T as 📱 Telegram Bot
    participant N as ⚙️ N8N
    participant D as 🤖 DeepSeek AI
    participant W as 📝 WordPress

    U->>T: Nhắn nội dung chủ đề bài viết
    T->>N: Trigger webhook (message.text)
    N->>D: Gửi Prompt + yêu cầu trả JSON
    D-->>N: {"post_title": "...", "post_content": "..."}
    N->>N: Code JS parse JSON
    N->>W: Create Post (title + HTML content)
    W-->>N: Post ID + URL
    Note over W: Bài viết Published 🎉
```

---

## 💻 Yêu Cầu Hệ Thống

| Thành phần | Phiên bản tối thiểu |
|---|---|
| Ubuntu | 24.04.4 LTS |
| Docker | 29.4.0 |

**Tài khoản cần có trước:**
- [x] Cloudflare account + domain đã cấu hình
- [x] Telegram account
- [x] DeepSeek account (https://platform.deepseek.com)

---

## 📁 Cấu Trúc Thư Mục

```
BT4-WordPress-N8N/
├── 📄 docker-compose.yml       # Định nghĩa 5 services
├── 📄 .env                     # Biến môi trường (không commit)
├── 📄 .env.example             # Template biến môi trường
├── 📄 .gitignore
├── 📁 volumes/                 # Dữ liệu persistent (auto tạo)
│   ├── mariadb/
│   ├── wordpress/
│   └── n8n/
└── 📄 README.md
```

---

## 🚀 Hướng Dẫn Triển Khai

### Bước 1 – Clone & Cấu Hình Môi Trường

```bash
# Clone repository
git clone https://github.com/nhukhiem3143/BT4-WordPress-Dev-App-OpenSrc.git
cd wordpress-project

# Tạo file .env từ template
cp .env.example .env
```

Mở `.env` và điền đầy đủ thông tin:

```env
# === MARIADB DATABASE ===
MYSQL_ROOT_PASSWORD=
MYSQL_DATABASE=
MYSQL_USER=
MYSQL_PASSWORD=

# === WORDPRESS ===
WORDPRESS_DB_HOST=mariadb:3306
WORDPRESS_TABLE_PREFIX=wp_

# === PORTS ===
WORDPRESS_PORT=8080
PHPMYADMIN_PORT=8083
N8N_PORT=5678

# === N8N ===
N8N_WEBHOOK_URL=

# === CLOUDFLARED TUNNEL ===
CLOUDFLARED_TUNNEL_TOKEN=

# === TIMEZONE (Optional) ===
TZ=Asia/Ho_Chi_Minh
```

> ⚠️ **Lưu ý:** Không commit file `.env` lên GitHub.

---

### Bước 2 – Chạy Docker Compose


#### Pull tất cả images
```bash
docker compose pull
```

<img width="1145" height="274" alt="image" src="https://github.com/user-attachments/assets/82c61cd1-132f-425a-885c-8640f8382ad7" />

#### Khởi chạy tất cả services ở chế độ nền
```bash
docker compose up -d
```

##### Kiểm tra trạng thái các service
```bash
docker compose ps
```

**Kết quả:**

<img width="1595" height="261" alt="image" src="https://github.com/user-attachments/assets/49750236-f469-4e93-b6b8-dd4086382886" />

---

### Bước 3 – Cấu Hình Cloudflare Tunnel

Truy cập **Cloudflare Dashboard → Zero Trust → Networks → Tunnels → wordpress-tunnel → Configure → Public Hostnames**

Thêm **3 router** như sau:

| Subdomain | Service | Port |
|---|---|---|
| `wordpress.nhukhiem.id.vn` | `http://wordpress:80` | *(đã có từ BT3)* |
| `phpmyadmin.nhukhiem.id.vn` | `http://phpmyadmin:80` | Thêm mới |
| `n8n.nhukhiem.id.vn` | `http://n8n:5678` | Thêm mới |

##### Thêm tunnel cho phpmyadmin
<img width="1913" height="1079" alt="image" src="https://github.com/user-attachments/assets/9070cdf8-a285-4995-b71e-fc87b8f0e4eb" />

##### Thêm tunnel cho n8n
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/3598f577-6cfa-4cf0-b4f3-3ce2dff90703" />

##### Kết quả 
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/9b9b08ff-29f3-4519-80f6-ccafc474e3c3" />


> 💡 Tên service trong URL tunnel phải khớp với tên service trong `docker-compose.yml`.

---

## ⚙️ Cấu Hình N8N

### Bước 1 – Tạo Tài Khoản & Activate License

**Tạo tài khoản admin:**
1. Truy cập `https://n8n.nhukhiem.id.vn`
2. Điền đầy đủ thông tin: First name, Last name, **Email** (quan trọng!), Password
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/9b6aaee0-ac8d-423d-8e60-552a50b09a6e" />

3. Chọn **"Send me a License key"** → điền thông tin → Submit
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/d012496c-09c4-4bf9-aa84-989dc7bc8099" />

4. Kiểm tra email để lấy **License Key**
<img width="1919" height="1079" alt="Screenshot 2026-05-21 173029" src="https://github.com/user-attachments/assets/46d266ed-d02d-468c-b13c-7ed3170907d2" />

**Activate License:**
```
SETTING (góc dưới trái) → Usage and Plan → Enter activation key → Paste key → Activate
```
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/871a02e0-dd5e-4285-a9e8-272217efbd1d" />

> ✅ Thông báo thành công: *"Your Registered Community Edition has been successfully activated."*

<img width="1919" height="1079" alt="Screenshot 2026-05-21 195650" src="https://github.com/user-attachments/assets/9c5bb02d-cc06-47d3-a98e-72ff2a18d764" />

---

### Bước 2 – Tạo Telegram Bot

1. Mở Telegram → tìm kiếm **@BotFather** → bắt đầu chat
2. Gõ lệnh `/newbot`
3. Đặt tên bot (vd: `Bot_Wordpress`)
4. Đặt username bot (phải kết thúc bằng `bot`, vd: `nhukhiem_wp_bot`)
5. **Copy Token** được cấp (dạng: `1234567890:AAxxxxxxxxxxxxxx`)

<img width="1257" height="997" alt="Screenshot 2026-05-21 200809" src="https://github.com/user-attachments/assets/3eb89597-72e4-4751-bd74-bd4a5a3c9f83" />

> ⚠️ **Quan trọng:** Sau khi tạo bot, phải **chat lần đầu** với bot (nội dung bất kỳ) trước khi dùng trong n8n, nếu không webhook sẽ không nhận được message.

#### Bot vừa tạo
<img width="1225" height="994" alt="image" src="https://github.com/user-attachments/assets/df58bb00-dd81-46ef-a8e0-1831325a7cbb" />

---

### Bước 3 – Lấy DeepSeek API Key

1. Truy cập `https://platform.deepseek.com`
2. Đăng nhập / Đăng ký tài khoản
3. Vào **API Keys** → **Create new API key**
4. **Copy API Key** (chỉ hiển thị một lần)

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/e9c42a4c-3c9a-4b0f-a24f-4fedbe099c89" />

---

### Bước 4 – Tạo WordPress Application Password

1. Truy cập `https://wordpress.nhukhiem.id.vn/wp-admin`
2. **Users → Profile** (hoặc vào tài khoản admin)
3. Kéo xuống phần **Application Passwords**
4. Nhập tên: `n8n` → bấm **"Add New Application Password"**
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/78eb87bb-7370-465a-b1d3-e0409c97cdca" />

5. **Copy chuỗi 24 ký tự** được tạo ra (chỉ hiển thị một lần)
<img width="1919" height="1079" alt="Screenshot 2026-05-21 201839" src="https://github.com/user-attachments/assets/6ccf2b1e-b355-4e0b-aae8-e28718bd1cf7" />

---

### Bước 5 – Build Workflow

Truy cập `https://n8n.nhukhiem.id.vn` → **Overview → Create Workflow**

#### 🔷 Node 1: Telegram Trigger

| Trường | Giá trị |
|---|---|
| Authentication | `Credential to connect with` |
| Credential | Tạo mới → Paste **Telegram Bot Token** |
| Trigger On | `Message` |
| Additional Fields | Để trống |

*Thêm key đã tạo từ @BotFather*
<img width="1853" height="1079" alt="image" src="https://github.com/user-attachments/assets/03bf9a27-9551-4d59-936b-d1b77a1eabff" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/fdeac4eb-c52d-4ee0-9eef-c77ac30000c2" />


##### 📌 Test Trigger

1. Nhấn `Test This Trigger`
2. Mở Telegram
3. Gửi cho bot:
   ```txt
   /start
4. Gửi tiếp: ```hello```
<img width="1913" height="1030" alt="image" src="https://github.com/user-attachments/assets/f534dc27-d917-4ee8-94ec-e5ad209d3883" />

5. Nếu thành công, node sẽ xuất hiện Output
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/5e5fc1df-6a23-4d7d-b6a0-de5a0baa365a" />

#### 🔷 Node 2: DeepSeek – Message a Model


| Trường | Giá trị |
|---|---|
| Credential | Chọn `DeepSeek account` |
| Resource | `Chat` |
| Operation | `Complete` |
| Model | `deepseek-v4-flash` hoặc `deepseek-chat` |
| Simplify | `ON` |

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/7ae1a408-ab5e-404b-a545-6abb2db2145d" />


---

##### 🔷 Prompt Message 1

| Trường | Giá trị |
|---|---|
| Role | `System` |
| Content | Nội dung bên dưới |

```text
Bạn là trợ lý viết bài blog chuyên nghiệp bằng tiếng Việt.  Nhiệm vụ: - Viết bài chuẩn SEO - Nội dung chi tiết - Có HTML đầy đủ - Có CSS inline đẹp - Chỉ trả về JSON thuần túy - Không dùng markdown - Không dùng ```json  JSON phải có đúng 2 field: {   "post_title": "...",   "post_content": "..." }
```
<img width="1140" height="963" alt="image" src="https://github.com/user-attachments/assets/03e66e92-798c-4f39-be5b-c978ca0e72f6" />

---

##### 🔷 Prompt Message 2

Bấm:

```text
Add Message
```

rồi điền:

| Trường | Giá trị |
|---|---|
| Role | `User` |
| Content | Nội dung bên dưới |

```javascript
{{ $json.message.text }}

Kết quả sinh ra ở định dạng HTML+CSS để tôi dùng HTML+CSS này tạo bài viết cho wordpress.

Trả về JSON với 2 trường:
- post_title
- post_content
```

<img width="1262" height="1007" alt="image" src="https://github.com/user-attachments/assets/d86c9e2a-002f-4fa3-9f34-af04829ac016" />

---

##### 📌 Test Deepseek

1. Nhấn `Execute step`
2. Mở Telegram
3. Gửi cho bot: Ví dụ :  ```wordpess là gì```
<img width="1210" height="956" alt="image" src="https://github.com/user-attachments/assets/1c32ffbe-e256-43e2-8246-6288a252e1c6" />

4. Nếu thành công, node sẽ xuất hiện Output
<img width="1919" height="1079" alt="Screenshot 2026-05-21 235219" src="https://github.com/user-attachments/assets/65b273da-5906-44c1-88d6-137dca822381" />


#### 🔷 Node 3: Code in JavaScript

```javascript
// 1. Lấy dữ liệu gốc từ DeepSeek trả về
const rawText = $json.message.content;

// 2. Làm sạch JSON (loại bỏ markdown code fence nếu có)
const cleaned = rawText
  .replace(/```json\s*/gi, "")
  .replace(/```\s*/g, "")
  .trim();

// 3. Parse chuỗi JSON thành Object JavaScript
const cleanData = JSON.parse(cleaned);

// 4. Trả về title và content cho node WordPress
return {
  title: cleanData.post_title,
  content: cleanData.post_content,
};
```
<img width="1696" height="1019" alt="image" src="https://github.com/user-attachments/assets/9580722e-1f7d-42d3-835e-9c133dbe35e8" />

##### 📌 Test Output
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/c2631b4f-0315-48da-b517-36ac51da0f96" />


#### 🔷 Node 4: WordPress – Create a Post

| Trường | Giá trị |
|---|---|
| Credential | Tạo mới → WordPress account |
| Resource | `Post` |
| Operation | `Create` |
| WordPress URL | `https://wordpress.nhukhiem.id.vn/` |
| Ignore SSL Issues | `true` |
| Title | `{{ $json.title }}` |
| Content | `{{ $json.content }}` |
| Status | `Publish` hoặc `Draft` |

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/84b3b66f-0a2d-4295-9667-cee02e7eba5f" />

---

### 🔷 Cấu hình WordPress Credential

| Trường | Giá trị |
|---|---|
| WordPress URL | `https://wordpress.nhukhiem.id.vn/` |
| Username | Username admin WordPress |
| Password | Application Password WordPress |
| Ignore SSL Issues | `true` |

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/4c96feeb-db5f-4be2-8439-dc591b4c36d5" />

---

### 🔷 Mapping dữ liệu

| Field WordPress | Giá trị |
|---|---|
| Title | `{{ $json.title }}` |
| Content | `{{ $json.content }}` |

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/ad3e5079-6f89-402d-b34d-8b96942936e0" />


---

### 🔷 Workflow hoàn chỉnh

```text
Telegram Trigger
    ↓
DeepSeek Create Chat Completion
    ↓
Code
    ↓
WordPress Create Post
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/160a15b8-4d49-493a-ae2d-ffdc87210a58" />

---

### 🔷 Publish Workflow

Bấm:

```text
Publish
```

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/5c357fdf-67ee-4051-997a-d70ce70875d0" />


---

### 🔷 Cách hoạt động

1. Gửi tin nhắn Telegram
2. DeepSeek sinh bài viết HTML
3. Code node parse JSON
4. WordPress tự đăng bài

## 🎯 Kết Quả Đạt Được

### Workflow tổng thể

```mermaid
graph LR
    A["📱 Nhắn tin\nTelegram Bot"] -->|"Webhook Trigger"| B["⚙️ N8N\nWorkflow"]
    B -->|"Prompt"| C["🤖 DeepSeek AI"]
    C -->|"JSON Response"| B
    B -->|"REST API"| D["📝 WordPress"]
    D -->|"Bài viết Published"| E["🌐 wordpress\n.nhukhiem.id.vn"]
```

### Checklist hoàn thành

- [x] 5 service Docker chạy ổn định, không restart
- [x] WordPress public tại `https://wordpress.nhukhiem.id.vn`
- [x] phpMyAdmin public tại `https://phpmyadmin.nhukhiem.id.vn`
- [x] N8N public tại `https://n8n.nhukhiem.id.vn`
- [x] N8N License đã được Activate
- [x] Workflow 4 nodes đã được Publish
- [x] Chat Telegram Bot → bài viết tự động lên WordPress

---

## 💬 Nhận Xét

**Những gì đạt được:**
- Xây dựng thành công pipeline tự động hóa hoàn chỉnh từ đầu đến cuối
- Hiểu được cách tích hợp nhiều công nghệ: Docker, Cloudflare Tunnel, Telegram Bot API, DeepSeek AI API, WordPress REST API
- N8N là công cụ mạnh mẽ cho phép kết nối các service khác nhau mà không cần viết code phức tạp

**Khó khăn gặp phải:**
- Cấu hình Cloudflare Tunnel cần thêm router cho từng subdomain riêng lẻ
- DeepSeek đôi khi trả về JSON bọc trong markdown fence (```json...```) cần xử lý thêm ở Code Node
- Phải chat lần đầu với Telegram Bot trước khi webhook hoạt động

**Điểm cải tiến có thể làm thêm:**
- Thêm node xử lý lỗi (Error Trigger) khi AI trả về JSON không hợp lệ
- Thêm Telegram bot phản hồi xác nhận khi bài viết đã được đăng thành công
- Thêm System Message tốt hơn để AI sinh nội dung chuẩn SEO


<div align="center">

**BT3 Repository:** [nhukhiem3143/BT3-WordPress--Dev-App-OpenSrc](https://github.com/nhukhiem3143/BT3-WordPress--Dev-App-OpenSrc)

Made with ❤️ by Như Khiêm – 58KTPM

</div>
