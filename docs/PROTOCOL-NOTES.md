# Protokol Notları — AulaBridge

Her protokol için: taşıma, çerçeve, komutlar, kimlik, yetenekler, doğrulama durumu, kaynak.
Kod yazarken başvuru belgesi. `protocols/*.js` modülleri bu tabloların birebir karşılığıdır.

Durum: **V** = gerçek donanımda doğrulanmış · **C** = satıcı sürücüsünden/koddan çıkarılmış, donanımda denenmemiş · **?** = eksik.

---

## 0. Ortak yardımcılar

```
tag(buf, off)        buf[off]=0xAA, buf[off+1]=0x55          // blok sonu imzası
sum8(buf, from, to)  toplam & 0xFF                            // basit checksum
chunks(data, size)   ceil(len/size) parça, her biri size'a zero-pad
hex(buf, n)          ilk n byte, "aa bb cc" biçiminde         // log için
```

Yazma kuralı (hepsi için): **oku → değişen byte'ı yaz → bilinmeyeni koru → imza/checksum doğrula**.
Okuma başarısızsa yazma reddedilir.

---

## 1. `g5kb` — SinoWealth BYCOMBO4 (F75 ailesi)

**Modeller:** F75, F75 Pro, F87, F87 Pro, F99, F99 Pro, F108, H108 · **Durum:** V (F75)

### Taşıma
| | |
|---|---|
| Kablo | `258A:010C`, usage page `0xFF00`, usage `0x0001` |
| Dongle | `3554:FA09`, aynı sayfa |
| Rapor | **520 byte feature**, report ID **6** |
| Hız | paket başına ~20 ms bekleme, yazma sonrası `GetFeature` ile teyit |

Chrome `receiveFeatureReport(6)` çağrısında baştaki report-ID byte'ını da döner → önce strip et.

### Çerçeve
```
06  cmd  sub  00  total  idx  lenLo  lenHi   payload[≤512]
```
- Yazma: `cmd` düz. Okuma: `cmd | 0x80`, sonra `getFeature(6, 0x208)`.
- 512 byte'lık parçalar; `total` parça sayısı, `idx` sıra.
- Bazı bloklarda son 2 byte `5A A5` imzası (LED bloğu ve RGB tablosu).

### Komutlar
| Yaz | Oku | Blok | İçerik |
|---|---|---|---|
| `03` sub=layer | `83` | 512 | tuş matrisi, `MatrixLen`×4 byte (128 giriş) |
| `04` | `84` | 128 | LED/ayar bloğu, imza `5A A5` @126 |
| `05` | `85` | değişken | makro tamponu |
| `06` sub=grup | `86` | — | renk grubu (3 düzlem) |
| `0A` | `8A` | 512 | efekt renk tablosu, 21 byte/slot @21 |
| `0B` | — | — | ekran parametreleri |
| `11` | — | 1 | fabrika sıfırlama (değeri bilinmiyor → kullanılmıyor) |
| — | `02` sub=1 | 6 | **kimlik parolası** |

### Kimlik (model kapısı)
Kablo ve dongle aynı VID:PID'yi paylaşır; model **yalnızca** 6-byte parola ile ayrılır:

| Parola | Model | Tuş | LED |
|---|---|---|---|
| `03 00 00 00 00 CD` | F75 | 82 | 90 |
| `03 00 00 00 00 E1` | F75 wired | 82 | 90 |
| `03 00 00 00 00 8F` | F87 | 87 | 102 |
| `03 00 00 00 01 0B` | F87 Pro | 87 | 102 |
| `03 00 00 00 00 A9` | F87 Pro wired | 87 | 102 |
| `03 00 00 00 00 A4` | F99 | 99 | 113 |
| `03 00 00 00 01 58` | F99 Pro | 100 | 113 |
| `03 00 00 00 00 9A` | F99 wired | 99 | 113 |
| `03 00 00 00 03 7C` | H108 | 108 | 125 |

Bilinmeyen parola → salt-okunur mod + "model doğrulanamadı" uyarısı.

