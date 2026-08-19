# Driver WiFi RTL8189ES/RTL8189FS untuk STB HG680P (Armbian)

Repo ini berisi source code driver WiFi Realtek **RTL8189ES / RTL8189FS** yang sudah disesuaikan agar bisa dibuild dan berjalan di STB **HG680P** yang menjalankan **Armbian** dengan kernel versi **6.18.41**.

Driver bawaan vendor biasanya tidak kompatibel dengan kernel modern (Armbian mainline), sehingga source di repo ini sudah dipatch supaya cocok dengan API `cfg80211` terbaru dan bisa dibuild ulang (di-*compile* sendiri) sebagai kernel module (`.ko`).

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

## Troubleshooting

- **Build gagal / error path kernel**: pastikan path `/lib/modules/$(uname -r)/build` benar-benar ada. Jika tidak ada, berarti header kernel untuk versi tersebut belum terpasang di Armbian.
- **`modprobe: FATAL: Module 8189fs not found`**: biasanya karena lupa menjalankan `depmod -a` setelah copy file `.ko`, atau file `.ko` di-copy ke path yang salah.
- **Interface WiFi tidak muncul setelah `modprobe`**: cek log kernel dengan `dmesg | tail -50` untuk melihat pesan error dari driver.

## Referensi

Repo ini merupakan modifikasi/patch dari source driver Realtek RTL8189ES/RTL8189FS agar kompatibel dengan API `cfg80211` pada kernel Linux modern (Armbian, kernel 6.18.41).
