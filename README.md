# Asisten Kesiapan Akreditasi Program Studi

Aplikasi web untuk membantu program studi **mengukur kesiapan akreditasi sebelum asesmen
lapangan**. Program studi mengunggah LED, LKPS, ekspor PDDIKTI, dan dokumen bukti. Sistem
membaca dokumen, mencocokkannya dengan butir instrumen akreditasi, lalu menampilkan nilai saat
ini, temuan, dan tindak lanjut yang perlu dikerjakan. Reviewer internal memeriksa setiap
usulan sebelum menjadi hasil akhir.

Isi instrumen saat ini: **LAM INFOKOM, Instrumen 2.1** (jalur Terakreditasi dan Unggul, S1 dan
S2). Antarmuka tidak terikat pada satu lembaga: nama lembaga, ambang status, dan butir dibaca
dari berkas profil instrumen di `rubrics/`.

| Masuk | Dasbor |
|---|---|
| ![Halaman masuk](docs/img/masuk.png) | ![Dasbor proyek](docs/img/ringkasan.png) |
| **Telaah butir** | **Kelola pengguna** |
| ![Telaah butir](docs/img/telaah.png) | ![Kelola pengguna](docs/img/pengguna.png) |
| **Tanya AI** | **Uji Akurasi** |
| ![Tanya AI](docs/img/tanya-ai.png) | ![Uji Akurasi](docs/img/gladi_uji_akurasi.png) |

*Semua data pada tangkapan layar adalah fiktif.*

---

## Daftar isi

