# Manifest Şeması

Bir cihazın **ne olduğunu ve ne yapabildiğini** tanımlayan JSON. Uygulama açılışta bu dosyaları
okur, cihaz bağlanınca kimliği eşleştirir ve arayüzü **tamamen bu dosyadan** üretir.
Model başına UI kodu yoktur.

Bu belge manifest'in sözleşmesidir: hangi alan zorunlu, hangi değerler geçerli, hangi bilgi
JSON'a girer, hangisi koda kalır.

---

## 0. Neden her şey JSON değil

İstek şuydu: protokoller dahil her şey JSON'da olsun. O yola tam girilirse JSON bir programlama
diline dönüşür — kendi değişkenleri, koşulları, döngüleri olan kötü bir dil. Bakımı imkânsızlaşır.

Doğru bölünme:

| Bilgi türü | Nerede | Neden |
|---|---|---|
| Model listesi, VID:PID, kimlik baytları | **JSON** | Veri. Model eklemek için kod değişmez. |
| Yetenekler (ışık var mı, ekran kaç piksel) | **JSON** | Veri + UI'yı sürer. |
| Efekt tablosu (isim, firmware değeri, hangi alanlar aktif) | **JSON** | Donanımın tablosu, mantık değil. |
| Tuş yerleşimi (x, y, w, h, matris indeksi, LED indeksi) | **JSON** | Veri. |
| Ayar tanımları (debounce seçenekleri, uyku süreleri) | **JSON** | Veri. |
| Sınırlar (matris boyu, LED sayısı, makro belleği) | **JSON** | Veri. |
| Çerçeve biçimi, checksum, komut sırası, parçalama | **Kod** | Algoritma. Her aile için 1 dosya. |
| Okuma-yazma-değiştirme mantığı, hata yönetimi | **Kod** | Algoritma. Aile başına 1 kez. |
| Arayüz düzeni | **Kod** (tek, genel) | Bir kez yazılır, tüm modeller kullanır. |

Sonuç: **var olan bir aileye yeni model eklemek = sadece JSON.** Yeni bir aile eklemek =
1 protokol dosyası + JSON. Uygulamada 24 klavye × 1 manifest, 12 fare × 1 manifest.

Bir modelin JSON'unda protokol kodu yoktur; yalnızca `"family": "g5kb"` yazar. Protokolün
bayt düzeni ve kaynak depo notu `protocols/g5kb.js`'in başında.

---

## 1. Dosya düzeni

```
models/
├─ index.json                  katalog: hangi manifest var, hangi ailede
├─ keyboards/
│  ├─ f75.json  f75pro.json  f87.json  ...
└─ mice/
   ├─ sc620.json  sc900.json  ...
layouts/
├─ f75.json                    tuş yerleşimi (birden çok model paylaşabilir)
└─ s75pro.json
```

- Manifest yolu `models/index.json`'dan gelir; uygulama hepsini birden indirmez, cihaz
  bağlanınca aday manifestleri indirir.
- Yerleşim dosyaları paylaşılır: F75 ve F75 Pro aynı `layouts/f75.json`'u kullanır.
- Her dosya aynı origin'den gelir. Uzak URL'den manifest yüklenmez (`assets/`, `layouts/`
  dışına çıkan yol reddedilir).

---

## 2. Üst düzey alanlar

```json
{
  "schema": 1,
  "id": "f75",
  "name": "AULA F75",
  "family": "g5kb",
  "kind": "keyboard",
  "status": "verified",
  "usb": [ /* aşağıda */ ],
  "probe": { /* aşağıda */ },
  "caps": { /* aşağıda */ },
  "layout": "layouts/f75.json",
  "matrix": { "len": 128 },
  "light": { /* §4 */ },
  "perKey": { "leds": 90 },
  "macros": { "maxBytes": 3584, "nameMax": 30, "playModes": ["repeatN", "hold", "untilAny"] },
  "settings": [ /* §6 */ ],
  "battery": { "level": true, "charging": false },
  "specs": { "layout": "75% · 80 keys", "connectivity": "USB-C · 2.4 GHz" }
}
```