### LED bloğu (read-modify-write)
| Offset | Alan |
|---|---|
| `0x03` | debounce |
| `0x09` | firmware "uygulandı" bayrağı — 1 ise yazma yok sayılır |
| `0x0A` | aktif donanım efekti |
| `0x0B` | knob işlevi (0 ışık, 1 ses) — uygulamadan yazılmaz |
| `0x18` | uyku süresi |
| `0x3A + 2i` | efekt *i* parlaklığı (0–9) |
| `0x3B + 2i` | `hız<<4 | renkIndex` (0–6 palet, 7 rastgele) |

### Efektler (18 + kapalı)
Efekt tablosu `KB.ini` `LedOpt` sırasındadır; UI indeksi ≠ firmware değeri. Eşleme model dosyasında:
`{mode: 1, hw: 1, name:'Fixed on'}` … `{mode:18, hw:29, name:'Rotating storm'}`.

### Tuş girişi (4 byte)
| Byte'lar | Anlam |
|---|---|
| `00 00 00 00` | **hiçbir işlev yok** — bu yüzden her tuşun varsayılanı yazılmalı |
| `00 mods hid 00` | klavye tuşu |
| `02 00 hi lo` | tüketici (medya) tuşu |
| `01 btn 01 00` | fare tuşu (1 L, 2 R, 3 M, 4/5 yan) |
| `03 mode count id` | makro (mode 1×N, 2 basılı tut, 4 herhangi tuşa kadar) |
| `0D 00 00 00` | Fn |
| `07 00 00 1D` | knob |

### Makro tamponu
- Tablo: makro başına `u16 addr, u16 size`, adresler tampon başından.
- Her makro: ad uzunluğu (1 byte), UTF-16LE ad, sonra 4-byte olaylar.
- Olay: `type<<4 | up<<7 | delay[19:16], delay[15:8], delay[7:0], code`. Tip 1 tuş, 2 fare, 5 tekerlek.

### Yetenekler
`lighting, perkey, keymap, fnLayer, macros, settings, battery(sorgu), debounce, sleep`
Ekran **yok**. 2.4G üzerinden ekran/GIF aktarımı yok.

---

## 2. `aula04` — Sonix "04" (S75 Pro ailesi)

**Modeller:** S75 Pro, L99, F108 Pro, F75 Max, F98 Pro, S98 Pro, S99 Pro · **Durum:** V (S75 Pro)

### Taşıma
| | |
|---|---|
| Kablo | `0C45:800A`; config = **64-byte feature**, report ID 0 |
| Kablo (ekran) | aynı cihaz, **4096-byte output** koleksiyonu |
| Dongle | `0C45:FEFE` / `05AC:024F` MI_03, **32-byte** input+output |
| Gecikme | komut başına 35 ms, ardından 1 feature okuması (ack) |

### USB çerçevesi
```
64-byte feature:  04  op  b2  00 00 00 00 00  countLo  countHi  …  [veri]  …  AA 55
```
`count` = ardından gelecek 64-byte veri paketi sayısı.
Yazma dizisi: `18` (başla) → `<op>` → veri → `02` (bitir) → `F0` (uygula).

### Komutlar
| Op | Anlam | Veri |
|---|---|---|
| `18` / `02` / `F0` | başla / bitir / uygula | — |
| `13` | ışık | `mode r g b 0 0 0 0 colorful bright speed dir … AA55` |
| `17` b2=1 | ayarlar | `00 01 winLock altF4 altTab fnSwitch sleep 00 response … AA55` |
| `28` | saat | `00 slot 5A yy mm dd hh mi ss 00 wday … AA55` |
| `11` / `27` | tuş haritası / Fn | 9 paket, 576 byte, 4 byte/tuş |
| `23` | per-key renk (mode `0x80`) | 9 paket, `[light r g b]` |
| `F5` | per-key renk oku | 9 feature okuması |
| `20` count=8 | canlı per-key (müzik) | 512 byte |
| `19`→`15` | makrolar | aşağıda |
| `72` b2=slot | ekran | `count` × 4096-byte output |
| — | kimlik | **yok**; model ürün adından |

