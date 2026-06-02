# Spec — Pertemuan 5: REST API CRUD (Pengganti SQLite Pertemuan 4)

**Tanggal:** 2026-06-02
**Status:** Draft → menunggu review user
**Konteks:** Lanjutan Pertemuan 4 (Flutter + SQLite). Mengganti persistensi lokal dengan backend Laravel via REST API.

---

## 1. Tujuan

Membuat materi praktikum Pertemuan 5 yang strukturnya **paralel dengan Pertemuan 4**, tetapi sumber data dipindah dari SQLite lokal (`sqflite`) ke **REST API** (Laravel di server publik). Fokus mahasiswa: sisi **Flutter** (client). Backend disediakan dosen dalam bentuk siap-deploy.

### Outcome
1. Mahasiswa memahami perbedaan I/O lokal (disk) vs jaringan (HTTP) dalam konteks async/await.
2. Mahasiswa bisa konsumsi endpoint REST CRUD dengan `package:http`.
3. Mahasiswa bisa serialisasi JSON ↔ Dart object (`toJson` / `fromJson`).
4. Mahasiswa bisa kirim header kustom (`X-API-Key`) dan memahami autentikasi sederhana.
5. Mahasiswa bisa menangani 3 kelas error baru: timeout, no-internet, HTTP 4xx/5xx.
6. UI/UX pengguna akhir **identik** dengan Pertemuan 4 (Home + Form Create/Edit + Detail + delete dgn konfirmasi).

### Non-goals
- Login/registrasi user (auth per-user). Pakai shared API key saja.
- Offline cache / sinkronisasi.
- State management lanjutan (Provider/Bloc/Riverpod). Tetap `setState` + `FutureBuilder`.
- Real-time (WebSocket/SSE).

---

## 2. Keputusan Desain (ringkas)

| Aspek | Pilihan | Alasan |
|---|---|---|
| Backend | Laravel 11 baru (`pertemuan-5-be/`) terpisah dari `catat-emas-laravel` | Tidak mengganggu proyek existing |
| Auth | `X-API-Key` shared (1 key untuk semua client) | Konsisten dgn pola `catat-emas-laravel`; fokus mahasiswa di CRUD HTTP |
| Deploy | Public URL (Railway/Fly.io/VPS), dideploy dosen | Setup mahasiswa nol; konsisten lintas device |
| HTTP client | `package:http` (resmi) | API sederhana, cukup untuk CRUD basic |
| Scope client | Ganti `DbHelper` → `ApiClient`, signature sama | UI tidak berubah; fokus pada perubahan I/O |
| Database BE | Postgres di production (Railway), SQLite untuk dev lokal | Default Laravel friendly |

---

## 3. Arsitektur

```
┌─────────────────────────────────────┐   HTTPS    ┌──────────────────────────┐
│   Flutter App (pertemuan_5)         │   X-API-   │  Laravel API (public)    │
│                                     │   Key      │                          │
│   HomePage  ─ FutureBuilder         │ ─────────▶ │  routes/api.php          │
│   Form      ─ insert / update       │            │   └─ middleware api.key  │
│   Detail    ─ view + tombol Edit    │   JSON     │  CatatanController       │
│                                     │ ◀───────── │   (resource controller)  │
│   ApiClient (singleton)             │            │  Model Catatan           │
│    ├─ getAll()  GET    /api/catatan │            │  migration catatan       │
│    ├─ getById() GET    /{id}        │            │  VerifyApiKey middleware │
│    ├─ create()  POST   /            │            │                          │
│    ├─ update()  PUT    /{id}        │            │  DB: Postgres (prod)     │
│    └─ delete()  DELETE /{id}        │            │      SQLite (dev)        │
└─────────────────────────────────────┘            └──────────────────────────┘
```

### Boundary
- **Mahasiswa hanya menyentuh** folder `pertemuan_5/` (Flutter).
- **Dosen menyiapkan & deploy** `pertemuan-5-be/` (Laravel).
- Kontrak antara keduanya = **REST API + JSON** (didokumentasikan di modul).

---

## 4. Backend Laravel (`pertemuan-5-be/`)

### 4.1 Struktur
```
pertemuan-5-be/
├── app/
│   ├── Http/
│   │   ├── Controllers/Api/CatatanController.php
│   │   └── Middleware/VerifyApiKey.php
│   └── Models/Catatan.php
├── database/
│   ├── migrations/2026_06_02_000001_create_catatan_table.php
│   └── seeders/CatatanSeeder.php
├── routes/api.php
├── bootstrap/app.php       # registrasi alias 'api.key'
├── .env.example            # API_KEY=dev-secret-123
└── README.md               # setup lokal + deploy Railway
```

