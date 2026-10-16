# Plan: Setup zmk-config Chocofi (nice!nano v1) — Keymap Editor + ZMK Studio

Tanggal: 16-10-2026
Repo: `DimasPradana/zmk-config` (remote GitHub baru, kosong — sudah diverifikasi `git ls-remote` tanpa refs)
Keyboard: Chocofi split 36-key (3 baris × 5 kolom + 3 thumb per sisi), 2× nice!nano **v1** (nRF52840), OLED SSD1306.

---

## 1. Tujuan

1. Repo ini bisa build firmware Chocofi via GitHub Actions.
2. Layout keymap dapat diedit via Keymap Editor (`nickcoutsos.github.io/keymap-editor`).
3. Keymap dapat diedit runtime via ZMK Studio (`zmk.studio` atau fork `danielsvane/zmk-studio`, tanpa re-flash).
4. Visual keymap (SVG) auto-generate tiap push via keymap-drawer.
5. OLED aktif (firmware lama tidak mengaktifkan display).
6. Firmware lama di-backup sebelum ditimpa (restore point).

## 2. Fakta Hasil Investigasi (sudah diverifikasi, jangan diulang)

- **Controller**: nice!nano **v1** — `INFO_UF2.TXT`: Model `nice!nano`, Board-ID `nRF52840-nicenano`, UF2 bootloader 0.6.0. UF2 family `0x239a00b3`. Board ZMK: `nice_nano` (bukan `nice_nano_v2`).
- **Firmware lama**: ZMK berbasis main (Zephyr OS v3.5.0), `CONFIG_ZMK_STUDIO=y` + `snippet_studio_rpc_usb_uart`, physical layout "5 Column"/"6 Column", nama `Chocofi`, VID/PID default `1d50:615e`, MAC random-static, **tanpa display**, punya EXT_POWER.
- **Koneksi saat ini**: sisi **KIRI** tersambung USB/BLE. Drive bootloader `NICENANO` masuk mode UF2. `CURRENT.UF2` sisi kiri sudah di-backup ke `plan/16-10-2026-chocofi-zmk-setup/backup/CURRENT-left.uf2`; SHA-256 salinan cocok dengan sumber (`7706a7bf6fda655a3639564f3710654b4b1c7f427b43703d31a6bcf3f9578fc0`). Backup sisi **KANAN** masih perlu sebelum flash.
- **Repo lokal**: branch `main`, 1 commit template. Remote baru kosong → `git push -u origin main` langsung berhasil tanpa force.
- **Chocofi tidak punya shield resmi di ZMK** → shield self-contained di repo ini.
- Wiring matrix chocofi yang teruji (referensi `sudoish/chocofi-config`): rows `pro_micro 4,5,6,7`; cols kiri `20,19,18,15,14`; cols kanan `14,15,18,19,20` + `col-offset = <5>`; `diode-direction = "col2row"`.
- Keymap Editor butuh `config/info.json` (format QMK-like, key dengan `row`/`col` grid orto) untuk layout kustom; alternatif auto-parse matrix transform.
- ZMK Studio butuh: firmware dengan Studio + physical layout + binding `&studio_unlock` di keymap.
- Tool remap runtime (fork danielsvane) fungsional = ZMK Studio biasa + UI refresh; tidak butuh konfigurasi firmware khusus.

## 3. Keputusan Desain

| Keputusan | Pilihan | Alasan |
|---|---|---|
| ZMK revision | `main` | v0.3 tidak punya Studio/physical layouts; firmware lama juga main-era |
| Shield | Self-contained `boards/shields/chocofi/` | Tanpa dependensi eksternal; wiring teruji |
| Board target | `nice_nano` (v1) | Dikonfirmasi user dari label PCB |
| Homerow mods | YA (GACS: GUI-ALT-CTRL-SHIFT) | Permintaan user; wajib agar 36-key punya modifier |
| Studio | Tetap diaktifkan (left/central + USB-UART snippet) | Permintaan user; firmware lama juga punya |
| OLED | Aktifkan `CONFIG_ZMK_DISPLAY` | Ada OLED, firmware lama tidak enable |
| Layout visual | keymap-drawer (SVG, CI) | PNG opsional via konversi manual |
| File corne lama | Commit dulu (backup), lalu pindah `archive/corne/` setelah chocofi jalan | Permintaan user |