### Efektler (19 + kapalı)
1 Static, 2 Single On, 3 Single Off, 4 Glittering, 5 Fluttering, 6 Colourful, 7 Breath, 8 Spectrum,
9 Outward, 10 Scrolling, 11 Rolling, 12 Rotating, 13 Explode, 14 Launch, 15 Ripples, 16 Flowing,
17 Pulsating, 18 Tilt, 19 Shuttle, 0 Off, `0x80` per-key.
Parlaklık/hız 0–5; yön 0 sol, 1 sağ, 2 yukarı, 3 aşağı.
F87 Pro ailesinde ek: `17` Rainbow Windmill, `18` Colour Gather, `19` Neon Overlay.

### Yan ışık (deneysel)
Aynı `13` komutu, `mode` = `0x20`…`0x24`. S75 Pro'da firmware sürücünün listesini izlemiyor:
`22`→mavi, `23`→pembe/neon, `24`→kapalı; `25`–`29` etkisiz. UI yalnızca 20–24 gösterir.

### Tuş girişi (4 byte) — g5kb'den **farklı**
| Byte'lar | Anlam |
|---|---|
| `00 00 00 00` | varsayılan |
| `05 03 00 00` | devre dışı |
| `02 mods hid 00` | tuş / kombinasyon |
| `01 01/02/04/08/10 …` | fare tuşu (sol, sağ, orta, 4, 5); `01 03` çift tık |
| `01 03 01/FF 00` | tekerlek yukarı / aşağı |
| `03 lo hi 00` | tüketici |
| `06 index mode loops` | makro (0 bir kez, 1 N kez, 2 aç/kapa, 3 basılı, 4 tıkla) |
| `05 02 id 00` | sürücü işlevleri (program/URL/metin) — taşınabilir değil, atlanır |

### Makrolar
- Tampon `count × 64`'e padlenir, `AA55` sonda; `count = len/64 + (len%64 ? 2 : 1)`.
- Byte 0–399: makro başına `u16 LE` offset (`FF FF FF FF` = boş).
- Offset'te: `u16 adım sayısı`, 6 sıfır byte, sonra 4-byte adımlar:
  `00 00 hid b0` bas, `00 00 hid 30` bırak, `00 00 btn 90/10` fare, `lo hi 00 50` gecikme (min 10 ms).

### Ekran (S75 Pro, 135×240)
- 256-byte başlık + kareler, 4096'nın katına `FF` ile padlenir.
- Başlık byte 0 = kare sayısı (≤141); byte 1..n = kare gecikmesi / 2 ms.
- Kareler RGB565 little-endian, yukarıdan aşağı.
- Dizi: `18` → `72 slot count` → `count` × 4096 output (her biri ≤300 ms input bekler) → `02`.
- L99: 320×480, ≤200 kare. **L99'un ekranı ayrıca `EEEF:268A` COM cihazı** olarak da görünüyor
  (`12 34 56 78 02` imzası) — iki yol var, hangisinin doğru olduğu donanımda belirlenecek.

### 2.4 GHz çerçevesi
```
32-byte output:  cmd  len  seq  data[28]  checksum        checksum = ilk 31 byte toplamı & 0xFF
```
Dongle `cmd len seq`'i input olarak yansıtır.