### 4.2 Skema tabel `catatan`
| Kolom        | Tipe           | Nullable | Catatan                       |
|--------------|----------------|----------|-------------------------------|
| id           | bigint PK auto | tidak    |                               |
| judul        | string(150)    | tidak    |                               |
| isi          | text           | tidak    |                               |
| kategori     | string(50)     | tidak    | enum di sisi UI: Kuliah/Tugas/Pribadi/Lainnya |
| dibuat_pada  | timestamp      | tidak    | diisi server saat insert kalau tidak dikirim  |
| created_at   | timestamp      | tidak    | Laravel default               |
| updated_at   | timestamp      | tidak    | Laravel default               |

### 4.3 Endpoint
| Method | URL                  | Body                                           | Sukses                | Error                                  |
|--------|----------------------|------------------------------------------------|-----------------------|----------------------------------------|
| GET    | `/api/catatan`       | —                                              | 200 `{success, data:[…]}` | 401 (key salah)                     |
| GET    | `/api/catatan/{id}`  | —                                              | 200 `{success, data}` | 404                                    |
| POST   | `/api/catatan`       | `{judul, isi, kategori, dibuat_pada?}`         | 201 `{success, data}` | 422 (validasi), 401                    |
| PUT    | `/api/catatan/{id}`  | `{judul, isi, kategori}`                       | 200 `{success, data}` | 404, 422, 401                          |
| DELETE | `/api/catatan/{id}`  | —                                              | 200 `{success, message}` | 404, 401                            |

Semua endpoint **wajib** header `X-API-Key: <env API_KEY>`.

### 4.4 Format JSON `Catatan`
```json
{
  "id": 7,
  "judul": "Tugas Mobile",
  "isi": "Selesaikan modul P5",
  "kategori": "Tugas",
  "dibuat_pada": "2026-06-02T10:30:00Z"
}
```

### 4.5 Validasi (Laravel `FormRequest` atau inline)
- `judul`: required, string, max:150
- `isi`: required, string
- `kategori`: required, string, max:50
- `dibuat_pada`: optional, ISO-8601 (kalau kosong → server isi `now()`)

### 4.6 Response standar
- Sukses: `{ "success": true, "data": <object|array>, "message"?: <string> }`
- Error: `{ "success": false, "message": <string>, "errors"?: <object> }` (422 mengikut default Laravel di key `errors`)

### 4.7 Middleware `VerifyApiKey`
Salin pola dari `catat-emas-laravel/app/Http/Middleware/VerifyApiKey.php`. Bandingkan header `X-API-Key` dengan `config('app.api_key')`/env, jika tidak cocok → 401 JSON.

---

## 5. Frontend Flutter (`pertemuan_5/`)

### 5.1 Struktur
```
pertemuan_5/
├── lib/
│   ├── main.dart           # model Catatan + UI (HomePage, Detail, Form)
│   └── api_client.dart     # ← GANTIKAN db_helper.dart
└── pubspec.yaml            # http: ^1.2.0   (sqflite & path DIHAPUS)
```

### 5.2 Perubahan vs Pertemuan 4
| File            | Pertemuan 4              | Pertemuan 5                                   |
|-----------------|--------------------------|-----------------------------------------------|
| `pubspec.yaml`  | `sqflite`, `path`        | **`http`** saja                               |
| `main.dart`     | `WidgetsFlutterBinding.ensureInitialized()` wajib | Tidak wajib (no native plugin) |
| `Catatan` model | `toMap()` int epoch      | `toJson()/fromJson()` ISO-8601 string         |
| Repository      | `db_helper.dart`         | `api_client.dart` (signature 5 method sama)   |
| UI (Home/Form/Detail/Delete) | — | **Identik** — hanya nama panggilan repo berubah |

### 5.3 `ApiClient` (kontrak)
```dart
class ApiClient {
  ApiClient._();
  static final ApiClient instance = ApiClient._();

  static const _baseUrl = 'https://<deploy-url>/api';   // diatur dosen
  static const _apiKey  = 'dev-secret-123';             // sama dgn server

  Map<String, String> get _headers => {
    'X-API-Key': _apiKey,
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  };

  Future<List<Catatan>> getAll();
  Future<Catatan>       getById(int id);
  Future<Catatan>       insert(Catatan c);   // POST → return Catatan dgn id
  Future<Catatan>       update(Catatan c);   // PUT
  Future<void>          delete(int id);
}
```

### 5.4 Error handling
- HTTP timeout 10 detik → `TimeoutException` → tampilkan "Server tidak merespons".
- Status 4xx/5xx → throw `ApiException(statusCode, message)` → ditangkap di UI sebagai SnackBar / `snapshot.hasError`.
- `SocketException` (no internet) → "Periksa koneksi internet".
- Parsing error → throw generic + log.

