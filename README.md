# Aplikasi Peminjaman Alat — Panduan Setup (v3)

Aplikasi terdiri dari:
- **`Code.gs`** — backend di Google Sheets (Apps Script), berfungsi sebagai API.
- **`index.html`, `manifest.json`, `sw.js`, `qr-scanner-worker.min.js`, `icon-*.png`, `robots.txt`** — aplikasi web (PWA), siap di-hosting ke GitHub Pages.

> **Catatan performa (v3):** scanner sekarang pakai library `qr-scanner` (QR-only, jauh lebih kecil) menggantikan `html5-qrcode` sebelumnya, dan font sudah pakai font bawaan sistem HP (tidak lagi memuat Google Fonts dari internet). Hasilnya: halaman lebih cepat dibuka, lebih hemat kuota, dan lebih ringan di HP dengan spek rendah. File `qr-scanner-worker.min.js` **wajib ikut di-upload** persis di folder yang sama dengan `index.html` — ini fallback decode QR untuk browser yang belum punya fitur *native barcode detection* (kebanyakan Chrome Android modern sebenarnya sudah punya fitur ini secara bawaan, jadi file ini hanya dipakai sebagai cadangan).

## 1. Siapkan Google Sheet

1. Buat Google Sheet baru.
2. **Extensions > Apps Script** → hapus isi default → tempel seluruh isi `Code.gs`.
3. **Ganti baris `var APP_KEY = 'GANTI_DENGAN_KODE_RAHASIA_ANDA';`** dengan kode rahasia buatan Anda sendiri (bebas, contoh: `'lab-tif-2026-x7k'`). Catat kode ini — akan dipakai lagi di langkah 3.
4. Simpan, lalu jalankan fungsi **`setupSheets`** sekali (Run). Izinkan akses saat diminta.
5. Cek spreadsheet: sheet **Database**, **Peminjaman**, **Log** otomatis terbuat.
6. Isi data alat di sheet **Database**: `Barcode` (isi sesuai nilai QR code yang tertempel di alat), `Nama Alat`, `Posisi` (Lab A/B/C), `Kondisi` (isi `Ada` untuk alat yang siap dipinjam).

## 2. Deploy sebagai Web App

1. **Deploy > New deployment > Web app**.
2. **Execute as**: `Me`. **Who has access**: `Anyone`.
3. **Deploy**, otorisasi jika diminta, lalu salin **Web app URL** (`.../exec`).

> Setiap kali mengedit ulang `Code.gs`, buat deployment baru lewat **Deploy > Manage deployments > Edit > New version > Deploy** agar perubahan aktif.

## 3. Deploy ke GitHub Pages dengan GitHub Secrets (kunci tidak ter-commit ke repo)

File `index.html` yang saya buatkan sudah berisi placeholder (`__WEBAPP_URL__`, `__APP_KEY__`, `__ACCESS_PIN__`) alih-alih nilai asli, dan sudah disertakan workflow `.github/workflows/deploy.yml` yang otomatis mengisi nilai-nilai itu **saat proses deploy**, bukan disimpan permanen di file yang Anda commit. Jadi kalau ada orang membuka repo Anda di github.com, mereka hanya melihat tulisan `__APP_KEY__`, bukan kode aslinya.

1. Buat repository baru di GitHub (boleh **Private** — lihat catatan keamanan di bawah).
2. Upload **semua isi folder** `peminjaman-alat-app/` apa adanya — termasuk folder tersembunyi `.github/workflows/deploy.yml`. Kalau upload lewat web github.com, pastikan folder `.github` ikut ter-upload (drag seluruh folder hasil ekstrak zip, jangan file satu-satu, supaya struktur foldernya ikut).
3. Masuk ke **Settings > Secrets and variables > Actions > New repository secret**, tambahkan tiga secret ini satu per satu:
   - `WEBAPP_URL` → isi dengan Web App URL dari langkah 2 (`.../exec`)
   - `APP_KEY` → isi sama persis dengan `APP_KEY` di `Code.gs`
   - `ACCESS_PIN` → isi PIN akses aplikasi (boleh dikosongkan isi dengan string kosong kalau tidak mau pakai PIN)
4. Masuk ke **Settings > Pages**, bagian **Source** pilih **GitHub Actions** (bukan "Deploy from a branch").
5. Buka tab **Actions** di repo Anda — workflow "Deploy to GitHub Pages" akan otomatis berjalan (atau klik **Run workflow** kalau belum jalan otomatis). Tunggu sampai selesai (tanda centang hijau).
6. GitHub akan memberi URL di **Settings > Pages**, seperti `https://username.github.io/nama-repo/`.
7. Buka URL itu di Chrome Android → menu **⋮** → **Instal aplikasi**.

Setiap kali Anda mengganti nilai secret (misalnya ganti PIN), buka tab **Actions** → pilih workflow terakhir → **Re-run all jobs** supaya situs ter-deploy ulang dengan nilai baru.

> **Tidak mau ribet dengan GitHub Actions?** Boleh juga isi `WEBAPP_URL`, `APP_KEY`, `ACCESS_PIN` langsung menggantikan placeholder di `index.html` sebelum upload (edit manual, tanpa workflow). Lebih simpel, tapi nilainya akan ter-commit apa adanya ke repo — cocok kalau repo-nya sudah Private dan hanya Anda yang kelola.

