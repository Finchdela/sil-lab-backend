# Sistem Informasi Laboratorium (SIL-Lab) - Backend API Documentation

Backend ini dibangun menggunakan **Node.js**, **Express**, dan **Supabase** sebagai Database & Authentication Service. Sistem ini dirancang untuk mengelola peminjaman laboratorium, inventaris peralatan/bahan, pelaporan kerusakan, serta analitik statistik penggunaan lab.

---

## 🚀 Fitur Utama

- **Authentication & Authorization**: Registrasi dibatasi untuk civitas akademika UIN Raden Mas Said Surakarta (`@mhs.uinsaid.ac.id` dan `@staff.uinsaid.ac.id`). Otentikasi berbasis JWT menggunakan Supabase Auth.
- **Peminjaman Lab & Alat**:
  - Validasi bentrok jadwal peminjaman ruangan secara otomatis.
  - Pembuatan jadwal perkuliahan berulang (*recurring*) oleh Admin.
  - Pengurangan stok bahan habis pakai (*consumable*) secara otomatis ketika peminjaman disetujui.
- **Manajemen Inventaris**: Pengelolaan data ruangan dan peralatan/bahan oleh Admin.
- **Pelaporan Kerusakan**: Pengguna dapat melaporkan kerusakan alat, dan admin dapat memperbarui status penanganannya.
- **Dashboard Analitik**: Menyediakan data statistik peminjaman, rasio pengguna, laporan kerusakan baru, serta tren penggunaan ruangan dan alat terpopuler.

---

## 🛠️ Prasyarat & Instalasi

### 1. Salin Repositori & Pasang Dependensi
```bash
git clone https://github.com/Finchdela/sil-lab-backend.git
cd sil-lab-backend
npm install
```

### 2. Konfigurasi Environment Variables (`.env`)
Buat file bernama `.env` di direktori utama (sejajar dengan `package.json`) dan sesuaikan nilainya:
```env
PORT=3000
SUPABASE_URL=https://your-project-id.supabase.co
SUPABASE_KEY=your-supabase-anon-or-service-role-key
```

### 3. Setup Database (Supabase)
Jalankan perintah SQL yang ada pada file `schema.sql` di SQL Editor pada Dashboard Supabase Anda. Skrip ini akan membuat tabel berikut:
*   `users`: Menyimpan profil tambahan pengguna (nama, role).
*   `ruangan`: Menyimpan data ruangan laboratorium.
*   `peralatan`: Menyimpan aset alat (dapat dikembalikan) dan bahan (habis pakai).
*   `peminjaman`: Menyimpan data transaksi pengajuan peminjaman lab.
*   `peminjaman_detail_alat`: Detail alat/bahan yang dipinjam dalam satu transaksi.
*   `laporan_kerusakan`: Laporan kerusakan alat oleh pengguna.

> [!IMPORTANT]
> Pastikan **Row Level Security (RLS)** dinonaktifkan untuk tabel-tabel di atas agar backend NodeJS dapat membaca dan menulis data menggunakan Anon Key. Skrip `schema.sql` sudah otomatis menonaktifkan RLS tersebut.

### 4. Menjalankan Server
*   **Mode Pengembangan (dengan Nodemon):**
    ```bash
    npm run dev
    ```
*   **Mode Produksi:**
    ```bash
    npm start
    ```

---

## 🔑 Otentikasi & Otorisasi

Semua endpoint yang membutuhkan otentikasi mengharuskan Anda mengirimkan **JSON Web Token (JWT)** melalui Header HTTP:
```http
Authorization: Bearer <your_access_token>
```

### Role Pengguna
Sistem mengenal 3 jenis role:
1.  `mahasiswa` (Default saat registrasi)
2.  `staff`
3.  `admin`

---

## 📌 Dokumentasi Endpoint API

Semua URL endpoint diawali dengan base path `/api`.

### 1. Otentikasi (`/api/auth`)

#### **Registrasi Akun Baru**
Mendaftarkan pengguna baru ke Supabase Auth dan tabel profil `users`.
*   **URL:** `/api/auth/register`
*   **Method:** `POST`
*   **Headers:** `Content-Type: application/json`
*   **Request Body:**
    ```json
    {
      "email": "user@mhs.uinsaid.ac.id",
      "password": "securepassword123",
      "nama": "Ahmad Dani",
      "role": "mahasiswa" // opsional, default: "mahasiswa"
    }
    ```