| Alan | Zorunlu | Açıklama |
|---|---|---|
| `schema` | ✔ | Şema sürümü. Şu an `1`. Uyumsuzda yükleme reddedilir. |
| `id` | ✔ | Benzersiz, küçük harf, `[a-z0-9-]`. Dosya adıyla aynı olmalı. |
| `name` | ✔ | Kullanıcıya görünen ad. `i18n` anahtarı da olabilir (`"@model.f75"`). |
| `family` | ✔ | `protocols/` altındaki dosya adı: `g5kb`, `aula04`, `aa`, `mouse55`, `incott`, `yxt`, `evision`, `hfd`, `bytetech`, `xsyd`, `oem1ca5`, `hubmouse`, `i8`. |
| `kind` | ✔ | `keyboard` \| `mouse`. Hangi sekme setinin açılacağını belirler. |
| `status` | ✔ | `verified` (donanımda test edildi) \| `experimental` \| `exploring` (salt-okunur). |
| `usb` | ✔ | Aday bağlantılar. Boş olamaz. |
| `probe` | ✔ | Kimlik doğrulama. `status: verified` ise `strong` olmalı. |
| `caps` | ✔ | Yetenek sözlüğü. |
| `layout` | keyboard ✔ | Yerleşim dosyası yolu. |
| `matrix` | keyboard ✔ | `len` = tuş matrisi giriş sayısı (4 byte/giriş). |
| `light` | `caps.lighting` ✔ | Efekt tablosu + sınırlar. |
| `perKey` | `caps.perKey` ✔ | `leds` = adreslenebilir LED sayısı. |
| `macros` | `caps.macros` ✔ | Bellek ve mod sınırları. |
| `settings` | `caps.settings` ✔ | Ayar tanımları. |
| `battery` | `caps.battery` ✔ | `level`, `charging` bayrakları. |
| `specs` | — | Yalnız gösterim; Dashboard'daki künye. |
| `notes` | — | Tek satır uyarı; arayüzde bilgi ikonu olarak görünür. |

Bilinmeyen üst düzey alan **reddedilir** (yazım hatası sessizce yutulmasın).

---

## 3. `usb` ve `probe`

### usb

```json
"usb": [
  { "vid": "258a", "pid": "010c", "role": "usb",    "usagePage": 65280 },
  { "vid": "3554", "pid": "fa09", "role": "dongle", "usagePage": 65280 }
]
```

| Alan | Açıklama |
|---|---|
| `vid`, `pid` | 4 haneli onaltılı, tırnak içinde. |
| `role` | `usb` (kablo) \| `dongle` (2.4 GHz). |
| `usagePage` | Config koleksiyonunun sayfa numarası (ondalık). Zorunlu değil ama önerilir: aynı VID:PID altında birden çok arayüz var. |
| `interface` | MI_xx numarası, biliniyorsa. Yalnız ipucu; asıl seçim `usagePage` + rapor boyu ile. |

VID:PID **tek başına kanıt değildir**; yalnızca aday üretir. Kesin seçim `probe` ile yapılır.

### probe

Kimlik doğrulama, manifest'te bildirilir; çözme kodu ailede.

```json
"probe": { "op": "identify", "accept": ["0300000000cd"], "strong": true }
```

| Alan | Açıklama |
|---|---|
| `op` | Ailenin dışa açtığı kimlik fonksiyonu. Çoğunda `identify`. |
| `accept` | Fonksiyonun döndürebileceği değerler. Biri eşleşirse model bu. |
| `strong` | `true` → eşleşmezse bu manifest kesinlikle reddedilir. `false` → diğer adaylara geçilir. |
| `fallback` | `true` → kimlik okunamadıysa (cihaz cevap vermiyor) salt-okunur modda bu manifest kullanılır. `strong` ile birlikte kullanılamaz. |

Ailelerin `identify` dönüşleri:

| Aile | Dönüş | Örnek |
|---|---|---|
| `g5kb` | 6-byte parola, bitişik onaltılı | `"0300000000cd"` |
| `aula04` | ürün adından çıkarılan anahtar | `"s75"`, `"l99"` |
| `aa` | cihaz bilgisi baytlarından anahtar | `"2711"` |
| `bytetech` | 6-byte UUID, bitişik onaltılı | `"110000000003"` |
| `xsyd` | VID:PID eşi | `"1ca2:1901"` |
| `evision` | configId → model anahtarı | `"v9pro"` |
| `hfd` | cihazın bildirdiği VID:PID | `"38a6:2729"` |
| `incott` | kimlik yanıtı baytları | `"0c02f0f0"` |
| diğerleri | ürün adı | `"AULA SC620"` |