## 4. Struktur & Perubahan File

```
zmk-config/
├── .github/workflows/
│   ├── build.yml          # EDIT: @v0.3 → @main
│   └── draw.yml           # BARU: reusable workflow keymap-drawer
├── boards/shields/chocofi/            # BARU
│   ├── Kconfig.shield
│   ├── Kconfig.defconfig
│   ├── chocofi.dtsi
│   ├── chocofi-layouts.dtsi           # opsional bila pakai layout bawaan ZMK
│   ├── chocofi_left.overlay
│   ├── chocofi_right.overlay
│   └── chocofi.zmk.yml                # metadata (opsional)
├── build.yaml             # EDIT: isi target build
├── config/
│   ├── west.yml           # EDIT: revision v0.3 → main
│   ├── chocofi.keymap     # BARU
│   ├── chocofi.conf       # BARU
│   ├── info.json          # BARU (layout untuk Keymap Editor)
│   ├── corne.conf         # tetap (backup fase awal)
│   └── corne.keymap       # tetap (backup fase awal)
├── archive/corne/         # BARU (fase akhir: pindahkan 2 file corne)
├── keymap_drawer.config.yaml   # BARU (parse_config.raw_binding_map: BOOT/UNLOCK)
└── plan/16-10-2026-chocofi-zmk-setup/plan.md  # file ini
```

### 4.1 `.github/workflows/build.yml` (edit)

```yaml
name: Build ZMK firmware
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

### 4.2 `config/west.yml` (edit)

Ganti `revision: v0.3` → `revision: main`. Sisanya tetap.

### 4.3 `build.yaml` (edit)

```yaml
---
include:
  - board: nice_nano
    shield: chocofi_left
    snippet: studio-rpc-usb-uart
    cmake-args: -DCONFIG_ZMK_STUDIO=y
    artifact-name: chocofi_left_studio
  - board: nice_nano
    shield: chocofi_right
  - board: nice_nano
    shield: settings_reset
```

Catatan: `settings_reset` berguna untuk membersihkan bond BLE/split lama. Studio RPC BLE ikut aktif saat `CONFIG_ZMK_STUDIO=y`.

### 4.4 `boards/shields/chocofi/Kconfig.shield` (baru)

```kconfig
config SHIELD_CHOCOFI_LEFT
	def_bool $(shields_list_contains,chocofi_left)

config SHIELD_CHOCOFI_RIGHT
	def_bool $(shields_list_contains,chocofi_right)
```

### 4.5 `boards/shields/chocofi/Kconfig.defconfig` (baru)

```kconfig
if SHIELD_CHOCOFI_LEFT

config ZMK_KEYBOARD_NAME
	default "Chocofi"

config ZMK_SPLIT_BLE_ROLE_CENTRAL
	default y

endif

if SHIELD_CHOCOFI_LEFT || SHIELD_CHOCOFI_RIGHT

config ZMK_SPLIT
	default y

endif
```

### 4.6 `boards/shields/chocofi/chocofi.dtsi` (baru)

- Include `dt-bindings/zmk/matrix_transform.h` dan `layouts/foostan/corne/5column.dtsi` (tersedia di ZMK main; posisi fisik chocofi ≈ corne 5-col; urutan key position sama dengan transform kita).
- `chosen`: `zmk,kscan = &kscan0;`, `zmk,physical-layout = &foostan_corne_5col_layout;`, `zephyr,display = &oled;`
- `default_transform` 10 kolom × 4 baris:

```
RC(0,0) RC(0,1) RC(0,2) RC(0,3) RC(0,4)  RC(0,5) RC(0,6) RC(0,7) RC(0,8) RC(0,9)
RC(1,0) RC(1,1) RC(1,2) RC(1,3) RC(1,4)  RC(1,5) RC(1,6) RC(1,7) RC(1,8) RC(1,9)
RC(2,0) RC(2,1) RC(2,2) RC(2,3) RC(2,4)  RC(2,5) RC(2,6) RC(2,7) RC(2,8) RC(2,9)
                RC(3,2) RC(3,3) RC(3,4)  RC(3,5) RC(3,6) RC(3,7)