*   **Aturan Email:** Harus berakhiran dengan `@mhs.uinsaid.ac.id` atau `@staff.uinsaid.ac.id`.
*   **Response (201 Created):**
    ```json
    {
      "message": "Registrasi berhasil! Silakan cek email Anda untuk verifikasi.",
      "user": { ... }
    }
    ```

#### **Login Pengguna**
Mendapatkan token akses JWT untuk berinteraksi dengan API yang terproteksi.
*   **URL:** `/api/auth/login`
*   **Method:** `POST`
*   **Request Body:**
    ```json
    {
      "email": "user@mhs.uinsaid.ac.id",
      "password": "securepassword123"
    }
    ```
*   **Response (200 OK):**
    ```json
    {
      "message": "Login berhasil",
      "token": "eyJhbGciOiJIUzI1NiIsIn...",
      "user": {
        "id": "uuid-user",
        "email": "user@mhs.uinsaid.ac.id",
        "nama": "Ahmad Dani",
        "role": "mahasiswa",
        "created_at": "2026-06-09T04:36:23Z"
      }
    }
    ```

#### **Permintaan Lupa Password (Forgot Password)**
Mengirimkan link reset password ke email terdaftar.
*   **URL:** `/api/auth/forgot-password`
*   **Method:** `POST`
*   **Request Body:**
    ```json
    {
      "email": "user@mhs.uinsaid.ac.id",
      "redirectUrl": "http://localhost:5500/update-password.html"
    }
    ```
*   **Response (200 OK):**
    ```json
    {
      "message": "Link reset password telah dikirim ke email Anda."
    }
    ```

#### **Perbarui Password Baru (Update Password)**
Memperbarui password pengguna yang sedang masuk/mengakses lewat token pemulihan.
*   **URL:** `/api/auth/update-password`
*   **Method:** `POST`
*   **Headers:** `Authorization: Bearer <token>`
*   **Request Body:**
    ```json
    {
      "new_password": "newsecurepassword123"
    }
    ```
*   **Response (200 OK):**
    ```json
    {
      "message": "Password berhasil diperbarui. Silakan login kembali."
    }
    ```

---

### 2. Peminjaman Lab & Peralatan (`/api/booking`)

#### **Ajukan Peminjaman Baru (User)**
Mengajukan peminjaman ruangan lab dan/atau peralatan/bahan. Status awal akan berupa `pending`.
*   **URL:** `/api/booking`
*   **Method:** `POST`
*   **Headers:** `Authorization: Bearer <token>`
*   **Request Body:**
    ```json
    {
      "ruang_id": 1, 
      "waktu_mulai": "2026-06-10T08:00:00.000Z",
      "waktu_selesai": "2026-06-10T10:00:00.000Z",
      "tujuan_peminjaman": "Praktikum Kimia Organik",
      "alat_ids": [
        { "id": 2, "jumlah": 3 },
        { "id": 5, "jumlah": 1 }
      ]
    }
    ```
*   **Response (201 Created):**
    ```json
    {
      "message": "Peminjaman berhasil diajukan",
      "data": {
        "id": 15,
        "user_id": "uuid-user",
        "ruang_id": 1,
        "waktu_mulai": "2026-06-10T08:00:00.000Z",
        "waktu_selesai": "2026-06-10T10:00:00.000Z",
        "tujuan_peminjaman": "Praktikum Kimia Organik",
        "status": "pending",
        "created_at": "2026-06-09T11:36:21.000Z"
      }
    }
    ```
*   **Validasi Bentrok:** Jika pada waktu tersebut ruangan sudah dibooking dan disetujui, API mengembalikan status `409 Conflict` dengan pesan `"Ruangan bentrok dengan jadwal lain!"`.

#### **Dapatkan Riwayat Peminjaman**
Mengambil daftar riwayat peminjaman.
*   **URL:** `/api/booking`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>`
*   **Akses:**
    - **Mahasiswa/Staff:** Hanya melihat peminjaman miliknya sendiri.
    - **Admin:** Melihat peminjaman dari semua pengguna.
*   **Response (200 OK):** Menghasilkan array objek peminjaman lengkap beserta data relasi `users`, `ruangan`, dan detail peralatan yang dipinjam.

#### **Jadwal Publik Laboratorium**
Mendapatkan jadwal penggunaan laboratorium yang disetujui mulai hari ini ke depan.
*   **URL:** `/api/booking/schedule`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>`
*   **Response (200 OK):** Menyajikan data transparan meliputi waktu, nama peminjam, tujuan, dan ruangan untuk ditampilkan pada kalender/jadwal publik.