Kanıtlanmamış bir değeri `accept`'e yazmak yanlış modele yazma riski doğurur. Emin değilsen
`status: "exploring"` kullan: cihaz tanınır, yazma kilitli kalır.

---

## 4. `caps` — yetenek sözlüğü

Kapalı liste. Yeni anahtar eklemek kod değişikliği gerektirir (bilinçli).

| Anahtar | Anlamı | Manifest bölümü | Açtığı arayüz |
|---|---|---|---|
| `lighting` | Efekt tabanlı arka ışık | `light` | Lighting sekmesi: efekt galerisi |
| `perKey` | LED başına renk | `perKey` | Lighting → Per-key fırçası |
| `keymap` | Tuş yeniden atama | `layout`, `matrix` | Keys sekmesi |
| `fnLayer` | İkinci katman | `layout` (`idxFn`) | Keys → Fn |
| `macros` | Cihazda makro | `macros` | Macros sekmesi |
| `settings` | Cihaz ayarları | `settings` | Ayarlar bloğu |
| `battery` | Pil seviyesi | `battery` | Dashboard pil kartı |
| `charging` | Şarj durumu | `battery.charging` | Pil kartı şarj ikonu |
| `screen` | TFT ekran | `screen` | Screen sekmesi |
| `clock` | Saat senkronu | — | Screen → Saat |
| `dpi` | DPI kademeleri | `mouse.dpi` | Mouse → DPI |
| `polling` | Yoklama hızı | `mouse.polling` | Mouse → Performans |
| `sensor` | LOD, angle snap, motion sync | `mouse.sensor` | Mouse → Sensör |
| `power` | Uyku, debounce | `mouse.power` | Mouse → Güç |
| `buttons` | Buton yeniden atama | `mouse.buttons` | Mouse → Butonlar |
| `hall` | Manyetik anahtar telemetrisi | `hall` | HE sekmesi |
| `matrix` | LED matris animasyonu | `matrixFx` | Lighting → Matris |
| `reset` | Fabrika sıfırlama | — | Settings → Tehlikeli |
| `readBack` | Yazılanı geri okuyabilir | — | "Doğrulandı" rozeti |
| `liveLight` | Anlık LED durumu okunabilir | — | Canlı önizleme |

`true` yazılan her yeteneğin bölümü zorunlu; bölüm varsa yetenek de `true` olmalı.
İkisinden biri eksikse **doğrulama hatası** — bu, "JSON'a göre her şey çalışır" güvencesini tutan mekanizma.

---

## 5. `light`

```json
"light": {
  "brightness": { "max": 9, "default": 5 },
  "speed":      { "max": 4, "default": 2 },
  "modes": [
    { "id": "fixed",   "hw": 1,  "label": "Fixed on",   "color": true, "speed": true,  "dir": false },
    { "id": "rainbow", "hw": 3,  "label": "Rainbow",    "color": false, "speed": true, "dir": true },
    { "id": "off",     "hw": 0,  "label": "Off" }
  ]
}
```

| Efekt alanı | Anlamı |
|---|---|
| `id` | Kararlı anahtar; profiller bunu saklar. Firmware değeri değişse de profil bozulmaz. |
| `hw` | Firmware'e yazılan değer. UI sırası ile firmware sırası farklıysa buradadır. |
| `label` | İngilizce etiket; `i18n` anahtarı `lighting.effect.<id>` varsa o kullanılır. |
| `color` | Renk seçici gösterilsin mi. |
| `colorful` | "Çok renkli" anahtarı. |
| `speed` / `dir` | Hız / yön kontrolü. |
| `brightnessOnly` | Efekt yalnız parlaklık kullanıyorsa (yan ışık gibi). |
| `profileOnly` | Efekt seçilemez, yalnız profil içinden uygulanır (`0x80` per-key gibi). |

Mode listesinde `off` bulunmalı. `color: false` olan efektte renk seçici gizlenir —
hangi efektin neyi desteklediği JSON'da, kodda değil.

### Yan ışık / ikinci bölge

```json
"light": { "zones": [ { "id": "main", "label": "Keys" }, { "id": "bar", "label": "Light bar", "modes": [ /* ... */ ] } ] }
```

`zones` yoksa tek bölge vardır. Bölge başına ayrı efekt tablosu verilebilir (S75 Pro yan ışık).

