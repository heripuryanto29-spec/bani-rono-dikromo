# Silsilah Keluarga — Vercel frontend + Google Apps Script API

Migrasi frontend dari Google Apps Script HTML Service ke hosting statis Vercel. Google Apps Script tetap menjadi backend; Google Sheets tetap menyimpan data dan Google Drive tetap menyimpan foto. Browser hanya berkomunikasi dengan `/api/gas` di Vercel, sehingga `GAS_API_SECRET` tidak pernah ditaruh di HTML.

## Struktur proyek

```text
silsilah-keluarga-vercel/
├── index.html          # frontend responsif
├── api/
│   └── gas.js          # Vercel Serverless Function / proxy API
├── gas/
│   └── Code.gs         # backend Google Apps Script
├── vercel.json
├── .gitignore
└── README.md
```

## 1. Siapkan Google Sheets dan Apps Script

1. Buka spreadsheet yang sudah dipakai aplikasi Silsilah Keluarga.
2. Buka **Ekstensi → Apps Script**.
3. Cadangkan kode lama. Ganti `Code.gs` dengan isi `gas/Code.gs` dari paket ini. Backend mempertahankan nama sheet dan skema dari versi sebelumnya: `Anggota`, `Users`, `Keluarga`, dan `LogAktivitas`.
4. Jalankan `setupDatabase()` sekali dari editor bila belum pernah dijalankan. Berikan izin Google Sheets dan Google Drive. Jika database lama sudah dipakai, jangan menghapus sheet atau mengubah urutan header.
5. Buka **Project Settings / Setelan project → Script Properties / Properti skrip** dan tambahkan:
   - Property: `API_SECRET`
   - Value: string acak rahasia minimal 32 karakter, misalnya hasil password manager. Jangan gunakan contoh yang dipublikasikan dan jangan masukkan nilai ini ke `index.html` atau repository GitHub.
6. Pastikan properti `SPREADSHEET_ID` dan `PHOTO_FOLDER_ID` sudah ada. Biasanya dibuat oleh `setupDatabase()`.
7. Klik **Deploy → Manage deployments → Edit** pada deployment Web App yang digunakan, atau buat deployment baru. Pilih **Execute as: Me**. Karena Vercel memanggil backend dari server, setelan akses Web App perlu mengizinkan permintaan tanpa login Google (umumnya **Anyone**). Akses API tetap harus melewati `API_SECRET`; jangan membagikan secret. Jika kebijakan Google Workspace melarang akses ini, deployment model ini tidak dapat dipakai tanpa backend/autentikasi lain.
8. Klik **Deploy** dan salin URL Web App yang berakhiran `/exec`. Jangan gunakan URL `/dev`.
9. Buka URL `/exec` langsung untuk memastikan tampil pesan bahwa API aktif. URL ini tidak lagi menyajikan frontend lama.

> Jika menggunakan project Apps Script baru, jalankan `setupDatabase()` di project yang benar-benar terhubung ke spreadsheet yang ingin dipakai. Jangan menjalankan setup pada spreadsheet kosong jika data lama perlu dipertahankan.

## 2. Unggah ke GitHub

1. Ekstrak ZIP ini.
2. Buat repository GitHub baru, lalu unggah semua isi folder (termasuk folder `api` dan `gas`).
3. Pastikan `.env` atau nilai `API_SECRET` tidak pernah diunggah. File `.gitignore` sudah mengabaikan `.env` dan `.env.local`.

## 3. Deploy ke Vercel