1. [Cara kerja singkat](#cara-kerja-singkat)
2. [Prinsip yang tidak boleh dilanggar](#prinsip-yang-tidak-boleh-dilanggar)
3. [Coba cepat: mode demo (5 menit)](#coba-cepat-mode-demo-5-menit)
4. [Menjalankan versi lengkap](#menjalankan-versi-lengkap)
5. [Model AI](#model-ai)
6. [Peran pengguna](#peran-pengguna)
7. [Struktur folder](#struktur-folder)
8. [Untuk pengembang](#untuk-pengembang)
9. [Status dan pekerjaan berikutnya](#status-dan-pekerjaan-berikutnya)
10. [Masalah yang sering muncul](#masalah-yang-sering-muncul)

---

## Cara kerja singkat

```mermaid
flowchart LR
    A[Unggah LED, LKPS,<br/>PDDIKTI, bukti] --> B[Baca dokumen<br/>sel LKPS, teks, OCR]
    B --> C[Fakta<br/>+ cek konflik]
    B --> D[Cari bukti<br/>BM25 + bge-m3]
    D --> E[Model AI mengutip bukti<br/>qwen3:8b + cek ganda]
    E --> F[Validator kutipan]
    C --> G[Mesin aturan<br/>hitung skor dari rubrik]
    F --> G
    G --> H[Temuan + tindak lanjut<br/>Ringkasan AI]
    H --> I[Reviewer menelaah]
    I --> J[Laporan resmi atau draf<br/>PDF / XLSX]
    I --> K[Uji akurasi<br/>vs reviewer dan hasil resmi]
```

1. **Membaca dokumen.** Angka LKPS dibaca langsung dari sel Excel (bukan oleh AI). PDF dan
   DOCX diambil teksnya. Halaman hasil pindai dan gambar tertempel dibaca dengan OCR
   (Tesseract) dan model visual.
2. **Mengecek fakta.** Angka dari LKPS dibandingkan dengan ekspor PDDIKTI. Bila berbeda, butir
   ditandai **konflik** dan menunggu keputusan manusia. Proyek yang belum punya reviewer boleh
   diputuskan penyusun dengan memilih salah satu nilai sumber beserta alasan.
3. **Mencari bukti.** Untuk butir yang perlu bukti tertulis (misalnya dokumen SPMI), sistem
   mencari potongan dokumen yang relevan, lalu model AI menunjuk kutipan buktinya. Jawaban yang
   meragukan ditanyakan ulang (**cek ganda**). Bila tidak konsisten, butir wajib ditelaah.
4. **Menghitung skor.** Skor dihitung **mesin aturan** dari rubrik JSON berversi, bukan oleh AI.
   Dasbor menampilkan **satu angka nilai**: nilai bila diajukan sekarang (butir yang belum
   terbukti dihitung skor terendah rubrik) dan seberapa tinggi nilai itu bisa naik.
5. **Ringkasan AI.** Setelah pemeriksaan selesai, Ringkasan AI disusun otomatis sebagai satu
   pesan percakapan. Setiap angka dan kode butir diperiksa terhadap data run.
6. **Telaah dan laporan.** Reviewer menyetujui, menolak, atau mengoreksi. Laporan resmi
   diterbitkan reviewer dan tidak pernah ditimpa. Laporan draf bercap "DRAF" dapat diunduh
   kapan saja, juga tanpa reviewer.
7. **Uji akurasi.** Menu Uji Akurasi menghitung kesepakatan sistem dengan reviewer (beserta
   penyebab selisih) dan membandingkan sistem serta reviewer dengan hasil asesmen resmi yang
   diunggah lewat templat XLSX. Telaah buta dapat diaktifkan per proyek (ADR-49).
8. **Tanya AI.** Asisten percakapan: sapaan dan pertanyaan umum, pengetahuan akreditasi dari
   instrumen resmi, hasil pemeriksaan, perbandingan antar pemeriksaan, dan kemajuan proyek.
   Tanya AI hanya membaca dan tidak pernah mengubah skor.

## Prinsip yang tidak boleh dilanggar

- **AI tidak pernah memberi skor.** Skor hanya dari `akreditasi/rules/` atas rubrik di `rubrics/`.
- **Angka LKPS dibaca dari sel**, bukan ditebak AI.
- **Konflik fakta tidak diselesaikan otomatis.** Butir yang bergantung ditangguhkan.
- **Bukti belum ditemukan tidak sama dengan skor nol.** Skornya berupa rentang (misalnya 1
  sampai 4) dan nilai saat ini memakai skor terendah rubrik, bukan nol.
- **Isi dokumen adalah data, bukan perintah** (kebal terhadap kalimat jebakan di dokumen).
- **Dokumen prodi asli tidak boleh dikirim ke server model publik** dan tidak boleh ditaruh di
  repositori ini. Folder `data/` diabaikan git.

---

## Coba cepat: mode demo (5 menit)

Mode demo menjalankan **tampilan web saja** dengan data contoh di dalam peramban. Tidak perlu
basis data, Docker, atau server AI. Cocok untuk melihat-lihat dan mengerjakan tampilan.

**Yang perlu dipasang:** [Git](https://git-scm.com/downloads) dan
[Node.js 22 atau lebih baru](https://nodejs.org/) (pilih versi LTS).

Buka **PowerShell** (Windows) atau **Terminal** (macOS/Linux), lalu jalankan:

```bash
git clone https://github.com/sofwanrsd/ai-accreditation-assistant.git
cd ai-accreditation-assistant/web
npm ci
npm run dev:mock
```

Buka alamat yang muncul (biasanya **http://localhost:5173**), lalu masuk dengan:

| Surel | Kata sandi |
|---|---|
| surel apa saja, misalnya `demo@contoh.ac.id` | `demo-akreditasi` |

Di mode demo ada tombol **Mode demo** di kanan atas untuk berganti peran (penyusun, reviewer,
pimpinan, admin) dan melihat perbedaan hak aksesnya. Hentikan dengan `Ctrl + C`.

---

## Menjalankan versi lengkap

Versi lengkap menjalankan semuanya: basis data, penyimpanan berkas, API, pekerja latar
belakang, dan web. Ada dua tempat pemasangan:

| Tempat | Alamat | Panduan |
|---|---|---|
| Laptop pengembang (data fiktif) | **https://localhost**, hanya dari laptop itu | Langkah di bawah |
| **Komputer CoE** (data asli, model AI lokal, dibuka dari PC lain di LAN) | misalnya **https://akreditasi.coe.lan** | [`docs/PASANG_DI_KOMPUTER_COE.md`](docs/PASANG_DI_KOMPUTER_COE.md) |

Langkah berikut untuk laptop pengembang.

### 1. Pasang perangkat

| Perangkat | Untuk apa | Unduh |
|---|---|---|
| Git | Mengunduh kode | https://git-scm.com/downloads |
| Docker Desktop | Menjalankan basis data, API, web | https://www.docker.com/products/docker-desktop/ |
| Node.js 22+ (LTS) | Membangun tampilan web | https://nodejs.org/ |
| uv | Menjalankan Python 3.13 (otomatis dipasang uv) | https://docs.astral.sh/uv/getting-started/installation/ |

Setelah memasang Docker Desktop, **buka aplikasinya** dan tunggu sampai statusnya *running*.

### 2. Unduh kode dan siapkan pengaturan

```bash
git clone https://github.com/sofwanrsd/ai-accreditation-assistant.git
cd ai-accreditation-assistant
uv sync --all-extras --group dev
uv run python deploy/scripts/siapkan_env.py
```

Perintah terakhir membuat berkas `deploy/.env` berisi kata sandi dan kunci acak. Mode laptop
ini membuat Caddy hanya mendengar di `127.0.0.1` dan mengaktifkan masuk pengembangan
(dev-login) untuk skrip data contoh. **Jangan pernah meng-commit atau membagikan berkas ini.**

| Pilihan `siapkan_env.py` | Hasil |
|---|---|
| (tanpa pilihan) | Laptop, tanpa model AI, dev-login aktif hanya lewat localhost |
| `--model-demo` | Server model demo publik. **Hanya data fiktif** |
| `--model-lokal [URL]` | Ollama di komputer ini atau LAN (bawaan `http://host.docker.internal:11434/v1`), dev-login mati |
| `--alamat HOST` | Nama situs untuk LAN, misalnya `akreditasi.coe.lan`. Caddy terbuka untuk LAN, dev-login mati |
| `--timpa` | Membuat ulang `deploy/.env` yang sudah ada |

### 3. Bangun tampilan web

```bash
cd web
npm ci
npm run build
cd ..
```

### 4. Nyalakan basis data dan penyimpanan berkas

```bash
docker compose -f deploy/compose.yaml --env-file deploy/.env up -d postgres garage
```

Lalu siapkan penyimpanan berkas (cukup sekali):

- **Windows (PowerShell):** `powershell -ExecutionPolicy Bypass -File deploy\scripts\garage-init.ps1`
- **macOS/Linux/Git Bash:** `sh deploy/scripts/garage-init.sh`

### 5. Nyalakan semua layanan

```bash
docker compose -f deploy/compose.yaml --env-file deploy/.env up -d --build
```

Pertama kali bisa 5 sampai 10 menit karena Docker membangun citra. Cek statusnya:

```bash
docker compose -f deploy/compose.yaml --env-file deploy/.env ps
```

Tunggu sampai `api` dan `worker` berstatus **healthy**.

### 6. Buat akun admin pertama

```bash
docker compose -f deploy/compose.yaml --env-file deploy/.env exec api akreditasi pengguna buat --surel admin@kampus.ac.id --nama "Admin Sistem" --admin-sistem
```

Masukkan kata sandi (minimal 10 karakter) dua kali. Karakter yang diketik memang tidak tampil.

### 7. (Opsional) Isi data contoh

```bash
uv run python deploy/scripts/smoke.py
```

Skrip ini membuat proyek contoh berisi LKPS dan LED **fiktif**, menjalankan pemeriksaan, dan
menerbitkan laporan. Keluaran `0` berarti semua langkah lulus. Skrip memerlukan dev-login,
jadi hanya berjalan di mode laptop.

### 8. Buka aplikasi

1. Buka **https://localhost**. Peramban akan memperingatkan sertifikat tidak dikenal (wajar
   untuk laptop sendiri): pilih **Advanced → Proceed to localhost**.
2. Masuk dengan akun admin dari langkah 6.
3. Siapkan proyek dan timnya langsung dari web, tanpa skrip:
   1. **Buat proyek.** Bila belum ada proyek, halaman awal menampilkan tombol **Buat proyek**.
      Setelahnya tombol yang sama ada di pemilih proyek (pojok kiri atas). Pilih **lembaga
      akreditasi** (daftarnya berasal dari profil instrumen yang sudah diunggah), isi program
      studi, jenjang, dan periode, lalu pilih profil instrumen dan peta sel LKPS. Target tidak
      dipilih: status Terakreditasi atau Unggul ditentukan otomatis dari nilai akhir (ADR-53). Proyek
      baru langsung menjadi proyek aktif. Bila belum ada profil atau peta sel yang cocok, minta
      admin instrumen mengunggahnya di **Pengaturan → Profil instrumen** (profil draf boleh
      dipakai, proyeksi status baru tampil setelah profil disahkan).
   2. **Buat akun.** Buka **Pengaturan → Pengguna → Tambah pengguna** untuk akun penyusun,
      reviewer, atau pimpinan. Kata sandi yang Anda buat bersifat sementara. Pengguna wajib
      menggantinya saat masuk pertama, dan server menolak permintaan lain sampai diganti. Nama
      dan peran global akun dapat diubah lewat menu baris **⋯ → Ubah data akun**.
   3. **Tambahkan anggota.** Buka **Pengaturan → Anggota proyek → Tambah anggota**, pilih akun
      (atau undang lewat surel) dan perannya. Di halaman yang sama peran anggota dapat diubah dan
      anggota dapat dikeluarkan. Setiap proyek wajib memiliki minimal satu reviewer.

> Admin sistem mengelola proyek, akun, dan keanggotaan, bukan isi penilaian. Untuk melihat
> ringkasan, hasil, dan telaah, masuk dengan akun anggota proyek (penyusun, reviewer, atau
> pimpinan).

### Mematikan dan menyalakan lagi

```bash
docker compose -f deploy/compose.yaml --env-file deploy/.env stop     # matikan
docker compose -f deploy/compose.yaml --env-file deploy/.env start    # nyalakan lagi
```

Data tersimpan di volume Docker dan tetap ada setelah laptop dimatikan.

---

## Model AI

Tanpa model AI, aplikasi tetap berjalan: angka LKPS/PDDIKTI tetap dinilai, sedangkan butir yang
perlu bukti tertulis tampil sebagai **rentang skor**. Dengan model AI, sistem mencari dan
mengutip bukti dari dokumen, menyusun Ringkasan AI dan Saran AI, dan menjawab Tanya AI.

| Tugas | Model | Keterangan |
|---|---|---|
| Mencari dan mengutip bukti, Ringkasan AI, Saran AI, Tanya AI | `qwen3:8b` | Mode berpikir aktif, sekitar 1 sampai 2 menit per butir atau per pertanyaan |
| Membaca halaman pindaian atau foto | `gemma3:4b` | Satu-satunya model visual, hasilnya dibandingkan dengan OCR |
| Pencarian berdasarkan makna | `bge-m3:latest` | Digabung dengan pencarian kata kunci (BM25) |

Pengaturan bawaan di `deploy/.env.example`:

- **Prompt v3** (`AKREDITASI_MODEL_PROMPT_VERSION=v3`) dan **cek ganda**
  (`AKREDITASI_MODEL_DOUBLE_CHECK=1`). Gladi fiktif 17 Oktober 2026: 6 dari 6 benar tanpa
  positif palsu.
- **Satu panggilan model sekaligus** (`AKREDITASI_MODEL_MAX_CONCURRENCY=1`) dan run penilaian
  satu per satu (`AKREDITASI_JOBS_SERIAL_RUNS=1`).
- **Langkah diulang sampai berhasil.** Galat sementara server model (HTTP 5xx, 524, koneksi
  putus, keluaran rusak) diulang dengan jeda bertambah. Langkah run (pemetaan bukti per butir,
  ekstraksi fakta, embeddings) diulang selama `AKREDITASI_MODEL_STEP_RETRY_BUDGET_S` (2 jam).
  Bila server model lama mati, run menunggu lalu melanjutkan butir yang belum selesai.

**GPU sekitar 6 GB VRAM.** Ketiga model tidak muat bersamaan, sehingga Ollama menukar model.
Atur Ollama dengan `OLLAMA_KEEP_ALIVE=30m` (model tidak dimuat ulang di antara butir),
`OLLAMA_MAX_LOADED_MODELS=1`, dan `OLLAMA_NUM_PARALLEL=1`, lalu jalankan ulang Ollama. Catatan
pengujian: [`docs/LOG_SERVER_MODEL.md`](docs/LOG_SERVER_MODEL.md) §3f.

**Data asli: model lokal.** Pasang Ollama di komputer yang sama (atau di LAN) dan buat
pengaturan dengan:

```bash
ollama pull qwen3:8b
ollama pull gemma3:4b
ollama pull bge-m3
uv run python deploy/scripts/siapkan_env.py --model-lokal --timpa
```

Panduan lengkap untuk komputer CoE: [`docs/PASANG_DI_KOMPUTER_COE.md`](docs/PASANG_DI_KOMPUTER_COE.md).

**Latihan dengan data fiktif: model demo.**

```bash
uv run python deploy/scripts/siapkan_env.py --model-demo --timpa
```

> ⚠️ **Server model demo terbuka ke internet tanpa autentikasi.** Jangan pernah mengunggah
> dokumen prodi asli saat memakai `--model-demo`. Untuk data asli, gunakan `--model-lokal`.

Jalankan `--timpa` hanya **sebelum** langkah 4 pertama kali (kata sandi basis data ikut
berubah). Bila layanan sudah pernah dinyalakan, ubah saja baris `AKREDITASI_MODEL_*` dan
`AKREDITASI_API_MODEL_ENABLED` di `deploy/.env` secara manual lalu jalankan ulang `api` dan
`worker`.

---

## Peran pengguna

| Peran | Dapat melakukan |
|---|---|
| **Penyusun** | Mengunggah dokumen, menjalankan pemeriksaan, mengerjakan tindak lanjut, mengunduh laporan draf. Memutuskan konflik fakta hanya bila proyek belum memiliki reviewer |
| **Reviewer** | Semua milik penyusun, ditambah menyetujui, menolak, atau mengoreksi skor, memutuskan konflik fakta, menerbitkan laporan resmi, mengisi pembanding resmi di Uji Akurasi, membuat ulang Ringkasan AI |
| **Pimpinan** | Melihat ringkasan, hasil, laporan resmi, dan laporan draf (baca saja), memakai Tanya AI |
| **Admin instrumen** | Mengunggah dan mengesahkan profil instrumen (rubrik) |
| **Admin sistem** | Membuat proyek dan akun, mengubah data akun, mengatur ulang kata sandi, mengatur anggota proyek dan perannya |

Proyek tanpa reviewer tetap dapat dipakai untuk cek mandiri sebelum pengajuan: laporan draf
bercap "DRAF: belum ditelaah reviewer" dapat diunduh kapan saja, dan konflik fakta diputuskan
penyusun (ditandai perlu ditinjau bila reviewer ditambahkan). Telaah dan laporan resmi menunggu
reviewer (ADR-48).

Panduan pemakaian lengkap: [`docs/PANDUAN_PENGGUNA.md`](docs/PANDUAN_PENGGUNA.md).

---

## Struktur folder

| Folder | Isi |
|---|---|
| `akreditasi/` | Kode inti Python: pembaca dokumen, LKPS, PDDIKTI, mesin aturan, API, pekerja |
| `web/` | Tampilan web (React + Vite + Tailwind + shadcn/ui) |
| `rubrics/` | Profil instrumen (rubrik JSON), peta sel LKPS, templat LKPS, contoh fiktif |
| `prompts/` | Templat perintah untuk model AI (berversi, bawaan `v3`) |
| `deploy/` | Docker Compose, Caddy, skrip penyiapan, cadangan, pemantauan |
| `migrations/` | Perubahan skema basis data (Alembic) |
| `tests/` | Uji otomatis Python |
| `docs/` | Dokumentasi, catatan keputusan, panduan, telaah keamanan |
| `instrumen_resmi/` | Paket resmi LAM INFOKOM (jangan diubah) |
| `eval/` | Jawaban kunci dan metrik akurasi |
| `AKR-0*.pdf` | Dokumen rancangan asli (baca urut: 00 → 01 → 02 → 05 → 04 → 03, 06, 07) |

Inventaris berkas lengkap: [`docs/DAFTAR_BERKAS.md`](docs/DAFTAR_BERKAS.md).

---

## Untuk pengembang

**Baca dulu:** [`CLAUDE.md`](CLAUDE.md) (prinsip, konvensi, perintah) dan
[`docs/PERUBAHAN_RANCANGAN.md`](docs/PERUBAHAN_RANCANGAN.md) (catatan keputusan ADR-19 sampai
ADR-50).

```bash
# Python (sama dengan CI)
uv run pytest -m "not live and not db"          # uji (tanpa jaringan, tanpa basis data)
uv run ruff check .                             # gaya kode
uv run ruff format --check akreditasi tests scripts eval rubrics/tools   # rapikan: tanpa --check
uv run mypy akreditasi                          # cek tipe
uv run python scripts/lint_istilah.py           # istilah baku, tanpa ; dan tanda pisah
uv run akreditasi --help                        # CLI

# Uji basis data (butuh Docker)
docker run -d --name akr-pg-test -e POSTGRES_PASSWORD=test -p 55432:5432 pgvector/pgvector:pg18
AKREDITASI_TEST_DATABASE_URL=postgresql://postgres:test@localhost:55432/postgres uv run pytest -m db

# Web
cd web
npm run dev:mock        # tampilan dengan data contoh
npm run lint
npm run typecheck
npm test                # uji unit (Vitest)
npm run check:api       # schema.d.ts sinkron dengan openapi.json
npm run build
npm run test:e2e        # uji ujung ke ujung + aksesibilitas (Playwright)
```

CI (`.github/workflows/ci.yml`) menjalankan semua perintah di atas, ditambah uji basis data
PostgreSQL 18 + pgvector, Playwright, `caddy validate`, `docker compose config`, dan
pemeriksaan konfigurasi Prometheus serta Alertmanager.

Aturan singkat:

- Semua teks untuk pengguna memakai **Bahasa Indonesia baku** (EYD V) dengan istilah: unggah,
  berkas, telaah, temuan, butir, persyaratan, rentang skor. Tanpa titik koma dan tanda pisah.
- Bila mengubah API, perbarui kontrak web:
  `uv run python -m akreditasi.api.openapi --output web/openapi.json` lalu `cd web && npm run gen:api`.
- Setiap hasil membawa `run_id` dan `rubric_version`. Hasil lama tidak pernah ditimpa.

Referensi teknis lengkap (semua perintah CLI, API, CI, penampil tahap 0):
[`docs/REFERENSI.md`](docs/REFERENSI.md). Penerapan di server:
[`deploy/README.md`](deploy/README.md). Telaah keamanan: [`docs/KEAMANAN.md`](docs/KEAMANAN.md).

---

## Status dan pekerjaan berikutnya

**Sudah jadi:** pembacaan LKPS, PDDIKTI, dokumen, pindaian, dan gambar tertempel, mesin aturan
dengan satu angka nilai, temuan, telaah (termasuk telaah buta dan antrean berprioritas), laporan
resmi dan laporan draf, Uji Akurasi dengan pembanding resmi, Ringkasan AI otomatis, Saran AI,
Tanya AI (umum, pengetahuan, hasil, perbandingan, kemajuan proyek), cek ganda, ulangan langkah
model untuk GPU kecil, API, pekerja latar belakang, aplikasi web, masuk dengan surel dan kata
sandi, kelola proyek, anggota, dan pengguna dari web, pencarian bge-m3, pemantauan, cadangan,
dan pemasangan on-premise dengan model lokal. Semua diuji dengan data fiktif.

**Menunggu pihak lain:**

- Dokumen prodi asli untuk uji akurasi tahap 0 ([`docs/PROSEDUR_TAHAP0.md`](docs/PROSEDUR_TAHAP0.md)).
- Penelaah domain untuk jawaban kunci dan keputusan terbuka
  ([`docs/TEMUAN_INSTRUMEN_2.1.md`](docs/TEMUAN_INSTRUMEN_2.1.md)).
- Bobot resmi jalur Terakreditasi (`bobot_verified: false`).
- Komputer CoE dengan GPU untuk pemasangan on-premise dan uji kinerja model lokal.

**Bisa dikerjakan berikutnya:**

- Tampilkan nomor gambar pada lokasi kutipan di web (`Locator.image`).
- Layar reviewer untuk memutuskan laporan kesalahan Tanya AI.
- Pratinjau dan pemetaan kolom ekspor PDDIKTI di layar, dan pembatalan run dari layar.
- Pisahkan berkas env per layanan Docker (temuan 11 di [`docs/KEAMANAN.md`](docs/KEAMANAN.md)).

---

## Masalah yang sering muncul

| Gejala | Penyebab dan solusi |
|---|---|
| `npm` atau `node` tidak dikenal | Node.js belum terpasang atau terminal belum dibuka ulang setelah pemasangan. |
| `docker` tidak dikenal atau *cannot connect to the Docker daemon* | Buka aplikasi Docker Desktop dan tunggu sampai *running*. |
| `api` tidak pernah *healthy* | Langkah 4 (`garage-init`) belum dijalankan. Jalankan, lalu ulangi langkah 5. |
| Peramban memperingatkan sertifikat | Wajar di laptop sendiri. Pilih *Advanced → Proceed to localhost*. Untuk komputer CoE, pasang sertifikat akar Caddy (panduan CoE langkah 9). |
| Port 443 sudah dipakai | Matikan aplikasi lain yang memakai port 443 (misalnya XAMPP, IIS, Skype lama). |
| PC lain tidak dapat membuka aplikasi di laptop | Benar demikian: mode laptop hanya mendengar di `127.0.0.1`. Untuk LAN, ikuti panduan CoE. |
| Tampilan tidak berubah setelah mengubah kode web | Jalankan ulang `npm run build`, lalu muat ulang peramban dengan `Ctrl + Shift + R`. |
| Masuk sebagai admin tetapi muncul "belum menjadi anggota proyek" | Benar demikian: admin hanya mengelola akun. Masuk dengan akun penyusun atau reviewer. |
| Muncul "Proyek ini belum memiliki reviewer" dan tombol "Terbitkan laporan" tidak ada | Wajar untuk cek mandiri: pakai **Unduh laporan draf**. Untuk laporan resmi dan telaah, admin sistem menambahkan reviewer di **Anggota proyek**. |
| Pemeriksaan lama berstatus berjalan saat server model mati | Run menunggu server model dan melanjutkan butir yang belum selesai setelah server kembali. Tanpa model, sistem tetap memberi rentang skor. |
| `smoke.py` gagal di dev-login (404) | dev-login mati (mode CoE atau `--alamat`). Skrip data contoh hanya untuk mode laptop. |