### 5.5 Model `Catatan` — perubahan minimal
- `toJson()` mengirim `dibuat_pada` sebagai `dibuatPada.toUtc().toIso8601String()`.
- `fromJson()` parse `DateTime.parse(m['dibuat_pada'])`.
- Field `id` tetap nullable (server yang assign).

---

## 6. Struktur Modul Markdown (`pertemuan-5/MODUL_PERTEMUAN_5.md`)

Paralel 8 langkah dengan P4, durasi 120 menit:

| # | Langkah                                     | Durasi | Perubahan vs P4                                  |
|---|---------------------------------------------|--------|--------------------------------------------------|
| 1 | Setup project + dependency `http`           | 10'    | Copy dari P4 (atau P3), hapus sqflite/path       |
| 2 | Konsep REST & JSON (gantikan SQLite intro)  | 15'    | HTTP method, status code, JSON, header           |
| 3 | Refactor model: `toJson/fromJson`           | 10'    | ISO-8601 vs int epoch                            |
| 4 | `ApiClient` singleton + 5 method CRUD       | 25'    | Tabel kontrak endpoint + contoh request/response |
| 5 | Home pakai `FutureBuilder`                  | 15'    | **Identik P4** (highlight tetap sama)            |
| 6 | Form CREATE + EDIT                          | 20'    | **Identik P4** (highlight tetap sama)            |
| 7 | Delete + dialog konfirmasi                  | 10'    | **Identik P4**                                   |
| 8 | Polish + error handling network + tes manual| 15'    | Skenario: matikan Wi-Fi → muncul error           |

Modul juga akan menyertakan:
- Bagian "Yang berubah dari Pertemuan 4" (tabel perbandingan).
- Kontrak API (copy dari spec ini, ringkas).
- Catatan emulator: Android emulator akses `10.0.2.2` untuk localhost; production URL public bisa langsung.
- Checklist tes manual paralel P4 + 3 tes baru: offline, server mati, key salah.

---

## 7. Repo BE — Setup & Deploy (catatan dosen)

### Lokal
```bash
cd pertemuan-5-be
composer install
cp .env.example .env
php artisan key:generate
touch database/database.sqlite
php artisan migrate --seed
php artisan serve   # http://127.0.0.1:8000
```

### Deploy Railway (rekomendasi)
1. Push repo ke GitHub.
2. Railway → New Project → Deploy from GitHub → pilih repo.
3. Add Postgres plugin → Railway inject `DATABASE_URL`.
4. Set env: `APP_KEY`, `API_KEY`, `APP_ENV=production`, `DB_CONNECTION=pgsql`.
5. Jalankan migration via Railway shell: `php artisan migrate --force`.
6. Salin URL publik ke modul mahasiswa.

README BE akan berisi versi lengkap ini.

---

## 8. Rencana Pembuatan

1. **Backend** — scaffold Laravel 11 di `pertemuan-5-be/`, implement model + migration + controller + middleware + seeder + README.
2. **Modul Markdown** — `pertemuan-5/MODUL_PERTEMUAN_5.md` (8 langkah, paralel P4).
3. **(Opsional) Starter Flutter** — `pertemuan-5/starter/` dengan struktur kosong + pubspec siap (mahasiswa fokus isi `api_client.dart`). Akan diputuskan di plan.
4. **Smoke test** — jalankan BE lokal, test 5 endpoint via curl, dokumentasikan di README.

---

## 9. Risiko & Mitigasi

| Risiko                                   | Mitigasi                                                  |
|------------------------------------------|-----------------------------------------------------------|
| Server publik down saat praktikum        | Sediakan fallback: instruksi run BE lokal di README BE    |
| Mahasiswa salah base URL (emulator)      | Tabel khusus di modul: Android emulator vs iOS sim vs web |
| API key bocor                            | Key di modul = key dev. Production rotate setelah praktikum |
| CORS error (kalau flutter web)           | BE expose `Access-Control-Allow-Origin: *` di middleware  |
| Mahasiswa lupa header `Content-Type`     | Tunjukkan di langkah 4 + checklist tes                    |

---

## 10. Definition of Done

- [ ] Folder `pertemuan-5-be/` jalan lokal (5 endpoint pass curl test)
- [ ] BE punya README setup lokal + deploy Railway
- [ ] Modul `MODUL_PERTEMUAN_5.md` lengkap 8 langkah, format paralel P4
- [ ] Modul menyertakan kontrak API + tabel perbedaan vs P4
- [ ] Modul punya checklist tes manual (10+ skenario)
- [ ] Dokumen ini di-commit ke repo `materi-pengajaran`
