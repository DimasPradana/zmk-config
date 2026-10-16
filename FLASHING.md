# Panduan Flash Firmware Chocofi (nice!nano v1)

Panduan langkah demi langkah: dari sebelum kabel USB dipasang sampai keyboard
siap dipakai lagi. Berlaku untuk kedua sisi (kiri/kanan).

> **Aturan emas:** kerjakan **satu sisi dalam satu waktu**. Jangan colok dua sisi
> bersamaan — drive bootloader keduanya bernama sama (`NICENANO`) sehingga mudah
> tertukar.

## 0. Siapkan dulu (sebelum menyentuh kabel)

1. Download artifact build terakhir dari GitHub:
   repo → tab **Actions** → run **Build ZMK firmware** terbaru yang hijau →
   **Artifacts** → **firmware**. Ekstrak ZIP-nya. Isinya tiga file:
   - `chocofi_left_studio.uf2` — untuk sisi **kiri** (central)
   - `chocofi_right-nice_nano@1__zmk-zmk.uf2` — untuk sisi **kanan**
     (nama persisnya bisa sedikit berbeda tergantung versi workflow,
     yang penting ada kata `chocofi_right`)
   - `settings_reset-nice_nano@1__zmk-zmk.uf2` — pembersih bond BLE
     (dipakai di langkah 3, cukup sekali per sisi)
2. Pastikan backup firmware lama aman:
   `plan/16-10-2026-chocofi-zmk-setup/backup/CURRENT-left.uf2` dan
   `CURRENT-right.uf2`. Kalau salah flash, dua file ini adalah jalan pulang.
3. Hapus pairing lama di komputer (wajib setelah ganti firmware,
   karena identitas BLE berubah):
   ```
   bluetoothctl remove E6:E8:7E:47:E3:B3
   ```
   (ganti MAC dengan milikmu; lihat via `bluetoothctl devices | grep -i chocofi`).
   Di pengaturan Bluetooth desktop, "lupakan"/"remove" perangkat `Chocofi`.
4. Siapkan kabel USB-C data (bukan kabel charge saja) dan catat sisi mana yang
   sedang dipegang: **kiri = central**, kanan = peripheral.

## 1. Colok USB ke SISI KIRI dulu

1. Dalam keadaan keyboard **menyala normal** (atau mati, sama saja),
   colokkan kabel USB-C ke controller sisi **kiri**.
2. Di Linux akan muncul sebagai keyboard USB `Chocofi` + port serial
   (`/dev/ttyACM0`). Ini mode normal, **bukan** mode flash.

## 2. Masuk mode bootloader (UF2)

Ada dua cara — pilih yang paling mudah:

**Cara A — tombol BOOT di keymap (tanpa menyentuh hardware, disarankan):**
1. Tahan `LWR` (thumb kiri) + `RSE` (thumb kanan) **bersamaan** → masuk layer ADJ.
2. Sambil menahan keduanya, tekan tombol `BOOT` (baris bawah, kiri atau kanan).
3. Keyboard langsung masuk bootloader → drive `NICENANO` muncul.
   (Perilaku `&bootloader` di nRF52840 masuk ke mode UF2 yang sama persis
   seperti double-tap reset fisik.)

**Cara B — tombol reset fisik (double-tap):**
1. Tombol reset adalah **tombol SMD kecil di papan nice!nano itu sendiri**
   (papan kecil tempat kabel USB-C menancap, bukan di PCB chocofi).
   Bentuknya tombol taktil mini — tekan dengan kuku/ujung pulpen.
2. Tekan **dua kali cepat** (jeda < 0,5 detik).
   - Satu kali tekan = restart biasa (bukan itu yang kita mau).
   - Dua kali cepat = masuk bootloader.
3. Kalau tombolnya tertutup case: selipkan pinset logam dan sentuhkan
   pin **RST ke GND** dua kali cepat (lubang pin di tepi papan nice!nano).
   Fungsinya identik dengan double-tap tombol reset.

Tandanya (kedua cara sama):
   - Keyboard berhenti mengetik sementara.
   - Muncul drive USB bernama **`NICENANO`** (di file manager, atau
     `/run/media/$USER/NICENANO` di terminal).
   - Verifikasi via terminal:
     ```
     lsblk -o NAME,LABEL,MOUNTPOINTS | grep -i nicenano
     cat /run/media/$USER/NICENANO/INFO_UF2.TXT
     ```
     Harus tertulis `Model: nice!nano`.
4. Kalau drive tidak muncul: ulangi double-tap (timing-nya yang paling sering
   salah), coba port/kabel lain, pastikan kabelnya kabel data.