#### **Cek Peminjaman Alat Aktif (Admin)**
*   **URL:** `/api/booking/active-tools`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Response (200 OK):** Daftar peminjaman alat yang statusnya `disetujui` dan waktu selesainya lebih besar dari saat ini. Digunakan untuk menghitung sisa stok dinamis.

#### **Buat Jadwal Perkuliahan / Manual (Admin)**
Membuat peminjaman langsung disetujui, mendukung fitur jadwal berulang setiap minggu.
*   **URL:** `/api/booking/admin`
*   **Method:** `POST`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "ruang_id": 1,
      "waktu_mulai": "2026-06-10T08:00:00.000Z",
      "waktu_selesai": "2026-06-10T10:00:00.000Z",
      "tujuan_peminjaman": "Kuliah Pemrograman Web",
      "is_repeat": true,
      "repeat_count": 8 // Mengulangi jadwal ini setiap minggu selama 8 pertemuan
    }
    ```
*   **Response (201 Created):**
    ```json
    {
      "message": "Jadwal kuliah berhasil dibuat",
      "data": [ ... ]
    }
    ```

#### **Persetujuan / Perubahan Status Peminjaman (Admin)**
Menyetujui, menolak, atau menyelesaikan peminjaman.
*   **URL:** `/api/booking/:id/status`
*   **Method:** `PATCH`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "status": "disetujui" // Nilai lain: 'ditolak', 'selesai', 'dibatalkan'
    }
    ```
*   **Catatan Penting Pengurangan Stok:**
    - Jika status diubah menjadi `disetujui`, sistem akan memeriksa peralatan yang diajukan.
    - Jika ada item dengan tipe `jenis = "bahan"` (habis pakai), jumlah stok di database (`jumlah_total` & `jumlah_tersedia`) akan otomatis dipotong permanen.
    - Jika stok bahan tidak cukup, proses persetujuan akan dibatalkan otomatis dan mengembalikan status `400 Bad Request` dengan pesan error stok kurang.
*   **Response (200 OK):**
    ```json
    {
      "message": "Status berhasil diubah menjadi disetujui",
      "data": { ... }
    }
    ```

#### **Hapus Data Peminjaman (Admin)**
*   **URL:** `/api/booking/:id`
*   **Method:** `DELETE`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Response (200 OK):** `{"message": "Data peminjaman berhasil dihapus"}`

---

### 3. Manajemen Ruangan (`/api/rooms`)

#### **Daftar Ruangan**
Mendapatkan semua ruangan lab.
*   **URL:** `/api/rooms`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>`
*   **Response (200 OK):** Array dari objek ruangan.

#### **Tambah Ruangan Baru (Admin)**
*   **URL:** `/api/rooms`
*   **Method:** `POST`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "nama_ruang": "Laboratorium Komputer Terpadu",
      "kapasitas": 40,
      "lokasi": "Gedung C Lantai 2"
    }
    ```
*   **Response (201 Created):**
    ```json
    {
      "message": "Ruangan berhasil ditambahkan",
      "data": { ... }
    }
    ```

#### **Ubah Status Ruangan / Pemeliharaan (Admin)**
Mengubah status ketersediaan ruangan (misal: masuk tahap perbaikan).
*   **URL:** `/api/rooms/:id/status`
*   **Method:** `PATCH`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "status": "pemeliharaan" // 'tersedia', 'pemeliharaan'
    }
    ```
*   **Response (200 OK):** `{"message": "Status ruangan berhasil diubah", "data": { ... }}`

#### **Edit Informasi Ruangan (Admin)**
*   **URL:** `/api/rooms/:id`
*   **Method:** `PUT`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "nama_ruang": "Laboratorium Komputer Update",
      "kapasitas": 45,
      "lokasi": "Gedung C Lantai 2"
    }
    ```
*   **Response (200 OK):** `{"message": "Data ruangan berhasil diperbarui", "data": { ... }}`

#### **Hapus Ruangan (Admin)**
*   **URL:** `/api/rooms/:id`
*   **Method:** `DELETE`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Response (200 OK):** `{"message": "Ruangan berhasil dihapus"}`

---

### 4. Inventaris Alat & Bahan (`/api/equipment`)

#### **Daftar Peralatan & Bahan**
*   **URL:** `/api/equipment`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>`
*   **Response (200 OK):** Array data alat & bahan diurutkan alfabetis berdasarkan `nama_alat`.

#### **Tambah Peralatan/Bahan Baru (Admin)**
*   **URL:** `/api/equipment`
*   **Method:** `POST`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "nama_alat": "Mikroskop Binokuler Olympus",
      "kategori": "Optik",
      "jumlah_total": 10,
      "kondisi": "Baik",
      "jenis": "alat" // 'alat' (dipinjam dan kembali) atau 'bahan' (habis pakai)
    }
    ```
