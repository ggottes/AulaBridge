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
│  ├─ driver.json              bildirim (§2–§9) + kaynak listesi (§2a)
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
| `source` | ✔ | Bu sürücünün uyarlandığı **kaynaklar**. En az iki: karşılaştırma zorunlu (§8). Örn. `[{"repo":"paulcoding810/open-aula","license":"none","file":"web/PROTOCOL.md"},{"repo":"livvaa/AulaToolkit","license":"GPL-3.0","file":"src/shared/protocols/aula04.ts"}]`. Her kayıtta `repo`, `license`, `file` zorunlu. |

Bilinmeyen üst düzey alan **reddedilir** (yazım hatası sessizce yutulmasın).

`family` alanı yoktur. Sürücüler arasında paylaşılan bir aile kavramı yok — her model
kendi `driver.js`'ini taşır.

---

## 2a. Kaynak çapraz kontrolü

Her sürücü **en az iki bağımsız kaynak** üzerinden yazılır. Tek kaynağa bağlı kalmak, o
kaynaktaki bir hatayı doğru sanmak demektir. İki proje aynı cihazı bağımsız olarak çözdüyse,
ikisinin ayrıştığı yer gerçek bir soru işaretidir.

### Kaynak sayısı

| Durum | Kaynak | `status` tavanı |
|---|---|---|
| En az iki, uyuşuyor | 2+ | `verified` olabilir |
| En az iki, ayrışıyor (aşağıda çözüldü) | 2+ | `verified` olabilir |
| En az iki, ayrışıyor (çözülmedi) | 2+ | `experimental`, `divergence` alanında not |
| Yalnız bir kaynak var | 1 | `experimental` — kimse bağımsız doğrulamadı |
| Kaynak yok | 0 | `exploring` — yazma yok |

`driver.test.js` `source.length >= 2` kuralını `experimental` altında da zorunlu tutmaz ama
`verified` için tutar. Tek kaynaklı sürücü tanınır, denenebilir, ama "doğrulandı" sayılmaz.

### Ayrışma kaydı

İki kaynak aynı bayt için farklı şey söylüyorsa bu **yazılır**, gizlenmez:

```json
"divergence": [
  { "field": "light.modes[7].hw", "sources": ["open-aula:19", "AulaToolkit:20"],
    "chosen": 19, "reason": "F75 donanımında 19 çalıştı (open-aula testi)" },
  { "field": "macros.nameMax", "sources": ["open-aula:30", "AulaSofwareLinux:32"],
    "chosen": 30, "reason": "diğer iki kaynak 30 diyor" }
]
```

| Alan | Anlamı |
|---|---|
| `field` | `driver.json` içindeki yol. |
| `sources` | `repo:değer` biçiminde, her kaynağın ne dediği. |
| `chosen` | Seçilen değer. `field`'daki değerle aynı olmalı (test zorlar). |
| `reason` | Neden bu. Donanım testi > çoğunluk > yeni tarihli kaynak. |

Ayrışma kaydı olmayan ama kaynakları uyuşmayan bir sürücü `driver.test.js`'te **uyarı**
üretir. Birden çok modelde aynı ayrışma varsa bu, kaynaklardan birinin sistematik hatasıdır.

### Özellik envanteri (birlik)

Bir modelde bir özellik var, diğerinde yok — bu bilgi de kaydedilir. `caps` yalnız `true`
olanları taşır; ama "acaba bu modelde de var mıydı" sorusu her sürücüde ayrı yanıtlanır.
Kaynak karşılaştırma tablosu her sürücüde tutulur:

```json
"sourceMatrix": {
  "open-aula":     { "lighting": true, "perKey": true, "keymap": true, "macros": true, "screen": false },
  "AulaToolkit":   { "lighting": true, "perKey": true, "keymap": true, "macros": true, "screen": true }
}
```

