# StudySpark

Platform e-learning berbasis kuis dengan gamifikasi untuk siswa SMP/SMA/SMK.
Implementasi MVP dari PRD v1.0.

**🌐 Live:** <https://studyspark-snowy.vercel.app>
(frontend statis + FastAPI serverless, keduanya di Vercel)

> ⚠️ **Data masih sementara.** Belum ada `DATABASE_URL`, jadi aplikasi memakai
> SQLite di `/tmp` yang ikut terhapus setiap instance serverless dingin —
> pendaftaran dan progres bisa hilang sewaktu-waktu. Aplikasi tetap berfungsi
> penuh karena konten mata pelajaran terisi ulang otomatis. Satu perintah untuk
> menjadikannya permanen: [Menjadikan data permanen](#menjadikan-data-permanen).
>
> Cek status kapan saja di `/api/health` — field `data_permanen` akan bernilai
> `true` begitu database sungguhan tersambung.

**Stack:** FastAPI + SQLAlchemy + SQLite/PostgreSQL (backend), HTML + Bootstrap 5 + JavaScript vanilla (frontend), JWT (auth).

---

## Menjalankan secara lokal

```bash
cd backend && python -m venv .venv && .venv/Scripts/python.exe -m pip install -r requirements.txt
```

```bash
cd backend && cp .env.example .env && .venv/Scripts/python.exe seed.py
```

```bash
cd backend && .venv/Scripts/python.exe -m uvicorn app.main:app --reload --port 8000
```

Buka <http://127.0.0.1:8000> — FastAPI sekaligus menyajikan folder `frontend/`,
jadi cukup satu perintah untuk seluruh aplikasi. Dokumentasi API interaktif ada
di <http://127.0.0.1:8000/docs>.

**Akun demo** (semua password `password123`):

| Email | Peran |
|---|---|
| `cakra@studyspark.id` | siswa (akun kosong, untuk mencoba dari nol) |
| `nadia@studyspark.id` | siswa (sudah punya XP, mengisi leaderboard) |
| `admin@studyspark.id` | admin |

Konten awal: **5 mata pelajaran, 13 materi, 156 soal**, semuanya lengkap dengan
penjelasan dan referensi.

---

## Keputusan yang diambil untuk celah di PRD

PRD tidak mengatur beberapa hal yang harus punya jawaban pasti sebelum bisa
dikodekan. Berikut yang dipakai — semuanya terpusat di `backend/app/config.py`
sehingga bisa diubah di satu tempat.

### 1. Anti-farming XP (tidak ada di PRD)

Tanpa aturan ini, siswa bisa mengerjakan kuis → menghafal kunci dari halaman
pembahasan → mengulang → XP tak terbatas, dan leaderboard kehilangan makna.

- Percobaan **pertama** pada sebuah materi: XP penuh.
- Percobaan **kedua dan seterusnya**: XP jawaban benar dikali **0.25**, dan bonus
  (nilai sempurna, materi selesai) tidak diberikan lagi.
- Tiap materi punya **12 soal**, kuis mengambil **10 secara acak**, sehingga dua
  percobaan tidak pernah identik.

### 2. Kunci jawaban tidak pernah dikirim sebelum submit

`GET`/`POST` yang memulai kuis mengembalikan schema `QuestionPublic` yang tidak
memuat field `jawaban`, `penjelasan`, maupun `referensi`. Penilaian sepenuhnya di
server. Kunci dan pembahasan baru muncul di response submit.

### 3. Batas waktu ditegakkan server

`quiz_attempts.expires_at` diisi saat kuis dimulai. Timer JavaScript hanya untuk
tampilan — menghentikannya lewat DevTools tidak memberi keuntungan. Submit yang
lewat batas (+30 detik toleransi jaringan) tetap dinilai tapi ditandai
`is_expired`.

### 4. Streak

- Dipicu oleh **menyelesaikan kuis**, bukan sekadar login — supaya streak benar-benar
  mengukur aktivitas belajar. Bonus XP login harian (+20) tetap terpisah.
- Semua perhitungan "hari" memakai zona **Asia/Jakarta**, bukan UTC. Server
  produksi biasanya UTC, dan itu akan memutus streak siswa pada jam 07:00 WIB.
- Melewatkan satu hari mengembalikan streak ke 0, sesuai PRD §16.

### 5. Leaderboard berperiode

Tabel `Leaderboard` di PRD §23 dihapus (tidak punya kolom waktu, jadi filter
harian/mingguan/bulanan mustahil dihitung) dan diganti tabel event
**`xp_transactions`** — tiap perolehan XP dicatat dengan timestamp. Filter periode
menjadi query agregasi biasa, dan sebagai bonus ada jejak audit XP.

### 6. Kurva level

PRD hanya mendefinisikan level 1–4 lalu "dst". Yang dipakai:

| Level | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| XP | 0 | 500 | 1.000 | 2.000 | 3.500 | 5.500 | 8.000 | 11.000 | 14.500 | 18.500 |

Di atas level 10, tiap level butuh tambahan 5.000 XP.

### 7. Kuis yang ditinggalkan

Kalau siswa menutup browser di tengah kuis lalu kembali:
- Masih dalam batas waktu → sesi **dilanjutkan** dengan soal dan jawaban yang sama
  (jawaban sementara disimpan di `sessionStorage`).
- Sudah lewat batas waktu → sesi ditandai `batal` dan **tidak dihitung** sebagai
  percobaan, jadi siswa tidak dirugikan.

### 8. Rumus nilai

`nilai = benar / total_soal × 100` (bukan `benar × 10` yang hanya benar untuk 10
soal). Materi dianggap selesai bila nilai ≥ **80**.

### 9. Contoh XP di PRD §12 tidak konsisten

PRD mencontohkan 8 benar → +120 XP, padahal aturan §14 menghasilkan 8 × 10 = 80.
Yang diimplementasikan adalah aturan §14, karena itu yang konsisten.

---

## Struktur proyek

```
backend/
  app/
    config.py         Semua aturan gamifikasi + setting (satu sumber kebenaran)
    database.py       Engine & session SQLAlchemy
    models.py         Skema database
    schemas.py        Request/response Pydantic
    security.py       Hash password (bcrypt) & JWT
    deps.py           Dependency auth
    gamification.py   Level, XP, streak, badge
    queries.py        Query yang dipakai beberapa router
    ratelimit.py      Pembatas percobaan login
    timeutil.py       Helper UTC vs WIB
    routers/          auth, subjects, quiz, leaderboard, dashboard, profile
  seed_data.py        Isi konten: mata pelajaran, materi, 156 soal, badge
  seed.py             Script pengisian database
frontend/
  index.html          Landing
  login.html  register.html
  dashboard.html
  subjects.html  materials.html
  quiz.html  result.html
  leaderboard.html  achievements.html  profile.html
  css/style.css       Tema glassmorphism
  js/api.js           Wrapper fetch + sesi
  js/ui.js            Navbar, toast, escaping, format
  js/dashboard.js  js/quiz.js  js/result.js  js/profile.js
```

---

## Catatan keamanan

- Password di-hash dengan **bcrypt**; kolomnya bernama `password_hash`.
- Login dibatasi **8 percobaan per 5 menit** per kombinasi IP+email (in-memory —
  ganti ke Redis kalau backend di-scale ke banyak instance).
- Pesan error login sengaja sama untuk email salah maupun password salah, supaya
  tidak bisa dipakai menebak email yang terdaftar.
- Semua data dari API dirender lewat fungsi `esc()` untuk mencegah XSS.
- Token disimpan di `localStorage`. Untuk produksi pertimbangkan httpOnly cookie
  (lebih aman terhadap XSS, tapi butuh konfigurasi CORS + `SameSite=None; Secure`
  karena frontend dan backend beda domain).

---

## Deployment

### Vercel (yang sedang dipakai)

Vercel menghosting **keduanya** dalam satu domain, jadi frontend dan API berada di
origin yang sama dan tidak butuh konfigurasi CORS.

| File | Fungsi |
|---|---|
| `vercel.json` | `/api/*`, `/docs` → serverless function; sisanya → `frontend/` |
| `api/index.py` | Entrypoint ASGI; menambahkan `backend/` ke `sys.path` lalu meng-export `app` |
| `requirements.txt` (root) | Dependency untuk Python runtime Vercel |
| `.vercelignore` | Menahan `.venv`, `.env`, dan file `.db` agar tidak ikut ter-upload |

Deploy ulang:

```bash
vercel deploy --prod
```

Environment variable yang sudah diset di project: `JWT_SECRET` (production +
preview). `VERCEL=1` diisi otomatis oleh platform dan dipakai `config.py` untuk
mematikan `SERVE_FRONTEND` serta memindahkan SQLite ke `/tmp`.

### Menjadikan data permanen

Jalankan satu perintah ini, lalu **pilih plan Free** saat diminta:

```bash
vercel integration add neon
```

Perintah ini harus dijalankan manusia di terminal interaktif — Vercel
mensyaratkan persetujuan legal Marketplace tidak boleh diotomatiskan. Neon punya
plan Free (0,5 GB) yang tidak meminta kartu kredit.

Sesudahnya deploy ulang, dan selesai:

```bash
vercel deploy --prod
```

Tidak ada langkah manual lain. Aplikasi sudah menangani sisanya sendiri:

- **Nama variabel** — `config.py` mengenali `DATABASE_URL`, `POSTGRES_URL`,
  `POSTGRES_PRISMA_URL`, dan `DATABASE_POSTGRES_URL`, jadi penyedia mana pun cocok.
- **Format URL** — `postgres://` dan `postgresql://` otomatis dinormalkan menjadi
  `postgresql+psycopg://`. Tanpa ini SQLAlchemy mencari psycopg2 yang tidak dipasang.
- **Connection pool** — memakai `NullPool` di serverless supaya koneksi basi tidak
  menumpuk dan menghabiskan kuota koneksi database.
- **Isi konten** — tabel dibuat dan 156 soal diisi otomatis saat startup pertama
  bila database masih kosong. Tidak perlu menjalankan `seed.py`.

Alternatif penyedia lain (Supabase, Prisma Postgres) juga bisa — ganti `neon`
dengan slug yang muncul di `vercel integration discover postgres`.

### Alternatif: backend terpisah (rencana asli PRD §5)

Kalau nanti backend dipindah ke Railway/Render:

1. Pakai `backend/Procfile` sebagai start command.
2. Set `DATABASE_URL`, `JWT_SECRET`, `CORS_ORIGINS` (domain Vercel), `SERVE_FRONTEND=false`.
3. Di frontend, isi `js/config.js` dengan URL backend lalu sisipkan
   `<script src="js/config.js"></script>` sebelum `js/api.js` di tiap halaman.

Catatan free tier: Render menidurkan instance setelah 15 menit idle (cold start
~50 detik) dan menghapus database PostgreSQL gratis setelah 90 hari.

---

## Yang belum ada (sesuai batasan MVP di PRD §28)

| Fitur | Status | Catatan |
|---|---|---|
| Forgot password / OTP | belum | Butuh layanan email (SendGrid/Resend) yang belum ada di tech stack |
| Admin dashboard | belum | Sementara pakai `seed.py`; skema sudah punya `User.role` |
| Daily challenge | belum | Perlu definisi: acak dari mana, sama untuk semua siswa atau tidak, reward berapa |
| Notifikasi | belum | Untuk V1 sebaiknya banner in-app saja; push butuh service worker + scheduler |
| Upload foto ke Cloudinary | sebagian | Profil menerima URL gambar; upload bertanda tangan belum dibuat |
| Migrasi database | belum | Sekarang pakai `create_all`. Ganti ke Alembic saat skema mulai berubah di produksi |

**Perlu perhatian sebelum rilis publik:** target pengguna utama adalah siswa SMP
(12–15 tahun, secara hukum anak-anak). UU No. 27/2022 tentang Perlindungan Data
Pribadi mensyaratkan persetujuan orang tua untuk pemrosesan data anak. Belum ada
privacy policy, mekanisme consent, maupun opsi memakai nama samaran di
leaderboard.
