# pertemuan_4 — Catatan Mahasiswa (Persistensi SQLite + CRUD)

Lanjutan dari Pertemuan 3. Sekarang data catatan **tidak hilang** saat aplikasi ditutup,
karena disimpan di database SQLite lokal lewat package `sqflite`.

Mengangkat konsep inti:

1. **Async / await & Future** — operasi I/O (database) berjalan asinkron
2. **`FutureBuilder`** — menampilkan UI berbeda untuk state loading / error / data
3. **SQLite lewat `sqflite`** — tabel, query, parameter binding, migrasi versi
4. **CRUD penuh** — Create, Read, Update, Delete
5. **Repository pattern sederhana** — pisahkan akses DB ke `db_helper.dart`
6. **`toMap()` / `fromMap()`** — bridge antara objek Dart ↔ row database

## Cara menjalankan

```bash
cd pertemuan_4
flutter create .          # generate folder native (android/ios/...) jika belum ada
flutter pub get
flutter run
```

> ⚠️ Jika menjalankan di **desktop** (macOS/Linux/Windows), `sqflite` butuh
> `sqflite_common_ffi`. Praktikum ini diasumsikan jalan di **emulator Android**
> atau **simulator iOS** / device fisik.

## Modul

Lihat penjelasan lengkap & langkah praktikum di
`../materi-pengajaran/pertemuan-4/MODUL_PERTEMUAN_4.md`.