Tabloda bir yetenek bir kaynakta `true`, diğerinde `false` ise ve `caps`'te ne yazıldığı
tartışmalıysa, `divergence`'a da girer. `caps`'teki değer, tablonun **kesişimi değil**:
kaynakların birinde `true` olması "denenmeli" demektir; `status` bunun doğrulanıp
doğrulanmadığını söyler.

### Kaynak seçimi

Aynı cihaz için kaynak ararken sıra:

1. **Donanımda test edilmiş** proje (open-aula S75 Pro, FaizAsta F75, AulaToolkit SC900 Pro, OpenAula HERO 68/84, aulaar V9)
2. **Satıcı kodundan çıkarılmış** proje (parsiya/f108-pro, Dahuum fallback)
3. **Kaptür/decompile notu** (Salamor L99, marcoslor F87)
4. **README düzeyinde iddia** (yalnız bilgi kaynağı, kanıt değil)

Bu sıralama `divergence.reason` içinde gerekçe olarak kullanılır.

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
| 15 | `source` bir dizi; her elemanında `repo`, `license`, `file` dolu. |
| 16 | `status: verified` ise `source.length >= 2`. |
| 17 | `divergence[].chosen`, `field`'ın gerçek değeriyle aynı. |
| 18 | Her `caps: true`, `sourceMatrix`'in **en az bir** kaynağında `true`. |

Doğrulayıcı `core/driver.js` içinde, ~180 satır, bağımlılıksız. Testleri gerçek bildirimleri
ve bozuk örnekleri (`tests/fixtures/bad-drivers/`) koşar.

---

## 10. Yeni özellik ekleme akışı

Bir kaynakta bu projede olmayan bir özellik bulunduğunda (ör. bir klavyede "tilt" efekti,
bir farede "motion sync"), sıra **değişmez**:

```
1. Kaynakları karşılaştır   → aynı özellik başka kaynakta var mı, aynı bayt mı
2. Şemaya ekle              → yeni bir veri alanı; doğrulama kuralıyla birlikte
3. Arayüze bağla            → JSON'dan okuyan tek, genel kontrol
4. Sürücüye yaz             → yalnız o modelin driver.js'inde kodlama
5. Bildirime yaz            → driver.json'da değer
6. Test yaz                 → fixture + "alan yoksa görünmez" testi
```

### 1. Kaynakları karşılaştır
Özellik tek kaynakta mı görünüyor? İkinci bir kaynakta aynı şeyi ara. Bulunamazsa
`sourceMatrix`'e `false` yaz, `divergence`'a "yalnız tek kaynakta" notu düş. Özellik
doğrulanmamış sayılır; `caps`'te `true` yapıp `status`'u `verified`'a çıkarma.

### 2. Şemaya ekle
Şema **kapalı bir listedir**: bilinmeyen anahtar reddedilir. Bu bilinçli, çünkü her yeni
anahtar arayüzde bir kontrol, sürücüde bir fonksiyon demek. Yeni özellik eklerken:

- Alanı **veri** olarak tanımla: sayı, liste, bayrak. Koşul veya formül değil.
- Hangi bölüme gireceğine karar ver: mevcut bölüm (ör. `settings[]`) yeterliyse yeni bölüm açma.
- Yeni bir `caps` anahtarı gerekiyorsa, o anahtarın açtığı arayüzü de yaz.
- Doğrulama kuralını aynı commit'te ekle: alan varsa bölüm, bölüm varsa `caps`.

Örnek: bir klavyede "yan ışık parlaklığı" bulundu →
`light.zones[].brightness` alanı + kural "zone varsa brightness > 0" + `caps.lighting` zaten var.

### 3. Arayüze bağla
Kontrol **tek kez** yazılır, tüm modeller kullanır. Kural: kontrol, JSON'da alan varsa görünür;
yoksa görünmez. Model adı geçen `if` yazma.

```js
// doğru: alan varsa göster
if (driver.light?.zones?.some(z => z.brightness)) renderZoneBrightness(...);

// yanlış: modele göre özel durum
if (driver.id === 's75pro') renderZoneBrightness(...);
```