---

## 6. `settings` — cihaz ayarları

```json
"settings": [
  { "key": "debounce", "type": "select", "default": 1,
    "options": [[1, "1"], [2, "2"], [3, "3"], [4, "4"]] },
  { "key": "sleep", "type": "select", "default": 10,
    "options": [[0, "Never"], [10, "5 min"], [20, "10 min"]] },
  { "key": "winLock", "type": "check", "default": 0 }
]
```

| Tip | Kontrol | Zorunlu alan |
|---|---|---|
| `check` | Anahtar | `default` (0/1) |
| `select` | Açılır liste | `options`: `[değer, etiket]` |
| `number` | Sayı alanı | `min`, `max`, `step`, `default` |
| `range` | Kaydırıcı | `min`, `max`, `step`, `default` |

`label` verilmezse `i18n` anahtarı `settings.<key>` kullanılır; o da yoksa `key` gösterilir.
Etiketler Türkçeleştirme için JSON'a gömülmez.

---

## 7. Yerleşim dosyası

`layouts/*.json` — bir veya daha çok modelin paylaştığı tuş haritası.

```json
{
  "schema": 1,
  "name": "75% ANSI · 82 keys",
  "width": 679, "height": 284,
  "keys": [
    { "code": 41, "label": "Esc", "idx": 0, "light": 0, "x": 30, "y": 30, "w": 25, "h": 27, "def": [0,0,41,0] },
    { "code": 30, "label": "1",   "idx": 7, "light": 7, "x": 70, "y": 81, "w": 25, "h": 27, "def": [0,0,30,0] }
  ]
}
```

| Alan | Zorunlu | Açıklama |
|---|---|---|
| `code` | ✔ | HID usage. Tuşun kimliği; atamalar buna bağlanır. |
| `label` | ✔ | Kapak üstü yazı. Çok satır için `\n`. |
| `idx` | ✔ | Tuş matrisindeki giriş (`idx × 4` offset). `< matrix.len` olmalı. |
| `light` | perKey ✔ | LED indeksi. `< perKey.leds` olmalı. |
| `x y w h` | ✔ | Yerleşim kutusu, yerleşim biriminde. |
| `def` | — | Fabrika baytları; "sıfırla" bunları yazar. Yoksa `[0,0,code,0]`. |
| `idxFn` | fnLayer ✔ | Fn katmanındaki indeks, farklıysa. |
| `group` | — | Grup etiketi (`"nav"`, `"num"`), sadece görsel. |
| `side` | — | `left` \| `right`; geniş tuşların etiket hizası. |

Doğrulama: aynı `idx` iki kez kullanılamaz; `code` tekil olmalı; tüm `idx` < `matrix.len`;
tüm `light` < `perKey.leds`.

---

## 8. Fare bölümleri

```json
"mouse": {
  "dpi":     { "min": 100, "max": 26000, "step": 50, "stages": 6,
               "defaults": [800,1600,3200,4800,8000,12000],
               "separateXY": false, "colors": true },
  "polling": { "wired": [125,250,500,1000], "wireless": [125,250,500,1000,2000,4000,8000] },
  "sensor":  { "lod": [1,2], "angleSnap": true, "motionSync": true, "ripple": true, "rotation": [-30,30] },
  "power":   { "sleep": [1,3,5,10,20,30,60], "debounce": [1,2,4,6,8,10] },
  "buttons": { "slots": [ { "id": 0, "label": "Left", "locked": true } ] }
}
```

| Alan | Anlamı |
|---|---|
| `dpi.stages` | Kademe sayısı. `defaults` uzunluğu buna eşit olmalı. |
| `dpi.separateXY` | X ve Y ayrı girilebiliyor mu. |
| `polling.wired` / `wireless` | O bağlantıda seçilebilen Hz değerleri. Boş dizi = kontrol yok. |
| `sensor.lod` | mm cinsinden seçenekler; boş dizi = desteklenmiyor. |
| `power.sleep` | Dakika seçenekleri. |
| `buttons.slots[].locked` | Yeniden atanamaz (sol tık gibi). |

Bir alt anahtar yoksa o kontrol gösterilmez. `"sensor": { "lod": [] }` → LOD gizli.

---

## 9. Doğrulama kuralları

