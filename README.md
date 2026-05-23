# 💾 mrimgx_file_layout

_Macrium Reflect X (.mrimgx) file format dokumentáció + macOS / Linux port a kibontáshoz._

[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](#licenc)
[![Last commit](https://img.shields.io/github/last-commit/dodisrs/mrimgx_file_layout?style=flat-square)](https://github.com/dodisrs/mrimgx_file_layout/commits)
[![Top language](https://img.shields.io/github/languages/top/dodisrs/mrimgx_file_layout?style=flat-square)](https://github.com/dodisrs/mrimgx_file_layout)

## Bevezetés

Ez a repo a Macrium Reflect X (`.mrimgx`) és Macrium Reflect X File and Folder (`.mrbakx`) backup-fájlok bináris layoutját dokumentálja, és tartalmaz egy POSIX-portolt C++ kibontó-eszközt, az `img_to_raw`-t. A felső repo Visual Studio 2022 megoldása csak Windowson fordul (Win32 storage API-kra épül) — ezzel a forkkal `.mrimgx` fájlok natívan kibonthatók macOS-en és Linuxon, anélkül hogy Wine, VM vagy Macrium Reflect telepítés kéne.

---

## Mire való

Ha egy Windows gép Macrium Reflect X imaget készített (`.mrimgx`), és a fájlokhoz egy másik OS-en (macOS / Linux) kell hozzáférni, az upstream `img_to_vhdx.exe` nem futtatható natívan és Wine alatt sem (mert a VHDX-mountoláshoz a Windows kernel `virtdisk.dll` API-ja kell). Az `img_to_raw` ehelyett egy nyers blokk-szintű disk image-et (`.raw`) ír ki, amit a célgép user-space NTFS-eszközei (sleuthkit / ntfs-3g) tudnak olvasni.

## Telepítés

**Követelmények:**
- CMake 3.16+, C++17 fordító (Apple clang, GCC, Clang)
- A POSIX porthoz: macOS 12+ vagy modern Linux disztribúció
- A kibontott NTFS-tartalom olvasásához: `sleuthkit` (`brew install sleuthkit` / `apt install sleuthkit`)
- Beágyazott függőségek (a `src/libs/` és `src/dependencies/` alatt mellékelve): Zstandard, OpenSSL, zlib, nlohmann/json

Build:
```bash
git clone https://github.com/dodisrs/mrimgx_file_layout.git
cd mrimgx_file_layout
cmake -B build -S src
cmake --build build -j
# binárisok: build/img_to_raw  és  build/img_to_vhdx (utóbbi csak Windows alatt)
```

## Használat

### Alapeset — full image kibontása raw disk image-be

```bash
./build/img_to_raw input.mrimgx output.raw
```

Kimenet (rövidítve):
```
Reading backup metadata: input.mrimgx
Disk size:    128849018880 bytes (120.00 GB)
Sector size:  512 bytes
Output:       output.raw
Created sparse raw file (122880 MB logical size).
Restoring blocks...
  100.0%  93 / 93 MB  74.4 MB/s
Done! Raw image written to: output.raw
Mount with: hdiutil attach -nomount -readonly "output.raw"
Then NTFS mount the partition device.
```

A raw fájl sparse — a logikai mérete megegyezik az eredeti diszk méretével, fizikailag annyi helyet foglal, amennyi adat ténylegesen használatban volt.

### Backup tartalom megtekintése írás nélkül

```bash
./build/img_to_raw input.mrimgx -desc
```

Kilistázza a backupban tárolt diszkeket, partíciókat, méreteket — output fájl írása nélkül.

### Konkrét diszk kiválasztása multi-disk backupból

```bash
./build/img_to_raw input.mrimgx output.raw -d 4
```

### Titkosított backup

```bash
./build/img_to_raw input.mrimgx output.raw -p <password>
```

### NTFS partíció kibontása a raw fájlból (sleuthkit)

```bash
mmls output.raw                                       # partíció-tábla → start offset (sector)
fls -o <sector_offset> output.raw                     # root-listázás
tsk_recover -a -o <sector_offset> output.raw outdir/  # teljes fájlrendszer kibontása
```

## Bemeneti / Kimeneti formátum

A `.mrimgx` (image) és `.mrbakx` (file-and-folder backup) fájl-formátum részletes specifikációja a [`docs/FILE_LAYOUT.md`](docs/FILE_LAYOUT.md) fájlban. A főbb metaadat-blokkok:

| Mező | Típus | Példa |
|---|---|---|
| `$JSON` | kötelező root metaadat-blokk | backup-meta + disk/partition leíró JSON |
| `$AUXDATA` | opcionális root | belső backup-info, kibontáshoz nem szükséges |
| `$TRACK0` | kötelező / disk-szint | MBR / GPT első 1 MB (boot-szektor + partíció-tábla) |
| `$EPT` | opcionális / disk-szint | extended partition table (csak MBR) |
| `$INDEX` | kötelező / partíció-szint | DataBlockIndex (file offset + MD5 + méret) |
| `$BITMAP` | opcionális / partíció-szint | csak exFAT / ReFS partíciókhoz |
| Data block | partíció-szint | nyers fájlrendszer-adat, blokkonként zstd-tömöríthető és/vagy AES-titkosítható |

Részletes struct-definíciók, JSON-séma és titkosítási leírás:
- [`docs/FILE_LAYOUT.md`](docs/FILE_LAYOUT.md) — bináris layout
- [`docs/ENCRYPTION.md`](docs/ENCRYPTION.md) — titkosítás (key derivation, IV, AES-mode)
- [`schema/complete_schema.json`](schema/complete_schema.json) — JSON metaadat-séma

## Példák

**Példa 1 — egyszerű full backup kibontása:**
```bash
./build/img_to_raw demo.mrimgx demo.raw
# demo.raw: sparse, 120 GB logikai méret, fizikailag ~93 MB
mmls demo.raw
fls -o 32768 demo.raw
```

**Példa 2 — titkosított multi-disk backup, csak disk 2:**
```bash
./build/img_to_raw secret.mrimgx disk2.raw -p mypassword -d 2
```

**Példa 3 — backup-tartalom megtekintése írás nélkül:**
```bash
./build/img_to_raw input.mrimgx -desc
# === Backup Info ===
# Disks: 1
#   Disk 4:  size=128849018880 (120.00 GB), sector=512
#     Partitions: 1
```

## Tipikus teljesítmény

M-szériás Mac, 16 GB RAM, USB-C SSD: ~110 MB/s átlagos kibontási sebesség, ~9 perc / 40 GB tömörített → 60 GB raw output. `tsk_recover` NTFS-extract: ~15 perc / 425k fájl / 71 GB.

## Hivatkozások

- Upstream Macrium repo: https://github.com/macrium/mrimgx_file_layout
- File format eredeti dokumentáció: [`docs/FILE_LAYOUT.md`](docs/FILE_LAYOUT.md)
- macOS port pull request: https://github.com/macrium/mrimgx_file_layout/pull/9

---

## Fejlesztés / Hozzájárulás

Karbantartó: **Seres Dominik** ([nagymate@sajoauto.hu](mailto:nagymate@sajoauto.hu)).

PR-okhoz először nyiss issue-t. A POSIX porthoz a legfontosabb javítások: `unsigned long` → `uint32_t` cseréje fix-méretű struct-mezőkben (LP64 / LLP64 különbség), `wcstombs_s` / `strerror_s` (Microsoft) → POSIX megfelelők, `<windows.h>` / `<objbase.h>` / `<winioctl.h>` include-ok `#ifdef _WIN32` mögé.

## Licenc

[MIT](LICENSE) — szabadon felhasználható az MIT licenc feltételei szerint.

A repo eredeti forrása a Macrium Reflect X file layout dokumentáció (Copyright © 2024 Paramount Software UK Limited, MIT). A portolási patch-ek és az `img_to_raw` POSIX target ugyanazon MIT licenc alatt elérhetők.

A repo a következő harmadik-fél könyvtárakat használja: Zstandard (BSD / GPLv2), OpenSSL (Apache-style), zlib (zlib license), nlohmann/json (MIT). Részletek: [`LICENSE.txt`](LICENSE.txt).