### 4. Sürücüye yaz
Yalnız o modelin `driver.js`'inde. Özellik iki modelde de varsa iki sürücüye ayrı yazılır
(bkz. §14 kural 5: sürücüler birbirine referans vermez).

### 5. Bildirime yaz
`driver.json`'a değer. Etiket çevirisi gerekiyorsa `i18n` anahtarı, dosyaya gömülü metin değil.

### 6. Test yaz
İki test zorunlu:
- **Fixture testi**: sürücü, özelliği kaynaktaki baytlarla aynı kodluyor.
- **Devamsızlık testi**: alan `driver.json`'dan silinince arayüz o kontrolü göstermiyor,
  sürücü hata vermiyor. Bu, "JSON'a göre her şey çalışır" güvencesini koruyan tek test.

### Yeni özellik gerçekten yeni bölüm gerektiriyorsa
Ör. hiç olmayan bir "makro gecikme profili" alanı. Sıra aynı, ama şema sürümü artar:
`schema: 1` → `2`. Eski bildirimler reddedilir; 36 dosya elle güncellenir (geçiş betiği yok,
bkz. §13). Bunu yapmadan önce sor: bu alan mevcut bir bölüme sığar mı? Genelde sığar.

---

## 11. Yeni model ekleme akışı

### 1. Kaynağı bul — **en az iki**
Kalıcı bir "aile" katmanı yok; ama birçok model aynı firmware ailesinden çıkar, o yüzden
başlangıç noktası genelde aynı repolardır. Her model için o modelin sürücüsü ayrıca okunur,
ve **her zaman en az iki bağımsız kaynaktan**.

| Benzer model | Birinci kaynak | İkinci kaynak |
|---|---|---|
| F75 türevi | open-aula `PROTOCOL.md` | AulaSofwareLinux `proto.rs` / `KB.ini`, AulaToolkit `g5kb.ts` |
| S75 Pro türevi | open-aula `PROTOCOL.md` | FaizAsta `max.js`, AulaToolkit `aula04.ts`, parsiya/f108-pro |
| HERO/WIN HE türevi | shizunavn/OpenAula `hub-aulacn-protocol.md` | FaizAsta `he.js`, FaizAsta `max.js` |
| V9/X9 türevi | shDavlatbek/aulaar `PROTOCOL.md` | xinzore `protocol.ts` |

İki kaynak aynı fikri söylüyorsa güven artar. Ayrışıyorsa **`divergence`'a yazılır** (§2a) ve
donanım testi olmadan "doğrulandı" denmez. Tek kaynak varsa `experimental` tavanı aşılmaz.

### 2. Ayrışmaları çıkar
Kaynakları yan yana koy ve farkları listele. Her fark için:

- **Bayt düzeyi**: aynı komutun farklı payload'u, farklı checksum, farklı offset
- **Yetenek düzeyi**: bir kaynakta olan özellik diğerinde yok
- **Sınır düzeyi**: LED sayısı, matris boyu, makro belleği

Her fark `divergence[]`'a girer (§2a). Uyuşan ama hiçbirinin test etmediği bir alan varsa
`sourceMatrix`'te `false` kalır.

### 3. Klasörü aç
`models/<id>/`. Üç dosya: `driver.json`, `driver.js`, `layout.json`.
Yakın bir modelin dosyalarını **başlangıç noktası** olarak kopyala, sonra o modelin farklarını
yaz. Kopyalamak "paylaşmak" değildir: kopyadan sonra iki model bağımsızdır.

### 4. Doğrula
```bash
node --test tests/driver.test.js tests/<id>.test.js
```
Şema, çapraz referanslar, `caps` ↔ fonksiyon eşleşmesi, kaynak sayısı ve `divergence`
tutarlılığı kontrol edilir.

### 5. Simülatörde dene
`?sim=<id>` ile aç. Arayüz doğru sekmeleri açıyor mu, yanlış alan görünüyor mu?

