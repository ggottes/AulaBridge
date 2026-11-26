# Sürücü Bildirimi Şeması

Her model kendi sürücüsüdür: `models/<id>/` klasöründe bir bildirim (`driver.json`), bir kod
dosyası (`driver.js`) ve bir tuş yerleşimi (`layout.json`). AULA her model için ayrı sürücü
yayınlıyor; bu proje de öyle yapıyor. Ortak bir "aile" katmanı yoktur — iki model aynı firmware
ailesinden gelse bile iki ayrı klasörde, iki ayrı `driver.js` ile durur.

Bu belge `driver.json`'un sözleşmesidir: hangi alan zorunlu, hangi değerler geçerli, hangi bilgi
bildirime girer, hangisi `driver.js`'e kalır.

---

## 0. Neden her şey JSON değil

İstek şuydu: protokoller dahil her şey JSON'da olsun. O yola tam girilirse JSON bir programlama
diline dönüşür — kendi değişkenleri, koşulları, döngüleri olan kötü bir dil. Bakımı imkânsızlaşır.

Doğru bölünme, **model başına**:

| Bilgi türü | Nerede | Neden |
|---|---|---|
| VID:PID, kimlik baytları | **`driver.json`** | Veri. Modelin kendi USB kimliği. |
| Yetenekler (ışık var mı, ekran kaç piksel) | **`driver.json`** | Veri + UI'yı sürer. |
| Efekt tablosu (isim, firmware değeri, hangi alanlar aktif) | **`driver.json`** | Donanımın tablosu, mantık değil. |
| Tuş yerleşimi (x, y, w, h, matris indeksi, LED indeksi) | **`layout.json`** | Veri. |
| Ayar tanımları (debounce seçenekleri, uyku süreleri) | **`driver.json`** | Veri. |
| Sınırlar (matris boyu, LED sayısı, makro belleği) | **`driver.json`** | Veri. |
| Çerçeve biçimi, checksum, komut sırası, parçalama | **`driver.js`** | Algoritma. O modelin protokolü. |
| Kimlik sorgusu | **`driver.js`** | Algoritma; beklenen cevap `driver.json`'da `probe.accept`. |
| Okuma-yazma-değiştirme mantığı, hata yönetimi | **`driver.js`** | Algoritma. |
| Arayüz düzeni | **Kod** (tek, genel) | Bir kez yazılır, tüm modeller kullanır. |

Sonuç: **yeni model = yeni klasör.** `driver.json` + `driver.js` + `layout.json` + test.
Var olan bir sürücüyü başka modele "uyarlamak" yoktur; kaynak repo okunur, kod o modelin
klasörüne yazılır.

---

## 1. Dosya düzeni

```
models/
├─ index.json                  katalog: id → klasör, durum
├─ f75/
│  ├─ driver.json              bildirim (§2–§9)
│  ├─ driver.js                o modelin protokolü (§10)
│  └─ layout.json              tuş yerleşimi (§7)
├─ f87/
│  ├─ driver.json
│  ├─ driver.js
│  └─ layout.json
├─ s75pro/  l99/  sc620/  …    her model aynı üçlü
```

- Klasör yolu `models/index.json`'dan gelir; uygulama hepsini birden indirmez, cihaz
  bağlanınca aday sürücüleri indirir.
- Yerleşim **paylaşılmaz**: F75 ile F75 Pro'nun tuş haritaları ayrı dosyalardır, ayrı ayrı
  değiştirilebilir. İki modelin yerleşimi aynıysa iki dosyaya aynı içerik yazılır — bu bir
  sorun değil, kopyadır.
- Her dosya aynı origin'den gelir. Uzak URL'den bildirim yüklenmez (`models/` dışına çıkan
  yol reddedilir).

---

## 2. Üst düzey alanlar