```

- Set `&foostan_corne_5col_layout { transform = <&default_transform>; };`
- `kscan0` col2row, rows 4–7 pull-down aktif-high.
- `&pro_micro_i2c { status = "okay"; oled: ssd1306@3c { compatible = "solomon,ssd1306fb"; reg = <0x3c>; width = <128>; height = <32>; segment-offset = <0>; page-offset = <0>; display-offset = <0>; multiplex-ratio = <31>; segment-remap; com-invdir; com-sequential; inversion-on; prechargep = <0x22>; }; };`

### 4.7 Overlay kiri/kanan (baru)

`chocofi_left.overlay`:

```dts
#include "chocofi.dtsi"

&kscan0 {
	col-gpios
		= <&pro_micro 20 GPIO_ACTIVE_HIGH>
		, <&pro_micro 19 GPIO_ACTIVE_HIGH>
		, <&pro_micro 18 GPIO_ACTIVE_HIGH>
		, <&pro_micro 15 GPIO_ACTIVE_HIGH>
		, <&pro_micro 14 GPIO_ACTIVE_HIGH>
		;
};
```

`chocofi_right.overlay`:

```dts
#include "chocofi.dtsi"

&default_transform { col-offset = <5>; };

&kscan0 {
	col-gpios
		= <&pro_micro 14 GPIO_ACTIVE_HIGH>
		, <&pro_micro 15 GPIO_ACTIVE_HIGH>
		, <&pro_micro 18 GPIO_ACTIVE_HIGH>
		, <&pro_micro 19 GPIO_ACTIVE_HIGH>
		, <&pro_micro 20 GPIO_ACTIVE_HIGH>
		;
};
```

### 4.8 `config/chocofi.conf` (baru)

```conf
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_TIMEOUT=300000
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000
CONFIG_BT_MAX_CONN=3
CONFIG_BT_MAX_PAIRED=3
```

### 4.9 `config/chocofi.keymap` (baru) — spesifikasi lengkap

Include: `behaviors.dtsi`, `dt-bindings/zmk/keys.h`, `dt-bindings/zmk/bt.h`. **Tidak termasuk `&hm`/modul urob** — perilaku hanyalah `&kp`, `&mo`, `&mt`, `&bt`, `&bootloader`, `&studio_unlock` (zero-arg, tersedia saat `CONFIG_ZMK_STUDIO=y`).
Layer: 0 BASE, 1 LOWER, 2 RAISE, 3 ADJUST (aktif saat LOWER+RAISE di-hold bersamaan). Urutan key position persis transform (0–35).
Semua layer harus **36 binding**.

**BASE (0)** — QWERTY + homerow mods GACS pakai `&mt` (mod-tap, zero-param) yang sudah built-in di ZMK (tanpa helper eksternal). Tapping-term default 200 ms; tuning via behavior kustom `&mt_*` jika perlu.

```
Q      W      E      R      T       Y      U      I      O      P
&mt LGUI A    &mt LALT S    &mt LCTRL D   &mt LSHFT F   G    H  &mt RSHFT J  &mt RCTRL K  &mt RALT L  &mt RGUI SEMI
Z      X      C      V      B       N      M      COMMA  DOT    FSLH
                 ESC    &mo 1   SPACE      RET    &mo 2   BSPC
```

ADJUST (layer 3) diaktifkan implisit via ZMK layer mask: aktif hanya saat layer 0, 1, 2 semuanya aktif (hold LOWER + hold RAISE bersamaan). Tidak butuh binding tambahan.

**LOWER (1)** — angka, navigasi, Bluetooth, TAB. Binding persis:

```
N1 N2 N3 N4 N5          N6 N7 N8 N9 N0
&bt BT_SEL 0 &bt BT_SEL 1 &bt BT_SEL 2 &bt BT_SEL 3 &bt BT_SEL 4   LEFT DOWN UP RIGHT DEL
&bt BT_CLR trans trans trans trans       HOME PGDN PGUP END trans
              TAB trans trans        trans trans trans
