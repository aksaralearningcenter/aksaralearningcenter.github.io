# Aksara Learning Center — Frontend GitHub Pages + Backend API Spreadsheet

**Seluruh frontend berjalan di GitHub Pages** — landing page publik **dan** aplikasi admin internal. Apps Script murni menjadi **API backend** dengan Google Sheets sebagai database.

```
GitHub Pages                          Apps Script (API only)         Google Sheets
├─ index.html        (landing publik) → ?api=registrations  ──►  sheet Pendaftaran
├─ admin/index.html  (login + admin SPA) → ?api=call&token= ──►  Murid, Tabungan, Users, dst.
└─ .nojekyll                                              ──►  Email notifikasi otomatis
```

## Login admin dari GitHub

Semua user (Admin, Guru, Orang Tua) login dengan **username + password** via `?api=login` — sesi token 12 jam disimpan di `localStorage` browser. Login Google **tidak tersedia** lewat frontend eksternal (keterbatasan platform Apps Script).

Kredensial dibuat lewat menu spreadsheet:
- **Staf (Admin & Guru)**: `LMS & Tabungan → User → 🔑 Buat Akun Admin & Guru` (email kredensial terkirim otomatis)
- **Orang Tua**: otomatis saat pendaftaran disetujui admin

> ⚠️ Setelah deploy versi API ini, jalankan menu Buat Akun Admin & Guru agar akun Anda punya username/password untuk login dari GitHub.

---

## Langkah 1 — Deploy backend (Apps Script)

Jika web app Anda sudah pernah deploy, **loncat ke Langkah 2** (hanya butuh URL-nya).

1. Buka spreadsheet LMS & Tabungan → **Extensions → Apps Script**
2. Pastikan `Code.gs` versi terbaru sudah tersimpan (berisi `doGet` dengan `?api=`)
3. **Deploy → Manage deployments → Edit (ikon pensil) → Version: New version**
4. **Who has access: "Anyone"** ← **WAJIB**, agar form publik bisa mengirim pendaftaran
5. Copy **Web app URL** — bentuknya:
   ```
   https://script.google.com/macros/s/AKfycb.../exec
   ```

## Langkah 2 — Isi URL API di frontend

**`admin/index.html`** (aplikasi admin):
```js
const API_URL = 'https://script.google.com/macros/s/AKfycb.../exec';
```

**`index.html`** (landing page):
```js
const LP_API_URL = 'https://script.google.com/macros/s/AKfycb.../exec';
```

Dan ganti semua tulisan `APPS_SCRIPT_WEBAPP_URL` di `index.html` (4 tempat — tombol "Masuk Sistem") dengan URL yang sama.

## Langkah 3 — Push ke GitHub

```bash
cd setup_github_fe
git init
git add .
git commit -m "Landing page Aksara + API spreadsheet"
git branch -M main
# Buat repo baru di github.com (nama bebas, mis. aksara-landing), lalu:
git remote add origin https://github.com/aksaralearningcenter/aksaralearningcenter.github.io.git
git push -u origin main
```

## Langkah 4 — Aktifkan GitHub Pages

1. Repo → **Settings → Pages**
2. **Source: Deploy from a branch** → Branch: `main`, folder: `/ (root)` → **Save**
3. ±1 menit, situs hidup di **https://aksaralearningcenter.github.io/** (repo `username.github.io` = domain root)

`.nojekyll` sudah disertakan agar semua file dilayani apa adanya.

---

## Struktur folder

```
setup_github_fe/
├── index.html          ← landing page publik + form pendaftaran
├── admin/
│   └── index.html      ← aplikasi admin (login + dashboard SPA)
├── .nojekyll
└── README.md
```

Aplikasi admin diakses di **https://aksaralearningcenter.github.io/admin/**.

---

## API

### `POST {WEBAPP_URL}?api=registrations`

Kirim pendaftaran baru. Body JSON (`Content-Type: text/plain` disarankan — menghindari preflight CORS):

```json
{
  "nama": "Budi Santoso",
  "tanggalLahir": "2015-04-12",
  "program": "Aksara Kids (TK-SD)",
  "namaOrangTua": "Ibu Sari",
  "noHP": "081234567890",
  "email": "ortu@contoh.com",
  "catatan": "Ingin kelas coba hari Sabtu",
  "idemKey": "lpABC123"
}
```

Respons:

```json
{ "success": true, "id": "REG-...", "message": "Pendaftaran berhasil! ..." }
```

Proteksi bawaan (sama dengan web app): idempotency key, lock anti race-condition, cooldown 60 detik per nomor HP, batas 3/hari, dedupe 30 hari. Setiap pendaftaran otomatis memicu **email ke admin** + masuk sheet **Pendaftaran**.

### `GET {WEBAPP_URL}?api=stats`

Ringkasan publik non-sensitif (hanya angka & daftar program — tanpa data pribadi):

```json
{ "success": true, "totalPendaftar": 42, "program": ["Aksara Kids (TK-SD)", "..."], "updatedAt": "..." }
```

### Fallback JSONP

Jika jaringan browser memblokir fetch, form otomatis mencoba JSONP: `?api=registrations&callback=fn&payload={...}` — didukung penuh di sisi Apps Script.

### `POST {WEBAPP_URL}?api=login`

Login username/password (Admin, Guru, Orang Tua). Body:

```json
{ "login": "budisantoso427", "password": "K7mpQ2xR" }
```

Respons: `{ "success": true, "token": "<64-hex>", "peran": "...", "nama": "..." }` — token berlaku 12 jam. Brute-force guard: maks 5 gagal per username per 10 menit.

### `POST {WEBAPP_URL}?api=me` / `?api=logout` / `?api=change-password`

- `me` — info sesi `{ token }` → email, nama, peran, nama sekolah
- `logout` — hancurkan token `{ token }`
- `change-password` — ganti password sendiri `{ token, oldPass, newPass }`

### `POST {WEBAPP_URL}?api=call&token=<TOKEN>`

Endpoint utama aplikasi admin — memanggil fungsi internal dengan identitas dari token sesi. Body:

```json
{ "fn": "getStudents", "args": [] }
```

Contoh dengan argumen: `{ "fn": "addTransaction", "args": [{ "studentId": "...", "savingsId": "...", "jenis": "Setoran", "jumlah": 50000 }] }`

Hanya fungsi dalam **allowlist** server yang bisa dipanggil (murid, kelas, absensi, tabungan, transaksi, users, pendaftar, laporan). Fungsi auth & sensitif tidak ada di daftar.

---

## Update konten

Edit `index.html` (teks, harga, program, foto) → `git add . && git commit -m "..." && git push` → GitHub Pages diperbarui otomatis dalam ±1 menit.

Domain sendiri (opsional): tambahkan file `CNAME` berisi domain Anda, lalu arahkan DNS `CNAME` → `USERNAME.github.io`.

## Catatan keamanan

- Endpoint publik hanya: kirim pendaftaran & statistik agregat. Semua akses data butuh **token sesi** dari login username/password.
- Hak akses tetap ditegakkan di server: Orang Tua hanya melihat anaknya (via identitas token), Guru tidak bisa menu admin, Admin terakhir tidak bisa dihapus/dinonaktifkan.
- Password disimpan sebagai hash SHA-256 + salt; salt/hash tidak pernah dikirim ke browser.
- Kuota Apps Script: 20.000 URL-fetch call/hari (akun gratis) — untuk sekolah, jauh lebih dari cukup.