Uygulama her manifesti yüklemeden önce doğrular. Hata → manifest yüklenmez, günlüğe yazılır,
cihaz "tanınmıyor" görünür. **Doğrulamadan geçmemiş manifest ile cihaza yazılmaz.**

| # | Kural |
|---|---|
| 1 | `schema` desteklenen sürüm olmalı. |
| 2 | Bilinmeyen üst düzey anahtar yok. |
| 3 | `id` dosya adıyla aynı, `[a-z0-9-]+`. |
| 4 | `family` `protocols/` altında var. |
| 5 | `usb` boş değil; `vid`/`pid` 4 haneli onaltılı. |
| 6 | Her `caps: true` için ilgili bölüm var; her bölüm için `caps: true`. |
| 7 | `light.modes` içinde `off` var; `hw` değerleri tekil. |
| 8 | `perKey.leds` > 0; layout'taki tüm `light` bundan küçük. |
| 9 | `matrix.len` > 0; layout'taki tüm `idx` bundan küçük. |
| 10 | `layout` yolu `layouts/` altında; `..` yok. |
| 11 | `dpi.defaults` uzunluğu `stages`'a eşit; `min < max`, `step > 0`. |
| 12 | `probe.accept` boş değil. |
| 13 | `status: verified` ise `probe.strong: true`. |
| 14 | Aynı `vid:pid:role` birden çok manifestte `strong` probe ile talep edilemez — biri `fallback`, biri `strong` olmalı ya da VID:PID'ler ayrılmalı. |

Doğrulayıcı `core/manifest.js` içinde, ~150 satır, bağımlılıksız. Testleri gerçek manifestleri
ve bozuk örnekleri (`tests/fixtures/bad-manifests/`) koşar.

---

## 10. Yeni model ekleme akışı

Senin tarif ettiğin süreç, adım adım:

### 1. Kaynağı bul
Cihazın hangi ailede olduğunu belirle. Aile biliniyorsa gerisi veri toplama.

| Aile | Bakılacak repo | Ne alınır |
|---|---|---|
| `g5kb` | open-aula `PROTOCOL.md`, AulaSofwareLinux `KB.ini` | 6-byte parola, tuş/LED sayısı, efekt tablosu |
| `aula04` | open-aula `PROTOCOL.md`, parsiya/f108-pro | ürün adı, ayar byte'ları, efekt listesi |
| `bytetech` | shizunavn/OpenAula `hub-aulacn-protocol.md` | UUID, tuş yerleşimi, AP/RT sınırları |
| `evision` | shDavlatbek/aulaar `PROTOCOL.md` | configId, 128-byte blok alanları |
| … | `protocols/<aile>.js` başındaki kaynak notu | aynı ailenin diğer modelleri |

### 2. Manifest'i yaz
`models/keyboards/<id>.json`. Var olan bir modeli kopyala, kimlik ve sınırları değiştir.

### 3. Doğrula
```bash
node --test tests/manifest.test.js
```
Şema, çapraz referanslar ve yerleşim sınırları kontrol edilir.

### 4. Simülatörde dene
`?sim=<id>` ile aç. Arayüz doğru sekmeleri açıyor mu, yanlış alan görünüyor mu?

### 5. Donanımda dene
Cihaz takılıysa Log sekmesinde kimlik cevabı `accept` değerlerinden biriyle eşleşiyor mu.
Eşleşiyorsa ışık → tuş → makro sırasıyla dene, `docs/HARDWARE-TEST.md`'yi işaretle.

### 6. Durumu yükselt
Test geçtiyse `status: "verified"` + `probe.strong: true`. Geçmediyse `experimental` kalır,
`notes` alanına neyin çalışmadığı yazılır.

**Kod değişikliği gerekmez.** Yeni aile ise `protocols/<family>.js` yazılır (bir kez, o ailedeki
tüm modeller için).

---

## 11. Örnek: F75

`models/keyboards/f75.json`:

