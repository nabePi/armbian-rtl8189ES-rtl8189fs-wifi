# Driver WiFi RTL8189ES/RTL8189FS untuk STB HG680P (Armbian Resolute)

Repo ini berisi source code driver WiFi Realtek **RTL8189ES / RTL8189FS** yang sudah disesuaikan agar bisa dibuild dan berjalan di STB **HG680P** yang menjalankan **Armbian Resolute** dengan kernel versi **6.18.41**.

Driver bawaan vendor biasanya tidak kompatibel dengan kernel modern (Armbian mainline), sehingga source di repo ini sudah dipatch supaya cocok dengan API `cfg80211` terbaru dan bisa dibuild ulang (di-*compile* sendiri) sebagai kernel module (`.ko`).

## Prasyarat: STB Harus Sudah Menjalankan Armbian

Proses build/compile dilakukan **langsung di dalam STB HG680P**, bukan di komputer lain, jadi STB harus sudah berjalan dengan Armbian terlebih dahulu. Ada dua kondisi yang mungkin:

1. **Armbian di-boot dari USB Flashdisk / MicroSD**
   Masuk ke STB melalui Armbian yang di-boot dari media eksternal (USB flashdisk atau microSD), lalu login ke sistem tersebut (via SSH atau langsung terminal) sebelum lanjut ke langkah build di bawah.

2. **Armbian sudah ter-install permanen di eMMC STB**
   Jika Armbian sudah di-install ke penyimpanan internal (eMMC) STB, cukup nyalakan STB seperti biasa lalu login (SSH/terminal) ke sistem Armbian tersebut.

Pastikan sudah masuk (login) ke shell Armbian di STB sebelum melanjutkan ke langkah build.

## Kebutuhan

Sebelum build, pastikan di STB HG680P sudah tersedia:

- Header kernel yang sesuai dengan kernel yang sedang berjalan (`/lib/modules/$(uname -r)/build`)
- Tools build dasar: `make`, `gcc`, `git`

Cek versi kernel yang sedang dipakai:

```bash
uname -r
```

## Cara Build

1. Clone repo ini:

   ```bash
   git clone <url-repo-ini>
   cd armbian-rtl8189ES-rtl8189fs-wifi
   ```

2. Build modul driver:

   ```bash
   make -j4 ARCH=arm64 KSRC=/lib/modules/$(uname -r)/build
   ```

   Penjelasan opsi:

   | Opsi | Keterangan |
   |------|------------|
   | `-j4` | Menjalankan proses build hingga 4 job secara paralel (mempercepat compile). |
   | `ARCH=arm64` | STB HG680P menggunakan arsitektur ARM 64-bit. |
   | `KSRC=/lib/modules/$(uname -r)/build` | Menunjuk ke direktori header/build kernel yang sedang aktif di sistem. |

   Jika proses build berhasil, akan menghasilkan file modul `8189fs.ko` di direktori repo.

## Cara Install / Aktifkan Driver

1. Copy hasil compile (`8189fs.ko`) ke direktori modul kernel:

   ```bash
   sudo cp 8189fs.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/
   ```

2. Perbarui dependency modul agar `8189fs.ko` bisa dikenali dan dimuat oleh `modprobe`:

   ```bash
   sudo depmod -a
   ```

3. Muat (load) driver WiFi:

   ```bash
   sudo modprobe 8189fs
   ```

4. Cek apakah interface WiFi sudah muncul:

   ```bash
   ip link
   # atau
   iwconfig
   ```

   Jika berhasil, akan muncul interface baru seperti `wlan0`.

## Agar Driver Otomatis Dimuat Saat Boot

Supaya tidak perlu menjalankan `modprobe 8189fs` setiap kali STB di-restart, tambahkan modul ke daftar autoload:

```bash
echo "8189fs" | sudo tee -a /etc/modules
```

## CI: Test Build Otomatis (GitHub Actions)

Repo ini punya workflow GitHub Actions di `.github/workflows/build.yml` untuk memastikan source driver ini tetap bisa di-*compile* terhadap kernel target STB HG680P, yaitu **Linux 6.18.41 arm64**.

- Runner yang dipakai adalah **arm64 native** (`ubuntu-24.04-arm`), jadi build dilakukan native (bukan cross-compile).
- Workflow mendownload header kernel `6.18.41` arm64 (varian `-generic`, 4k page size, sesuai Armbian) dari mainline PPA Ubuntu, lalu menjalankan build yang sama seperti perintah di bagian [Cara Build](#cara-build) (`make ARCH=arm64 KSRC=... CC=cc`).
- Jika versi kernel `6.18.41` sudah tidak tersedia lagi di mainline PPA (misal karena dibersihkan/diarsipkan), job otomatis di-skip (bukan dianggap gagal).

**Cara menjalankan:** workflow ini di-trigger **manual**, tidak otomatis jalan saat push atau pull request. Untuk menjalankannya:

1. Buka tab **Actions** di halaman GitHub repo ini.
2. Pilih workflow **Build** di sidebar kiri.
3. Klik tombol **Run workflow**.

Gunakan ini untuk mengecek apakah source driver masih kompatibel dengan kernel `6.18.41` setelah ada perubahan kode.

Setelah workflow selesai, tabel di bawah ini akan **otomatis diperbarui** (commit sendiri oleh workflow) dengan status build per versi kernel dan tanggal terakhir dicek — tidak perlu diedit manual.

<!-- KERNEL_COMPAT_TABLE_START -->
| Versi Kernel (arm64) | Status | Terakhir Dicek |
|---|---|---|
| 6.18.41 | ✅ Berhasil | 2026-08-19 |
<!-- KERNEL_COMPAT_TABLE_END -->

## Troubleshooting

- **Build gagal / error path kernel**: pastikan path `/lib/modules/$(uname -r)/build` benar-benar ada. Jika tidak ada, berarti header kernel untuk versi tersebut belum terpasang di Armbian.
- **`modprobe: FATAL: Module 8189fs not found`**: biasanya karena lupa menjalankan `depmod -a` setelah copy file `.ko`, atau file `.ko` di-copy ke path yang salah.
- **Interface WiFi tidak muncul setelah `modprobe`**: cek log kernel dengan `dmesg | tail -50` untuk melihat pesan error dari driver.

## Referensi

Repo ini merupakan modifikasi/patch dari source driver Realtek RTL8189ES/RTL8189FS agar kompatibel dengan API `cfg80211` pada kernel Linux modern (Armbian, kernel 6.18.41).
