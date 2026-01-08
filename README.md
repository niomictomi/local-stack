# Local Development Stack with Docker Compose

Stack ini menyediakan semua service pendukung untuk development lokal di MacBook kamu. Cocok untuk project full-stack (PHP/Laravel, Node.js, Go, dll) yang butuh database, cache, dan object storage.

## Tech Stack / Services yang Tersedia

| Service       | Versi          | Fungsi Utama                          | Port Default (Host) |
|---------------|----------------|---------------------------------------|---------------------|
| **Redis**     | latest         | In-memory cache / key-value store     | 6379                |
| **MySQL**     | 8.0            | Relational database                   | 3306                |
| **phpMyAdmin**| latest         | Web interface untuk manage MySQL       | 8080                |
| **MinIO**     | latest         | S3-compatible object storage          | 9000 (API) <br> 9001 (Console) |

Semua service dijalankan menggunakan **Docker Compose** sehingga mudah di-start, stop, dan maintain.

## Requirements

- Docker Desktop untuk macOS (sudah terinstall Docker & Docker Compose)
- Terminal (iTerm/Terminal bawaan macOS)

Cek versi:
```bash
docker --version
docker-compose --version
```

## Folder Structure Rekomendasi

```
local-stack/
├── docker-compose.yml
├── README.md          ← file ini
└── .env               ← (opsional, untuk password lebih aman)
```

## docker-compose.yml (Config Lengkap)

```yaml
version: '3.8'

services:
  redis:
    image: redis:latest
    container_name: redis-local
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    command: redis-server --save 60 1 --loglevel warning

  mysql:
    image: mysql:8.0
    container_name: mysql-local
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    restart: unless-stopped
    command: --default-authentication-plugin=mysql_native_password

  minio:
    image: minio/minio:latest
    container_name: minio-local
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minio12345678   # minimal 8 karakter
    ports:
      - "9000:9000"    # API endpoint
      - "9001:9001"    # Web Console
    volumes:
      - minio-data:/data
    command: server /data --console-address ":9001"
    restart: unless-stopped

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    container_name: phpmyadmin-local
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_ARBITRARY: 1
    ports:
      - "8080:80"
    restart: unless-stopped
    depends_on:
      - mysql

volumes:
  redis-data:
  mysql-data:
  minio-data:
```

## Cara Pakai (Step by Step)

### 1. Pertama Kali Setup
```bash
# Buat folder
mkdir ~/local-stack && cd ~/local-stack

# Buat file docker-compose.yml (copy dari atas)
# Buat file README.md ini juga

# Jalankan semua service
docker-compose up -d
```

### 2. Perintah Harian
```bash
# Start / restart semua service
docker-compose up -d

# Stop semua service (data tetap aman)
docker-compose down

# Stop + hapus semua data (hati-hati!)
docker-compose down -v

# Lihat status container
docker-compose ps

# Lihat log salah satu service (contoh mysql)
docker-compose logs -f mysql
```

### 3. Akses Setiap Service

| Service         | Cara Akses                                              | Credential / Catatan                                      |
|-----------------|---------------------------------------------------------|-----------------------------------------------------------|
| **Redis**       | `localhost:6379`                                        | Tanpa password                                            |
| **MySQL**       | `localhost:3306`                                        | Root: `root` / `rootpassword`<br>User: `appuser` / `apppassword`<br>Database default: `myapp` |
| **phpMyAdmin**  | Buka browser → http://localhost:8080                    | Username: `root` atau `appuser`<br>Password: sesuai MySQL<br>Server: biarkan default/kosong |
| **MinIO API**   | `http://localhost:9000`                                 | Digunakan oleh SDK (AWS S3 SDK dengan custom endpoint)    |
| **MinIO Console**| Buka browser → http://localhost:9001                    | Username: `minioadmin`<br>Password: `minio12345678`       |

### 4. Contoh Koneksi dari Code Project

**Redis (Node.js - ioredis)**

```js
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });
```

**MySQL (PHP - PDO)**

```php
$db = new PDO('mysql:host=localhost;dbname=myapp', 'appuser', 'apppassword');
```

**MinIO (AWS SDK for PHP)**

```sh
$s3 = new Aws\S3\S3Client([
    'version'     => 'latest',
    'region'      => 'us-east-1',
    'endpoint'    => 'http://localhost:9000',
    'credentials' => [
        'key'     => 'minioadmin',
        'secret'  => 'minio12345678',
    ],
]);
```

## Troubleshooting Umum

- **phpMyAdmin tidak bisa connect** → Tunggu 20-30 detik setelah `up -d`. MySQL butuh waktu inisialisasi.
- **Port sudah dipakai** → Ubah port di `docker-compose.yml` (misal `3306:3306` jadi `3307:3306`).
- **Data hilang setelah down** → Jangan pakai `down -v` kalau ingin data tetap ada.
- **MinIO console blank** → Refresh halaman atau cek log: `docker-compose logs minio`.

## Upgrade Keamanan (Opsional)

Buat file `.env` di folder yang sama:

```sh
MYSQL_ROOT_PASSWORD=rahasia123
MYSQL_PASSWORD=rahasia456
MINIO_ROOT_PASSWORD=rahasiaminIO890
```

Lalu ubah `docker-compose.yml` jadi pakai `${VARIABEL}`.

Selamat developing! 🚀  
Kalau butuh tambahan service (Mailhog, PostgreSQL, MongoDB, Nginx, dll), tinggal bilang ya.

Sekarang README kamu sudah **super lengkap**: deskripsi jelas, stack terdaftar, config full, cara pakai step-by-step, contoh koneksi code, sampai troubleshooting. Tinggal copy-paste ke folder project kamu! 😄