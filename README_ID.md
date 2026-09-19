## ⚠️ Peringatan

Alat ini disediakan hanya untuk penggunaan pribadi. Gunakan hanya untuk mendekripsi file atau preset milik kamu sendiri. Jangan gunakan alat ini untuk merugikan orang lain atau untuk tujuan yang tidak baik. Gunakan dengan risiko sendiri.


# Dekripsi Preset KLWP yang Terkunci: Panduan Teknis Lengkap

> Didokumentasikan oleh **DGameXO (dgxo / dgamexo)**
> Panduan ini mencakup seluruh proses mulai dari memahami skema enkripsi hingga berhasil memulihkan preset yang terkunci.

---

## Daftar Isi

1. [Gambaran Umum](#gambaran-umum)
2. [Cara Kerja Enkripsi KLWP](#cara-kerja-enkripsi-klwp)
3. [Yang Kamu Butuhkan](#yang-kamu-butuhkan)
4. [Langkah 1: Identifikasi Versi Preset](#langkah-1-identifikasi-versi-preset)
5. [Langkah 2: Download APK yang Sesuai](#langkah-2-download-apk-yang-sesuai)
6. [Langkah 3: Decode APK dengan Apktool](#langkah-3-decode-apk-dengan-apktool)
7. [Langkah 4: Temukan Encryption Seed di Native Library](#langkah-4-temukan-encryption-seed-di-native-library)
8. [Langkah 5: Pahami Proses Key Derivation](#langkah-5-pahami-proses-key-derivation)
9. [Langkah 6: Ekstrak Data Terenkripsi](#langkah-6-ekstrak-data-terenkripsi)
10. [Langkah 7: Brute Force Key](#langkah-7-brute-force-key)
11. [Langkah 8: Dekripsi dan Rebuild Preset](#langkah-8-dekripsi-dan-rebuild-preset)
12. [Langkah 9: Repack dan Import](#langkah-9-repack-dan-import)
13. [Catatan Per Versi](#catatan-per-versi)
14. [Temuan Lintas Versi](#temuan-lintas-versi)
15. [Troubleshooting](#troubleshooting)

---

## Gambaran Umum

KLWP (Kustom Live Wallpaper) memungkinkan kreator mengunci preset mereka sebelum diekspor, sehingga pengguna lain tidak bisa mengedit atau melihat layer di dalamnya. Ketika preset dikunci, data layer-nya dienkripsi dan disimpan di dalam file `.klwp` dengan key `internal_readonly`.

Panduan ini mendokumentasikan proses reverse engineering lengkap yang digunakan untuk memulihkan skema enkripsi dan berhasil mendekripsi preset yang terkunci.

### Alur Enkripsi

```
Kreator mengunci preset di KLWP
        |
        v
Data layer (JSON array) --> Enkripsi DES/ECB --> Encode Base64 URL-safe
        |
        v
Disimpan sebagai nilai "internal_readonly" di preset.json dalam file .klwp
```

### Alur Dekripsi

```
Baca "internal_readonly" dari preset.json
        |
        v
Decode Base64 URL-safe --> Dekripsi DES/ECB (dengan key yang sudah diturunkan)
        |
        v
JSON array semua layer --> inject ke preset_root.viewgroup_items
        |
        v
Repack sebagai .klwp --> import ke KLWP
```

---

## Cara Kerja Enkripsi KLWP

### Algoritma

KLWP menggunakan **DES (Data Encryption Standard)** dalam mode **ECB (Electronic Codebook)** dengan padding **PKCS7**.

Data terenkripsi dikodekan menggunakan **Base64 URL-safe** (Android flag `0xa` = `Base64.URL_SAFE`). Artinya, string Base64 menggunakan `-` sebagai pengganti `+` dan `_` sebagai pengganti `/`.

Hal ini dikonfirmasi dengan menemukan `DESHelper.kt` yang dikompilasi ke dalam APK di:

```
smali_classes5/z6/a.smali
```

Inisialisasi cipher yang relevan dalam smali:

```smali
const-string v0, "DES"
invoke-static {v0}, Ljavax/crypto/Cipher;->getInstance(Ljava/lang/String;)Ljavax/crypto/Cipher;
const/4 v3, 0x2   # DECRYPT_MODE
invoke-virtual {v0, v3, p0}, Ljavax/crypto/Cipher;->init(ILjava/security/Key;)V
```

### Key Derivation

DES key tidak disimpan sebagai string biasa. Melainkan diturunkan lewat proses berikut:

```
key = String.format("%08d", hashCode(seed + author + email))
```

Di mana:
- `seed` adalah string yang dikembalikan oleh fungsi native `getPresetUnlockSeed()` dari `liblocal-config-lib.so`
- `author` adalah nilai field `author` di `preset_info` saat preset dikunci
- `email` adalah nilai field `email` di `preset_info` saat preset dikunci
- `hashCode()` adalah implementasi standar `String.hashCode()` milik Java

String key akhir di-encode dalam UTF-8 dan hanya **8 byte pertama** yang digunakan sebagai DES key.

### Implementasi Java hashCode

`String.hashCode()` milik Java bersifat deterministik dan bisa direplikasi dalam Python:

```python
def java_hashcode(s):
    h = 0
    for c in s:
        h = (31 * h + ord(c)) & 0xFFFFFFFF
    if h >= 0x80000000:
        h -= 0x100000000
    return h
```

Perlu dicatat bahwa hasilnya bisa negatif. `String.format("%08d", negative_number)` di Java menghasilkan string yang lebih panjang dari 8 karakter (misalnya `-236174758`), dan hanya 8 byte pertama yang digunakan sebagai DES key.

### Dari Mana Seed Berasal

Seed dikembalikan oleh fungsi native C yang dimuat dari `liblocal-config-lib.so`. Library ini dimuat via:

```java
System.loadLibrary("local-config-lib")
```

Tiga fungsi native di `SeedHelper.kt` adalah:

| Fungsi | Tujuan |
|---|---|
| `getPresetUnlockSeed()` | Seed untuk KLWP preset lock |
| `getKomponentUnlockSeed()` | Seed untuk KLWP komponent lock |
| `getServiceDESSeed()` | Seed untuk service-level DES |

Untuk KLWP v3.63 (build `363228708`), seed yang diekstrak dari `lib/x86_64/liblocal-config-lib.so`:

| Fungsi | Seed |
|---|---|
| `getPresetUnlockSeed` | `poiuyrqoispsx` |
| `getKomponentUnlockSeed` | `askoeruqwoie` |
| `getServiceDESSeed` | `ouweirit72idn` |

---

## Yang Kamu Butuhkan

- Environment Linux atau macOS (WSL juga bisa di Windows)
- `apktool` (versi 2.x) yang sudah terinstall atau tersedia di `~/bin/apktool`
- `objdump`, `strings`, dan `readelf` (biasanya sudah pre-installed di Linux)
- `python3` dengan `pycryptodome`: `pip install pycryptodome`
- File `.klwp` yang terkunci dan ingin kamu dekripsi
- Tebakan terbaik untuk **nama author** dan **email** yang digunakan saat preset dikunci
- APK KLWP untuk **versi yang sama** dengan yang digunakan saat preset dikunci

---

## Langkah 1: Identifikasi Versi Preset

File `.klwp` adalah arsip ZIP. Ekstrak langsung dengan `unzip`:

```bash
unzip your_preset.klwp -d preset_extracted
cat preset_extracted/preset.json
```

Cari blok `preset_info`:

```json
{
  "preset_info": {
    "version": 12,
    "title": "Test",
    "author": "dgamexo",
    "release": 363228708,
    "locked": true
  }
}
```

Field `release` adalah Android `versionCode` dari build KLWP yang digunakan untuk mengunci preset.

### Membaca Angka Release

| nilai release | versi KLWP |
|---|---|
| `363228708` | 3.63b228708 |
| `374331712` | 3.74b331712 |
| `382xxxxxx` | 3.82 |

Format: 3 digit pertama = versi major.minor, digit selanjutnya = nomor build.

Perhatikan juga field `author` dan `email`. Ini adalah nilai yang diset saat penguncian. Kalau file sudah didistribusikan ulang dengan `preset_info` yang dimodifikasi, nilainya mungkin berbeda dari aslinya.

---

## Langkah 2: Download APK yang Sesuai

Download APK KLWP yang sesuai dengan versi `release`.

Arsip resmi: `https://docs.kustom.rocks/docs/downloads/download-klwp/`

APKMirror: `https://www.apkmirror.com/apk/kustom-industries/klwp-live-wallpaper-maker/`

Download varian **Google Play**, bukan AOSP atau Huawei, karena native library-nya bisa berbeda.

Untuk v3.63: `https://kustom.rocks/download/klwp/363228708/google_release`

> **Catatan:** Berdasarkan temuan di v3.63, v3.74, dan v3.82, nilai seed di dalam `liblocal-config-lib.so` ternyata identik di semua versi yang sudah diuji. Dalam praktiknya, APK mana pun dari range v3.6x hingga v3.8x bisa digunakan untuk ekstraksi seed. Lihat [Temuan Lintas Versi](#temuan-lintas-versi) untuk detailnya.

---

## Langkah 3: Decode APK dengan Apktool

```bash
~/bin/apktool d klwp_google_release_363.apk -o klwp_363_decoded
cd klwp_363_decoded
```

Ini akan menghasilkan:
- `smali/` sampai `smali_classes5/` - bytecode yang sudah didecompile
- `lib/` - native libraries
- `assets/` - aset yang dibundel

---

## Langkah 4: Temukan Encryption Seed di Native Library

Cari library-nya:

```bash
find . -name "liblocal-config-lib.so"
```

Gunakan varian `x86_64` untuk analisis.

### Ekstrak String Kandidat

```bash
strings ./lib/x86_64/liblocal-config-lib.so | grep -E ".{8,20}" | head -30
```

Di v3.63, cari:

```
poiuyrqoispsx
ouweirit72idn
askoeruqwoie
```

### Peta String ke Fungsi

Metode pemetaan berbeda tergantung versi karena ada perubahan cara library menyimpan string-nya:

**v3.6x** — Seed disimpan di section `.comment` (tanpa virtual address). Pemetaan vaddr langsung tidak akan berhasil. Gunakan pendekatan dump section `objdump` sebagai gantinya:

```bash
objdump -s -j .rodata ./lib/x86_64/liblocal-config-lib.so
```

**v3.7x dan v3.8x** — Seed disimpan di `.rodata` (memiliki virtual address yang valid). Pemetaan vaddr standar bisa digunakan:

```bash
# Dapatkan file offsets
strings -o ./lib/x86_64/liblocal-config-lib.so | grep -E "poiuyrqoispsx|ouweirit72idn|askoeruqwoie"

# Disassemble untuk menemukan vaddr yang dimuat tiap fungsi
objdump -d ./lib/x86_64/liblocal-config-lib.so 2>/dev/null | grep -A8 "getPresetUnlockSeed"

# Cross-reference vaddr dengan section map
readelf -S --wide ./lib/x86_64/liblocal-config-lib.so | grep rodata
```

Instruksi `lea` di setiap function body menampilkan komentar dengan vaddr yang dimuat. Untuk v3.74+, vaddr ini ada di dalam `.rodata` dan bisa langsung dipetakan ke string.

Untuk v3.63 x86_64, pemetaan yang sudah dikonfirmasi adalah:

| vaddr | String | Fungsi |
|---|---|---|
| `0x598` | `poiuyrqoispsx` | `getPresetUnlockSeed` |
| `0x5a6` | `ouweirit72idn` | `getServiceDESSeed` |
| `0x5b4` | `askoeruqwoie` | `getKomponentUnlockSeed` |

---

## Langkah 5: Pahami Proses Key Derivation

Direkonstruksi dari `smali_classes5/org/kustom/lib/render/RootLayerModule.smali`:

```java
String seed   = SeedHelper.getPresetUnlockSeed();
String author = presetInfo.getAuthor();
String email  = presetInfo.getEmail();

String combined = seed + (author != null ? author : "") + (email != null ? email : "");
int    hash     = combined.hashCode();
String key      = String.format("%08d", hash);
// Hanya 8 byte pertama yang digunakan sebagai DES key
```

Ekuivalen Python:

```python
def java_hashcode(s):
    h = 0
    for c in s:
        h = (31 * h + ord(c)) & 0xFFFFFFFF
    if h >= 0x80000000:
        h -= 0x100000000
    return h

seed   = "poiuyrqoispsx"
author = "dgamexo"
email  = ""

combined = seed + author + email
h        = java_hashcode(combined)
key_str  = f"{h:08d}"
key      = key_str.encode('utf-8')[:8]
```

---

## Langkah 6: Ekstrak Data Terenkripsi

```bash
grep -o 'internal_readonly[^,}]*' preset_extracted/preset.json > internal_readonly_values.txt
head -c 200 internal_readonly_values.txt
```

Nilainya akan terlihat seperti Base64 URL-safe yang menggunakan karakter `-` dan `_`:

```
internal_readonly": "LajhAAincgvul-7Qluu6YHuXE9XTeqS97dkKe...
```

---

## Langkah 7: Brute Force Key

Dekripsi yang benar akan menghasilkan JSON yang terbaca dengan skor 80/80 karakter printable di 80 byte pertama.

```python
from Crypto.Cipher import DES
import base64

def java_hashcode(s):
    h = 0
    for c in s:
        h = (31 * h + ord(c)) & 0xFFFFFFFF
    if h >= 0x80000000:
        h -= 0x100000000
    return h

with open('internal_readonly_values.txt', 'r') as f:
    raw = f.read().strip()

data = raw.split(': "', 1)[1].rstrip('"')
data = data.replace('-', '+').replace('_', '/')
pad  = (4 - len(data) % 4) % 4
data += "=" * pad
encrypted = base64.b64decode(data)

seed = "poiuyrqoispsx"

authors = ["your_username", "creator_name", ""]
emails  = ["your@email.com", "creator@email.com", ""]

best = []
for author in authors:
    for email in emails:
        combined = seed + author + email
        h        = java_hashcode(combined)
        key_str  = f"{h:08d}"
        key      = key_str.encode('utf-8')[:8]
        try:
            cipher    = DES.new(key, DES.MODE_ECB)
            decrypted = cipher.decrypt(encrypted)
            score     = sum(1 for b in decrypted[:80] if 32 <= b < 127)
            best.append((score, author, email, key_str, decrypted))
        except Exception:
            pass

best.sort(reverse=True)
for score, author, email, key_str, decrypted in best[:5]:
    print(f"score={score}/80 | author={author!r} | email={email!r} | key={key_str}")
    print(f"  preview: {decrypted[:80]}")
    print()
```

Hasil yang benar menunjukkan `score=80/80` dan preview diawali dengan `[{"internal_type":`.

---

## Langkah 8: Dekripsi dan Rebuild Preset

```python
from Crypto.Cipher import DES
import base64, json

def java_hashcode(s):
    h = 0
    for c in s:
        h = (31 * h + ord(c)) & 0xFFFFFFFF
    if h >= 0x80000000:
        h -= 0x100000000
    return h

with open('internal_readonly_values.txt', 'r') as f:
    raw = f.read().strip()

data = raw.split(': "', 1)[1].rstrip('"')
data = data.replace('-', '+').replace('_', '/')
pad  = (4 - len(data) % 4) % 4
data += "=" * pad
encrypted = base64.b64decode(data)

seed   = "poiuyrqoispsx"
author = "dgamexo"
email  = ""

h       = java_hashcode(seed + author + email)
key_str = f"{h:08d}"
key     = key_str.encode('utf-8')[:8]

cipher    = DES.new(key, DES.MODE_ECB)
decrypted = cipher.decrypt(encrypted)

pad_len = decrypted[-1]
if pad_len < 16:
    decrypted = decrypted[:-pad_len]

layers = json.loads(decrypted)
with open('decrypted_layers.json', 'w') as f:
    json.dump(layers, f, ensure_ascii=False)

print(f"Berhasil mendekripsi {len(layers)} objek layer")
```

### Rebuild Menggunakan Shell Preset Kosong

Buat preset kosong baru di KLWP, ekspor sebagai `blank.klwp`, lalu:

```bash
unzip blank.klwp -d blank_ex
```

```python
import json

with open('blank_ex/preset.json') as f:
    shell = json.load(f)

with open('preset_extracted/preset.json') as f:
    original = json.load(f)

with open('decrypted_layers.json') as f:
    layers = json.load(f)

shell['preset_info'] = original['preset_info']
shell['preset_info']['locked'] = False

shell['preset_root'] = original['preset_root']
shell['preset_root'].pop('internal_readonly', None)
shell['preset_root']['viewgroup_items'] = layers

with open('blank_ex/preset.json', 'w') as f:
    json.dump(shell, f, ensure_ascii=False)

print("Berhasil digabung. Jumlah layer:", len(layers))
```

---

## Langkah 9: Repack dan Import

```bash
cp -r preset_extracted/bitmaps blank_ex/
cp -r preset_extracted/fonts   blank_ex/
cp -r preset_extracted/icons   blank_ex/

cd blank_ex
zip -r ../preset_unlocked.klwp .

adb push preset_unlocked.klwp /sdcard/Kustom/wallpapers/preset_unlocked.klwp
```

Buka KLWP, import preset-nya, dan verifikasi bahwa semua layer sudah terlihat dan bisa diedit.

---

## Catatan Per Versi

### Lokasi Penyimpanan Seed Berdasarkan Versi

Ditemukan perbedaan struktural penting antara build v3.6x dan v3.7x+:

| Range versi | Lokasi seed di `.so` | Pemetaan vaddr |
|---|---|---|
| v3.6x | Section `.comment` (vaddr = 0) | gagal — tidak ada virtual address |
| v3.7x dan v3.8x | Section `.rodata` (vaddr valid) | berhasil via peta section header |

Di v3.6x, seed disimpan di section ELF `.comment` yang tidak memiliki virtual address, sehingga konversi vaddr-ke-file-offset via section header `readelf` tidak akan bisa memecahkannya. Solusi alternatif yang andal untuk v3.6x adalah membaca `.rodata` langsung dengan `objdump -s -j .rodata` dan mem-parse string yang diakhiri null secara berurutan.

Di v3.7x dan v3.8x, seed sudah dipindahkan ke `.rodata` yang punya virtual address yang proper. Instruksi `lea` dalam output `objdump -d` langsung menunjuk ke `.rodata`, sehingga pemetaannya jadi mudah.

### Seed yang Sudah Diketahui (semua versi yang diuji, build Google Play)

| Fungsi | Seed |
|---|---|
| `getPresetUnlockSeed` | `poiuyrqoispsx` |
| `getKomponentUnlockSeed` | `askoeruqwoie` |
| `getServiceDESSeed` | `ouweirit72idn` |

### Ekstraksi Seed untuk Versi Lain

```bash
# Baca section .rodata langsung (berfungsi untuk semua versi)
objdump -s -j .rodata ./lib/x86_64/liblocal-config-lib.so

# Disassemble untuk konfirmasi pemetaan fungsi-ke-string
objdump -d ./lib/x86_64/liblocal-config-lib.so 2>/dev/null \
  | grep -A8 "getPresetUnlockSeed\|getKomponentUnlockSeed\|getServiceDESSeed"
```

---

## Temuan Lintas Versi

Hal berikut dikonfirmasi dengan menganalisis `liblocal-config-lib.so` yang diekstrak dari tiga build APK terpisah:

| Versi | Kode build | Lokasi seed | Seed identik |
|---|---|---|---|
| v3.63 | `363228708` | `.comment` | ya |
| v3.74 | `374331712` | `.rodata` | ya |
| v3.82 | `382xxxxxx` | `.rodata` | ya |

**Nilai seed untuk ketiga fungsi `SeedHelper` identik di v3.63, v3.74, dan v3.82.** Ini berarti seed tidak berubah setidaknya dalam range versi ini. Dalam praktiknya, seed `poiuyrqoispsx` bisa langsung digunakan tanpa perlu ekstraksi APK untuk preset mana pun dalam range versi ini.

String keempat `pcq834pqmaicp` muncul di `.rodata` milik v3.74 dan v3.82, dimuat oleh fungsi tambahan yang tidak ada di v3.63. Tujuannya saat ini belum diketahui dan tidak terlibat dalam preset unlock.

**Versi di bawah v3.6x belum diuji.** Investigasi masih berlangsung. Pantau repositori ini untuk update saat versi yang lebih lama dianalisis.

---

## Troubleshooting

### Skor di bawah 30/80

Key salah. Penyebab umum:
- Author atau email berbeda dari saat penguncian
- Preset didistribusikan ulang dengan `preset_info` yang dimodifikasi
- Versi atau varian APK salah (AOSP vs Google Play)

### JSONDecodeError setelah dekripsi

Strip padding PKCS7:

```python
pad_len = decrypted[-1]
if pad_len < 16:
    decrypted = decrypted[:-pad_len]
```

### Preset berhasil dimuat tapi layer kosong

Cek apakah `viewgroup_items` sudah diinjeksi:

```python
print(len(shell['preset_root'].get('viewgroup_items', [])))
```

Nilainya harus lebih dari 0.

### Error padding Base64

Konversi URL-safe Base64 sebelum decoding:

```python
data = data.replace('-', '+').replace('_', '/')
pad  = (4 - len(data) % 4) % 4
data += "=" * pad
```

### Ekstraksi seed menghasilkan pemetaan yang salah (v3.6x)

Di v3.6x, seed ada di `.comment` yang tidak memiliki virtual address. Gunakan dump section langsung:

```bash
objdump -s -j .rodata ./lib/x86_64/liblocal-config-lib.so
```

Parse string yang diakhiri null secara berurutan dari output. Urutan di `.rodata` untuk v3.63 adalah: `askoeruqwoie`, `ouweirit72idn`, `poiuyrqoispsx`.

---

## Ringkasan

```
Field release di preset.json
        |
        v
Download APK KLWP yang sesuai (build sama, varian Google Play)
        |
        v
apktool d --> smali + lib/
        |
        v
strings + objdump pada liblocal-config-lib.so --> seed string
(v3.6x: baca dari .comment via objdump -s -j .rodata)
(v3.7x+: baca dari .rodata via pemetaan vaddr)
        |
        v
java_hashcode(seed + author + email) --> format %08d --> 8-byte DES key
        |
        v
Decode Base64 URL-safe --> Dekripsi DES ECB --> strip padding PKCS7
        |
        v
JSON array layer --> inject ke blank preset shell sebagai viewgroup_items
        |
        v
copy bitmaps/fonts/icons --> repack sebagai .klwp --> import dan verifikasi
```

Proses ini diteliti dan didokumentasikan melalui reverse engineering penuh KLWP v3.63, menelusuri jalur enkripsi dari `RootLayerModule.smali` melalui `SeedHelper.smali` ke native `liblocal-config-lib.so`, dan akhirnya ke `DESHelper.kt` (dikompilasi sebagai `z6/a.smali`). Analisis lintas versi diperluas ke v3.74 dan v3.82 untuk mengkonfirmasi konsistensi seed dan mendokumentasikan migrasi dari `.comment` ke `.rodata`.