1. Masuk ke [Vercel](https://vercel.com/), pilih **Add New → Project**, lalu impor repository GitHub.
2. Framework preset: **Other**. Build command dan output directory boleh dikosongkan untuk proyek statis ini.
3. Sebelum/ketika deploy, buka **Settings → Environment Variables** dan tambahkan dua variabel berikut untuk Production (tambahkan juga Preview bila diperlukan):
   - `GAS_WEB_APP_URL` = URL Web App Apps Script yang berakhiran `/exec`
   - `GAS_API_SECRET` = nilai yang sama persis dengan `API_SECRET` pada Script Properties Apps Script
4. Deploy. Setelah mengubah environment variables, lakukan **Redeploy** agar variabel baru dimuat.
5. Buka URL `https://nama-proyek.vercel.app`.

## 4. API yang tersedia

Semua permintaan browser memakai `POST /api/gas` dengan JSON `{ "action": "...", "args": [...] }`. Secret API ditambahkan hanya oleh fungsi server Vercel.

| Action | Argumen frontend | Fungsi |
|---|---|---|
| `login` | `[username, password]` | Login dan membuat token sesi |
| `logout` | `[token]` | Logout |
| `getInitialData` | `[token]` | Memuat pengguna, anggota, cabang, statistik |
| `saveMember` | `[token, dataAnggota]` | Tambah/ubah anggota, unggah foto |
| `deleteMember` | `[token, memberId]` | Hapus anggota dengan validasi relasi |
| `saveFamily` | `[token, dataKeluarga]` | Tambah/ubah cabang keluarga |
| `deleteFamily` | `[token, familyId]` | Hapus cabang kosong |
| `changePassword` | `[token, passwordLama, passwordBaru]` | Ganti password |
| `createUser` | `[token, dataPengguna]` | Buat pengguna (Admin) |
| `getActivityLogs` | `[token]` | Log aktivitas (Admin) |

Otorisasi Admin/Pengguna dan validasi tetap dijalankan oleh Apps Script. Menyembunyikan menu di frontend bukan pengganti pemeriksaan akses backend.

## 5. Pengujian setelah deploy

- [ ] Buka situs Vercel melalui HTTPS di PC dan smartphone.
- [ ] Login dengan akun yang sudah ada pada sheet `Users`.
- [ ] Coba password salah; pesan error harus muncul tanpa menampilkan detail internal server.
- [ ] Tambah anggota, muat ulang data, lalu pastikan baris muncul di sheet `Anggota`.
- [ ] Ubah dan hapus anggota uji. Uji validasi jika anggota masih menjadi orang tua/pasangan.
- [ ] Buat cabang keluarga, ubah, lalu hapus setelah kosong.
- [ ] Uji relasi ayah/ibu/pasangan dan diagram pohon.
- [ ] Unggah foto kecil (maksimum 3 MB) dan pastikan file tersimpan di folder Drive.
- [ ] Ganti password dan coba login ulang.
- [ ] Buat akun Pengguna; pastikan akun tersebut tidak dapat mengubah data atau membuka log pengguna.
- [ ] Uji logout dan sesi kedaluwarsa.
- [ ] Periksa **Vercel → Logs** jika muncul error API. Jangan mencatat atau membagikan secret/password/token dalam laporan.

## Keamanan dan catatan operasional

- Data keluarga adalah data pribadi. Batasi akses pengguna, gunakan password kuat, jangan membagikan secret, dan hindari mengunggah data nyata sebelum seluruh tes keamanan selesai.
- Secret hanya berada di Script Properties Apps Script dan Environment Variables Vercel. Jika bocor, ganti di kedua tempat dan redeploy.
- Token sesi disimpan sementara di Apps Script Cache dengan TTL 6 jam dan frontend saat ini menyimpannya di memori halaman; pengguna mungkin perlu login lagi setelah refresh. Jangan simpan token/password ke `localStorage` tanpa desain keamanan tambahan.
- Hash password backend awal memakai SHA-256 + salt. Untuk sistem produksi dengan risiko tinggi, pertimbangkan penyedia identitas yang terkelola dan audit keamanan profesional.
- Foto disimpan di Drive dan UI menggunakan URL foto Drive yang tersimpan pada data anggota. Jangan menjadikan seluruh folder Drive publik demi menampilkan foto. Pastikan izin file/folder sesuai kebijakan keluarga; jika foto perlu diakses pengguna tanpa login Google, implementasikan endpoint proxy foto terautentikasi sebelum digunakan dengan data nyata.
- Vercel Functions dan Apps Script memiliki batas ukuran/waktu. Foto dibatasi 3 MB; jika unggahan gagal, uji dengan file lebih kecil.
- Jangan menghapus atau mengubah sheet/kolom yang sudah ada. Lakukan backup spreadsheet sebelum migrasi.

## Pemecahan masalah

- **`Konfigurasi server belum lengkap`**: periksa `GAS_WEB_APP_URL` dan `GAS_API_SECRET` di Vercel, lalu redeploy.
- **`Akses API ditolak`**: nilai `API_SECRET` di Script Properties tidak sama dengan `GAS_API_SECRET` di Vercel, atau properti belum dibuat.
- **Apps Script tidak mengembalikan JSON**: pastikan URL `/exec` benar, deployment versi terbaru sudah diterbitkan, dan Web App dapat dipanggil tanpa login Google.
- **Sheet tidak ditemukan**: jalankan `setupDatabase()` pada project Apps Script yang benar dan periksa `SPREADSHEET_ID`.
- **Foto tidak tampil**: periksa izin akses file Drive; jangan membuat folder publik sebagai jalan pintas.