> **`settings_reset` bukan tombol** — itu file firmware `.uf2` dari artifact CI
> (langkah 3) yang di-flash seperti firmware biasa untuk menghapus bond BLE.
> Untuk hapus bond ringan tanpa flash, pakai tombol `BTCLR` di layer LWR
> (tahan `LWR`, tekan `BTCLR` baris bawah kiri).

## 3. Flash `settings_reset` (pembersih bond — sekali per sisi)

> Langkah ini menghapus semua bond BLE lama (komputer + antar-sisi split).
> Wajib saat pertama kali ganti firmware; boleh dilewati untuk update rutin.

1. Saat drive `NICENANO` terbuka, copy file `settings_reset-*.uf2` ke dalamnya:
   ```
   cp settings_reset-nice_nano@1__zmk-zmk.uf2 /run/media/$USER/NICENANO/
   sync
   ```
   atau drag-and-drop via file manager.
2. Drive akan **hilang sendiri** — itu normal, artinya flashing berjalan.
3. Tunggu ±10 detik sampai keyboard boot kembali (mode normal).
4. Lanjut ke langkah 4 tanpa mencabut kabel.

## 4. Flash firmware kiri

1. Double-tap reset lagi → drive `NICENANO` muncul kembali.
2. Copy file **`chocofi_left_studio.uf2`** ke drive, lalu `sync`.
3. Drive hilang sendiri → tunggu boot (±5–10 detik).
4. Verifikasi cepat: keyboard mengetik lagi; OLED kiri menyala
   (tampil layer/baterai); di USB terbaca sebagai `Chocofi`.
5. **Cabut kabel USB** dari sisi kiri. Sisi kiri selesai.

## 5. Ulangi untuk SISI KANAN

1. Colok kabel ke sisi **kanan**.
2. Double-tap reset → drive `NICENANO` muncul.
3. (Sekali saja, jika belum) flash `settings_reset-*.uf2` → tunggu boot →
   double-tap lagi.
4. Copy file **`chocofi_right-*.uf2`** → `sync` → tunggu boot.
5. Cabut kabel. Kedua sisi selesai.

## 6. Pairing ulang Bluetooth

1. Pastikan kedua sisi menyala (baterai terisi / switch on).
2. Sisi kanan otomatis tersambung ke kiri (split). Tunggu ±10 detik.
   Kalau setelah 30 detik tombol di kanan tidak merespons: ulangi langkah 3
   (`settings_reset`) di **kedua** sisi, lalu flash ulang.
3. Di komputer: scan dan pair perangkat `Chocofi` seperti biasa.
4. Pilih slot profil BT via layer **LWR**: tahan `LWR` (thumb kiri),
   tekan `BT1`..`BT5` (baris home kiri) untuk memilih slot, lalu pair.
   Tombol `BTCLR` (baris bawah kiri) untuk menghapus bond di slot aktif.

## 7. Verifikasi akhir

- [ ] Semua 36 tombol mengeluarkan keycode yang benar
      (`sudo libinput debug-events --show-keycodes`, tekan satu per satu).
- [ ] Homerow mods: tap = huruf, tahan = modifier (uji `tahan F + J` = Shift+J).
- [ ] OLED menyala di sisi yang punya layar.
- [ ] `bluetoothctl info <MAC>` menunjukkan `Chocofi`, icon keyboard,
      Battery Service.
- [ ] ZMK Studio bisa connect (USB/BLE) — unlock via layer **ADJ**
      (tahan LWR+RSE bersamaan) → tombol `UNLOCK` kanan-bawah.

## Troubleshooting

| Gejala | Penyebab umum | Solusi |
|---|---|---|
| Drive `NICENANO` tidak muncul | Tap kurang cepat / kabel charge-only | Ulangi double-tap < 0,5 dtk; ganti kabel data |
| Copy UF2 gagal / drive read-only | Mount bermasalah | Cabut-colok, masuk bootloader lagi |
| Sisi kanan tidak merespons | Bond split lama | `settings_reset` kedua sisi, flash ulang |
| Kolom tombol tertukar | Revisi PCB beda wiring | Laporkan — urutan `col-gpios` di overlay perlu dikoreksi + rebuild |
| OLED mati total | Alamat I2C beda (0x3C vs 0x3D) | Laporkan — `reg` di `chocofi.dtsi` perlu diubah + rebuild |
| Komputer tidak menemukan `Chocofi` | Bond lama belum dihapus | `bluetoothctl remove <MAC>`, pair ulang |

## Rollback (kembali ke firmware lama)

1. Masuk bootloader di sisi yang bermasalah (langkah 2).
2. Copy file backup yang sesuai (`CURRENT-left.uf2` / `CURRENT-right.uf2`)
   ke drive `NICENANO` → `sync` → tunggu boot.
3. Pairing ulang seperti langkah 6 (bond ikut ter-reset).
