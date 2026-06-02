# Panduan Master LKS Cloud Computing: Teori, Kode Sumber & Praktik Konsol AWS (Versi Standar PostgreSQL)
*Modul Terpadu Pembelajaran Mandiri Persiapan Lomba Kompetensi Siswa (LKS) Bidang Cloud Computing*

Halo! Berkas ini adalah **panduan standar LKS** yang menggunakan database **PostgreSQL** sesuai dengan spesifikasi resmi lomba. Panduan ini dilengkapi dengan solusi kendala teknis *dropdown* kosong di AWS Console dan pencegahan eror *S3 Lifecycle*.

---

## 📑 Daftar Isi Modul Terpadu
1. [Konsep Dasar: Apa itu Serverless & Event-Driven?](#1-konsep-dasar-apa-itu-serverless--event-driven)
2. [Modul 1: Jaringan Aman & VPC](#modul-1-jaringan-aman--vpc)
3. [Modul 2: Database & Penyimpanan (S3 & RDS)](#modul-2-database--penyimpanan-s3--rds)
4. [Modul 3: Logika Bisnis & Dependency (Lambda & Layers)](#modul-3-logika-bisnis--dependency-lambda--layers)
5. [Modul 4: API & Notifikasi (API Gateway & SNS)](#modul-4-api--notifikasi-api-gateway--sns)
6. [Modul 5: Orkestrasi Alur Kerja (AWS Step Functions)](#modul-5-orkestrasi-alur-kerja-aws-step-functions)
7. [Modul 6: Otomatisasi & Pemantauan (EventBridge & CloudWatch)](#modul-6-otomatisasi--pemantauan-eventbridge--cloudwatch)
8. [Modul 7: Frontend & CI/CD Pipeline (Amplify & GitHub Actions)](#modul-7-frontend--cicd-pipeline-amplify--github-actions)
9. [Tips & Trik Juara LKS Cloud Computing](#tips--trik-juara-lks-cloud-computing)

---

## 1. Konsep Dasar: Apa itu Serverless & Event-Driven?

Sebelum mulai praktik, mari kita pahami dulu dua istilah utama ini:
*   **Serverless (Tanpa Server Tradisional):** Kita tidak perlu menyewa dan mengelola server virtual (seperti EC2). AWS yang otomatis mengatur skalabilitas dan keamanannya. Kita hanya fokus pada penulisan kode.
*   **Event-Driven (Digerakkan Kejadian):** Komponen-komponen sistem berkomunikasi lewat pesan atau peristiwa (*events*). 
    *   *Analogi:* Bel pintu berbunyi (Event) -> Kamu berjalan membukakan pintu (Response/Action). Di proyek ini: Pesanan masuk (Event) -> Sistem otomatis memicu proses pembayaran dan pengurangan stok (Response).

---

## Modul 1: Jaringan Aman & VPC

Di modul ini, kita akan membangun jaringan virtual terisolasi yang disebut **VPC (Virtual Private Cloud)** lengkap dengan subnet publik dan privat, serta jalur komunikasi privat terisolasi (VPC Endpoints).

### 1.1. Konsep & Arsitektur Jaringan
*   **Subnetting**: Kita membagi blok IP `20.1.0.0/20` menjadi:
    *   *Subnet Publik 1 & 2* (`20.1.0.0/25` & `20.1.1.0/25`): Terhubung langsung ke internet.
    *   *Subnet Privat 1 & 2* (`20.1.11.0/25` & `20.1.12.0/25`): Terisolasi untuk meletakkan Database RDS dan Lambda backend.
*   **Security Groups**:
    *   `lks-sg-lambda`: Dipasang pada Lambda (Tanpa inbound rule/aturan masuk).
    *   `lks-sg-rds`: Hanya membuka port database PostgreSQL `5432` dari `lks-sg-lambda`.
    *   `lks-sg-vpc-endpoint`: Membuka port HTTPS (443) khusus untuk traffic IP dalam VPC (`20.1.0.0/20`).
*   **VPC Endpoints**: "Jalan tol" privat di dalam jaringan AWS agar Lambda privat bisa mengakses S3, SNS, Step Functions, dan EventBridge secara langsung dan cepat tanpa perlu rute internet publik.

---

### 1.2. Langkah Praktik Konsol AWS (Klik-demi-Klik)

#### A. Membuat VPC dengan Wizard "VPC and More"
1.  Buka browser, login ke AWS Console, pastikan wilayah di pojok kanan atas teratur ke **N. Virginia (us-east-1)** atau **N. California (us-west-1)**.
2.  Di search bar atas, ketik **VPC** lalu pilih layanan **VPC**.
3.  Klik tombol **Create VPC** berwarna oranye.
4.  Pilih opsi **VPC and more** dan konfigurasikan bagian atas formulir:
    *   **Name tag auto-generation**: Isi `lks`.
    *   **IPv4 CIDR block**: `20.1.0.0/20`.
    *   **IPv6 CIDR block**: Pilih **Amazon-provided IPv6 CIDR block**.
    *   **Number of Availability Zones (AZs)**: Pilih **2**.
    *   **Number of Public Subnets**: Pilih **2**.
    *   **Number of Private Subnets**: Pilih **2**.
    *   **NAT Gateways ($)**: Pilih **None** (Kita akan memakai VPC Endpoints privat).
    *   **VPC Endpoints**: Pilih **None** (Kita akan membuatnya manual agar penamaannya tepat).
    *   Centang **Enable DNS resolution** dan **Enable DNS hostnames**.
5.  > ⚠️ **LANGKAH KRITIS — JANGAN DILEWATKAN!**
    > AWS akan otomatis mengisi CIDR subnet dengan nilai default yang **SALAH** (misalnya `/24`). Kamu **WAJIB** mengubahnya secara manual di bagian **"Customize subnets CIDR blocks"** sebelum menekan Create VPC.
    >
    > Gulir ke bawah di formulir yang sama hingga menemukan tabel **Subnet CIDR blocks**, lalu ubah setiap isian CIDR berikut:
    >
    > | Subnet | CIDR Default AWS (Salah) | ✅ CIDR yang Harus Diisi |
    > | :--- | :--- | :--- |
    > | Public Subnet 1 | `20.1.0.0/24` | **`20.1.0.0/25`** |
    > | Public Subnet 2 | `20.1.1.0/24` | **`20.1.1.0/25`** |
    > | Private Subnet 1 | `20.1.8.0/24` | **`20.1.11.0/25`** |
    > | Private Subnet 2 | `20.1.9.0/24` | **`20.1.12.0/25`** |
6.  Setelah seluruh CIDR sudah diubah dengan benar seperti tabel di atas, klik **Create VPC** di bagian paling bawah. Tunggu hingga proses selesai, lalu klik **View VPC**.

#### A.1. ⚠️ Wajib: Rename VPC ke Nama yang Benar
> **Perhatian!** Wizard AWS akan memberi nama VPC secara otomatis menjadi **`lks-vpc`** (bukan `lks-vpc-serverless`). Ini harus segera diubah karena nama resource diperiksa oleh mesin penilaian otomatis LKS!
>
> Langkah mengganti nama:
> 1. Di halaman **Your VPCs**, cari dan klik VPC bernama **`lks-vpc`** yang baru saja dibuat.
> 2. Di bagian bawah halaman, klik tab **Details**.
> 3. Di kolom **Name**, arahkan kursor ke nama `lks-vpc`, lalu klik ikon **pensil** ✏️ yang muncul.
> 4. Hapus nama lama, ketik **`lks-vpc-serverless`**, lalu tekan **Enter** atau klik ikon **centang** untuk menyimpan.
> 5. Refresh halaman. Pastikan nama sudah berubah menjadi `lks-vpc-serverless` sebelum melanjutkan.

#### B. Menyesuaikan Penamaan Subnet
1.  Pada menu kiri, klik **Subnets**.
2.  Arahkan kursor ke kolom nama subnet, klik ikon **pensil**, sesuaikan namanya jika belum tepat:
    *   `lks-subnet-public1-us-east-1a` (CIDR: `20.1.0.0/25`)
    *   `lks-subnet-public2-us-east-1b` (CIDR: `20.1.1.0/25`)
    *   `lks-subnet-private1-us-east-1a` (CIDR: `20.1.11.0/25`)
    *   `lks-subnet-private2-us-east-1b` (CIDR: `20.1.12.0/25`)
    *   Klik **Save** setiap mengubah nama.

#### C. Membuat Security Groups (Satpam Jaringan)
1.  Pada menu kiri VPC, scroll ke bawah ke bagian **Security** -> klik **Security Groups**.
2.  Klik **Create security group** (Pojok kanan atas).
3.  **Grup 1: `lks-sg-lambda`**:
    *   **Name**: `lks-sg-lambda`, **Description**: `Security group for Lambda`
    *   **VPC**: Pilih **lks-vpc-serverless** (hapus default VPC dengan klik `x`).
    *   *Inbound rules*: Biarkan kosong. Klik **Create security group**.
4.  **Grup 2: `lks-sg-rds`**:
    *   Klik **Create security group** lagi.
    *   **Name**: `lks-sg-rds`, **Description**: `Security for RDS PostgreSQL`
    *   **VPC**: Pilih **lks-vpc-serverless**.
    *   *Inbound rules* (Klik **Add rule**):
        *   **Type**: Pilih **PostgreSQL** (port otomatis terisi 5432).
        *   **Source**: Pilih **Custom**, lalu cari dan pilih Security Group **`lks-sg-lambda`**.
    *   Klik **Create security group**.
5.  **Grup 3: `lks-sg-vpc-endpoint`**:
    *   Klik **Create security group** sekali lagi.
    *   **Name**: `lks-sg-vpc-endpoint`, **Description**: `Security for VPC Endpoints`
    *   **VPC**: Pilih **lks-vpc-serverless**.
    *   *Inbound rules* (Klik **Add rule**):
        *   **Type**: Pilih **HTTPS** (port 443).
        *   **Source**: Pilih **Custom**, lalu masukkan IP VPC kita: `20.1.0.0/20`.
    *   Klik **Create security group**.

#### D. Membuat VPC Endpoints
1.  Pada menu kiri VPC, klik **Endpoints** -> klik **Create endpoint**.
2.  **S3 Gateway Endpoint**:
    *   **Name**: `lks-s3-endpoints`
    *   **Service category**: **AWS services**.
    *   **Services**: Ketik `s3`, centang `com.amazonaws.us-east-1.s3` (**Type = Gateway**).
    *   **VPC**: Pilih **lks-vpc-serverless**.
    *   **Route tables**: Di sini akan muncul **4 route table** — ini normal. Centang hanya **kedua route table privat** yaitu:
        *   `lks-rtb-private1-us-east-1a`
        *   `lks-rtb-private2-us-east-1b`
        *   *(Jangan centang route table utama/Main dan route table publik)*
    *   Klik **Create endpoint**.
3.  **Interface Endpoints**:
    *   Ulangi klik **Create endpoint** untuk:
        *   `lks-eventbridge-endpoints` (layanan: `com.amazonaws.us-east-1.events`)
        *   `lks-steps-endpoints` (layanan: `com.amazonaws.us-east-1.states`)
        *   `lks-sns-endpoints` (layanan: `com.amazonaws.us-east-1.sns`)
    *   Untuk ketiga layanan di atas, atur **VPC** = `lks-vpc-serverless`, centang **kedua Subnet Privat**, dan pada **Security groups**, centang **`lks-sg-vpc-endpoint`** (hapus centang default).

---

### 1.3. Potongan Template CloudFormation Modul 1
```yaml
Resources:
  LKSVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 20.1.0.0/20
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: lks-vpc-serverless
```

---

## Modul 2: Database & Penyimpanan (S3 & RDS)

Di modul ini kita akan menyiapkan ruang penyimpanan data tak terstruktur (S3) dan database relasional aman (PostgreSQL RDS).

### 2.1. Konsep Database & Siklus Log
*   **Lifecycle S3**: Kita menerapkan aturan pengarsipan otomatis: Intelligent-Tiering (hari ke-30), Glacier (hari ke-90), dan Hapus Permanen (hari ke-365) guna meminimalisasi biaya penyimpanan cloud.
*   **Database Isolasi**: RDS dipasang di subnet privat agar tidak bisa diakses langsung dari internet publik. Kita juga membuat skema otomatis lewat Lambda agar database siap digunakan saat deployment selesai.

---

### 2.2. Langkah Praktik Konsol AWS (Klik-demi-Klik)

#### A. Membuat S3 Bucket & Pengaturan CORS
1.  Ketik **S3** di search bar atas konsol AWS, klik layanan **S3**.
2.  Klik **Create bucket**.
    *   **Bucket name**: `lks-orders-namamu-2026` *(Ingat: ganti 'namamu' dengan nama lengkapmu)*.
    *   **Bucket Versioning**: Pilih **Enable**.
    *   Klik **Create bucket** di paling bawah.
3.  Masuk ke bucket yang baru dibuat, klik tab **Management** -> klik **Create lifecycle rule**.
    *   **Name**: `lks-orders-lifecycle`
    *   **Choose a rule scope**: Pilih **Apply to all objects in the bucket**, centang persetujuan di bawahnya.
    *   **Lifecycle rule actions**: Centang opsi **`Transition current versions of objects between storage classes`** dan **`Expire current versions of objects`**.
    *   **Transitions**:
        *   Pilih **Intelligent-Tiering**, isi Days = `30`.
        *   Klik **Add transition**, pilih **S3 Glacier Flexible Retrieval**, isi Days = `90`.
    *   **Expire current versions**: Isi Days = `365`.
    *   > ⚠️ **PERINGATAN EROR AWS (PENTING):**
        > Jangan mengisi jumlah hari (Days) untuk **Glacier** sama dengan atau kurang dari **Intelligent-Tiering** (misalnya sama-sama diisi `30`).
        > AWS akan menolak konfigurasi ini dan menampilkan **Unknown Error / API Response Error**. Jumlah hari untuk **Glacier** harus selalu **lebih besar** daripada **Intelligent-Tiering** (sesuai panduan: Intelligent-Tiering = `30`, Glacier = `90`).
    *   Klik **Create rule**.
4.  Klik tab **Permissions** di bagian atas bucket.
    *   Di bagian **Block public access (bucket settings)**, klik **Edit**.
    *   **Hapus centang** pada *Block all public access*, lalu klik **Save changes**. Ketik `confirm` pada dialog konfirmasi yang muncul, lalu klik **Confirm**.
    *   Scroll ke bawah ke bagian *Cross-origin resource sharing (CORS)*, klik **Edit**. Tempelkan JSON berikut:
```json
[
    {"AllowedHeaders": ["*"], "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"], "AllowedOrigins": ["*"], "ExposeHeaders": []}
]
```
    *   Klik **Save changes**.

#### B. Membuat Database RDS PostgreSQL
1.  Ketik **RDS** di search bar atas konsol, klik layanan **RDS**.
2.  Di menu navigasi kiri, klik **Subnet groups** -> **Create DB subnet group**.
    *   **Name**: `lks-db-subnet-group`, **Description**: `Subnet Group for LKS database`
    *   **VPC**: Pilih **lks-vpc-serverless**.
    *   **Add subnets**: Pilih Availability Zones `us-east-1a` dan `us-east-1b` (atau zona yang sesuai di region Anda). Centang kedua Subnet Privat kita (cek CIDR `20.1.11.0/25` dan `20.1.12.0/25`).
    *   Klik **Create**.
3.  Di menu kiri, klik **Databases** -> klik **Create database**.
    *   **Creation method**: Pilih **Standard create** (atau **Full configuration**).
    *   **Engine**: **PostgreSQL**.
    *   **Engine version**: Pilih **PostgreSQL 15.x** (atau versi `16.x` / `17.x` yang tersedia).
    *   **Templates**: Pilih **Free Tier**.
    *   > 🛠️ **SOLUSI PENTING BILA DROPDOWN SERVER (INSTANCE TYPE) KOSONG / MERAH:**
        > Akun AWS Academy melarang instans berbasis chip Graviton (`t4g`) yang dijadikan default oleh PostgreSQL AWS Console terbaru, dan hanya mengizinkan instans Intel standar seperti `db.t3.micro` atau `db.t2.micro`.
        >
        > **Langkah Menampilkan Server:**
        > 1. Pada bagian **Instance configuration**, geser sakelar **`Include previous generation classes`** ke kanan hingga **aktif (berwarna biru)**.
        > 2. Di bawahnya, pilih opsi **`Burstable classes`**.
        > 3. Klik dropdown **`Instance type`** yang tadinya kosong/eror. Instans **`db.t3.micro`** atau **`db.t2.micro`** kini dijamin akan muncul dan silakan Anda pilih.
    *   **Storage type**: `gp3`, **Allocated storage**: `20` GB.
    *   **Settings**:
        *   **DB instance identifier**: `lks-rds-orders`
        *   **Master username**: `dbadmin`, **Master password**: `TechnoCloud2026!`
    *   **Connectivity**:
        *   **VPC**: Pilih **lks-vpc-serverless**.
        *   **DB subnet group**: Pilih **lks-db-subnet-group**.
        *   **Public access**: Pilih **No** (sangat penting untuk keamanan database!).
        *   **VPC security group (firewall)**: Pilih **Choose existing**, pilih **`lks-sg-rds`**, hapus grup `default` dengan klik `x`.
    *   **Additional configuration** (Gulir ke paling bawah, buka dropdown):
        *   **Initial database name**: `ordersdb`
        *   Centang *Enable automated backups*, atur **Backup retention period = 7 days**.
    *   Klik **Create database**. Pembuatan memakan waktu sekitar 5-10 menit. Status akan berubah dari `creating` menjadi `available`.

---

### 2.3. Langkah Praktik & Kode Inisialisasi Database via Lambda (`lks-lambda-init-db`)

Untuk membuat tabel-tabel database RDS PostgreSQL secara otomatis dan aman (tanpa perlu membuka koneksi database ke publik), kita akan membuat satu fungsi Lambda pembantu bernama `lks-lambda-init-db`. Lambda ini akan masuk ke dalam VPC privat, menghubungkan ke RDS, lalu mengeksekusi skema database.

#### A. Persiapan: Membuat Lambda Layer (Dependencies ZIP)
Sebelum membuat fungsi Lambda, kita harus menyiapkan dan mengunggah library Python (seperti `psycopg2` dan `pandas`) sebagai Lambda Layer.
1.  **Buat ZIP Dependencies di Terminal/CMD lokal Anda:**
```bash
mkdir python
pip install psycopg2-binary pandas -t python/
zip -r lks-layer-dependencies.zip python/
```
> ⚠️ **PENTING — KESELARASAN VERSI PYTHON:**
> Library biner (seperti `psycopg2` dan `pandas`) dikompilasi sesuai versi Python lokal Anda. 
> Jika Python lokal Anda adalah **Python 3.12** (bisa dicek dengan `python3 --version`), maka berkas `.so` yang dihasilkan di dalam folder `python/psycopg2/` akan bernama `_psycopg.cpython-312-x86_64-linux-gnu.so`.
> 
> Jika Anda mengunggah zip tersebut tetapi memilih Runtime **Python 3.11** di Lambda, Anda akan mendapat error:
> `Runtime.ImportModuleError: No module named 'psycopg2._psycopg'`
> 
> **Solusi:** Ubah Runtime Lambda di AWS Console (baik saat membuat Layer maupun Fungsi Lambda) menjadi **Python 3.12** agar sesuai dengan versi biner yang diunggah.
>
2.  Buka layanan **Lambda** di AWS Console.
3.  Di menu navigasi kiri, klik **Layers** -> klik **Create layer**.
    *   **Name**: `lks-layer-dependencies`
    *   **Upload method**: Pilih **Upload a .zip file** dan unggah berkas `lks-layer-dependencies.zip` yang baru dibuat.
    *   **Compatible runtimes**: Pilih **Python 3.11** atau **Python 3.12** (sesuaikan dengan versi Python lokal Anda).
    *   Klik **Create**.

#### B. Langkah Pembuatan Fungsi Lambda (`lks-lambda-init-db`) di Konsol AWS
1.  Buka layanan **Lambda** di konsol AWS, klik **Create function** (Pilih *Author from scratch*).
    *   **Function name**: `lks-lambda-init-db`
    *   **Runtime**: **Python 3.11**
    *   **Permissions**: Pilih **Use an existing role** -> Pilih **`LabRole`** (role lab dengan akses VPC luas).
2.  **Advanced settings** (Penting):
    *   Centang **Enable VPC**.
    *   **VPC**: Pilih **lks-vpc-serverless**.
    *   **Subnets**: Centang **kedua Subnet Privat** kita (`lks-subnet-private1` & `lks-subnet-private2`).
    *   **Security groups**: Pilih **`lks-sg-lambda`**.
    *   Klik **Create function**.
3.  **Hubungkan Layer Dependencies:**
    *   Gulir ke bagian paling bawah halaman fungsi ke bagian **Layers** -> klik **Add a layer**.
    *   Pilih **Custom layers**, pilih **`lks-layer-dependencies`**, lalu klik **Add**.
4.  **Konfigurasi Tambahan (General & Environment):**
    *   Klik tab **Configuration** di bagian atas halaman fungsi Lambda Anda:
        *   Klik menu kiri **General configuration** -> klik **Edit**. Atur **Timeout = 5 minutes (300 seconds)** (inisialisasi membutuhkan waktu tunggu koneksi yang cukup lama). Klik **Save**.
        *   Klik menu kiri **Environment variables** -> klik **Edit**. Tambahkan 4 variabel lingkungan berikut:
            *   `DB_HOST` = *(Salin Endpoint database dari menu RDS Databases)*
            *   `DB_NAME` = `ordersdb`
            *   `DB_USER` = `dbadmin`
            *   `DB_PASSWORD` = `TechnoCloud2026!`
            *   Klik **Save**.

#### C. Kode Sumber Python Inisialisasi Database (`lks-lambda-init-db`)
Tempelkan kode Python berikut pada tab **Code** di editor Lambda Anda, lalu klik **Deploy** di pojok kanan atas:

```python
import json
import os
import psycopg2

DB_HOST = os.environ.get('DB_HOST')
DB_NAME = os.environ.get('DB_NAME')
DB_USER = os.environ.get('DB_USER')
DB_PASSWORD = os.environ.get('DB_PASSWORD')

def lambda_handler(event, context):
    print("Mulai proses inisialisasi skema database LKS...")
    
    # 1. Kueri DDL Pembuatan Tabel
    sql_schema = """
    CREATE TABLE IF NOT EXISTS customers (
        customer_id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL
    );

    CREATE TABLE IF NOT EXISTS orders (
        order_id SERIAL PRIMARY KEY,
        customer_id INT REFERENCES customers(customer_id),
        status VARCHAR(50) DEFAULT 'PENDING',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE TABLE IF NOT EXISTS order_items (
        item_id SERIAL PRIMARY KEY,
        order_id INT REFERENCES orders(order_id),
        product_id INT,
        quantity INT NOT NULL,
        price NUMERIC(10, 2) NOT NULL
    );

    CREATE TABLE IF NOT EXISTS inventory (
        product_id SERIAL PRIMARY KEY,
        product_name VARCHAR(100) NOT NULL,
        stock INT NOT NULL
    );
    """
    
    # 2. Kueri DML Data Contoh Awal (Seeding)
    sql_seed = """
    INSERT INTO inventory (product_name, stock) 
    VALUES 
        ('Laptop Asus', 50), 
        ('Mouse Wireless', 200), 
        ('Keyboard Mechanical', 100)
    ON CONFLICT DO NOTHING;

    INSERT INTO customers (name, email) 
    VALUES 
        ('Abdul Kholik', 'abdul.kholik@example.com'),
        ('Ricky Fahmi', 'ricky.fahmi@example.com')
    ON CONFLICT (email) DO NOTHING;
    """
    
    try:
        # Menghubungkan ke RDS PostgreSQL
        print(f"Menghubungkan ke database host: {DB_HOST}...")
        conn = psycopg2.connect(
            host=DB_HOST,
            database=DB_NAME,
            user=DB_USER,
            password=DB_PASSWORD
        )
        conn.autocommit = True
        cursor = conn.cursor()
        
        # Eksekusi pembuatan tabel
        print("Mengeksekusi kueri pembuatan tabel (DDL)...")
        cursor.execute(sql_schema)
        
        # Eksekusi pemasukan data awal
        print("Memasukkan data awal (Seeding)...")
        cursor.execute(sql_seed)
        
        cursor.close()
        conn.close()
        print("Inisialisasi skema database RDS sukses!")
        
        return {
            "statusCode": 200,
            "body": json.dumps({"message": "Database RDS PostgreSQL initialized successfully with schema and seed data!"})
        }
        
    except Exception as e:
        print(f"Error saat inisialisasi database: {str(e)}")
        return {
            "statusCode": 500,
            "body": json.dumps({"message": "Database initialization failed", "error": str(e)})
        }
```

#### D. Langkah Eksekusi (Memicu Database Creation)
1.  Setelah kode di-deploy, klik tombol **Test** (berwarna abu-abu, di samping tombol Deploy).
2.  Di pop-up *Configure test event*:
    *   **Event name**: `test`
    *   Biarkan payload JSON apa adanya (default).
    *   Klik **Save**.
3.  Klik kembali tombol **Test** (sekarang berwarna oranye).
4.  Perhatikan kotak hasil eksekusi yang muncul di layar:
    *   Status harus berupa **`Succeeded`** (Sukses).
    *   Hasil log respon menampilkan **`statusCode: 200`** dan pesan sukses.
    *   Selamat! Tabel database PostgreSQL Anda kini telah resmi terbuat di dalam RDS dan siap menerima data order.

---

## Modul 3: Logika Bisnis & Dependency (Lambda & Layers)

Di modul ini, kita menyusun 6 fungsi Lambda dengan Python 3.11 dan menyatukan library pendukung agar tidak terjadi duplikasi package.

### 3.1. Konsep Shared Layer & VPC Lambda
*   **Lambda Layer**: Menggabungkan package eksternal seperti `psycopg2` (konektor PostgreSQL) dan `pandas` (generator Excel) dalam satu ZIP.
*   **VPC Integration**: Lambda backend dipasang di subnet privat dan menggunakan security group `lks-sg-lambda` agar bisa terhubung ke database RDS PostgreSQL.

---

### 3.2. Langkah Praktik Konsol AWS (Klik-demi-Klik)

#### A. Membuat Fungsi Lambda Utama (`lks-lambda-order-management`)
1.  Di menu kiri Lambda, klik **Functions** -> klik **Create function** (Pilih *Author from scratch*).
    *   **Function name**: `lks-lambda-order-management`
    *   **Runtime**: **Python 3.11**.
    *   **Permissions** -> Buka *Change default execution role*: Pilih **Use an existing role** -> Pilih **`LabRole`** (role lab standar dengan akses luas).
2.  **Advanced settings** (Penting):
    *   Centang **Enable VPC**.
    *   **VPC**: Pilih **lks-vpc-serverless**.
    *   **Subnets**: Pilih kedua **Subnet Privat** kita.
    *   **Security groups**: Pilih **`lks-sg-lambda`**.
    *   Klik **Create function**.
3.  Di tab **Code** bagian bawah, tempelkan kode Python penanganan pesanan (lihat sub-bab 3.3 di bawah). Klik **Deploy**.
4.  Gulir ke paling bawah halaman fungsi ke bagian **Layers** -> klik **Add a layer**.
    *   Pilih **Custom layers**, pilih **`lks-layer-dependencies`**, klik **Add**.
5.  Klik tab **Configuration** di bagian atas halaman fungsi Lambda:
    *   Klik menu kiri **General configuration** -> klik **Edit**. Atur **Memory = 512 MB** dan **Timeout = 30 seconds**. Klik **Save**.
    *   Klik menu kiri **Environment variables** -> klik **Edit**. Tambahkan variabel berikut:
        *   `DB_HOST` = *(Salin Endpoint database dari menu RDS)*
        *   `DB_NAME` = `ordersdb`, `DB_USER` = `dbadmin`, `DB_PASSWORD` = `TechnoCloud2026!`
        *   `STATE_MACHINE_ARN` = *(Salin ARN Step Functions dari Modul 5)*
    *   Klik **Save**.

---

### 3.3. Kode Sumber Python (`lks-lambda-order-management` - PostgreSQL)
```python
import json
import os
import boto3
import psycopg2

DB_HOST = os.environ.get('DB_HOST')
DB_NAME = os.environ.get('DB_NAME')
DB_USER = os.environ.get('DB_USER')
DB_PASSWORD = os.environ.get('DB_PASSWORD')
SFN_ARN = os.environ.get('STATE_MACHINE_ARN')

sfn_client = boto3.client('stepfunctions')

def lambda_handler(event, context):
    path = event.get('path')
    http_method = event.get('httpMethod')
    
    if path == '/orders' and http_method == 'POST':
        body = json.loads(event.get('body', '{}'))
        customer_id = body.get('customer_id')
        items = body.get('items')
        
        # 1. Simpan pesanan ke RDS PostgreSQL
        conn = psycopg2.connect(host=DB_HOST, database=DB_NAME, user=DB_USER, password=DB_PASSWORD)
        cursor = conn.cursor()
        
        cursor.execute("INSERT INTO orders (customer_id, status) VALUES (%s, 'PENDING') RETURNING order_id;", (customer_id,))
        order_id = cursor.fetchone()[0]
        
        for item in items:
            cursor.execute(
                "INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (%s, %s, %s, %s);",
                (order_id, item['product_id'], item['quantity'], item['price'])
            )
        conn.commit()
        cursor.close()
        conn.close()
        
        # 2. Trigger Step Functions secara asinkron
        payload = {
            "order_id": order_id,
            "customer_id": customer_id,
            "items": items
        }
        
        response = sfn_client.start_execution(
            stateMachineArn=SFN_ARN,
            name=f"Order-{order_id}",
            input=json.dumps(payload)
        )
        
        return {
            "statusCode": 201,
            "headers": {"Content-Type": "application/json", "Access-Control-Allow-Origin": "*"},
            "body": json.dumps({"message": "Order created successfully", "order_id": order_id, "execution_arn": response['executionArn']})
        }
```

---

### 3.4. Langkah Praktik & Kode Fungsi Lambda Pendukung (Simulasi & Integrasi)

Selain fungsi utama di atas, Anda juga harus membuat 3 fungsi Lambda pendukung yang dipanggil oleh alur kerja Step Functions (`ValidateOrder` -> `ProcessPayment` -> `UpdateInventory` -> `SendConfirmation`).

#### A. Lambda 1: Pemrosesan Pembayaran (`lks-lambda-process-payment`)
Fungsi ini mensimulasikan pemrosesan pembayaran dan menghitung total harga dari pesanan.
1. Buat fungsi Lambda baru di AWS Console:
   * **Function name**: `lks-lambda-process-payment`
   * **Runtime**: **Python 3.12**
   * **Permissions**: Pilih **Use an existing role** -> Pilih **`LabRole`**
   * **Advanced settings**: Tidak perlu mencentang Enable VPC (karena fungsi ini hanya pemrosesan logis, tidak membutuhkan koneksi database).
   * Klik **Create function**.
2. Di bagian bawah, klik **Add a layer**, pilih **Custom layers** -> **`lks-layer-dependencies`**, lalu klik **Add**.
3. Tempelkan kode program berikut di tab **Code**, klik **Deploy**:

```python
import json

def lambda_handler(event, context):
    print("Menerima event untuk pemrosesan pembayaran:", json.dumps(event))
    
    order_id = event.get("order_id")
    items = event.get("items", [])
    
    # Hitung total harga
    total_amount = sum(float(item.get("price", 0)) * int(item.get("quantity", 0)) for item in items)
    print(f"Total pembayaran untuk Order #{order_id}: Rp {total_amount}")
    
    # Simulasi status pembayaran (Selalu SUCCESS jika total > 0)
    payment_status = "SUCCESS" if total_amount > 0 else "FAILED"
    
    # Masukkan status pembayaran ke event
    event["totalAmount"] = total_amount
    event["paymentStatus"] = payment_status
    
    return event
```

#### B. Lambda 2: Pembaruan Inventaris (`lks-lambda-update-inventory`)
Fungsi ini melakukan pengecekan stok di RDS PostgreSQL secara atomik. Jika stok memadai, stok akan dikurangi dan status order di-update menjadi `PAID`. Jika stok kurang, transaksi dibatalkan (rollback) dan status order diset menjadi `FAILED`.
1. Buat fungsi Lambda baru di AWS Console:
   * **Function name**: `lks-lambda-update-inventory`
   * **Runtime**: **Python 3.12**
   * **Permissions**: Pilih **Use an existing role** -> Pilih **`LabRole`**
2. **Advanced settings** (Wajib masuk VPC karena mengakses database RDS):
   * Centang **Enable VPC**.
   * **VPC**: Pilih **lks-vpc-serverless**.
   * **Subnets**: Centang **kedua Subnet Privat**.
   * **Security groups**: Pilih **`lks-sg-lambda`**.
   * Klik **Create function**.
3. Hubungkan **`lks-layer-dependencies`** sebagai Layer pendukung.
4. Klik tab **Configuration** -> **Environment variables**, tambahkan:
   * `DB_HOST` = *(Endpoint RDS Anda)*
   * `DB_NAME` = `ordersdb`
   * `DB_USER` = `dbadmin`
   * `DB_PASSWORD` = `TechnoCloud2026!`
   * Di bagian **General configuration**, ubah **Timeout menjadi 45 detik**.
5. Tempelkan kode program berikut di tab **Code**, klik **Deploy**:

```python
import os
import json
import psycopg2

DB_HOST = os.environ.get('DB_HOST')
DB_NAME = os.environ.get('DB_NAME')
DB_USER = os.environ.get('DB_USER')
DB_PASSWORD = os.environ.get('DB_PASSWORD')

def lambda_handler(event, context):
    print("Menerima event untuk update inventaris:", json.dumps(event))
    
    order_id = event.get("order_id")
    items = event.get("items", [])
    
    conn = None
    try:
        conn = psycopg2.connect(
            host=DB_HOST,
            database=DB_NAME,
            user=DB_USER,
            password=DB_PASSWORD
        )
        cursor = conn.cursor()
        
        # Mulai transaksi database secara manual
        conn.autocommit = False
        
        # 1. Cek stok untuk semua item terlebih dahulu
        for item in items:
            product_id = item.get("product_id")
            quantity = item.get("quantity")
            
            cursor.execute("SELECT stock, product_name FROM inventory WHERE product_id = %s FOR UPDATE;", (product_id,))
            row = cursor.fetchone()
            if not row:
                raise Exception(f"Produk ID {product_id} tidak ditemukan di inventaris.")
            
            current_stock, product_name = row[0], row[1]
            if current_stock < quantity:
                raise Exception(f"Stok produk '{product_name}' (ID {product_id}) tidak cukup. Dibutuhkan: {quantity}, Tersedia: {current_stock}.")
        
        # 2. Kurangi stok jika semua item lolos pengecekan
        for item in items:
            product_id = item.get("product_id")
            quantity = item.get("quantity")
            cursor.execute(
                "UPDATE inventory SET stock = stock - %s WHERE product_id = %s;",
                (quantity, product_id)
            )
            
        # 3. Update status order menjadi PAID
        cursor.execute(
            "UPDATE orders SET status = 'PAID' WHERE order_id = %s;",
            (order_id,)
        )
        
        # Commit seluruh transaksi
        conn.commit()
        cursor.close()
        
        event["inventoryStatus"] = "SUCCESS"
        print(f"Update inventaris sukses untuk Order #{order_id}.")
        
    except Exception as e:
        print(f"Gagal mengupdate inventaris: {str(e)}")
        if conn:
            conn.rollback()
            try:
                # Update status order menjadi FAILED jika gagal stok
                cursor = conn.cursor()
                cursor.execute(
                    "UPDATE orders SET status = 'FAILED' WHERE order_id = %s;",
                    (order_id,)
                )
                conn.commit()
                cursor.close()
            except Exception as db_err:
                print(f"Gagal mengupdate status order ke FAILED: {str(db_err)}")
                
        event["inventoryStatus"] = "FAILED"
        event["errorMessage"] = str(e)
        
    finally:
        if conn:
            conn.close()
            
    return event
```

#### C. Lambda 3: Kirim Notifikasi SNS (`lks-lambda-send-notification`)
Fungsi ini mempublikasikan detail pemrosesan pesanan (sukses maupun gagal) ke Amazon SNS Topic agar admin menerima pemberitahuan via email secara instan.
1. Buat fungsi Lambda baru di AWS Console:
   * **Function name**: `lks-lambda-send-notification`
   * **Runtime**: **Python 3.12**
   * **Permissions**: Pilih **Use an existing role** -> Pilih **`LabRole`**
   * **Advanced settings**: Tidak perlu mencentang Enable VPC (karena VPC Endpoint SNS sudah di-setup agar terhubung lewat jaringan internal AWS, namun jika ada kendala koneksi, melampirkan VPC `lks-vpc-serverless` dan Subnet Privat juga sangat aman).
   * Klik **Create function**.
2. Klik tab **Configuration** -> **Environment variables**, tambahkan:
   * `SNS_TOPIC_ARN` = *(Salin ARN dari Amazon SNS Topic `lks-sns-order-notifications` yang telah Anda buat di Modul 4)*
   * Di bagian **General configuration**, ubah **Timeout menjadi 60 detik**.
3. Tempelkan kode program berikut di tab **Code**, klik **Deploy**:

```python
import os
import json
import boto3

SNS_TOPIC_ARN = os.environ.get('SNS_TOPIC_ARN')
sns_client = boto3.client('sns')

def lambda_handler(event, context):
    print("Menerima event untuk pengiriman notifikasi:", json.dumps(event))
    
    order_id = event.get("order_id")
    payment_status = event.get("paymentStatus")
    inventory_status = event.get("inventoryStatus")
    err_message = event.get("errorMessage", "Unknown error")
    
    subject = f"Notifikasi Status Order #{order_id}"
    
    if payment_status == "SUCCESS" and inventory_status == "SUCCESS":
        message = (
            f"Halo Admin,\n\n"
            f"Pesanan Baru Berhasil Diproses!\n"
            f"---------------------------------\n"
            f"Order ID       : {order_id}\n"
            f"Customer ID    : {event.get('customer_id')}\n"
            f"Total Bayar    : Rp {event.get('totalAmount'):,.2f}\n"
            f"Status Order   : PAID (Pembayaran Sukses & Stok Dikurangi)\n\n"
            f"Sistem akan segera memproses pengiriman."
        )
    elif payment_status == "FAILED":
        message = (
            f"Halo Admin,\n\n"
            f"PERINGATAN: Transaksi Pembayaran Gagal!\n"
            f"---------------------------------\n"
            f"Order ID       : {order_id}\n"
            f"Customer ID    : {event.get('customer_id')}\n"
            f"Status Order   : FAILED\n\n"
            f"Detail Masalah : Proses pembayaran ditolak."
        )
    else:
        message = (
            f"Halo Admin,\n\n"
            f"PERINGATAN: Pembaruan Stok Gagal (Stok Kosong/Kurang)!\n"
            f"---------------------------------\n"
            f"Order ID       : {order_id}\n"
            f"Customer ID    : {event.get('customer_id')}\n"
            f"Status Order   : FAILED\n\n"
            f"Detail Masalah : {err_message}"
        )
        
    try:
        response = sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=message,
            Subject=subject
        )
        print(f"Notifikasi berhasil dikirim. MessageId: {response['MessageId']}")
        return {
            "statusCode": 200,
            "body": json.dumps({"message": "Notification sent successfully", "messageId": response['MessageId']})
        }
    except Exception as e:
        print(f"Gagal mengirim notifikasi SNS: {str(e)}")
        return {
            "statusCode": 500,
            "body": json.dumps({"message": "Failed to send notification", "error": str(e)})
        }
```

#### D. Lambda 4: Pembuatan Laporan Harian (`lks-lambda-generate-report`)
Fungsi ini dipicu secara berkala oleh EventBridge Scheduler pada pukul 23:59 UTC untuk menarik data pesanan harian dari RDS PostgreSQL, menyusunnya dalam format Excel (.xlsx), mengunggah berkas tersebut ke S3 Bucket, dan mengembalikan tautan unduh (pre-signed URL).
1. Buat fungsi Lambda baru di AWS Console:
   * **Function name**: `lks-lambda-generate-report`
   * **Runtime**: **Python 3.12**
   * **Permissions**: Pilih **Use an existing role** -> Pilih **`LabRole`**
2. **Advanced settings** (Wajib masuk VPC agar dapat query data dari RDS PostgreSQL):
   * Centang **Enable VPC**.
   * **VPC**: Pilih **lks-vpc-serverless**.
   * **Subnets**: Centang **kedua Subnet Privat**.
   * **Security groups**: Pilih **`lks-sg-lambda`**.
   * Klik **Create function**.
3. Hubungkan **`lks-layer-dependencies`** sebagai Layer pendukung.
4. Klik tab **Configuration** -> **Environment variables**, tambahkan:
   * `DB_HOST` = *(Endpoint RDS Anda)*
   * `DB_NAME` = `ordersdb`
   * `DB_USER` = `dbadmin`
   * `DB_PASSWORD` = `TechnoCloud2026!`
   * `S3_BUCKET` = *(Nama S3 bucket Anda, misal: `lks-orders-namamu-2026`)*
   * Di bagian **General configuration**, ubah **Memory menjadi 1024 MB** dan **Timeout menjadi 90 detik**.
5. Tempelkan kode program berikut di tab **Code**, klik **Deploy**:

```python
import os
import json
import datetime
import pandas as pd
import psycopg2
import boto3

DB_HOST = os.environ.get('DB_HOST')
DB_NAME = os.environ.get('DB_NAME')
DB_USER = os.environ.get('DB_USER')
DB_PASSWORD = os.environ.get('DB_PASSWORD')
S3_BUCKET = os.environ.get('S3_BUCKET')

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    print("Memulai pembuatan laporan harian...")
    
    # 1. Tarik data dari RDS PostgreSQL
    try:
        conn = psycopg2.connect(
            host=DB_HOST,
            database=DB_NAME,
            user=DB_USER,
            password=DB_PASSWORD
        )
        
        query = """
            SELECT 
                o.order_id, 
                c.name as customer_name, 
                c.email as customer_email, 
                o.status, 
                o.created_at,
                COALESCE(SUM(oi.quantity * oi.price), 0) as total_price
            FROM orders o
            JOIN customers c ON o.customer_id = c.customer_id
            LEFT JOIN order_items oi ON o.order_id = oi.order_id
            GROUP BY o.order_id, c.name, c.email, o.status, o.created_at
            ORDER BY o.created_at DESC;
        """
        
        df = pd.read_sql(query, conn)
        conn.close()
        
    except Exception as e:
        print(f"Gagal mengambil data dari database: {str(e)}")
        return {
            "statusCode": 500,
            "body": json.dumps({"message": "Failed to fetch data", "error": str(e)})
        }
        
    # 2. Buat file Excel menggunakan Pandas
    today_str = datetime.date.today().strftime('%Y-%m-%d')
    file_name = f"laporan-harian-{today_str}.xlsx"
    local_path = f"/tmp/{file_name}"
    
    try:
        with pd.ExcelWriter(local_path, engine='openpyxl') as writer:
            df.to_excel(writer, sheet_name='Daily Orders', index=False)
        print(f"File Excel berhasil dibuat secara lokal di {local_path}.")
        
    except Exception as e:
        print(f"Gagal menulis file Excel: {str(e)}")
        return {
            "statusCode": 500,
            "body": json.dumps({"message": "Failed to generate Excel", "error": str(e)})
        }
        
    # 3. Unggah ke S3 Bucket
    s3_key = f"reports/{file_name}"
    try:
        print(f"Mengunggah ke S3: {S3_BUCKET}/{s3_key}...")
        s3_client.upload_file(local_path, S3_BUCKET, s3_key)
        
        url = s3_client.generate_presigned_url(
            'get_object',
            Params={'Bucket': S3_BUCKET, 'Key': s3_key},
            ExpiresIn=86400  # Link aktif 24 jam
        )
        
        print(f"Unggahan sukses! Pre-signed URL: {url}")
        return {
            "statusCode": 200,
            "body": json.dumps({
                "message": "Daily report generated and uploaded successfully!",
                "s3_bucket": S3_BUCKET,
                "s3_key": s3_key,
                "download_url": url
            })
        }
        
    except Exception as e:
        print(f"Gagal mengunggah file ke S3: {str(e)}")
        return {
            "statusCode": 500,
            "body": json.dumps({"message": "Failed to upload to S3", "error": str(e)})
        }
```

> ✅ **Pada titik ini, seluruh 5 fungsi Lambda sudah selesai dibuat dan dideploy:**
> 1. `lks-lambda-init-db` (inisialisasi database)
> 2. `lks-lambda-order-management` (handler pesanan utama)
> 3. `lks-lambda-process-payment` (simulasi pembayaran)
> 4. `lks-lambda-update-inventory` (update stok)
> 5. `lks-lambda-send-notification` (notifikasi SNS)
> 6. `lks-lambda-generate-report` (laporan harian)
>
> Sebelum melanjutkan ke Modul 5, **salin ARN masing-masing fungsi** dari halaman **Lambda -> Functions** -> klik nama fungsi -> salin nilai **Function ARN** di pojok kanan atas. Anda membutuhkan ARN ke-4 fungsi berikut untuk dikonfigurasi di Step Functions: `order-management`, `process-payment`, `update-inventory`, `send-notification`.

---

## Modul 4: API & Notifikasi (API Gateway & SNS)

Di modul ini, kita membuat RESTful API Gateway sebagai gerbang masuk publik dan Amazon SNS untuk mempublikasikan notifikasi status.

### 4.1. Konsep Rate-Limiting & SNS Topics
*   **API Gateway REST API**: Menggunakan otentikasi **API Key** (`lks-api-key`) di header `x-api-key`.
*   **Usage Plan**: Mencegah serangan spam dengan **Rate Limit 1.000 req/sec** dan **monthly quota 100.000 req**.
*   **SNS Topic**: Menyebarkan pesan notifikasi ke email admin secara instan begitu terjadi perubahan status transaksi.

---

### 4.2. Langkah Praktik Konsol AWS (Klik-demi-Klik)

#### A. Membuat Amazon SNS Topic & Subscription
1.  Cari **SNS** di pencarian atas konsol, klik layanan **Simple Notification Service**.
2.  Di menu kiri, klik **Topics** -> **Create topic**.
    *   **Type**: **Standard**, **Name**: `lks-sns-order-notifications`
    *   Klik **Create topic**.
3.  Di dalam detail topik yang terbuat, gulir ke bawah ke bagian *Subscriptions*, klik **Create subscription**.
    *   **Protocol**: **Email**, **Endpoint**: Masukkan `handi@seamolec.org`.
    *   Klik **Create subscription**.
4.  *Sangat Penting:* Buka email tersebut, cari email konfirmasi dari AWS, lalu klik tautan **Confirm Subscription**.

#### B. Setup REST API di API Gateway
1.  Cari **API Gateway** di pencarian atas konsol, klik layanan **API Gateway**.
2.  Di bagian **REST API** (bukan HTTP API), klik **Build**.
    *   Pilih **New API**, **API name**: `lks-api-orders`, **Endpoint Type**: **Regional**.
    *   Klik **Create API**.
3.  **Membuat Resource & Method**:
    *   Klik dropdown **Actions** -> klik **Create Resource**.
        *   **Resource Name**: `orders`, centang **Enable API Gateway CORS**, klik **Create Resource**.
    *   Sorot `/orders`, klik **Actions** -> **Create Method**.
        *   Pilih **POST** pada dropdown, klik tanda centang.
        *   **Integration type**: **Lambda Function**.
        *   Centang **Use Lambda Proxy integration**.
        *   **Lambda Function**: Ketik dan pilih `lks-lambda-order-management`.
        *   Klik **Save**.
4.  **Deploy API**:
    *   Klik nama root API Anda, klik **Actions** -> **Deploy API**.
        *   **Deployment stage**: **[New Stage]**, **Stage name**: `production`.
        *   Klik **Deploy**. Salin **Invoke URL** yang muncul di atas.

#### C. Membuat API Key & Usage Plan
1.  Di menu kiri API Gateway, klik **API Keys** -> **Actions** -> **Create API Key**.
    *   **Name**: `lks-api-key`, pilih **Auto Generate**, klik **Save**. Salin string kunci rahasianya.
2.  Di menu kiri, klik **Usage Plans** -> klik **Create**.
    *   **Name**: `lks-usage-plan`.
    *   Centang **Enable Throttling** (Rate = `1000`, Burst = `2000`).
    *   Centang **Enable Quota** (Quota = `100000` per Month).
    *   Klik **Next**.
3.  **Add API Stages**: Pilih API `lks-api-orders` and Stage `production`. Klik tombol centang, lalu klik **Next**.
4.  **Add API Key**: Masukkan nama key `lks-api-key` yang telah kita buat, tambahkan ke usage plan, lalu klik **Done**.

---

## Modul 5: Orkestrasi Alur Kerja (AWS Step Functions)

AWS Step Functions berfungsi merakit logika pemrosesan pesanan dalam diagram alir visual deklaratif.

### 5.1. Alur Transisi State
Start -> `ValidateOrder` (Lambda) -> `ProcessPayment` (Lambda) -> `PaymentChoice` (Branching):
*   *Success* -> `UpdateInventory` (Lambda) -> `InventoryChoice` -> *Success* -> `SendConfirmation` -> End.
*   *Failed* -> Kirim Notifikasi Gagal -> `OrderFailed` (End state dengan kegagalan).

---

### 5.2. Langkah Praktik Konsol AWS (Klik-demi-Klik)
1.  Cari **Step Functions** di pencarian atas konsol, klik layanan **Step Functions**.
2.  Di menu kiri, klik **State machines** -> klik **Create state machine**.
3.  Di halaman pembuatan, pilih **Write your workflow in code**.
    *   **Type**: Pilih **Standard**.
4.  Di bagian editor **Definition**, hapus seluruh kode JSON yang ada, lalu tempelkan kode definisi **ASL (Amazon States Language)** JSON dari sub-bab 5.3 di bawah.
5.  *Perhatian:* Ganti setiap bagian `"Resource": "arn:aws:lambda:us-east-1:123456789012:function:..."` dengan ARN Lambda fungsi Anda yang sebenarnya. ARN Lambda bisa disalin dari halaman detail masing-masing fungsi Lambda di menu **Lambda -> Functions**.
6.  Klik **Next** di pojok kanan atas.
7.  Pada halaman **Specify details**:
    *   **State machine name**: `lks-stepfunctions-order-workflow`.
    *   **Permissions**: Pilih **Choose an existing role** -> Pilih **`LabRole`** dari daftar.
    *   Klik **Create state machine** di pojok kanan bawah.
8.  Setelah dibuat, salin nilai **ARN** dari halaman detail State Machine ini (tersedia di bagian atas halaman). ARN ini akan kita masukkan ke Environment Variable `STATE_MACHINE_ARN` pada Lambda di Modul 3.

---

### 5.3. Kode Definisi ASL JSON (Amazon States Language)
```json
{
  "Comment": "Alur Kerja Pemrosesan Pesanan LKS",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:lks-lambda-order-management",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:lks-lambda-process-payment",
      "Next": "PaymentChoice"
    },
    "PaymentChoice": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.paymentStatus",
          "StringEquals": "SUCCESS",
          "Next": "UpdateInventory"
        }
      ],
      "Default": "PaymentFailed"
    },
    "UpdateInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:lks-lambda-update-inventory",
      "Next": "InventoryChoice"
    },
    "InventoryChoice": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.inventoryStatus",
          "StringEquals": "SUCCESS",
          "Next": "SendConfirmation"
        }
      ],
      "Default": "InventoryFailed"
    },
    "SendConfirmation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:lks-lambda-send-notification",
      "End": true
    },
    "PaymentFailed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:lks-lambda-send-notification",
      "Next": "OrderFailed"
    },
    "InventoryFailed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:lks-lambda-send-notification",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingError",
      "Cause": "Proses pembayaran atau stok inventaris gagal."
    }
  }
}
```

---

## Modul 6: Otomatisasi & Pemantauan (EventBridge & CloudWatch)

Di modul ini, kita mengonfigurasi otomatisasi jadwal (EventBridge) dan pemantauan sistem otomatis (CloudWatch).

---

### 6.1. Langkah Praktik Konsol AWS (Klik-demi-Klik)

#### A. Membuat EventBridge Scheduled Rule (Cron Laporan)
1.  Cari **EventBridge** di pencarian atas konsol, klik layanan **Amazon EventBridge**.
2.  Di menu navigasi kiri, pilih **Schedules** -> **Create schedule**.
    *   **Schedule name**: `lks-eventbridge-daily-report`.
    *   **Schedule pattern**: Pilih **Recurring schedule** -> **Cron-based schedule**.
    *   **Cron expression**: Isi `59 23 * * ? *` (Trigger harian pukul 23:59 UTC). Klik **Next**.
    *   **Target service**: Pilih **AWS Lambda**.
    *   **Function**: Pilih **`lks-lambda-generate-report`**.
    *   Klik **Next**.
3.  Di halaman **Step 3: Settings** (atau halaman opsional):
    *   Cari bagian **Schedule state and permissions** -> **Execution role**.
    *   Pilih **Use existing role** (bukan *Create new role*).
    *   Pada dropdown, pilih **`LabRole`**.

    > ⚠️ **PENTING — Akun AWS Academy/Vocarium:**
    > Jangan biarkan AWS membuat role baru secara otomatis. Akun lab memiliki pembatasan izin `iam:CreateRole`, sehingga pembuatan role otomatis akan gagal dengan error:
    > *"is not authorized to perform: iam:CreateRole"*
    > Selalu pilih **Use existing role** -> **`LabRole`** pada setiap layanan yang meminta Execution Role (EventBridge Scheduler, Step Functions, dll).

    *   Klik **Next**, lalu klik **Create schedule**.


#### B. Setup CloudWatch Alarm untuk RDS CPU
1.  Cari **CloudWatch** di pencarian atas konsol, klik layanan **CloudWatch**.
2.  Di menu kiri, klik **Alarms** -> **All alarms** -> klik **Create alarm**.
3.  Klik **Select metric**.
    *   Pilih kategori **RDS**.
    *   Pilih sub-kategori **Per-DB instance metrics** (bukan Per-DB cluster).
    *   Pada kolom pencarian, ketik `lks-rds-orders`. Cari baris dengan **Metric Name = CPUUtilization** dan centang item tersebut.
    *   Klik **Select metric** di pojok kanan bawah.
4.  **Specify metric and conditions**:
    *   **Statistic**: `Average`, **Period**: `5 minutes`.
    *   Threshold type: **Static**.
    *   *Whenever CPUUtilization is...* Pilih **Greater than** -> isi `80`.
    *   Klik **Next**.
5.  **Configure Actions**:
    *   Alarm state trigger: **In alarm**.
    *   Send a notification: Pilih **Select an existing SNS topic** -> Pilih **`lks-sns-order-notifications`**.
    *   Klik **Next**.
6.  **Alarm name**: Isi `lks-alarm-rds-cpu`, klik **Create alarm**.

### 6.2. Panduan Konfigurasi EventBridge untuk `lks-lambda-generate-report`

> [!NOTE]
> Fungsi `lks-lambda-generate-report` sudah dibuat dan dideploy di **Modul 3.4.D**. Di sub-bab ini hanya perlu mengonfigurasi EventBridge Scheduler sebagai *trigger* otomatis harian.

Pastikan EventBridge Scheduler sudah dikonfigurasi sesuai langkah **6.1.A** di atas (Target Lambda: `lks-lambda-generate-report`). Tidak ada langkah tambahan yang diperlukan di sini.

---

## Modul 7: Frontend & CI/CD Pipeline (Amplify & GitHub Actions)

Modul terakhir adalah mendeploy aplikasi web frontend ke AWS Amplify dan mengotomatisasi alur kerja CI/CD menggunakan GitHub Actions.

---

### 7.1. Langkah Praktik Konsol AWS (Klik-demi-Klik)
1.  Cari **Amplify** di pencarian atas konsol, klik layanan **AWS Amplify**.
2.  Di halaman utama Amplify, klik tombol **Create new app** berwarna oranye di pojok kanan atas.
3.  Pada langkah *Add your code*, pilih **GitHub** lalu klik **Next**.
    *   Jika ini pertama kali, AWS akan meminta otorisasi ke akun GitHub. Klik **Authorize AWS Amplify** di halaman GitHub yang terbuka.
4.  Pilih **repositori GitHub** Anda dan pilih **Branch** `master` atau `main`. Klik **Next**.
5.  Pada langkah *App settings*:
    *   **App name**: Ubah menjadi `lks-amplify-order-app`.
    *   **Build and test settings**: AWS akan membaca dan menampilkan isi `amplify.yml` dari repositori Anda. Periksa apakah `baseDirectory` sudah menunjuk ke `frontend`.
    *   Buka bagian **Advanced settings**, klik **Add environment variables** dan tambahkan:
        *   Key: `API_ENDPOINT` -> Value: *(Tempelkan Invoke URL API Gateway dari Modul 4)*
        *   Key: `API_KEY` -> Value: *(Tempelkan string kunci API Key dari Modul 4)*
        *   Key: `AWS_REGION` -> Value: `us-east-1`
    *   Klik **Next**.
6.  Tinjau konfigurasi pada halaman *Review*, lalu klik **Save and deploy**.
    *   Amplify akan otomatis memulai proses build. Tunggu hingga status menjadi ✅ **Deployed** (sekitar 2-3 menit).
    *   Setelah selesai, klik tautan **Domain** yang muncul (contoh: `https://master.d12345.amplifyapp.com`) untuk membuka web Anda di browser.

---

### 7.2. Berkas Konfigurasi

#### A. Konfigurasi Build AWS Amplify (`amplify.yml`)
Letakkan berkas ini di root direktori repositori Anda:
```yaml
version: 1
frontend:
  phases:
    build:
      commands: []
  artifacts:
    baseDirectory: frontend
    files:
      - '**/*'
  cache:
    paths: []
```

#### B. GitHub Actions Workflow (`.github/workflows/lks-deploy.yml`)
Berkas ini berfungsi mendeploy secara otomatis saat Anda melakukan `git push` ke cabang master:
```yaml
name: Deploy Frontend to AWS Amplify

on:
  push:
    branches:
      - master
    paths:
      - 'frontend/**'
  workflow_dispatch:

jobs:
  deploy-frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: us-east-1

      - name: Validate Files
        run: |
          if [ ! -f frontend/index.html ] || [ ! -f frontend/app.js ] || [ ! -f amplify.yml ]; then
            echo "Error: File index.html, app.js, atau amplify.yml tidak ada!"
            exit 1
          fi

      - name: Trigger Amplify Build
        run: |
          APP_ID=$(aws amplify list-apps --query "apps[?name=='lks-amplify-order-app'].appId" --output text)
          if [ -z "$APP_ID" ]; then
            echo "Error: Aplikasi Amplify tidak ditemukan!"
            exit 1
          fi
          echo "Memicu deployment untuk App ID: $APP_ID"
          aws amplify start-job --app-id $APP_ID --branch-name master --job-type RELEASE
```

---

## Tips & Trik Juara LKS Cloud Computing

1.  **Disiplin Nama Resource (Auto-Grader)**: LKS menggunakan bot penilai otomatis yang mencari penamaan spesifik. Jangan mengubah prefix `lks-` atau memodifikasi karakter nama karena bisa berakibat nilai nol!
2.  **Perbarui Session Token**: Akun AWS Academy (Vocareum) memiliki token AWS yang kedaluwarsa dalam 3-4 jam. Ingatlah untuk memperbarui kredensial AWS di GitHub Secrets sebelum melakukan `git push` agar proses CI/CD tidak error.
3.  **Gunakan CloudWatch Logs**: Bila mendapati integrasi data di web statis Amplify tidak merespon, buka tab CloudWatch Logs fungsi Lambda untuk melacak pesan kesalahan seperti `connection timeout` (biasanya disebabkan salah meletakkan VPC Subnet atau Security Group).