```

Kolom kiri row 1: `&bt BT_SEL 0` ... `&bt BT_SEL 4`. Kolom kanan row 2: `HOME PGDN PGUP END trans`.

**RAISE (2)** — simbol + media + modifier kiri (plain `&kp` agar bisa di-hold untuk kombinasi simbol):

```
EXCL AT HASH DLLR PRCNT     CARET AMPS ASTRK LPAR RPAR
LGUI LALT LCTRL LSHFT trans GRAVE SQT MUTE VOLD VOLU
UNDER PLUS LBRC RBRC PIPE   TILDE trans trans trans trans
              trans trans trans     trans trans DEL
```

**ADJUST (3)** — aktif saat LOWER+RAISE di-hold bersamaan (layer mask ZMK). Fungsi fungsi sistem + unlock Studio:

```
F1 F2 F3 F4 F5            F6 F7 F8 F9 F10
F11 F12 trans trans trans trans trans trans trans &studio_unlock
&bootloader trans trans trans trans   trans trans trans trans &bootloader
              trans trans trans       trans trans trans
```

Aturan:
- `&studio_unlock` WAJIB ada (posisi: layer 3, kanan paling ujung) — tanpa ini Studio read-only.
- Jangan hapus BT_SEL/BT_CLR/BT_CLR_ALL minimal BT_CLR — pairing recovery butuh ini.
- Kombo/kombinasi non-standar minimal di awal; editor tool (Keymap Editor/Studio) dipakai untuk iterasi lanjutan.

### 4.10 `config/info.json` (baru)

Format QMK-like untuk Keymap Editor: object `{"layouts": {"LAYOUT": {"layout": [...]}}}`, tiap key berisi `x, y` (visual) dan **`row`, `col`** sebagai grid orto **bukan pin GPIO** — pemetaan posisi grid ke urutan binding keymap. Definisi grid:

- Row 0: kolom 0–9 (10 key)
- Row 1: kolom 0–9
- Row 2: kolom 0–9
- Row 3: kolom 2–7 (6 thumb)

Keymap Editor membaca `row/col` sebagai identifikasi posisi; referensi format: `nickcoutsos/zmk-config-corne-demo` → `config/info.json`.

### 4.11 `.github/workflows/draw.yml` (baru)

```yaml
name: Draw keymaps
on:
  workflow_dispatch:
  push:
    paths:
      - "config/*.keymap"
jobs:
  draw:
    uses: caksoylar/keymap-drawer/.github/workflows/draw-zmk.yml@main
    permissions:
      contents: write
```

### 4.12 `keymap_drawer.config.yaml` (baru)

```yaml
parse_config:
  raw_binding_map:
    "&bootloader": "BOOT"
    "&studio_unlock": "UNLOCK"
