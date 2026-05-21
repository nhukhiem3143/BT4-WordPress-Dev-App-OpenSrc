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
| Ubuntu | 20.04 LTS |
| Docker | 24.x |
| Docker Compose | v2.x |
| RAM | 2GB+ |
| Disk | 10GB+ |

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
├── 📁 n8n/
│   └── 📄 code-node.js         # Code JS dùng trong n8n Code Node
├── 📁 volumes/                 # Dữ liệu persistent (auto tạo)
│   ├── mariadb/
│   ├── wordpress/
│   ├── phpmyadmin/
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

> ⚠️ **Lưu ý:** Không commit file `.env` lên GitHub. File `.gitignore` đã loại trừ sẵn.

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

### Bước 4 – Cài Đặt WordPress

**1. Kiểm tra CSDL trước khi cài:**
- Truy cập `https://phpmyadmin.nhukhiem.id.vn`
- Đăng nhập với user/password từ `.env`
- Quan sát: database `wordpress_db` **chưa có bảng nào**

**2. Cài đặt WordPress:**
- Truy cập `https://wordpress.nhukhiem.id.vn`
- Làm theo wizard cài đặt của WordPress
- Điền thông tin: Tên site, admin, mật khẩu, email

**3. Kiểm tra CSDL sau khi cài:**
- Quay lại phpMyAdmin
- Quan sát: database đã có **11 bảng** do WordPress tạo tự động

```
wp_commentmeta    wp_comments      wp_links
wp_options        wp_postmeta      wp_posts
wp_term_relationships  wp_term_taxonomy  wp_termmeta
wp_terms          wp_usermeta      wp_users
```

**4. Tạo 2 bài viết thủ công trong WordPress:**
- Bài 1: Giới thiệu bản thân (thông tin cá nhân, sở thích, có hình ảnh/video)
- Bài 2: Kiến thức học được từ môn Phát triển ứng dụng với mã nguồn mở

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

---

### Bước 2 – Tạo Telegram Bot

1. Mở Telegram → tìm kiếm **@BotFather** → bắt đầu chat
2. Gõ lệnh `/newbot`
3. Đặt tên bot (vd: `NhuKhiem WordPress Bot`)
4. Đặt username bot (phải kết thúc bằng `bot`, vd: `nhukhiem_wp_bot`)
5. **Copy Token** được cấp (dạng: `1234567890:AAxxxxxxxxxxxxxx`)

> ⚠️ **Quan trọng:** Sau khi tạo bot, phải **chat lần đầu** với bot (nội dung bất kỳ) trước khi dùng trong n8n, nếu không webhook sẽ không nhận được message.

---

### Bước 3 – Lấy DeepSeek API Key

1. Truy cập `https://platform.deepseek.com`
2. Đăng nhập / Đăng ký tài khoản
3. Vào **API Keys** → **Create new API key**
4. **Copy API Key** (chỉ hiển thị một lần)

---

### Bước 4 – Tạo WordPress Application Password

1. Truy cập `https://wordpress.nhukhiem.id.vn/wp-admin`
2. **Users → Profile** (hoặc vào tài khoản admin)
3. Kéo xuống phần **Application Passwords**
4. Nhập tên: `n8n` → bấm **"Add New Application Password"**
5. **Copy chuỗi 24 ký tự** được tạo ra (chỉ hiển thị một lần)

---

### Bước 5 – Build Workflow

Truy cập `https://n8n.nhukhiem.id.vn` → **Overview → Create Workflow**

#### 🔷 Node 1: Telegram Trigger

| Trường | Giá trị |
|---|---|
| Authentication | `Credential` |
| Credential | Tạo mới → Paste **Telegram Bot Token** |
| Updates | `message` |

#### 🔷 Node 2: DeepSeek – Message a Model

| Trường | Giá trị |
|---|---|
| Credential | Tạo mới → Paste **DeepSeek API Key** |
| Model | `deepseek-chat` |
| Prompt | Kéo `message.text` từ panel trái vào → thêm hậu tố |

**Nội dung Prompt:**
```
{{ $json.message.text }}. Kết quả sinh ra ở định dạng HTML+CSS để tôi dùng HTML+CSS này tạo bài viết cho wordpress. Trả về JSON với 2 trường: post_title (tiêu đề bài viết) và post_content (nội dung HTML đầy đủ).
```

| Option | Giá trị |
|---|---|
| Output Content as JSON | **BẬT** |
| System Message (Add Option) | `Bạn là trợ lý viết bài blog chuyên nghiệp bằng tiếng Việt. Chỉ trả về JSON thuần túy, không có markdown fence.` |

#### 🔷 Node 3: Code in JavaScript

```javascript
// 1. Lấy dữ liệu gốc từ DeepSeek trả về
const rawText = $input.first().json.text;

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

#### 🔷 Node 4: WordPress – Create a Post

| Trường | Giá trị |
|---|---|
| Credential | Tạo mới (xem bên dưới) |
| WordPress URL | `https://wordpress.nhukhiem.id.vn/` |
| Ignore SSL Issues | **BẬT** |
| Title | Kéo `title` từ node trước vào |
| Content | Kéo `content` từ node trước vào |
| Status | `Publish` |

**Cấu hình WordPress Credential:**

| Trường | Giá trị |
|---|---|
| WordPress URL | `https://wordpress.nhukhiem.id.vn/` |
| Username | *(username admin WordPress)* |
| Password | *(chuỗi 24 ký tự Application Password)* |
| Ignore SSL Issues | `true` |

#### 🔷 Publish Workflow

Bấm nút **"Publish"** (góc trên phải) để workflow hoạt động.

> ✅ Sau khi Publish, mỗi tin nhắn gửi đến Telegram Bot sẽ tự động kích hoạt toàn bộ workflow.

---

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
- [x] 2 bài viết thủ công đã được tạo trong WordPress
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

---

## 📚 Tài Liệu Tham Khảo

- [N8N Documentation](https://docs.n8n.io/)
- [WordPress REST API](https://developer.wordpress.org/rest-api/)
- [DeepSeek API Reference](https://platform.deepseek.com/api-docs)
- [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Telegram Bot API](https://core.telegram.org/bots/api)

---

<div align="center">

**BT3 Repository:** [nhukhiem3143/BT3-WordPress--Dev-App-OpenSrc](https://github.com/nhukhiem3143/BT3-WordPress--Dev-App-OpenSrc)

Made with ❤️ by Như Khiêm – 58KTPM

</div>