### 6. Donanımda dene
Cihaz takılıysa Log sekmesinde kimlik cevabı `accept` değerlerinden biriyle eşleşiyor mu.
Eşleşiyorsa ışık → tuş → makro sırasıyla dene, `docs/HARDWARE-TEST.md`'yi işaretle.
Donanım testi, `divergence`'daki bir seçimi doğrularsa kaydın `reason`'ı güncellenir.

### 7. Durumu yükselt
Test geçtiyse `status: "verified"` (iki kaynak + donanım). Geçmediyse `experimental` kalır,
`notes` alanına neyin çalışmadığı yazılır.

**Ortak katmana dokunulmaz.** Yeni model ne kadar benzerse benzesin, mevcut bir sürücünün
`driver.js`'i değişmez. Bir hata birden çok modelde varsa her birinde ayrı düzeltilir —
ortak soyutlama yok, kopya kabul.

---

## 12. Örnek: F75

`models/f75/driver.json`:

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
  "source": [
    { "repo": "paulcoding810/open-aula", "license": "none", "file": "web/PROTOCOL.md" },
    { "repo": "yhuikzdtguioaert/AulaSofwareLinux", "license": "MIT", "file": "src/proto.rs" },
    { "repo": "livvaa/AulaToolkit", "license": "GPL-3.0", "file": "src/shared/protocols/g5kb.ts" }
  ],
  "sourceMatrix": {
    "open-aula":       { "lighting": true, "perKey": true, "keymap": true, "macros": true, "battery": true },
    "AulaSofwareLinux":{ "lighting": true, "perKey": true, "keymap": true, "macros": false, "battery": false },
    "AulaToolkit":     { "lighting": true, "perKey": true, "keymap": true, "macros": true, "battery": true }
  },
  "divergence": [
    { "field": "macros.maxBytes", "sources": ["open-aula:3584", "AulaSofwareLinux:4096"],
      "chosen": 3584, "reason": "open-aula F75 donanımında test etti; 3584 ile sınır korunuyor" }
  ],
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

## 13. Sürümleme

- `schema` artınca eski bildirimler reddedilir; geçiş betiği yazılmaz (az sayıda dosya, elle düzeltilir).
- `source`, `sourceMatrix` ve `divergence` bilgi alanlarıdır: uygulama davranışını değiştirmez,
  ama `driver.test.js` ve `tools/diff-sources.js` bunlara bakar.
- Profiller `driver.json`'un `id`'si + `light.modes[].id` saklar. Firmware değeri (`hw`) değişse
  bile kullanıcı profili çalışmaya devam eder. Bu yüzden `id` alanları **yeniden adlandırılmaz**;
  yeni efekt eklenirse yeni `id` verilir.
- Bildirimde `version` alanı yoktur: git geçmişi yeterli.
- Bir modelin `driver.js`'i değişince yalnız o modelin testleri koşar; diğerleri etkilenmez.

---

## 14. Bilinçli sınırlar

1. Bildirim **hangi işlemlerin sırasını** tanımlamaz. Yazma dizileri `driver.js`'te.
2. Bildirim **çerçeve biçimini** tanımlamaz. Farklı model = farklı `driver.js`.
3. Bildirim **hesaplama** yapmaz (koşul, formül, değişken). Sabit veri.
4. Bildirim **görsel düzen** tanımlamaz; yalnızca hangi kontrolün görüneceğini söyler
   (`caps` + bölümün varlığı). Yerleşim `styles/` + `app/views/` içinde bir kez.
5. Bildirim **başka bir sürücüye referans vermez**. Her klasör kendi kendine yeterli; bir
   modeli silmek diğerini bozmaz.
6. `sourceMatrix` **satıcı sürücüsünü listelemez**. Satıcının kendi binary'si bir kaynak değil,
   doğrulanacak şeydir; kaynak listesine yalnız okunabilir, lisansı belli projeler girer.
7. `divergence` **tartışmayı kapatmaz**, görünür kılar. Bir alanın iki kaynakta farklı olması
   ve seçimin gerekçelenmesi, seçimin doğru olduğu anlamına gelmez — yalnızca bilinçli
   yapıldığı anlamına gelir.
