# Desain Database – Evermos Mini

Database: MySQL (GORM AutoMigrate). Semua tabel memiliki `id` (PK), `created_at`, `updated_at`.

## ERD

```mermaid
erDiagram
    USERS ||--o| TOKO : "memiliki"
    USERS ||--o{ ALAMAT : "memiliki"
    USERS ||--o{ TRANSAKSI : "melakukan"
    TOKO ||--o{ PRODUK : "menjual"
    CATEGORY ||--o{ PRODUK : "mengelompokkan"
    PRODUK ||--o{ LOG_PRODUK : "snapshot perubahan"
    ALAMAT ||--o{ TRANSAKSI : "dikirim ke"
    TRANSAKSI ||--|{ DETAIL_TRANSAKSI : "berisi"
    LOG_PRODUK ||--o{ DETAIL_TRANSAKSI : "dibeli sebagai"

    USERS {
        uint id PK
        string nama_user
        string kata_sandi "bcrypt, tidak pernah dikirim ke client"
        string no_telp "UNIQUE"
        string email "UNIQUE"
        date tanggal_lahir
        string jenis_kelamin
        string pekerjaan
        string id_provinsi
        string id_kota
        string role "user | admin"
    }

    TOKO {
        uint id PK
        uint user_id FK "UNIQUE, dibuat otomatis saat register"
        string nama_toko
        string photo
    }

    ALAMAT {
        uint id PK
        uint user_id FK
        string judul_alamat
        string nama_penerima
        string no_telp
        string provinsi
        string kota
        string kecamatan
        string kelurahan
        string detail_alamat
    }

    CATEGORY {
        uint id PK
        string nama_category "dikelola admin"
    }

    PRODUK {
        uint id PK
        uint toko_id FK
        uint category_id FK
        string nama_produk
        string slug
        float harga_reseller
        float harga_konsumen
        int stok
        text deskripsi
        string gambar "path/URL hasil upload"
    }

    LOG_PRODUK {
        uint id PK
        uint produk_id FK
        uint toko_id
        uint category_id
        string nama_produk "snapshot"
        string slug "snapshot"
        float harga_reseller "snapshot"
        float harga_konsumen "snapshot"
        text deskripsi "snapshot"
    }

    TRANSAKSI {
        uint id PK
        uint user_id FK
        uint alamat_pengiriman FK
        string invoice "UNIQUE"
        string metode_bayar
        string status
        float harga_total
    }

    DETAIL_TRANSAKSI {
        uint id PK
        uint transaksi_id FK
        uint log_produk_id FK
        int kuantitas
        float harga_total
    }
```

## Daftar Tabel

### users
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | BIGINT UNSIGNED PK | |
| nama_user | VARCHAR | |
| kata_sandi | VARCHAR | hash bcrypt |
| no_telp | VARCHAR | UNIQUE (wajib tidak sama antar user) |
| email | VARCHAR | UNIQUE (wajib tidak sama antar user) |
| tanggal_lahir | DATE | |
| jenis_kelamin | VARCHAR | |
| pekerjaan | VARCHAR | |
| id_provinsi, id_kota | VARCHAR | kode dari emsifa API wilayah |
| role | VARCHAR | default `user`; `admin` diubah langsung di DB |

### toko
Dibuat otomatis saat register. Relasi 1:1 dengan users (`user_id` UNIQUE).

| Kolom | Tipe | Keterangan |
|---|---|---|
| user_id | FK → users.id | UNIQUE |
| nama_toko | VARCHAR | default `Toko <nama user>` |
| photo | VARCHAR | hasil upload |

### alamat
Alamat pengiriman milik user. Wajib dipilih saat membuat transaksi.

| Kolom | Tipe | Keterangan |
|---|---|---|
| user_id | FK → users.id | |
| judul_alamat, nama_penerima, no_telp | VARCHAR | |
| provinsi, kota, kecamatan, kelurahan | VARCHAR | |
| detail_alamat | TEXT | |

### category
| Kolom | Tipe | Keterangan |
|---|---|---|
| nama_category | VARCHAR | Hanya admin yang boleh CRUD |

### produk
| Kolom | Tipe | Keterangan |
|---|---|---|
| toko_id | FK → toko.id | pemilik produk |
| category_id | FK → category.id | |
| nama_produk, slug | VARCHAR | |
| harga_reseller, harga_konsumen | DECIMAL(15,2) | |
| stok | INT | dikurangi saat transaksi |
| deskripsi | TEXT | |
| gambar | VARCHAR | hasil upload |

### log_produk
Snapshot produk setiap kali produk dibuat, diubah, atau dibeli. Tabel ini adalah sumber `log_produk_id` pada `detail_transaksi`, sehingga harga dan nama produk pada riwayat transaksi tidak berubah walaupun produk diedit.

| Kolom | Tipe | Keterangan |
|---|---|---|
| produk_id | FK → produk.id | |
| (kolom lain) | | salinan kolom produk pada saat snapshot dibuat |

### transaksi
| Kolom | Tipe | Keterangan |
|---|---|---|
| user_id | FK → users.id | pembeli |
| alamat_pengiriman | FK → alamat.id | harus milik user yang sama |
| invoice | VARCHAR | UNIQUE, format `INV-<user_id>-<unix>` |
| metode_bayar | VARCHAR | |
| status | VARCHAR | contoh: `Menunggu Pembayaran`, `Dikirim`, `Selesai` |
| harga_total | DECIMAL(15,2) | jumlah subtotal detail |

### detail_transaksi
| Kolom | Tipe | Keterangan |
|---|---|---|
| transaksi_id | FK → transaksi.id | |
| log_produk_id | FK → log_produk.id | snapshot produk yang dibeli |
| kuantitas | INT | |
| harga_total | DECIMAL(15,2) | harga_konsumen × kuantitas |

## Aturan Integritas
- `users.email` dan `users.no_telp` UNIQUE.
- `toko.user_id` UNIQUE (satu user satu toko).
- `transaksi.invoice` UNIQUE.
- Pembuatan transaksi harus dalam satu DB transaction: validasi stok, kurangi stok, insert `transaksi` + `detail_transaksi`.
- Setiap perubahan produk dan setiap transaksi menulis ke `log_produk`. Produk tidak dihapus fisik jika sudah pernah masuk `log_produk` yang dipakai transaksi (gunakan soft delete atau tolak penghapusan).

## Hak Akses
| Resource | Aturan |
|---|---|
| users | hanya pemilik akun yang dapat melihat dan mengubah datanya |
| toko | hanya pemilik yang dapat mengubah toko |
| alamat | hanya pemilik alamat |
| produk | hanya pemilik toko yang dapat ubah/hapus produk |
| transaksi | hanya pembeli yang dapat melihat dan mengubah status transaksinya |
| category | create/update/delete hanya `role = admin` |