```json
{
  "schema": 1,
  "id": "f75",
  "name": "AULA F75",
  "kind": "keyboard",
  "status": "verified",
  "usb": [ /* aşağıda */ ],
  "probe": { /* aşağıda */ },
  "caps": { /* aşağıda */ },
  "layout": "layout.json",
  "matrix": { "len": 128 },
  "light": { /* §5 */ },
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
| `id` | ✔ | Benzersiz, küçük harf, `[a-z0-9-]`. Klasör adıyla aynı olmalı. |
| `name` | ✔ | Kullanıcıya görünen ad. `i18n` anahtarı da olabilir (`"@model.f75"`). |
| `kind` | ✔ | `keyboard` \| `mouse`. Hangi sekme setinin açılacağını belirler. |
| `status` | ✔ | `verified` (donanımda test edildi) \| `experimental` \| `exploring` (salt-okunur). |
| `usb` | ✔ | Aday bağlantılar. Boş olamaz. |
| `probe` | ✔ | Kimlik doğrulama; beklenen cevap `accept` listesinde. |
| `caps` | ✔ | Yetenek sözlüğü. |
| `layout` | keyboard ✔ | Yerleşim dosyası. Klasör içinde `layout.json`. |
| `matrix` | keyboard ✔ | `len` = tuş matrisi giriş sayısı (4 byte/giriş). |
| `light` | `caps.lighting` ✔ | Efekt tablosu + sınırlar. |
| `perKey` | `caps.perKey` ✔ | `leds` = adreslenebilir LED sayısı. |
| `macros` | `caps.macros` ✔ | Bellek ve mod sınırları. |
| `settings` | `caps.settings` ✔ | Ayar tanımları. |
| `battery` | `caps.battery` ✔ | `level`, `charging` bayrakları. |
| `specs` | — | Yalnız gösterim; Dashboard'daki künye. |
| `notes` | — | Tek satır uyarı; arayüzde bilgi ikonu olarak görünür. |
| `source` | ✔ | Bu sürücünün uyarlandığı depo ve lisans. Örn. `{"repo":"paulcoding810/open-aula","license":"none","file":"web/PROTOCOL.md"}`. |

Bilinmeyen üst düzey alan **reddedilir** (yazım hatası sessizce yutulmasın).

`family` alanı yoktur. Sürücüler arasında paylaşılan bir aile kavramı yok — her model
kendi `driver.js`'ini taşır.

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

Kimlik doğrulama: beklenen cevap bildirimde, çözme kodu `driver.js`'te.

```json
"probe": { "accept": ["0300000000cd"] }
```

| Alan | Açıklama |
|---|---|
| `accept` | `driver.js`'in `identify()` fonksiyonunun döndürebileceği değerler. Biri eşleşirse bu model seçilir. |
| `fallback` | `true` → kimlik okunamadıysa (cihaz cevap vermiyor) salt-okunur modda bu sürücü kullanılır. |

`accept` değeri ne olacağını `driver.js`'in `identify()`'ı belirler; biçim serbest metin.
Örnek: g5kb sürücüleri 6-byte parolayı bitişik onaltılı döner (`"0300000000cd"`), evision
sürücüleri configId anahtarını (`"v9pro"`), sonix sürücüleri ürün adı anahtarını (`"s75"`).

Kanıtlanmamış bir değeri `accept`'e yazmak yanlış modele yazma riski doğurur. Emin değilsen
`status: "exploring"` kullan: cihaz tanınır, yazma kilitli kalır.

---

## 4. `caps` — yetenek sözlüğü

Kapalı liste. Yeni anahtar eklemek kod değişikliği gerektirir (bilinçli).

| Anahtar | Anlamı | Bildirim bölümü | Açtığı arayüz |
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

`models/<id>/layout.json` — yalnız o modelin tuş haritası. Paylaşılmaz; iki modelin haritası
aynıysa iki dosyaya aynı içerik yazılır.

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

Uygulama her bildirimi yüklemeden önce doğrular. Hata → bildirim yüklenmez, günlüğe yazılır,
cihaz "tanınmıyor" görünür. **Doğrulamadan geçmemiş bildirimle cihaza yazılmaz.**

| # | Kural |
|---|---|
| 1 | `schema` desteklenen sürüm olmalı. |
| 2 | Bilinmeyen üst düzey anahtar yok. |
| 3 | `id` klasör adıyla aynı, `[a-z0-9-]+`. |
| 4 | `driver.js` aynı klasörde var ve `identify()` dışa açıyor. |
| 5 | `usb` boş değil; `vid`/`pid` 4 haneli onaltılı. |
| 6 | Her `caps: true` için ilgili bölüm var; her bölüm için `caps: true`. |
| 7 | Her `caps: true` için `driver.js`'te karşılık gelen fonksiyon var. |
| 8 | `light.modes` içinde `off` var; `hw` değerleri tekil. |
| 9 | `perKey.leds` > 0; `layout.json`'daki tüm `light` bundan küçük. |
| 10 | `matrix.len` > 0; `layout.json`'daki tüm `idx` bundan küçük. |
| 11 | `layout` yolu klasör içinde (`layout.json`); `..` yok. |
| 12 | `dpi.defaults` uzunluğu `stages`'a eşit; `min < max`, `step > 0`. |
| 13 | `probe.accept` boş değil. |
| 14 | Aynı `vid:pid:role` taşıyan iki sürücünün `accept` listeleri kesişmez; kesişiyorsa biri `fallback` işaretli olmalı. |
| 15 | `source.repo` ve `source.license` dolu. |

Doğrulayıcı `core/driver.js` içinde, ~150 satır, bağımlılıksız. Testleri gerçek bildirimleri
ve bozuk örnekleri (`tests/fixtures/bad-drivers/`) koşar.

---

## 10. Yeni model ekleme akışı

### 1. Kaynağı bul
Kalıcı bir "aile" katmanı yok; ama birçok model aynı firmware ailesinden çıkar, o yüzden
başlangıç noktası genelde aynı repolardır. Her model için o modelin sürücüsü ayrıca okunur.

| Benzer model | Bakılacak repo | Ne alınır |
|---|---|---|
| F75 türevi | open-aula `PROTOCOL.md`, AulaSofwareLinux `KB.ini` | 6-byte parola, tuş/LED sayısı, efekt tablosu |
| S75 Pro türevi | open-aula `PROTOCOL.md`, parsiya/f108-pro | ürün adı, ayar byte'ları, efekt listesi |
| HERO/WIN HE türevi | shizunavn/OpenAula `hub-aulacn-protocol.md` | UUID, tuş yerleşimi, AP/RT sınırları |
| V9/X9 türevi | shDavlatbek/aulaar `PROTOCOL.md` | configId, 128-byte blok alanları |
| … | modelin `driver.js` başındaki kaynak notu | yakın modellerin klasörleri |

### 2. Klasörü aç
`models/<id>/`. Üç dosya: `driver.json`, `driver.js`, `layout.json`.
Yakın bir modelin dosyalarını **başlangıç noktası** olarak kopyala, sonra o modelin farklarını
yaz. Kopyalamak "paylaşmak" değildir: kopyadan sonra iki model bağımsızdır.

### 3. Doğrula
```bash
node --test tests/driver.test.js tests/<id>.test.js
```
Şema, çapraz referanslar, `caps` ↔ fonksiyon eşleşmesi ve yerleşim sınırları kontrol edilir.

### 4. Simülatörde dene
`?sim=<id>` ile aç. Arayüz doğru sekmeleri açıyor mu, yanlış alan görünüyor mu?

### 5. Donanımda dene
Cihaz takılıysa Log sekmesinde kimlik cevabı `accept` değerlerinden biriyle eşleşiyor mu.
Eşleşiyorsa ışık → tuş → makro sırasıyla dene, `docs/HARDWARE-TEST.md`'yi işaretle.

### 6. Durumu yükselt
Test geçtiyse `status: "verified"`. Geçmediyse `experimental` kalır, `notes` alanına neyin
çalışmadığı yazılır.

**Ortak katmana dokunulmaz.** Yeni model ne kadar benzerse benzesin, mevcut bir sürücünün
`driver.js`'i değişmez. Bir hata birden çok modelde varsa her birinde ayrı düzeltilir —
ortak soyutlama yok, kopya kabul.

---

## 11. Örnek: F75

`models/keyboards/f75.json`:

```json
{
  "schema": 1,
  "id": "f75",
  "name": "AULA F75",
  "kind": "keyboard",
  "status": "verified",
  "usb": [
    { "vid": "258a", "pid": "010c", "role": "usb",    "usagePage": 65280 },
    { "vid": "3554", "pid": "fa09", "role": "dongle", "usagePage": 65280 }
  ],
  "probe": { "accept": ["0300000000cd", "030000000000e1"] },
  "source": { "repo": "paulcoding810/open-aula", "license": "none", "file": "web/PROTOCOL.md" },
  "caps": {
    "lighting": true, "perKey": true, "keymap": true, "fnLayer": true,
    "macros": true, "settings": true, "battery": true, "readBack": true
  },
  "layout": "layout.json",
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

- `schema` artınca eski bildirimler reddedilir; geçiş betiği yazılmaz (az sayıda dosya, elle düzeltilir).
- Profiller `driver.json`'un `id`'si + `light.modes[].id` saklar. Firmware değeri (`hw`) değişse
  bile kullanıcı profili çalışmaya devam eder. Bu yüzden `id` alanları **yeniden adlandırılmaz**;
  yeni efekt eklenirse yeni `id` verilir.
- Bildirimde `version` alanı yoktur: git geçmişi yeterli.
- Bir modelin `driver.js`'i değişince yalnız o modelin testleri koşar; diğerleri etkilenmez.

---

## 13. Bilinçli sınırlar

1. Bildirim **hangi işlemlerin sırasını** tanımlamaz. Yazma dizileri `driver.js`'te.
2. Bildirim **çerçeve biçimini** tanımlamaz. Farklı model = farklı `driver.js`.
3. Bildirim **hesaplama** yapmaz (koşul, formül, değişken). Sabit veri.
4. Bildirim **görsel düzen** tanımlamaz; yalnızca hangi kontrolün görüneceğini söyler
   (`caps` + bölümün varlığı). Yerleşim `styles/` + `app/views/` içinde bir kez.
5. Bildirim **başka bir sürücüye referans vermez**. Her klasör kendi kendine yeterli; bir
   modeli silmek diğerini bozmaz.