| cmd | Anlam |
|---|---|
| `05` | ışık (USB payload'u, `aa55` @data[14]) |
| `07` | ayarlar |
| `0C` | saat |
| `10` / `12` | tuş haritası / Fn — 21 × 28 byte |
| `14` | per-key renk — 21 parça |
| `09` | makro — 28 byte parçalar |
| `20` len=1 | **pil sorgusu**, cevap byte 3 = % |
| `7F`, `80+` | ekran (kullanılmıyor) |

### Model farkları
| Model | Fark |
|---|---|
| L99 | ayar byte'ları kaymış: winLock 4, altF4 3, altTab 2. Ekran 320×480. Yan ışık yok. |
| F108 Pro | 20 efekt; bar + yan ışık yalnızca Fn kombinasyonlarıyla (HID'den erişilemiyor) |
| F75 Max | **okuma komutu yok** → yaz-only, "gönderildi" denir "doğrulandı" denmez; per-key 576 byte |
| F98 Pro / S99 Pro | protokol varsayımı; önce salt-okunur keşif |

### Yetenekler
`lighting, perkey, keymap, fnLayer, macros, settings, clock, screen, bar, battery, displayBlackout`
F75 Max: `screen` var, `readback` yok.

---

## 3. `aa` — aynı Sinix firmware, farklı çerçeve (F87 Pro sonix varyantı)

**Durum:** C · Kaynak: satıcının WebHID uygulaması (`app.asar`, obje `Xn`)

### Taşıma
USB `38A6:2710/2711/2712/2713`, `0C45:8803`; dongle `0C45:FEFC` (page `FF80`), `0C45:FEFE` (`FF60`).
Report ID 0; boyut 32, `0xFF68` sayfasında 64.

### Çerçeve
```
AA cmd len offLo offHi o5 o6 o7 data[packLen-8]
```
Cihaz **her** paketi `cmd` yankılayarak yanıtlar; sıradaki paket ancak bundan sonra gider.
Okuma cevabı byte 8'de veri taşır, `len` ve `offset` byte 2–4'te.

| Oku | Yaz | Veri |
|---|---|---|
| `10` | — | cihaz bilgisi (32 B), bayt 2–3 makro alanı boyutu |
| `11` | `21` | ayarlar (16 B): [3] uyku dk, [4] yanıt 1–5, [5] yoklama (3=1K…6=8K), [6] OS, [7] matris saat biçimi |
| `12` | `22` | temel tuş haritası, 512 B = 128 × `[type a b c]` |
| `16` | `26` | Fn katmanı |
| `13` | `23` | ışık, 16 B |
| `14` | `24` | per-key, 512 B `[index r g b]` |
| `15` | `25` | makrolar |
| — | `2A` | 7×7 LED matris animasyonu (1–14) |
| — | `00` | kaydet (`00 00 00 00`) |
| — | `0F` | `AA 0F 05` tuş sıfırla, `AA 0F FF` fabrika |

Makro: 0–399 offset tablosu (`u32 LE`), ilk makro 400'de; her makro `u16 (2 × kayıt)`, `00 00`,
sonra `delayLo delayHi hid flag` (b0 bas, 30 bırak, 90/10 fare).

### Yetenekler
`lighting, perkey, keymap, fnLayer, macros, settings, matrix, reset`

---

## 4. `mouse55` — SC620

**Durum:** C (buton haritası V)

### Taşıma
`A8A4:2255` kablo, `A8A5:2255` dongle (MI_02). 64-byte output + input, report ID 0.

### Çerçeve
```
55 cmd flag chk len offLo offHi profile data[56]
```
`chk` = byte 4..63 toplamı & 0xFF. `flag` = `A5` yazma, `AE` parametre. Cevap `AA cmd …`.
100 ms sonra tekrar, 1 s sonra son tekrar.

| cmd | Veri | |
|---|---|---|
| `03` | — | firmware (cevap 3–5 ASCII) |
| `09` | 11 × 4 | buton haritası |
| `0D` | 56'lık parçalar | makro alanı, sonra `10` commit |
| `0F` len 46 | parametre bloğu | aşağıda |
| `21` len 3 | `00 00 mode` | ışık: 0 Wave, 1 Neon, 2 Cycling flash, 3 YOYO, 4 Single flash, 5 Breathing, 6 Off |
| `22` len 3 | `00 00 stage+1 dpiLo dpiHi` | aktif DPI kademesi |
| `23` len 3 | `00 00 rate` | 1=125 … 4=1000 Hz |
| `30` len 0x2E | — | pil: byte 8 %, byte 9 şarj |

### Parametre bloğu (`0F`)
`profile, lightMode, rate, 6, stage+1, dpi[6] u16 LE (50–26000), rgb[6], 0 0 0 0,
scrollReverse, liftoff (1/2 mm), sensor, debounce, sleep (dk), esports`
Sensör bitleri: `10` ripple, `01` angle snap, `20` motion sync.

### Buton girişi
`20 mask 00 00` fare tuşu (1 L, 2 R, 4 M, 8 geri, 10 ileri); `20 01 02` çift tık;
`21 38 01/FF` tekerlek; `21 55` DPI döngüsü; `10 mods hid 00` klavye; `30 lo hi 00` tüketici;
`70 idx n 00` makro; `00 00 00 00` devre dışı.

Slot sırası (donanımda doğrulandı): 0 sol, 1 sağ, 2 tekerlek, 3 arka yan, 4 ön yan,
5 alt DPI, 6 teker yukarı, 7 teker aşağı, 9 alt yan. 8 ve 10 fiziksel tuş yok.

---

## 5. `incott` — SC900 / SC900 Pro (PixArt 8K)

**Durum:** V (SC900 Pro)

### Taşıma
`093A:622C/522C`, `248A:5B4A/5B49`, `379A:0CF0/0CEF`; usage page `0xFF05`;
**9-byte feature**, report ID `0x09`. Sorgular aynı opcode + bit 7.

### Çerçeve
```
09  cmd  sub  args…
```
Cihaz tek bir cevap tamponu tutar; komut (ve yankılayan opcode'larda alt-komut) eşleşmeden güvenilmez.

| Komut | | |
|---|---|---|
| `0D` yoklama · `0C` DPI değeri · `0B` DPI döngüsü · `0A` sensör · `09` zamanlama | yazma |
| `06` buton · `07` makro parçası · `08` alıcı LED | yazma |
| `8D 8C 8B 8A 89 86 88` | aynılarının okuması |
| `81` durum · `80` sürüm · `8F` **kimlik** | okuma |

Alt-komutlar: LOD 1, ripple 2, angle snap 3, motion sync 4, performance 5; debounce 1, fire key 2, sleep 3, profile 9.

### Değer tabloları
- DPI 50–45000, adım 50, 6 kademe.
- Yoklama kablosu: `{0:1000, 1:500, 2:250, 3:125, 4:8000, 5:4000, 6:2000}`; **kabloda max 1000 Hz**.
- LOD: `{0:1.0 mm, 1:2.0 mm, 2:0.7 mm}`.
- Makro: 10 tampon × 320 byte, 32'lik parçalar, ≤71 adım.

---

## 6. `yxt` — SC390

**Durum:** C (sürücü kaptürü)

32-byte output, report ID `0x0E`, arayüz 2.
```
0e 20 <bölüm> …
```
- Bölüm `0x00`: byte 3 = byte 4 için nibble maskesi; byte 4: bit 7–6 USB hızı, bit 5–4 2.4G hızı, alt nibble DPI kademesi. Maske 0 → okuma.
- Bölüm `0x40`: byte 12 2.4G uyku, 13 BT uyku, 14 LOD (bit 1–0) + 4 anahtar (bit 4–7: ultraLowDelay, ultraLowPower, ripple, moveWakeup).

Sadece ayar var; DPI, buton, uyku kaptürde yok.

---

## 7. `evision` — V9 / X9 ailesi + 8K dongle

**Durum:** C (AULA HUB SDK'sından) · Kaynak: shDavlatbek/aulaar + xinzore

### Taşıma
`320F:22A0` 8K dongle, `22BA` eski dongle, `22B7` V9+ serisi, `22B8` V9 serisi, `22B6` X9 serisi.
Config arayüzü: application collection **`0xFF1C` / usage `0x92`**, **report ID 4**, 63 byte.
Arayüz numarasına göre arama yapılmaz, descriptor'a göre bulunur.

Parça boyu: kabloda 56, dongle'da 24.

### Çerçeve
```
basit:   00 00 CMD len 00 00 00 …            cevap data[7] (ve data[8])
blok:    cmd(0x01) başla
         her parça: 00 00 CMD len offLo offHi 00 payload
         cmd(0x02) bitir
```

| Cmd | Tür | Anlam |
|---|---|---|
| `01`/`02` | basit | blok başlat / bitir |
| `03` | oku 10 | temel bilgi, `[7]` = **configId** |
| `03` | oku 22 | `[20]`,`[21]` firmware |
| `05`/`06` | oku/yaz 128 | **yapılandırma bloğu** |
| `07`/`08` | oku 33 | varsayılan / güncel buton matrisi |
| `09` | yaz 33 | buton matrisi |
| `0D` | basit | fabrika sıfırlama |
| `15` | yaz | makro |
| `1A` | basit | pil %, şarj bayrağı yok |
| `AA` | basit | dongle: fare çevrimiçi mi |
| `B2` | basit | eşleştirme başlat |
| `B3` | basit | alıcı firmware |

### configId → model
0 X9 Pro · 1 X9 · 2 X9 Ultra · 3 V9+ · 4 V9 Pro+ · 5 V9 Ultra+ · 6 V9 · 7 V9 Pro ·
8 V9 Ultra · 9 V9 Max+ · 10 V9 Max · 11 X9 Max

### 128-byte config bloğu
| Offset | Alan |
|---|---|
| 0 | profil (yalnız 0) |
| 1 | ışık efekti (0 Static, 1 Breathing, 2 Blink, 3 Spectrum, 4 Wave, 5 Off) |
| 2 / 3 | parlaklık 0–4 / hız 0–4 |
| 4 | yön 1–2 |
| 5 | colorful (Spectrum/Wave için 1) |
| 6–8 | R G B |
| 9 | bilinmiyor → koru |
| 10 / 11 | 2.4G / kablo yoklama indeksi (0=125 … 6=8000) |
| 12 | aktif DPI kademesi |
| 13 | LOD (0=1 mm, 1=2 mm) |
| 14 + 9i | DPI kademesi: `enabled, xySeparate, Xlo, Xhi, Ylo, Yhi, R, G, B` (100–42000, adım 50) |
| 68–69 | uyku (sn): 180/240/300/600/900/1200 |
| 70 | debounce (ms): 1/2/4/6/8/10 |
| 71 / 72 | motion sync / angle snap |
| 73 | bilinmiyor → koru |
| 74 | sensör rotasyonu int8, −30…+30 |
| 75 | bilinmiyor → koru |
| 76–78 | DPI göstergesi: efekt / parlaklık / hız |
| 79–127 | bilinmiyor → koru |

Giriş report 3 olayları: `01 04` profil değişti, `02 04` sıfırlandı, `FF FF` bağlantı hatası.

---

## 8. `hfd` — F75 HE / HE ailesi (AULA'nın kendi HFD firmware'i)

**Durum:** C · Kaynak: FaizAsta `he.js`

### Taşıma
Kablo `38A6:2729` (64-byte), dongle `0C45:FEFE` (32-byte, page `0xFF60`).
Vendor sayfaları sırayla denenir: `FF68, FF80, FF60, FF00, FF01, FF5B`; usage `0x61`.

### Çerçeve
```
magic 0xAA … cevap 0x55     veri byte 8'den başlar
```
Komutlar: `10` INFO, `13` GET_LIGHT, `14` GET_KEYS, `23` SET_LIGHT, `24` SET_KEYS.
**Yalnızca bu beş komut üretilir** (reset/firmware/calibration asla).
Zaman aşımı 600 ms, 3 deneme (satıcı sürücüsü de 3 deniyor).
Kare başına ~5 ms ack; kabloda tablo 10 kare, dongle'da 22.

### Sabitler
`INFO_SIZE 0x38` · `LIGHT_SIZE 0x10` · `LED_COUNT 128` · `KEYS_SIZE 512` · `KEYS_READ 504` (126 kayıt) · `CUSTOM 20`

Yazmadan önce kimlik okunur; cihaz kendi VID:PID'sini ilk cevapta söyler.
Karıştırılmaması gereken HFD kartları: `0C45:80A2`, `0C45:80B2`, `0C45:80A1`, `38A6:2722`, `38A6:2723`, `38A6:2724`.

---

## 9. `bytetech` — HERO 68/84/75/99/87 HE, WIN68HE Ultra, MINI

**Durum:** V (HERO 68 ve 84)

### Taşıma
Kablo `372E:103E`, usage page `0xFF60`, usage `0x61`; alıcı `372E:1000` / `372E:106C` (8K).
**Report ID 9**, çerçeve 64 byte (63 payload).

### Çerçeve
```
reportId 9 | cmd | param | h2 | … | payload … | tail(1)
header 6 byte, tail 1 byte, veri 56 byte
```
`reportId` descriptor'dan okunur (`collections[].outputReports[0].reportId`), sabit değil.
Okuma `cmd | 0x80`. CRC `TPe.calculateCrc`.

### Komutlar (özet)
`0x82` sub `0x01` → **6-byte UUID** (model kimliği, big-endian u48).
`0x10` profil seç · `0x13/0x93` aktüasyon (11 × 5-byte kayıt/paket) · `0x14/0x94` Rapid Trigger ·
`0x03` makro · `0x05` makro komutu · RGB ve remap komutları `0x1x–0x2x` aralığında.

Birimler: **0.01 mm** (HERO68/84), mikron (XS, 0.005 mm).

### Alternatif: OEM 1CA5
`1CA5:0407` (HERO 68 HE ULTRA), `1CA5:0409` (HERO 68 HE PRO) — usage page `0xFFA0`,
`sendFeatureReport`/`receiveFeatureReport`, **report ID 3**, çoklu paket birleştirme.

---

## 10. `xsyd` — WIN 68 HE MAX

**Durum:** V · Kaynak: OpenAula, 8.257 karelik kaptür + RV32IMAC disassembly ile doğrulanmış

### Taşıma
`1CA2:1901` veya `1C4F:EE88`, usage page `0xFFA0`, **report ID 0**.

### Çerçeve
```
5C  LEN  OP  CK  payload[LEN]
CK = (0x35 + 0x5C + LEN + OP + payload[LEN-1]) & 0xFF
```
Yalnızca **son** payload byte'ı toplanır. İki istisna:
- `OP < 0x80` ve kesilmiş payload → `CK = 0x00`
- `OP = 0x30` sabit 0x43 slot → toplayıcı `frame[63]`

Cevap opcode'unda bit 7 set (`0x2C` → `0xAC`). 60 byte'tan büyük cevap birden çok input report gelir.

Birimler **mikrometre**; HERO68 belgelerindeki 0.01 mm değerleri buraya taşınmaz.

---

## 11. `hubmouse` — SC800 Max / L9 / V3 Ultra B / X5 Ultra

**Durum:** C (kısmi)

`38A6:28FC / 28FE / 2F09`, usage page `0xFFA8`, usage `0x02`, **report ID 11**.
Yalnızca AULA HUB bundle'ından çıkarıldı; komut tablosu eksik. İlk sürümde **yalnızca tanınır + pil sorgusu**.

---

## 12. `i8` — F2088

**Durum:** C (kısmi, yalnız ışık)

Dahuum'un bağımsız çözümü: 8-byte normal ışık protokolü. Tuş/makro yok.
İlk sürümde **yalnızca ışık**.

---

## 13. Kapsam dışı

**G7 Pro kulaklık**, **GS8 hoparlör**: HID yapılandırma protokolü yok. Cihaz listesinde
"tanınmıyor" olarak görünür, sekmeler kapalı.

**Firmware güncelleme**: hiçbir kaynakta güvenilir/geri alınabilir yol yok. Bilinçli olarak yok.

---

## 14. Kaynak → kod eşlemesi

| Modül | Birincil kaynak | Çapraz doğrulama |
|---|---|---|
| `g5kb.js` | open-aula `PROTOCOL.md` §F75 | AulaToolkit `g5kb.ts`, AulaSofwareLinux `proto.rs`, AULA-Studio |
| `aula04.js` | open-aula `PROTOCOL.md` §S75 Pro | AulaToolkit `aula04.ts`, F75-software `max.js`, parsiya/f108-pro |
| `aa.js` | open-aula `PROTOCOL.md` §F87 Pro | marcoslor (farklı çerçeve, aynı mantık) |
| `mouse55.js` | open-aula `PROTOCOL.md` §SC620 | AulaToolkit `mouse55` yok → tek kaynak |
| `incott.js` | AulaToolkit `incott.ts` | Incott/OpenMouse topluluk notları |
| `yxt.js` | AulaToolkit `yxt.ts` | — |
| `evision.js` | shDavlatbek `PROTOCOL.md` | xinzore `protocol.ts` |
| `hfd.js` | FaizAsta `he.js` | — |
| `bytetech.js` | OpenAula `bytetech/*`, `hub-aulacn-protocol.md` | — |
| `xsyd.js` | OpenAula `xsyd/*`, `xsyd-protocol.md` | — |
| `hubmouse.js` | OpenAula `hub-aulacn-protocol.md` | — |
| `i8.js` | Dahuum `F2088_RESEARCH.md` | — |

Lisanslar: çoğu GPL-2/3 → proje GPL-3.0-or-later. MIT/Apache kaynaklı kısımlar için dosya başında
kaynak ve lisans notu tutulur. Hiçbir satıcı binary'si depoya girmez.