```

Dibaca workflow draw via `config_path` default. Tanpa ini kedua binding ke-render mentah sebagai `&bootloader` / `&studio_unlock`.

### 4.13 Git & GitHub

1. **Commit backup dulu** (state lama): `git add -A` → commit `backup: template corne + keymap lama` → `git push -u origin main`.
2. Implementasi semua file di atas → commit `feat: chocofi 36-key setup (nice_nano v1, studio, oled)` → push → CI jalan.
3. Actions: aktif default; konfirmasi halaman Actions kalau ada prompt.
4. Keymap Editor: buka app → source GitHub → OAuth → pilih repo → load `config/chocofi.keymap`.
5. ZMK Studio (opsional segera, TANPA flash): firmware lama sudah ada RPC — bisa connect dulu untuk uji; perubahan runtime ini akan hilang setelah flash baru + settings_reset.

## 5. Prosedur Flash (urutan wajib)

1. **Backup firmware lama sisi kanan** (kiri sudah selesai, lihat §2): cabut USB kiri → colok USB kanan → double-tap reset → muncul drive `NICENANO` → salin `CURRENT.UF2` ke `plan/16-10-2026-chocofi-zmk-setup/backup/CURRENT-right.uf2` → keluar bootloader (cabut-colok atau tekan reset sekali). Jangan tulis apa pun ke drive. Lalu colok kembali sisi kiri untuk lanjut ke langkah 5.2.
2. Download artifact Actions → ekstrak 3 UF2.
3. Flash `settings_reset` ke kedua sisi (boot 10 detik) — membersihkan bond lama.
4. Flash `chocofi_left_studio` ke kiri (central), lalu `chocofi_right` ke kanan.
5. Pairing ulang BLE (nama tetap `Chocofi`).

## 6. Verifikasi

1. CI hijau; artifact berisi 3 UF2.
2. Semua 36 tombol benar: `sudo libinput debug-events --show-keycodes` atau tester browser — tekan tiap tombol satu per satu, kedua sisi.
3. Homerow mods: tap = huruf, hold = modifier, uji rolling cepat (tidak salah trigger).
4. OLED menyala, tampil layer/battery; kedua sisi (asumsi OLED ganda — kalau kanan tidak punya OLED, abaikan).
5. BLE: `bluetoothctl info` tetap `Chocofi`, HID, battery service.
6. Studio: connect via USB/BLE → unlock → ubah 1 tombol → save → uji; revert.
7. Keymap Editor: load repo → layout 36 tombol tampil → edit → commit → CI rebuild → flash → uji.

## 7. Risiko & Mitigasi

| Risiko | Mitigasi |
|---|---|
| Urutan col-gpios tidak cocok revisi PCB → kolom tertukar | Tes semua tombol; koreksi urutan di overlay, rebuild |
| v0.3→main breaking config | Workflow ikut `@main`; config minimal |
| Bond BLE lama mengganggu split | Flash `settings_reset` dulu (langkah 5.3) |
| OLED berbeda alamat I2C (0x3C vs 0x3D) | Kalau mati total, coba `reg = <0x3d>` |
| Homerow mods salah-trigger | Naikkan `tapping-term-ms` via custom hold-tap; edit via Keymap Editor |
| Flash salah file ke sisi salah | Urutan flash di §5; bootloader selalu bisa double-tap reset |
| Rollback penuh | Restore `CURRENT-left/right.uf2` dari folder backup |

## 8. Forensik OLED (17-10-2026)

- `backup-asli-kiri.UF2` identik (md5 sama) dengan `CURRENT-left.uf2` = firmware vendor lama.
- String `ssd1306@3c` + `i2c@40003000` di firmware lama **hanya nama node devicetree**
  (satu tabel dengan `adc@...`, `clock@...`), bukan bukti driver aktif.
  Tidak ada string display/LVGL/I2C-driver di firmware lama → display hampir pasti
  **tidak pernah di-enable** di firmware vendor (OLED lama juga gelap).
- Build baru terkonfirmasi display ON via `.config` CI
  (`CONFIG_ZMK_DISPLAY=y`, `CONFIG_SSD1306=y`, `CONFIG_DISPLAY=y`,
  `CONFIG_LVGL=y v9.3`, `CONFIG_I2C=y`, status screen built-in).
- Karena firmware sudah benar tapi OLED tetap gelap, kandidat penyebab:
  1. Panel sebenarnya **SH1106** (umum di modul OLED murah) → butuh compatible
     `sinowealth,sh1106`, bukan `solomon,ssd1306fb`.
  2. Alamat I2C **0x3D** bukan 0x3C.
  3. Resolusi panel **128x64** bukan 128x32.
  4. OLED hanya terpasang di satu sisi / masalah daya/wiring.
- Butuh info hardware: foto/tulisan di modul OLED (depan+belakang), sisi mana
  yang ada OLED, dan konfirmasi firmware baru sudah aktif (tes homerow mods).

## 9. Fase Lanjutan (di luar scope plan ini)

- Peripheral status screen kustom untuk OLED kanan.
- PNG export dari SVG (`rsvg-convert`).
- Combos, tap-dance, mouse keys, deep-sleep tuning.
- nice!view bila upgrade display.