*   **Response (201 Created):** `{"message": "Data berhasil ditambahkan", "data": [ ... ]}`

#### **Ubah Informasi Peralatan/Bahan (Admin)**
*   **URL:** `/api/equipment/:id`
*   **Method:** `PUT`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "nama_alat": "Mikroskop Binokuler Olympus (Edited)",
      "kategori": "Optik",
      "jumlah_total": 12,
      "kondisi": "Baik",
      "jenis": "alat"
    }
    ```
*   **Response (200 OK):** `{"message": "Data berhasil diperbarui", "data": [ ... ]}`

#### **Hapus Data Peralatan/Bahan (Admin)**
*   **URL:** `/api/equipment/:id`
*   **Method:** `DELETE`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Response (200 OK):** `{"message": "Data berhasil dihapus"}`

---

### 5. Laporan Kerusakan Alat (`/api/reports`)

#### **Buat Laporan Kerusakan Baru (User)**
*   **URL:** `/api/reports`
*   **Method:** `POST`
*   **Headers:** `Authorization: Bearer <token>`
*   **Request Body:**
    ```json
    {
      "alat_id": 3,
      "deskripsi_kerusakan": "Lensa obyektif berjamur dan buram saat digunakan."
    }
    ```
*   **Response (201 Created):** `{"message": "Laporan terkirim", "data": { ... }}`

#### **Daftar Laporan Saya (User)**
Melihat riwayat pelaporan kerusakan yang diajukan oleh user bersangkutan.
*   **URL:** `/api/reports/my`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>`
*   **Response (200 OK):** Array laporan kerusakan milik user tersebut.

#### **Daftar Semua Laporan Kerusakan (Admin)**
*   **URL:** `/api/reports/all`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Response (200 OK):** Array seluruh laporan kerusakan dari semua user beserta informasi detail peralatan dan nama pelapor.

#### **Ubah Status Penanganan Laporan (Admin)**
Memperbarui status penanganan atas kerusakan alat yang dilaporkan.
*   **URL:** `/api/reports/:id/status`
*   **Method:** `PATCH`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Request Body:**
    ```json
    {
      "status_laporan": "diproses" // Nilai: 'baru', 'diproses', 'selesai'
    }
    ```
*   **Response (200 OK):** `{"message": "Status laporan diperbarui", "data": { ... }}`

---

### 6. Analitik Dashboard (`/api/analytics`)

#### **Statistik Analitik Utama (Admin)**
Mendapatkan kumpulan data statistik teragregasi untuk widget dan grafik dashboard admin.
*   **URL:** `/api/analytics`
*   **Method:** `GET`
*   **Headers:** `Authorization: Bearer <token>` (Admin Only)
*   **Response (200 OK):**
    ```json
    {
      "totalBookings": 48,
      "countAdmin": 20,
      "countUser": 28,
      "pendingBookings": 3,
      "damageReports": 2,
      "totalTools": 150,
      "roomUsage": [
        { "ruang_id": 1, "ruangan": { "nama_ruang": "Lab Kimia" } },
        { "ruang_id": 1, "ruangan": { "nama_ruang": "Lab Kimia" } },
        { "ruang_id": 2, "ruangan": { "nama_ruang": "Lab Fisika" } }
      ],
      "toolUsage": [
        {
          "alat_id": 2,
          "jumlah_pinjam": 5,
          "peralatan": { "nama_alat": "Gelas Ukur" },
          "peminjaman": { "status": "disetujui" }
        }
      ]
    }
    ```

---

## 🛡️ Error Handling
Format respon error yang seragam digunakan ketika terjadi kesalahan input atau kegagalan server:
```json
{
  "error": "Pesan deskripsi kesalahan / error dari server."
}
```

HTTP Status Codes yang umum digunakan:
*   `400 Bad Request`: Input tidak valid, field wajib kurang, atau stok bahan tidak mencukupi.
*   `401 Unauthorized`: Token tidak dikirimkan, kedaluwarsa, atau tidak valid.
*   `403 Forbidden`: Otorisasi ditolak karena peran (role) pengguna tidak sesuai.
*   `409 Conflict`: Bentrok jadwal penggunaan ruangan.
*   `500 Internal Server Error`: Masalah koneksi database atau kesalahan sistem internal.