```json
{
  "schema": 1,
  "id": "f75",
  "name": "AULA F75",
  "family": "g5kb",
  "kind": "keyboard",
  "status": "verified",
  "usb": [
    { "vid": "258a", "pid": "010c", "role": "usb",    "usagePage": 65280 },
    { "vid": "3554", "pid": "fa09", "role": "dongle", "usagePage": 65280 }
  ],
  "probe": { "op": "identify", "accept": ["0300000000cd", "030000000000e1"], "strong": true },
  "caps": {
    "lighting": true, "perKey": true, "keymap": true, "fnLayer": true,
    "macros": true, "settings": true, "battery": true, "readBack": true
  },
  "layout": "layouts/f75.json",
  "matrix": { "len": 128 },
  "light": {
    "brightness": { "max": 9, "default": 5 },
    "speed": { "max": 4, "default": 2 },
    "modes": [
      { "id": "fixed",    "hw": 1,  "label": "Fixed on",     "color": true,  "speed": true },
      { "id": "respire",  "hw": 3,  "label": "Respire",      "color": true,  "speed": true },
      { "id": "rainbow",  "hw": 2,  "label": "Rainbow",      "color": false, "speed": true },
      { "id": "flash",    "hw": 19, "label": "Flash away",   "color": true,  "speed": true },
      { "id": "raindrops", "hw": 15, "label": "Raindrops",   "color": true,  "speed": true },
      { "id": "wheel",    "hw": 13, "label": "Rainbow wheel", "color": false, "speed": true },
      { "id": "ripples",  "hw": 20, "label": "Ripples shining", "color": true, "speed": true },
      { "id": "stars",    "hw": 16, "label": "Stars twinkle", "color": true, "speed": true },
      { "id": "shadow",   "hw": 18, "label": "Shadow disappear", "color": true, "speed": true },
      { "id": "snake",    "hw": 5,  "label": "Retro snake",  "color": true,  "speed": true },
      { "id": "neon",     "hw": 7,  "label": "Neon stream",  "color": true,  "speed": true },
      { "id": "reaction", "hw": 17, "label": "Reaction",     "color": true,  "speed": true },
      { "id": "sine",     "hw": 12, "label": "Sine wave",    "color": true,  "speed": true },
      { "id": "scan",     "hw": 8,  "label": "Retinue scanning", "color": true, "speed": true },
      { "id": "windmill", "hw": 28, "label": "Rotating windmill", "color": false, "speed": true },
      { "id": "waterfall","hw": 30, "label": "Colorful waterfall", "color": false, "speed": true },
      { "id": "bloom",    "hw": 14, "label": "Blossoming",   "color": false, "speed": true },
      { "id": "storm",    "hw": 29, "label": "Rotating storm", "color": true, "speed": true },
      { "id": "off",      "hw": 0,  "label": "Off" }
    ]
  },
  "perKey": { "leds": 90 },
  "macros": { "maxBytes": 3584, "nameMax": 30, "playModes": ["repeatN", "hold", "untilAny"] },
  "settings": [
    { "key": "debounce", "type": "select", "default": 1, "options": [[1,"1"],[2,"2"],[3,"3"],[4,"4"]] },
    { "key": "sleep", "type": "select", "default": 10,
      "options": [[0,"Never"],[2,"1 min"],[4,"2 min"],[6,"3 min"],[10,"5 min"],[20,"10 min"],[30,"15 min"]] }
  ],
  "battery": { "level": true, "charging": false },
  "specs": {
    "layout": "75% · 80 keys",
    "connectivity": "USB-C · 2.4 GHz · Bluetooth 5.0",
    "switch": "Hot-swap 3/5-pin",
    "lighting": "RGB · 18 effects"
  }
}
```

---

## 12. Sürümleme

- `schema` artınca eski manifestler reddedilir; geçiş betiği yazılmaz (az sayıda dosya, elle düzeltilir).
- Profiller `manifest.id` + `light.modes[].id` saklar. Firmware değeri (`hw`) değişse bile
  kullanıcı profili çalışmaya devam eder. Bu yüzden `id` alanları **yeniden adlandırılmaz**;
  yeni efekt eklenirse yeni `id` verilir.
- Manifest'te `version` alanı yoktur: git geçmişi yeterli.

---

## 13. Bilinçli sınırlar

1. Manifest **hangi işlemlerin sırasını** tanımlamaz. Yazma dizileri ailede.
2. Manifest **yeni bir çerçeve biçimi** tanımlamaz. Farklı çerçeve = yeni aile.
3. Manifest **hesaplama** yapmaz (koşul, formül, değişken). Sabit veri.
4. Manifest **görsel düzen** tanımlamaz; yalnızca hangi kontrolün görüneceğini söyler
   (`caps` + bölümün varlığı). Yerleşim `styles/` + `app/views/` içinde bir kez.