## Catatan keamanan (penting dibaca)

Aplikasi ini adalah **situs statis** (HTML+JS berjalan di browser pengguna), jadi ada batas yang tidak bisa dilewati:

- **GitHub Secrets menyembunyikan nilai dari repo & commit history** — orang yang membuka file di github.com tidak akan melihat kunci aslinya. Ini yang paling sering dimaksud orang dengan "supaya API key tidak kelihatan di GitHub", dan ini yang diselesaikan oleh langkah 3 di atas.
- **Tapi begitu situs live diakses**, nilai asli tetap ada di dalam halaman yang dimuat browser pengguna (karena aplikasi berjalan sepenuhnya di sisi pengguna) — orang yang sengaja buka "View Page Source"/DevTools di situs yang sudah live tetap bisa melihatnya. Tidak ada cara membuat situs statis murni benar-benar merahasiakan sesuatu dari penggunanya sendiri.
- Lapisan-lapisan lain yang tetap berguna sebagai penghalang tambahan:
  1. **`robots.txt` + `noindex`** → mencegah Google/mesin pencari mengindeks halaman, tidak muncul di pencarian publik.
  2. **Repository GitHub di-set Private** → mencegah orang membaca kode lewat halaman github.com (situs live tetap bisa diakses lewat link oleh siapa pun yang tahu; private Pages penuh perlu plan GitHub Pro/Team).
  3. **PIN akses (`ACCESS_PIN`)** → penghalang praktis untuk orang awam, bukan proteksi kuat untuk orang paham teknis.
  4. **`APP_KEY` di sisi server (`Code.gs`)** → yang paling penting: mencegah orang mengirim data ke spreadsheet Anda tanpa tahu kode ini, meski mereka menemukan URL Web App-nya secara terpisah.
- Untuk keamanan yang jauh lebih kuat (login akun Google sungguhan per pengguna), Apps Script Web App bisa dibatasi ke akun dalam organisasi Google Workspace — tapi ini butuh alur login tambahan dan mengubah cara pemanggilan API. Beri tahu saya kalau ini dibutuhkan.

Untuk kebutuhan tool crib internal dengan risiko rendah, kombinasi GitHub Secrets + robots.txt + repo private + PIN + APP_KEY di atas biasanya sudah cukup memadai.

## 4. Cara pakai

### Pinjam
1. Tap **Pinjam** → isi nama peminjam & keterangan (opsional).
2. Tap **Mulai Scan QR** — kamera tetap menyala untuk scan banyak alat sekaligus.
3. Tap **Simpan Peminjaman**.

### Kembalikan (bisa beberapa alat sekaligus)
1. Tap **Kembalikan** → **Mulai Scan QR**.
2. Scan semua alat yang mau dikembalikan satu per satu — setiap alat yang berhasil dikenali langsung masuk daftar lengkap dengan nama peminjamnya.
3. Bisa menghapus item dari daftar sebelum disimpan (tombol ✕).
4. Tap **Kembalikan Semua** untuk memproses seluruh daftar sekaligus.

### Lapor Alat Rusak
1. Dari menu utama, tap **Lapor Alat Rusak**.
2. Isi keterangan kerusakan (opsional), lalu **Mulai Scan QR** — bisa scan beberapa alat sekaligus.
3. Tap **Tandai Rusak**. Alat yang sedang berstatus "Dipinjam" tidak bisa langsung ditandai rusak — harus dikembalikan dulu.

### Menu utama
Menampilkan daftar semua alat yang **sedang dipinjam**, diperbarui otomatis tiap kembali ke halaman ini atau tap ikon refresh.

## Struktur data (referensi)

**Sheet "Database"**: `Barcode | Nama Alat | Posisi | Kondisi` (Kondisi: `Ada` / `Dipinjam` / `Rusak`)
**Sheet "Peminjaman"**: `ID Transaksi | Barcode | Nama Alat | Nama Peminjam | Tanggal Pinjam | Tanggal Kembali | Status | Keterangan`
**Sheet "Log"**: `Timestamp | ID Transaksi | Barcode | Nama Alat | Aksi | Nama Peminjam | Keterangan`

## Troubleshooting

- **"Gagal terhubung"**: cek `WEBAPP_URL` diakhiri `/exec`, dan `APP_KEY` di `index.html` sama persis dengan di `Code.gs`. Pastikan juga deploy "Who has access" = **Anyone**.
- **Kamera tidak muncul**: perlu HTTPS (otomatis terpenuhi di GitHub Pages) + izin kamera browser di-*allow*. Gunakan kamera belakang di HP.
- **QR tidak terbaca**: pastikan QR tercetak jelas, tidak buram, pencahayaan cukup. Aplikasi hanya membaca format QR Code.
- **Lupa PIN**: buka `index.html` di editor, cek nilai `ACCESS_PIN`, atau hapus data situs (`localStorage`) di browser pengguna untuk reset status "unlocked" (tetap perlu tahu PIN yang benar untuk masuk lagi).
- **Setelah edit `Code.gs` tidak ada perubahan**: buat deployment baru (lihat langkah 2).
