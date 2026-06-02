# Analisis Latihan Test Project: Serverless Event-Driven Automation with AWS
*Lomba Kompetensi Siswa (LKS) - Cloud Computing*

Dokumen ini berisi analisis detail dan komprehensif terhadap spesifikasi proyek **"Serverless Event-Driven Automation with AWS"** berdasarkan file [Latihan Test Project.pdf](file:///home/syntaxtrust/Documents/SMK/LKS/Latihan%20Test%20Project.pdf). Proyek ini dirancang sebagai simulasi ujian kompetensi (LKS) bidang Cloud Computing dengan durasi pengerjaan **5 Jam**.

---

## 1. Ringkasan Proyek (Project Overview)

Sistem yang dibangun adalah **Sistem Pemrosesan Pesanan Otomatis (Automated Order Processing System)** berbasis serverless dan *event-driven*. Seluruh komponen infrastruktur harus didefinisikan sebagai *Infrastructure as Code* (IaC) menggunakan **AWS CloudFormation** (kecuali beberapa komponen yang dinyatakan dideploy manual tanpa CloudFormation).

### Parameter Teknis Global:
*   **Wilayah Default (Region):** Virginia Utara (`us-east-1`)
*   **Prefix Layanan:** Semua nama resource wajib diawali dengan prefix `lks-` (misalnya: `lks-vpc-serverless`, `lks-lambda-order-management`).
*   **Sistem Operasi EC2:** Amazon Linux 2023 (jika diperlukan).
*   **Runtime Lambda:** Python 3.11
*   **Database:** MySQL 8.0.x (Amazon RDS)

---

## 2. Arsitektur Sistem & Aliran Data (Architecture & Data Flow)

Berikut adalah visualisasi alur kerja pemrosesan order berdasarkan diagram arsitektur di halaman 3 dokumen:

```mermaid
graph TD
    subgraph Frontend & API
        Amplify[AWS Amplify <br/> lks-amplify-order-app] -->|1. Request| APIGW[API Gateway <br/> lks-api-orders]
        APIGW -->|2. Invoke| LambdaOrder[Lambda: Order Management <br/> lks-lambda-order-management]
    end

    subgraph Orchestration & Logic
        LambdaOrder -->|3. Start Workflow| StepFunc{Step Functions <br/> lks-stepfunctions-order-workflow}
        
        StepFunc -->|Validate| LambdaOrder
        StepFunc -->|Process Payment| LambdaPay[Lambda: Process Payment <br/> lks-lambda-process-payment]
        StepFunc -->|Update Inventory| LambdaInv[Lambda: Update Inventory <br/> lks-lambda-update-inventory]
        StepFunc -->|Notifications| LambdaNotif[Lambda: Send Notification <br/> lks-lambda-send-notification]
    end

    subgraph Data & Storage
        LambdaOrder -->|Read/Write| RDS[(RDS MySQL <br/> lks-rds-orders)]
        LambdaInv -->|Read/Write| RDS
        LambdaOrder -->|Upload Invoice/Report| S3Docs[(S3 Bucket Documents <br/> lks-orders-yourname-2026)]
    end

    subgraph Events & Schedules
        EventBridge[EventBridge <br/> Scheduled & Status Rules] -->|Daily Report| LambdaReport[Lambda: Generate Report <br/> lks-lambda-generate-report]
        EventBridge -->|Low Stock Alert| LambdaLowStock[Lambda: Low Stock Check]
        EventBridge -->|Order Status Event| LambdaNotif
        LambdaNotif -->|Publish| SNS[SNS Topic <br/> lks-sns-order-notifications]
        SNS -->|Email Subscription| Admin[handi@seamolec.org]
    end

    subgraph Monitoring
        CloudWatch[CloudWatch Logs & Alarms] -.->|Monitor| APIGW
        CloudWatch -.->|Monitor| LambdaOrder
        CloudWatch -.->|Monitor| StepFunc
        CloudWatch -.->|Monitor| RDS
    end
```

### Penjelasan Alur Utama (End-to-End):
1.  **User Interaksi:** Pengguna mengakses UI di **AWS Amplify** untuk membuat pesanan.
2.  **API Gateway:** Request dikirim ke API Gateway `lks-api-orders` dan diteruskan ke Lambda `lks-lambda-order-management`.
3.  **Step Functions:** Lambda tersebut memicu State Machine `lks-stepfunctions-order-workflow` untuk memulai proses orkestrasi pemrosesan order.
4.  **Database & Storage:** Selama alur berjalan, data order dicatat ke **RDS MySQL** dan dokumen invoice disimpan ke **S3 Bucket**.
5.  **Notifikasi:** Jika terjadi perubahan status order atau error, Lambda akan mempublikasikan pesan ke **SNS Topic** untuk dikirim ke email admin.

---

## 3. Rincian Kebutuhan Komponen Infrastruktur (CloudFormation)

### 3.1. Jaringan & Keamanan (VPC & Security Groups)
*   **VPC Name:** `lks-vpc-serverless`
    *   **IPv4 CIDR:** `20.1.0.0/20`
    *   **IPv6 Range:** `/56` (Disediakan oleh Amazon)
    *   **Subnet:**
        *   2x Public Subnet: `20.1.0.0/25` & `20.1.1.0/25`
        *   2x Private Subnet: `20.1.11.0/25` & `20.1.12.0/25`
        *   Semua subnet menggunakan IPv6 `/64` dan mengaktifkan **Egress-only Internet Gateway** untuk IPv6.
    *   **Route Tables:** 1 untuk Public Subnets, 1 untuk Private Subnets.
*   **Security Groups:**
    1.  `lks-sg-lambda`: Keamanan untuk fungsi Lambda. Tidak memerlukan aturan inbound.
    2.  `lks-sg-rds`: Keamanan untuk database RDS. Membuka port `5432` khusus dari `lks-sg-lambda`.
    3.  `lks-sg-vpc-endpoint`: Keamanan untuk VPC Endpoint. Membuka port HTTP (80) & HTTPS (443) khusus dari CIDR `20.1.0.0/20`.
*   **VPC Endpoints (Interface & Gateway):**
    *   `lks-s3-endpoints` (Gateway): `com.amazonaws.us-east-1.s3`
    *   `lks-eventbridge-endpoints` (Interface): `com.amazonaws.us-east-1.events`
    *   `lks-steps-endpoints` (Interface): `com.amazonaws.us-east-1.states`
    *   `lks-sns-endpoints` (Interface): `com.amazonaws.us-east-1.sns`

### 3.2. S3 Buckets
1.  **S3 Bucket Dokumen Order:** `lks-orders-yourname-2026`
    *   Versioning: **Enabled**
    *   CORS: Diizinkan untuk akses frontend.
    *   Bucket Policy: Diizinkan untuk semua (public read/write sesuai instruksi teknis).
    *   **Lifecycle Rules:**
        *   Transition ke **Intelligent-Tiering** setelah **30 hari**.
        *   Transition ke **Glacier** setelah **90 hari**.
        *   Hapus permanen setelah **365 hari**.
2.  **S3 Bucket Log Aplikasi:** `lks-logs-yourname-2026`
    *   **Lifecycle Rules:**
        *   Transition ke **Intelligent-Tiering** setelah **7 hari**.
        *   Hapus permanen setelah **90 hari**.

### 3.3. Relational Database Service (RDS)
*   **DB Name:** `lks-rds-orders`
*   **Engine:** MySQL version 8.0.x
*   **Specs:** `db.t3.micro`, Storage 20 GB (`gp3` volume type).
*   **Network:** Hanya di **Private Subnets** (tidak ada akses publik).
*   **Credentials:** Database name `ordersdb`, master username `dbadmin`, master password `TechnoCloud2026!`.
*   **Backup:** Backup otomatis diaktifkan dengan retensi **7 hari**.
*   **Inisialisasi Schema:** Menggunakan fungsi Lambda khusus `lks-lambda-init-db` untuk membuat tabel: `customers`, `orders`, `order_items`, dan `inventory` secara otomatis saat pertama kali dideploy.

### 3.4. Fungsi Lambda (Python 3.11) & Layers
Semua fungsi Lambda menggunakan layer bersama: `lks-layer-dependencies` yang berisi library `pymysql`, `boto3`, `requests`, `pandas`, dan `openpyxl`.

| Nama Fungsi | Spesifikasi | Kegunaan | Environment Variables |
| :--- | :--- | :--- | :--- |
| `lks-lambda-order-management` | Memory: 512 MB <br> Timeout: 30s <br> Subnet: Private | Handler utama pemrosesan pesanan (CRUD). | RDS endpoint/credentials, S3 bucket name, Step Functions ARN. |
| `lks-lambda-process-payment` | Memory: 512 MB <br> Timeout: 30s | Memvalidasi dan memproses transaksi pembayaran. | - |
| `lks-lambda-update-inventory` | Memory: 256 MB <br> Timeout: 45s | Memperbarui stok barang di database RDS. | - |
| `lks-lambda-send-notification` | Memory: 256 MB <br> Timeout: 60s (1m) | Mengirim notifikasi email/status via SNS. | - |
| `lks-lambda-generate-report` | Memory: 1024 MB <br> Timeout: 60s | Agregasi data dan pembuatan laporan bulanan (Excel). | - |
| `lks-lambda-init-db` | Memory: 512 MB <br> Timeout: 300s | Inisialisasi tabel, constraint, index, dan data contoh di RDS. | RDS endpoint/credentials. |

### 3.5. Simple Notification Service (SNS)
*   **Topic Name:** `lks-sns-order-notifications`
*   **Subscribers:** Email subscription ke `handi@seamolec.org` (harus dikonfirmasi secara manual).
*   **Fungsi:** Mengirim notifikasi jika order berhasil dikonfirmasi, terjadi kegagalan pembayaran, stok menipis, system error, atau laporan harian telah selesai digenerate.

### 3.6. API Gateway
*   **API Name:** `lks-api-orders`
*   **Type:** REST API, Regional Endpoint, IP address `dualstack`, Security Policy `TLS 1.3`.
*   **Stage:** `production`
*   **Security:** Diperlukan **API Key** bernama `lks-api-key` dan diikat ke **Usage Plan** `lks-usage-plan` dengan aturan:
    *   Rate Limit: `1,000` requests/second
    *   Burst Limit: `2,000` requests
    *   Monthly Quota: `100,000` requests
*   **Endpoints:** Semua terintegrasi ke Lambda `lks-lambda-order-management`:
    *   `GET /orders` (List Orders)
    *   `POST /orders` (Create Order)
    *   `GET /orders/{id}` (Get Order Details)
    *   `PUT /orders/{id}` (Update Order)
    *   `DELETE /orders/{id}` (Delete Order)
    *   `GET /status/{id}` (Check Workflow Status) *— Catatan: Di PDF tertulis `/status{id}` tetapi secara standar RESTful adalah `/status/{id}`*
    *   `GET /customers` (Get Customers list)
    *   `GET /products` (Get Products list)

---

## 4. Implementasi Tanpa CloudFormation (Manual/Application Config)

### 4.1. AWS Step Functions State Machine
*   **Name:** `lks-stepfunctions-order-workflow`
*   **Type:** Standard Workflow
*   **IAM Role:** Menggunakan role `LabRole` yang sudah disediakan oleh AWS Academy/Class Account.
*   **Definisi Logika Workflow:**

```mermaid
stateDiagram-v2
    [*] --> ValidateOrder : Mulai
    note right of ValidateOrder: Lambda: lks-lambda-order-management

    ValidateOrder --> ProcessPayment
    note right of ProcessPayment: Lambda: lks-lambda-process-payment

    ProcessPayment --> PaymentChoice

    state PaymentChoice <<choice>>
    PaymentChoice --> UpdateInventory : paymentStatus == "SUCCESS"
    PaymentChoice --> PaymentFailed : paymentStatus == "FAILED" / "PENDING"

    state UpdateInventory {
        [*] --> PerformUpdate
        note right of PerformUpdate: Lambda: lks-lambda-update-inventory
    }

    UpdateInventory --> InventoryChoice

    state InventoryChoice <<choice>>
    InventoryChoice --> SendConfirmation : status == "SUCCESS"
    InventoryChoice --> InventoryFailed : status == "FAILED"

    SendConfirmation --> [*] : Order Selesai (Success)
    note right of SendConfirmation: Lambda: lks-lambda-send-notification

    PaymentFailed --> OrderFailed : Kirim Notif Gagal Bayar
    note right of PaymentFailed: Lambda: lks-lambda-send-notification

    InventoryFailed --> OrderFailed : Kirim Notif Gagal Stok
    note right of InventoryFailed: Lambda: lks-lambda-send-notification

    OrderFailed --> [*] : Selesai (Failed)
```

### 4.2. Amazon EventBridge Rules
1.  **Daily Report (`lks-eventbridge-daily-report`):**
    *   Schedule expression: `cron(59 23 * * ? *)` (Trigger setiap hari pukul 23:59 UTC).
    *   Target: `lks-lambda-generate-report`
2.  **Order Status Events (`lks-eventbridge-order-status`):**
    *   Event Pattern: Menangkap custom event status order (`created`, `paid`, `shipped`, `failed`).
    *   Target: `lks-lambda-send-notification`
3.  **Low Stock Alert (`lks-eventbridge-low-stock`):**
    *   Schedule expression: `rate(1 hour)` (Trigger setiap 1 jam sekali).
    *   Target: Fungsi Lambda khusus pemeriksaan stok untuk mendeteksi item di bawah batas minimum.

### 4.3. Amazon CloudWatch
*   **Log Groups:** Dibuat untuk seluruh fungsi Lambda, Step Functions, dan API Gateway dengan **retensi log 7 hari**.
*   **Alarms:**
    1.  **Lambda Alarm:** Pemicu ketika tingkat error Lambda > 5 kali dalam waktu 5 menit.
    2.  **API Gateway Alarm:** Pemicu ketika terjadi error klien (4XX) > 10 kali dalam waktu 5 menit.
    3.  **Step Functions Alarm:** Pemicu ketika kegagalan eksekusi workflow > 3 kali dalam waktu 10 menit.
    4.  **RDS Alarm:** Pemicu ketika penggunaan CPU RDS > 80% berturut-turut selama 5 menit.
*   **Dashboard:** `lks-dashboard-serverless` yang merangkum metrik performa Lambda, statistik request/error API Gateway, status Step Functions, dan utilisasi resource RDS.

### 4.4. AWS Amplify Frontend
*   **App Name:** `lks-amplify-order-app`
*   **Source:** Terkoneksi ke Repositori GitHub (`main` branch) dengan tipe Static HTML/CSS/JavaScript.
*   **Konfigurasi Build (`amplify.yml`):** Disesuaikan dengan struktur project static.
*   **Fitur Frontend:**
    *   Dashboard ringkasan statistik performa sistem.
    *   Form pembuatan pesanan baru.
    *   Tabel manajemen pesanan (View, Update, Delete).
    *   Halaman Laporan untuk mengakses file Excel hasil generate.
    *   Pemantauan status workflow sistem (status monitoring).
*   **Environment Variables:**
    *   `API_ENDPOINT`: URL API Gateway Production.
    *   `API_KEY`: Kunci otentikasi API Gateway.
    *   `AWS_REGION`: `us-east-1`

### 4.5. GitHub Actions CI/CD Pipeline
*   **File Alur Kerja:** `.github/workflows/lks-deploy.yml`
*   **Pemicu:** Push atau Pull Request ke branch `master` pada direktori `frontend/`, perubahan file workflow, atau manual trigger via `workflow_dispatch`.
*   **Job 1: `deploy-frontend`**
    *   Mengunduh kode (checkout).
    *   Konfigurasi AWS credentials menggunakan GitHub Secrets:
        *   `AWS_ACCESS_KEY_ID`
        *   `AWS_SECRET_ACCESS_KEY`
        *   `AWS_SESSION_TOKEN`
    *   Memvalidasi keberadaan file wajib: `index.html`, `app.js`, dan `amplify.yml`.
    *   Memeriksa keberadaan AWS Amplify app `lks-amplify-order-app`.
    *   Memicu deployment baru di Amplify dan memantau statusnya hingga sukses/gagal/timeout.
    *   Mencetak URL hasil deploy dan membuat ringkasan deploy di ringkasan GitHub Actions.
*   **Job 2: `notify`**
    *   Berjalan setelah `deploy-frontend` selesai (baik sukses maupun gagal) hanya pada branch `master`.
    *   Mengirim notifikasi status deployment detail (status, commit, author, link workflow) ke **SNS Topic** AWS.

---

## 5. Rencana Pengujian & Verifikasi (Testing Plan)

Untuk memastikan sistem berfungsi 100% dengan benar di LKS, ikuti daftar pengujian berikut:

1.  **API Testing (Postman/curl):**
    *   Panggil endpoint `/orders` dengan menyertakan `x-api-key` di header. Pastikan respon 200 OK.
    *   Panggil tanpa API Key atau dengan key salah. Pastikan respon 403 Forbidden atau 401 Unauthorized.
    *   Coba lakukan `POST /orders` dengan JSON payload yang tidak lengkap. Uji apakah skema validasi berjalan dengan baik di Lambda.
2.  **Workflow Testing (Step Functions):**
    *   Jalankan eksekusi Step Functions secara manual dengan payload valid. Pantau visualisasi grafis transisi state dari `ValidateOrder` -> `ProcessPayment` -> `UpdateInventory` -> `SendConfirmation` -> Selesai.
    *   Jalankan dengan status pembayaran bernilai `FAILED`. Pastikan alur bercabang ke `PaymentFailed` dan berakhir dengan status `OrderFailed`.
3.  **Event Testing (EventBridge & SNS):**
    *   Uji manual trigger EventBridge Rule untuk `Daily Report` dengan merubah waktu cron sementara atau memicunya secara langsung. Cek apakah Lambda menghasilkan file laporan di S3 dan email terkirim ke `handi@seamolec.org`.
    *   Cek inbox email `handi@seamolec.org` untuk memverifikasi email konfirmasi subscription SNS dari AWS.
4.  **Frontend & Integration Testing:**
    *   Buka URL AWS Amplify hasil deploy di browser.
    *   Gunakan form pemesanan untuk mengirim order baru, dan periksa apakah order langsung muncul di tabel manajemen pesanan.
    *   Cek log CloudWatch jika ada kegagalan komunikasi API.
5.  **CI/CD Pipeline Validation:**
    *   Lakukan perubahan kecil di file `frontend/index.html` dan push ke branch `master`.
    *   Pantau tab **Actions** di GitHub. Pastikan job `deploy-frontend` mendeteksi perubahan, mendeploy ke AWS Amplify, dan job `notify` berhasil mengirim email status deploy via SNS.

---

> [!NOTE]
> Proyek LKS Cloud Computing ini menuntut ketelitian tinggi terutama pada **Naming Convention** (prefix `lks-`), **Lifecycle Policy** S3 yang tepat, dan **Keamanan VPC** (menempatkan RDS di subnet private tanpa akses internet langsung). Pastikan aturan CIDR dan Security Group telah dikonfigurasi dengan presisi.
