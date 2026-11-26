# AulaBridge — Geliştirme Planı

Tarayıcıdan çalışan, kurulum gerektirmeyen AULA klavye ve fare yapılandırıcısı.
Tek sekmede model tanıma, ışık, tuş atama, makro, DPI, ekran, profil ve paket günlüğü.

> Resmî AULA ürünü değildir. AULA markası ve ürün adları sahiplerine aittir.
> Protokol bilgisi geri mühendislikle elde edilmiştir; kullanım riski kullanıcıya aittir.

Protokol detayları ayrı belgede: [docs/PROTOCOL-NOTES.md](docs/PROTOCOL-NOTES.md)

---

## İçindekiler

1. [Ürün tanımı](#1-ürün-tanımı)
2. [Kararlar](#2-kararlar)
3. [Destek matrisi](#3-destek-matrisi)
4. [Mimari](#4-mimari)
5. [Modül sözleşmeleri](#5-modül-sözleşmeleri)
6. [Manifest kataloğu](#6-manifest-kataloğu) → [docs/SCHEMA.md](docs/SCHEMA.md)
7. [Arayüz](#7-arayüz)
8. [Tasarım sistemi](#8-tasarım-sistemi)
9. [Güvenlik modeli](#9-güvenlik-modeli)
10. [i18n](#10-i18n)
11. [Test](#11-test)
12. [Fazlar](#12-fazlar)
13. [Riskler](#13-riskler)
14. [Açık sorular](#14-açık-sorular)
15. [Çalışma kuralları](#15-çalışma-kuralları)

---

## 1. Ürün tanımı

### Ne yapar
AULA'nın model başına dağıttığı onlarca Windows sürücüsünün yerine tek bir sayfa.
Cihaz takılır, tarayıcıda "Connect" denir, uygulama modeli **kendisi tanır** ve o modele
ait sekmeleri açar. Ayar değişikliği cihazın kendi hafızasına yazılır; bilgisayar değişse de kalır.

### Kimin için
- AULA klavyesi/faresi olup resmî sürücüyü kurmak istemeyen kullanıcı
- Linux/macOS'ta yaşayan, resmî Windows sürücüsü olmayan kullanıcı
- Birden çok AULA cihazı olup her biri için ayrı program istemeyen kullanıcı
- Protokolleri inceleyen geliştirici (Log sekmesi bir araştırma aracı)

### Farkı
| | Resmî sürücü | Bu proje |
|---|---|---|
| Kurulum | Her model için ayrı exe | Yok |
| Platform | Windows | Chromium masaüstü (Win/mac/Linux) |
| Model tanıma | Kendi modelini bilir | VID:PID + kimlik sorgusu (JSON'daki `probe`) |
| Yeni model ekleme | Yeni exe | Yeni `.json` dosyası — kod değişmez |
| Günlük/paket görünürlüğü | Yok | Var |
| Desteklenmeyen model | Yok | "Tanındı, deneysel" etiketiyle açılır |

### Model bilgisi nerede durur
Her cihazın özellikleri, yetenekleri, efekt tablosu, tuş yerleşimi ve sınırları tek bir
**JSON manifest** dosyasında. Uygulama cihazı tanıdıktan sonra arayüzü o dosyaya göre kurar.
Model başına arayüz kodu yoktur — bu projenin ayırt edici tasarım kararı.
Şema: [docs/SCHEMA.md](docs/SCHEMA.md) · Protokol baytları: [docs/PROTOCOL-NOTES.md](docs/PROTOCOL-NOTES.md)

### Kapsam dışı (bilinçli)
- **Firmware güncelleme.** Hiçbir kaynakta geri alınabilir güvenli yol yok.
- **Arka plan servisi / tepsi uygulaması.** WebHID sekme kapanınca biter; bu bir sınır, kusur değil.
- **Otomatik oyun profili değiştirme.** Servis gerektirir.
- **Ses/ritim senkronu** ilk sürümde yok (Faz 5'te, destekleyen modellerde).
- **Kulaklık/hoparlör.** HID yapılandırma protokolü yok.

---

## 2. Kararlar

| Konu | Karar | Gerekçe |
|---|---|---|
| Platform | Statik site + WebHID | Kurulum yok, GitHub Pages'te yayınlanır |
| Build | **Yok** — düz ES modülleri, JSDoc tipleri | `npm install` gerektirmez, kaynak = çalışan kod |
| Test | `node --test`, ek bağımlılık yok | Protokol modülleri saf olduğu için doğrudan koşar |
| Dil | İngilizce + i18n katmanı; `en`, `tr` | Sonradan dil eklemek tek JSON |
| Lisans | GPL-3.0-or-later | Kaynak protokol kodunun çoğu GPL; herkese açık kalır |
| Ad | AulaBridge | Klasör adıyla aynı |
| Tarayıcı | Chrome/Edge masaüstü, Chromium ≥ 100 | WebHID başka yerde yok |

---

## 3. Destek matrisi

Durum: **V** donanımda doğrulanmış · **C** sürücü kodundan çıkarılmış · **?** protokol eksik.
Ayrıntı ve bayt düzenleri: [PROTOCOL-NOTES.md](docs/PROTOCOL-NOTES.md)

### Klavyeler — 24 model, 6 protokol

| Model | Protokol | Kablo / dongle | Durum | Öne çıkan |
|---|---|---|---|---|
| F75 · F75 Pro | g5kb | 258A:010C / 3554:FA09 | **V** | 18 efekt, per-key, makro |
| F87 · F87 Pro | g5kb | aynı | C | 87 tuş |
| F99 · F99 Pro | g5kb | aynı | C | 96% düzen |
| F108 · H108 | g5kb | aynı | C | tam boy |
| F87 Pro (sonix) | aa | 38A6:271x / 0C45:FEFC | C | 7×7 LED matris |
| S75 Pro | aula04 | 0C45:800A / 0C45:FEFE, 05AC:024F | **V** | 19 efekt, TFT 135×240, yan ışık |
| L99 | aula04 | 0C45:800A / 05AC:024F | C | 84 tuş, TFT 320×480 |
| F108 Pro | aula04 | 0C45:800A / 05AC:024F | C | 20 efekt, 3 ışık bölgesi |
| F75 Max | aula04 | 0C45:800A / 05AC:024F | C | **yaz-only**, 20 efekt |
| F98 Pro · F98X · S99 Pro | aula04? | 0C45:800A | ? | salt-okunur keşif |
| S98 Pro | ? | 0C45:800A | ? | önce snif |
| F2088 | i8 | ? | C | yalnız ışık |
| F75 HE | hfd | 38A6:2729 / 0C45:FEFE | C | manyetik, per-key |
| HERO 68 · 84 · 75 · 99 · 87 HE | bytetech | 372E:103E / 372E:1000, 106C | HERO68/84 **V** | AP/RT, SOCD, DKS |
| HERO 68 HE ULTRA · PRO | oem1ca5 | 1CA5:0407 / 0409 | C | report ID 3 |
| WIN 68 HE MAX | xsyd | 1CA2:1901, 1C4F:EE88 | **V** | µm birimi, 4 slot |
| AG60 · AG75 · NEOS 75 · MINI 60HE · WIN60HE · HERO75HE | bytetech? | 372E / hub | ? | kimlik sorgusuyla teyit |

### Fareler — 12 model, 5 protokol

| Model | Protokol | USB ID | Durum |
|---|---|---|---|
| SC620 | mouse55 | A8A4:2255 / A8A5:2255 | buton haritası **V** |
| SC900 · SC900 Pro (+ 3 varyant) | incott | 093A / 248A / 379A | **V** |
| SC390 | yxt | 32C2:0024 | C |
| V9 / V9+ / Pro / Pro+ / Ultra / Max, X9 ailesi + 8K dongle | evision | 320F:22xx | C |
| SC800 Max · L9 · V3 Ultra B · X5 Ultra | hubmouse | 38A6:28FC/28FE/2F09 | C (kısmi) |
| SC580 · 580 X · S12 Pro · KUNLUN X5 | ? | ? | tanınır, sekme kapalı |

### Ortak yetenek bayrakları

Her model kaydı bu bayrakları taşır; arayüz sekmelerini buna göre açar.

```
lighting   perKey   keymap   fnLayer   macros   screen   clock
dpi        buttons  polling  sensor    battery  clock
readBack   reset    matrix    hall      analog
```

- `readBack: false` → yazma "gönderildi" olarak raporlanır, "doğrulandı" denmez (F75 Max).
- `hall` → "HE / Analog" sekmesi açılır.
- `experimental: true` → başlıkta sarı şerit.

---

## 4. Mimari

### Katmanlar

```
┌──────────────────────────────────────────────────────────┐
│ views/          ekranlar; yalnız store okur, session çağırır
├──────────────────────────────────────────────────────────┤
│ components/     keycap, slider, colorPicker, effectCard, modal, toast
├──────────────────────────────────────────────────────────┤
│ store.js        tek durum nesnesi + subscribe + persist (localStorage)
├──────────────────────────────────────────────────────────┤
│ session.js      bağlı cihaz oturumu: kuyruk, gecikme, retry, durum, olaylar
├──────────────────────────────────────────────────────────┤
│ registry.js     MODEL listesi; VID:PID → adaylar; kimlik çözümü
│ identity.js     model kapısı (parola / UUID / ürün adı)
├──────────────────────────────────────────────────────────┤
│ protocols/*.js  SAF kodlayıcı/çözücü: Uint8Array in/out, I/O yok
├──────────────────────────────────────────────────────────┤
│ transport.js    WebHID sarmalayıcı: request, open, sendFeature, receive, output
└──────────────────────────────────────────────────────────┘
                        │
                   WebHID API
                        │
                     cihaz
```

### Veri akışı

```
kullanıcı tıklaması
  → view: session.set('lighting', {mode: 7, brightness: 3})
  → session: kuyruğa al, model.protocol için codec'i çağır
  → protocol.lightPayload(...) → Uint8Array[] (saf, test edilebilir)
  → transport: 35 ms beklemeli feature yazımı
  → (readBack) protocol.lightParse(...) ile geri oku
  → store: {lighting: {...}, verified: true} + log kaydı
  → view yeniden çizilir
```

### Dosya düzeni

```
AulaBridge/
├─ index.html
├─ PLAN.md
├─ README.md
├─ LICENSE
├─ docs/
│  ├─ PROTOCOL-NOTES.md
│  ├─ SUPPORTED.md            model × yetenek tablosu (otomatik üretilir)
│  └─ HARDWARE-TEST.md        elle doğrulama kontrol listesi
├─ styles/
│  ├─ tokens.css              renk, boşluk, tipografi, gölge değişkenleri
│  ├─ base.css                reset, tipografi, odak
│  ├─ layout.css
│  └─ components.css
├─ app/
│  ├─ main.js                 boot, hash-router, klavye kısayolları
│  ├─ store.js
│  ├─ session.js
│  ├─ i18n/
│  │  ├─ index.js             t(), setLocale(), dir
│  │  ├─ en.json
│  │  └─ tr.json
│  ├─ views/
│  │  ├─ dashboard.js
│  │  ├─ lighting.js
│  │  ├─ keys.js
│  │  ├─ macros.js
│  │  ├─ mouse.js
│  │  ├─ screen.js
│  │  ├─ hall.js
│  │  ├─ profiles.js
│  │  ├─ log.js
│  │  └─ settings.js
│  └─ components/
│     ├─ keycap.js  colorPicker.js  slider.js  effectCard.js
│     ├─ modal.js   toast.js        deviceStrip.js  keyPreview.js
├─ core/
│  ├─ transport.js
│  ├─ manifest.js             JSON şema doğrulama + çapraz referans kontrolü
│  ├─ registry.js             manifest kataloğu → adaylar → probe → model
│  ├─ log.js
│  ├─ persist.js              localStorage, profil içe/dışa aktarma
│  └─ sim.js                  simülatör: manifest'i okur, sahte cevap üretir
├─ protocols/
│  ├─ util.js
│  ├─ g5kb.js     aula04.js   aa.js     mouse55.js   incott.js
│  ├─ yxt.js      evision.js  hfd.js    bytetech.js  xsyd.js
│  ├─ oem1ca5.js  hubmouse.js i8.js
├─ models/
│  ├─ index.json              katalog: id → dosya, aile, durum
│  ├─ keyboards/
│  │  ├─ f75.json  f75pro.json  f87.json  f87pro.json  f99.json  f99pro.json
│  │  ├─ f108.json h108.json    f87pro-sonix.json
│  │  ├─ s75pro.json l99.json   f108pro.json f75max.json f98pro.json s99pro.json s98pro.json f2088.json
│  │  ├─ f75he.json hero68he.json hero84he.json hero75he.json hero99he.json
│  │  ├─ win68hemax.json hero68ultra.json mini60he.json win60he.json
│  │  ├─ ag60.json ag75.json neos75.json
│  └─ mice/
│     ├─ sc620.json sc900.json sc390.json
│     ├─ v9.json x9.json sc800max.json sc580.json kunlunx5.json
├─ layouts/                   tuş yerleşimi; modeller paylaşır
│  ├─ f75.json f87.json f99.json f108.json
│  ├─ s75pro.json l99.json f75max.json
│  └─ hero68.json win68hemax.json …
├─ assets/
│  ├─ devices/                model fotoğrafı (koyu zemin, 3/4 açı)
│  └─ icons/                  efekt ve işlev ikonları
├─ tests/
│  ├─ g5kb.test.js  aula04.test.js  aa.test.js  mouse55.test.js
│  ├─ incott.test.js yxt.test.js evision.test.js hfd.test.js
│  ├─ bytetech.test.js xsyd.test.js
│  ├─ manifest.test.js        şema + çapraz referans + tüm manifestler
│  ├─ registry.test.js i18n.test.js
│  └─ fixtures/               gerçek paket baytları + bozuk manifest örnekleri
└─ .github/workflows/pages.yml
```

---

## 5. Modül sözleşmeleri

### `protocols/*.js` — saf, I/O yok

Her modül aynı şekli döndürür:

```js
export const REPORT_ID = 6;
export const CAPS = { lighting: true, perKey: true, keymap: true, macros: true };

export const identity     = () => Uint8Array[];         // kimlik okuma paketleri
export const parseIdentity= (bytes) => 'f75';            // 6-byte → model id | null

export const light        = { write(state), read() };
export const perKey       = { write(colors), read() };
export const keymap       = { write(layer, actions), read(layer) };
export const macros       = { write(list), read() };
export const settings     = { write(blob), read() };
export const battery      = { query(), parse(bytes) };
export const screen       = { upload(frames, slot) };    // opsiyonel
export const hall         = { readAP(), writeAP(list), readRT(), writeRT(list) };
```

- Fonksiyonlar **veri üretir**, göndermez. `session` gönderir.
- Sıralama/gecikme bilgisi modülden gelir: `write()` bir dizi döner, her adım bir paket ya da
  `{wait: 35}` belirteci olabilir.
- Tüm okuma fonksiyonları **eksik/bozuk veride `null`** döner; tahmin etmez.

### `transport.js`

```js
navigator.hid.requestDevice({ filters })     // kullanıcı seçer
open(device)                                  // exclusive; başka program tutuyorsa anlamlı hata
sendFeature(device, reportId, bytes)
receiveFeature(device, reportId, length)
sendOutput(device, reportId, bytes)
onInputReport(device, cb)
watchConnect / watchDisconnect                 // hid.connect / hid.disconnect
```

- Her çağrı zaman aşımına sahip (varsayılan 2 s, ekran için 5 s).
- Hata mesajları kullanıcı diliyle: "Cihaz başka bir program tarafından kullanılıyor.
  Resmî AULA sürücüsünü kapatın."

### `session.js`

```js
connect()                 // istek → aç → kimlik → model → yetenekler → ilk okuma
disconnect()
call(feature, action, args)   // kuyruğa alır, gecikme uygular, log yazar
state                     // { model, link, battery, charging, firmware, verified }
on('change' | 'log' | 'error', cb)
```

- **Tek kuyruk**: aynı cihaza eşzamanlı iki yazma gitmez.
- Retry: giden paket 3 kez, arada 70 ms. Kimlik okuma başarısızsa bağlantı kurulmaz.
- Bağlantı kopunca kuyruk temizlenir, store "bağlı değil"e döner; yarım yazma bırakılmaz.

### `store.js`

```js
{
  connected: false, model: null, link: null,
  lighting: {...}, keymap: { base: {}, fn: {} }, macros: [],
  mouse: { dpi: [], buttons: {}, settings: {} }, hall: {...},
  profiles: [{id, name, modelId, data}], activeProfileId,
  ui: { tab, theme, locale, reducedMotion },
  log: []
}
```

- `subscribe(fn)` → değişimde tam durum.
- Kalıcı olan: profiller, ui tercihleri. Cihaz durumu kalıcı değil.
- Profil içe/dışa aktarma: `{version: 1, model, data}` JSON.

---

## 6. Manifest kataloğu

Model bilgisinin tamamı JSON'da. Şema ve kurallar ayrı belgede: **[docs/SCHEMA.md](docs/SCHEMA.md)**

### Neyin JSON, neyin kod olduğu

| JSON | Kod |
|---|---|
| Model listesi, VID:PID, kimlik baytları | Çerçeve, checksum, komut sırası |
| Yetenekler, efekt tablosu, tuş yerleşimi | Parçalama/birleştirme |
| Ayar seçenekleri, sınırlar, pil bayrakları | Oku-değiştir-yaz mantığı |
| DPI kademeleri, yoklama listesi, buton slotları | Arayüz düzeni (tek kez, tüm modeller için) |

**Var olan aileye model eklemek = yalnızca JSON.** Yeni aile = 1 protokol dosyası + JSON.

### Katalog

`models/index.json` — hangi manifest var, hangi ailede:

```json
{
  "schema": 1,
  "models": [
    { "id": "f75", "file": "keyboards/f75.json", "family": "g5kb", "status": "verified" },
    { "id": "s75pro", "file": "keyboards/s75pro.json", "family": "aula04", "status": "verified" }
  ]
}
```

Uygulama açılışta yalnızca kataloğu indirir. Cihaz bağlanınca o VID:PID'ye uyan manifestleri
tembel yükler — 36 dosyayı baştan indirmez.

### Çözüm sırası (`registry.js`)

```
1. Cihazın tüm HID koleksiyonlarını topla (aynı VID:PID altında birden çok olabilir)
2. Katalogdan bu VID:PID'ye uyan manifestleri yükle ve doğrula (§SCHEMA 9)
3. Aday yoksa            → "tanınmıyor" kartı, hiçbir yazma yok
4. Aday tekse            → probe çalıştır; strong + eşleşti → bağlan
5. Aday birden çoksa     → her adayın probe'unu sırayla dene (aile identify fonksiyonu):
      g5kb     → 6-byte parola          evision  → configId
      bytetech → 0x82 UUID              hfd/xsyd → cihazın bildirdiği VID:PID
      aula04   → ürün adı               incott   → kimlik baytları
6. Kimlik çözüldü        → manifest yüklendi, sekmeler caps'ten üretilir
7. Çözülemedi            → fallback işaretli manifest varsa SALT OKUNABİLİR aç:
                            sekmeler görünür, yazma düğmeleri kilitli
                            Yoksa "tanınmıyor"
```

Kimlik okuması **her bağlantıda** yapılır, önbelleğe alınmaz. USB portu değişebilir.

### Manifest'in sürdüğü şeyler

Manifest değişince arayüz değişir; view koduna dokunulmaz:

| Manifest | Arayüzde karşılığı |
|---|---|
| `caps.lighting` | Lighting sekmesi açılır |
| `caps.perKey` | Per-key fırçası görünür |
| `light.modes[].color` | Renk seçici gösterilir/gizlenir |
| `light.modes[].dir` | Yön kontrolü |
| `caps.fnLayer` | Keys'te Fn düğmesi |
| `caps.macros` | Macros sekmesi + makro bellek göstergesi |
| `settings[]` | Ayar listesi; tip → kontrol |
| `caps.dpi` + `mouse.dpi` | Mouse → DPI bloğu, kademe sayısı |
| `mouse.polling.wired` | Kabloda seçilebilen Hz listesi |
| `mouse.sensor.lod: []` | LOD kontrolü gizli |
| `screen.width/height` | Kırpma alanı oranı |
| `caps.clock` | "Saatı senkronize et" düğmesi |
| `caps.readBack: false` | "Doğrulandı" rozeti yerine "gönderildi" |
| `status: experimental` | Sarı şerit |
| `status: exploring` | Tüm yazma kilitli |

### Paylaşılan VID:PID'ler (dikkat)

| VID:PID | Ait olduğu modeller |
|---|---|
| `0C45:800A` | S75 Pro, L99, F108 Pro, F75 Max, F98 Pro, S98 Pro, S99 Pro, F2088(?) |
| `0C45:FEFE` | S75 Pro dongle, F87 Pro dongle, F75 HE dongle |
| `05AC:024F` | S75 Pro dongle, F108 Pro dongle, F75 Max dongle |
| `258A:010C` | F75, F75 Pro, F87, F87 Pro, F99, F99 Pro, F108, H108 |
| `38A6:271x` | F87 Pro sonix varyantları |
| `38A6:2729` | F75 HE (kablo) |

Aynı VID:PID altındaki her manifest **farklı `accept` değeri** taşımalı; `manifest.test.js`
bunu zorunlu kılar (bkz. SCHEMA §9 kural 14).

---

## 7. Arayüz

### Yerleşim

```
┌──────────────────────────────────────────────────────────────┐
│ ◆ AulaBridge          [F75 · USB · 🔋82%]        [Log] [⚙]  │  üst çubuk
├──────┬───────────────────────────────────────────────────────┤
│      │  Dashboard  Lighting  Keys  Macros  Mouse  Screen …   │  sekmeler (modele göre)
│ cihaz│ ─────────────────────────────────────────────────────  │
│şeridi│                                                        │
│      │                                                        │
│ [F75]│              içerik                                   │
│ [SC9]│                                                        │
│      │                                                        │
│ [+]  │                                                        │
└──────┴───────────────────────────────────────────────────────┘
```

- Sol şerit yalnızca **birden çok cihaz bağlıyken** görünür; tek cihazda gizlenir.
- Sekmeler `caps` ve `status`'a göre üretilir; olmayan yetenek için sekme çıkmaz.
- `experimental` modellerde üst çubukta ince sarı şerit: "Experimental — settings may not apply as expected."
- Dar ekran (<900 px): şerit üstte yatay, sekmeler yatay kaydırmalı.

### Sekmeler

#### Dashboard
| Blok | İçerik |
|---|---|
| Cihaz kartı | model adı, bağlantı türü (USB / 2.4 GHz), firmware, durum rozeti (Verified / Experimental) |
| Pil | yüzde, şarj ikonu, "tahmini" notu (pil yüzdesi firmware'den geliyorsa) |
| Hızlı ışık | 6 favori efekt + parlaklık kaydırıcısı |
| Hızlı profil | kayıtlı profiller, "Uygula" |
| Bağlantı sorunu | cihaz yoksa: neden (tarayıcı desteklemiyor / sürücü açık / Bluetooth modu) |

#### Lighting
| Blok | İçerik |
|---|---|
| Efekt galerisi | her efekt için animasyonlu önizleme kartı; desteklenmeyen alanlar gri |
| Parametreler | parlaklık, hız, yön, renk, "colorful" anahtarı — seçilen efektin `caps`'ine göre |
| Per-key | klavye görseli üzerinde fırça; hazır paletler (gökkuşağı, sıcak, buz, tek renk, sıfırla) |
| Canlı önizleme | S75 Pro USB'de gerçek LED durumu okunur; diğerlerinde "gönderilen" |
| Yan ışık | yalnız S75 Pro; mode 20–24, gerçek davranışa göre etiketli |

#### Keys
| Blok | İçerik |
|---|---|
| Katman | Base / Fn düğmesi (caps'e göre) |
| Klavye görseli | tuşa tıkla → atama paneli; atanmış tuşlar rozetli |
| Atama paneli | Kategori: Tuş · Kısayol · Medya · Fare · Makro · Devre Dışı |
| Çakışma | aynı kombinasyon iki tuşa atanırsa uyarı |
| Uygula/Sıfırla | katman başına |

#### Macros
| Blok | İçerik |
|---|---|
| Liste | makrolar, ad, adım sayısı, bağlı tuş |
| Editör | adım listesi: tuş bas/bırak, gecikme, fare, tekerlek; sürükle-bırak sıralama |
| Mod | bir kez / N kez / basılı tut / herhangi tuşa kadar (protokole göre) |
| Kapasite | cihazın makro belleği göstergesi; doluysa yazma reddedilir |

#### Mouse
| Blok | İçerik |
|---|---|
| DPI | 6 kademe, her biri için değer + renk + etkin; X/Y ayrı (destekleyende) |
| Yoklama | kablo ve 2.4G ayrı; kabloda üst sınır protokole göre |
| Sensör | LOD, angle snap, motion sync, ripple, rotasyon |
| Güç | uyku, debounce |
| Butonlar | fare görseli üzerinde 11 slot; atama editörü Keys ile ortak |
| Pil | yüzde + şarj durumu (destekleyende) |

#### Screen
| Blok | İçerik |
|---|---|
| Yükleme | GIF/PNG/JPEG sürükle-bırak veya seç |
| Kırpma | cihaz çözünürlüğüne göre (135×240, 320×480); kare kare önizleme |
| Ayarlar | kare gecikmesi, kare sayısı sınırı, döndürme |
| Gönderim | ilerleme çubuğu, iptal, hata halinde "kısmi yazıldı" uyarısı |
| Saat | bilgisayar saatini cihaza yaz (S75 Pro, L99, F108 Pro) |

#### HE / Analog
| Blok | İçerik |
|---|---|
| Aktüasyon | tuş başına veya toplu, 0.10–3.40 mm (XS'te 5 µm adım) |
| Rapid Trigger | aç/kapa, press/release noktaları, deadzone |
| Canlı test | basılan tuşların anlık derinliği (Hall telemetrisi) |
| Gelişmiş | SOCD, DKS, Mod Tap — yalnız bytetech/xsyd'de |

#### Profiles
Yerel profiller: oluştur, kopyala, yeniden adlandır, sil, uygula, JSON içe/dışa aktar.
Profil `modelId`'ye bağlı; başka modele yüklenmeye çalışılırsa reddedilir.

#### Log
Giden/gelen her paket: zaman, yön, model, hex dökümü (ilk 32 byte + tamamını aç).
Filtre: yalnız hatalar, arama, temizle, dosyaya indir. Hata ayıklamanın ana aracı.

#### Settings
Dil, tema (koyu/açık/sistem), azaltılmış hareket, otomatik bağlanma, udev yardımı (Linux),
sürüm + bağlantılar, "tanılama paketi indir" (log + ortam bilgisi, cihaz verisi yok).

### Durum ekranları

| Durum | Gösterim |
|---|---|
| Tarayıcı desteklemiyor | Tam ekran açıklama + Chrome/Edge indirme bağlantısı |
| Cihaz yok | "Connect" düğmesi + model listesi (ne desteklendiği) |
| Tanınmayan cihaz | kart + "Bu model henüz desteklenmiyor" + talep formu bağlantısı |
| Model doğrulanamadı | salt-okunur şerit + yazma onayı |
| Cihaz başka programda | adım adım çözüm (sürücüyü kapat, sekmeyi yenile) |
| Yazma başarısız | toast + Log'a atlama bağlantısı + son paket |

---

## 8. Tasarım sistemi

### Yön
Koyu, sakin, alet gibi. Cihazın kendisi ekranın kahramanı — büyük görsel, geniş boşluk.
RGB renkleri yalnızca **efekt önizlemelerinde** doygun; arayüzün kendi vurgusu tek renk.
Sıfır süs, sıfır gradyan arka plan, sıfır gölge yığını.

### Token'lar

```css
--bg:        #0b0c0e;   --surface:  #131519;   --surface-2: #1a1d22;
--text:      #e8eaed;   --text-dim: #9aa0a8;   --line: #262a31;
--accent:    #e8a33d;   /* amber — RGB halkasıyla uyumlu */
--ok:        #4cb782;   --warn: #e8a33d;       --err: #e0574f;
--radius:    10px;      --radius-sm: 6px;
--space:     4px;  /* 4'ün katları: 4 8 12 16 24 32 48 64 */
--font:      ui-sans-serif, system-ui, 'Segoe UI', sans-serif;
--mono:      ui-monospace, 'Cascadia Mono', monospace;   /* sayısal değerler */
```

Aydınlık tema aynı değişken adlarıyla; `data-theme="light"` altında yeniden tanımlanır.
`prefers-color-scheme` ile otomatik.

### Tipografi
| Rol | Boyut / ağırlık |
|---|---|
| Ekran başlığı | 20 px / 600 |
| Bölüm başlığı | 13 px / 600, `--text-dim`, harf aralığı +0.02em |
| Gövde | 14 px / 400 |
| Etiket | 12 px / 500 |
| Sayısal değer | 14 px mono / 500 |
| Küçük not | 12 px / 400, `--text-dim` |

### Bileşen davranışları
- **Slider**: klavye ile ±1, Shift ile ±5; değer her zaman mono olarak yanında.
- **Renk seçici**: HSL tekeri + hex girişi + 8 hazır renk; klavye ile gezinilebilir.
- **Efekt kartı**: hover'da animasyon başlar, `prefers-reduced-motion`'da durur.
- **Toast**: 4 s, hata 8 s ve kapatma düğmeli.
- **Modal**: odak tuzağı, Esc kapatır, açılırken odağı ilk alana verir.

### Erişilebilirlik (pazarlıksız)
- Tüm etkileşim klavye ile; `:focus-visible` her yerde görünür.
- Kontrast: gövde metni ≥ 4.5:1, büyük metin ≥ 3:1.
- İkonlar `aria-label` taşır; renk tek başına anlam taşımaz.
- `prefers-reduced-motion` → tüm animasyonlar kapanır.
- Ekran okuyucu için sekmeler `role="tablist"`, paneller `role="tabpanel"`.

---

## 9. Güvenlik modeli

Yanlış yazma cihazı bozabilir (genelde sürücüyle düzeltilebilir, garanti yok). Bu yüzden:

1. **Kimlik olmadan yazma yok.** Model doğrulanmazsa tüm yazma düğmeleri kilitli.
2. **Oku-değiştir-yaz.** Bilinmeyen byte'lar korunur; asla sıfırlanmaz.
3. **İmza kontrolü.** `AA55` / `5A A5` / checksum yoksa yazma reddedilir — "bozuk okuma" sayılır.
4. **Sıfırlama ayrı onay.** Fabrika sıfırlama, `confirm` ile iki adım ister.
5. **Yazma sonrası doğrulama** (`readBack: true` ise). Doğrulama başarısızsa kullanıcı uyarılır,
   eski değer tekrar yazılmaz (kullanıcı karar verir).
6. **`SAFE` komut listesi.** Her protokol yalnızca beyaz listedeki komutları üretebilir;
   reset/firmware/test komutları modülde hiç bulunmaz.
7. **Tek sahip.** Bağlantı açıkken başka program erişemez; hata mesajı bunu açıklar.

---

## 10. i18n

- `t('lighting.effect.static')`, eksik anahtar → anahtar adı görünür, kırılmaz.
- `en.json` referans; testte her dil dosyası aynı anahtar kümesine sahip mi kontrol edilir.
- Sayı/tarih biçimi `Intl` ile, elle biçimlendirme yok.
- `dir` altyapıda var; RTL dil gelince `<html dir>` değişir.
- Başlangıç: `en`, `tr`. Yeni dil = `app/i18n/xx.json` + `index.js`'e bir satır.

Anahtar grupları: `app.*`, `nav.*`, `dash.*`, `lighting.*`, `keys.*`, `macros.*`,
`mouse.*`, `screen.*`, `hall.*`, `profiles.*`, `log.*`, `settings.*`, `error.*`, `device.*`.

---

## 11. Test

### Birim testleri (`node --test`)
| Dosya | Ne doğrular |
|---|---|
| `g5kb.test.js` | 520-byte başlık, parça bölme, tuş kodlama, LED bloğu read-modify-write, makro olay kodlama |
| `aula04.test.js` | 64-byte komut, `AA55` etiketi, 9 paketlik tuş/per-key bloğu, makro `count` formülü, ekran başlığı ve RGB565, 32-byte dongle checksum |
| `aa.test.js` | `AA cmd len …` başlığı, yankı bekleme sırası, offset'li blok okuma |
| `mouse55.test.js` | `55 cmd flag chk …` checksum, parametre bloğu 46 byte, buton girişi kodlama |
| `incott.test.js` | 9-byte çerçeve, alt-komut yankısı, yoklama tablosu, LOD eşlemesi |
| `yxt.test.js` | nibble maskesi, bölüm 0x40 byte 14 bit alanları |
| `evision.test.js` | report ID 4, blok transferi, configId → model, 128-byte blok offset'leri |
| `hfd.test.js` | 5 komutluk beyaz liste (başka komut üretilemez), kimlik doğrulama |
| `bytetech.test.js` | 63-byte payload başlığı, `cmd\|0x80`, 56-byte veri, birim dönüşümü (mm → 0.01 mm) |
| `xsyd.test.js` | `0x5C` çerçevesi, `CK` formülü, **iki istisna** (`OP<0x80` → CK 0, `OP=0x30` → `frame[63]`), µm birimi |
| **`manifest.test.js`** | **Depodaki tüm manifestler**: 14 kural (SCHEMA §9), `index.json` ile çift yönlü eşleşme, layout'ta `idx`/`light` sınırları, paylaşılan VID:PID'de tekil `accept`; bozuk örnekler reddediliyor |
| `registry.test.js` | Paylaşılan VID:PID'lerde aday listesi doğru; probe sırası; `fallback` yalnız kimlik okunamayınca; bilinmeyen ID'de boş |
| `i18n.test.js` | Tüm diller `en.json` ile aynı anahtar kümesinde |
| `tabs.test.js` | `caps` → beklenen sekme seti; eksik bölüm → sekme yok; `exploring` → tüm yazma kilitli |

Fixture'lar kaynaklardaki gerçek kaptür baytlarından gelir; "benim kodum kendini doğrulasın"
değil, "satıcının paketiyle aynı baytı üretsin" test edilir.

### Elle donanım testi
`docs/HARDWARE-TEST.md` — her aile için:
```
[ ] Cihaz tanındı, doğru model adı göründü
[ ] Işık efekti değişti ve cihazda görüldü
[ ] Parlaklık 1 kademe artır → cihazda arttı → geri al
[ ] Tuş atama: bir tuşa farklı tuş ata → test et → geri al
[ ] Makro yaz → tetikle → sil
[ ] Ayar (debounce/uyku) yaz → yeniden bağla → kalıcı mı
[ ] Log'da giden paket satıcı sürücününkiyle aynı
```

### CI
`node --test` + `node --check` (sözdizimi) + GitHub Pages yayını. Build adımı yok.

---

## 12. Fazlar

Her fazın sonunda **çalışan, yayınlanabilir site** var. Süre tahminleri tek geliştirici içindir.

### Faz 0 — İskelet + manifest motoru
**Süre:** 3–4 gün · **Bağımlılık:** yok

| # | İş | Kabul ölçütü |
|---|---|---|
| 0.1 | `index.html`, `styles/`, tema token'ları, açık/koyu | Tema değişiyor, `prefers-color-scheme` çalışıyor |
| 0.2 | `store.js` + subscribe + localStorage | Yenilemede ui tercihleri korunuyor |
| 0.3 | Hash router, sekme iskeletleri | Sekmeler gezilebiliyor |
| 0.4 | `i18n` + `en.json` + `tr.json` | Dil değişiyor, test geçiyor |
| 0.5 | **`manifest.js`**: şema + 14 çapraz kural | Geçerli manifestler yükleniyor, `tests/fixtures/bad-manifests/` hepsi reddediliyor |
| 0.6 | **`models/index.json`** katalog + tembel yükleme | Katalog → manifest → doğrula zinciri çalışıyor |
| 0.7 | **Sekmelerin manifest'ten üretilmesi** | Aynı sitede iki farklı manifest → iki farklı sekme seti, view koduna dokunmadan |
| 0.8 | `transport.js` — istek, aç, feature oku/yaz, zaman aşımı | Simülatörle uçtan uca |
| 0.9 | `registry.js` — aday listesi, probe sırası, salt-okunur yedek | Paylaşılan VID:PID'de doğru manifest |
| 0.10 | `sim.js` — manifest'i okuyup sahte cihaz üretir | Donanımsız her manifest gezilebiliyor, demo hiç yazmıyor |
| 0.11 | `log.js` + Log sekmesi | Her çağrı hex olarak görünüyor, indirilebiliyor |
| 0.12 | Dashboard + boş durumlar | "Connect" akışı çalışıyor |
| 0.13 | `tests/` koşucusu + CI + Pages | `node --test` yeşil, site yayında |

**Faz 0 çıkışı:** İki örnek manifest (`f75.json`, `sc620.json`) elle yazılır. Tarayıcıda `?sim=f75`
açıldığında Lighting/Keys/Macros sekmeleri, `?sim=sc620` açıldığında Mouse sekmesi görünür —
arayüz kodu tek, davranış manifest'ten. F75 takılıysa Connect → Log'da `06 82 01 00 …` →
cevap `03 00 00 00 00 cd`; katalogdan `f75.json` seçilir.

### Faz 1 — g5kb (F75 ailesi)
**Süre:** 5–7 gün · **Bağımlılık:** Faz 0

| # | İş | Kabul ölçütü |
|---|---|---|
| 1.1 | `protocols/g5kb.js`: kimlik, başlık, parçalama, `read`/`write` | Birim testleri fixture ile eşleşiyor |
| 1.2 | `identify()` → 6-byte parola; `probe.accept` ile eşleşme | Her parola doğru manifest'e çözülüyor |
| 1.3 | LED bloğu read-modify-write; `light.modes[].hw` eşlemesi | Parlaklık 1 artır → yalnız 1 byte değişiyor |
| 1.4 | Işık sekmesi manifest'ten: efektler, renk/hız/yön görünürlüğü | Cihazda efekt değişiyor; `color:false` efektte renk gizli |
| 1.5 | Per-key: `perKey.leds` kadar LED, fırça, paletler | 3 tuş boya → cihazda görünüyor |
| 1.6 | Tuş haritası: base + Fn, atama editörü | 1 tuş ata → test et → geri al |
| 1.7 | Makrolar: kodlayıcı + editör + `macros.maxBytes` sınırı | 3 adımlık makro çalışıyor; dolunca yazma reddediliyor |
| 1.8 | Ayarlar: `settings[]` → kontrol üretimi | Yeniden bağlanınca kalıcı |
| 1.9 | Pil sorgusu (`battery.level`) | Yüzde görünüyor |
| 1.10 | **8 manifest**: F75, F75 Pro, F87, F87 Pro, F99, F99 Pro, F108, H108 + paylaşılan `layouts/*.json` | 8 model listeleniyor; kod değişmedi |

**Çıkış:** En az bir fiziksel F75'te ışık + tuş + makro çalışıyor; diğer 7 model "experimental".
8 manifestin tamamı **yalnızca JSON** ile eklendi — bu, mimarinin asıl sınavı.

### Faz 2 — aula04 (S75 Pro ailesi)
**Süre:** 7–10 gün · **Bağımlılık:** Faz 1 (bileşenler hazır)

| # | İş | Kabul ölçütü |
|---|---|---|
| 2.1 | `protocols/aula04.js`: USB çerçevesi, ack, yazma dizisi | Birim testleri geçiyor |
| 2.2 | 2.4 GHz çerçevesi + checksum | Dongle üzerinden ışık değişiyor |
| 2.3 | Işık: 19 efekt, yön, yan ışık | S75 Pro'da doğrulandı |
| 2.4 | Per-key: 9×64 blok, oku/yaz | Gerçek renkler okunup geri yazılıyor |
| 2.5 | Tuş + Fn, aula04 kodlaması (g5kb'den farklı) | Atama çalışıyor |
| 2.6 | Makrolar (`count` formülü, `AA55` sonu) | Çalışıyor |
| 2.7 | Ayarlar + model farkları (L99 byte kayması) | Her modelde doğru byte |
| 2.8 | Saat senkronu (`caps.clock`) | Cihaz ekranında doğru saat |
| 2.9 | **Ekran sekmesi**: `screen.width/height` manifest'ten → kırpma + GIF → RGB565 → 4096'lık parçalar | S75 Pro'da GIF oynuyor; L99'da 320×480 oranı otomatik |
| 2.10 | `readBack:false` yolu (F75 Max) | UI "doğrulandı" demiyor |
| 2.11 | F98 Pro / S99 Pro / S98 Pro: `status:"exploring"`, salt-okunur | Log'da anlamlı cevap var mı not edildi |
| 2.12 | **5 manifest**: S75 Pro, L99, F108 Pro, F75 Max, F98 Pro (+ S99/S98 taslak) | Yalnız JSON; 5 model, tek protokol dosyası |

**Çıkış:** S75 Pro tam (ekran dahil), L99/F108 Pro experimental, F75 Max yaz-only — hepsi JSON farkı.

### Faz 3 — Fareler
**Süre:** 6–8 gün · **Bağımlılık:** Faz 0

| # | İş | Kabul ölçütü |
|---|---|---|
| 3.1 | `mouse55.js` (SC620) | DPI, yoklama, buton, pil |
| 3.2 | `incott.js` (SC900 ailesi) | 6 DPI kademesi, 8K yoklama, sensör |
| 3.3 | `yxt.js` (SC390) | Yoklama + LOD + anahtarlar |
| 3.4 | `evision.js` (V9/X9) | `identify()` → configId → manifest |
| 3.5 | `hubmouse.js` (SC800 vb.) | Tanıma + pil (kısmi) |
| 3.6 | **Mouse sekmesi tamamen manifest'ten**: `mouse.dpi`, `.polling`, `.sensor`, `.power` | Eksik alt anahtar → o kontrol gizli |
| 3.7 | Buton slotları `mouse.buttons.slots` → fare görseli | `locked` slotlar düzenlenemez |
| 3.8 | Fare makroları (mouse55, evision) | 1 makro çalışıyor |
| 3.9 | **8 manifest**: SC620, SC900, SC390, V9, X9, SC800 Max, SC580, KUNLUN X5 | KUNLUN X5 `status:"exploring"` kalır |

**Çıkış:** SC900 Pro ve SC620 fizikselde doğrulandı; DPI kademe sayısı manifest'ten geliyor.

### Faz 4 — HE / manyetik
**Süre:** 8–12 gün · **Bağımlılık:** Faz 1–2 · **En riskli faz**

| # | İş | Kabul ölçütü |
|---|---|---|
| 4.1 | `bytetech.js`: çerçeve, CRC, `0x82` UUID kimlik | HERO 68 tanınıyor |
| 4.2 | Aktüasyon okuma/yazma (11 kayıt/paket) | 1 tuş AP değişiyor |
| 4.3 | Rapid Trigger + deadzone | Çalışıyor |
| 4.4 | Canlı Hall telemetrisi | Basınca derinlik grafiği doluyor |
| 4.5 | HE sekmesi | Tüm kontroller |
| 4.6 | `xsyd.js` (WIN 68 HE MAX) — µm birimi, `0x5C` | Çerçeve testleri + okuma |
| 4.7 | `hfd.js` (F75 HE) — beyaz liste | Yalnız 5 komut; okuma çalışıyor |
| 4.8 | `oem1ca5.js` (HERO 68 ULTRA/PRO) | Tanıma |
| 4.9 | SOCD / DKS / Mod Tap editörü (bytetech) | Bir senaryo uçtan uca |
| 4.10 | AG60/AG75/NEOS/MINI/WIN60 kimlik teyidi | bytetech manifesti ile tanındı veya `exploring` |
| 4.11 | **6 manifest**: F75 HE, HERO 68/84/75/99 HE, WIN 68 HE MAX | `hall` bölümü manifest'ten (birim: 0.01 mm / µm) |

**Çıkış:** HERO 68/84 ve WIN 68 HE MAX doğrulandı; F75 HE experimental.
Aktüasyon birimi manifest'te (`"unit": "0.01mm"`), kodda sabit değil.

### Faz 5 — Derinlik
**Süre:** 5–8 gün

- Ekran editörü: kırpma arayüzü, kare kare gezinme, önizleme, kare gecikmesi.
- Profil yönetimi: kopyala, model dönüştürme, paylaşım bağlantısı (sıkıştırılmış JSON, URL fragment).
- Pil geçmişi (oturum içi, kalıcı değil).
- Ritim/müzik senkronu — yalnız `aula04` `0x20` canlı per-key'e sahip modeller.
- Tanılama paketi: log + ortam + model, cihaz verisi içermez.
- **Topluluk manifestleri**: kullanıcı kendi JSON'unu yükleyip dener (yerel, doğrulanmış,
  `status:"user"` etiketiyle, paylaşılamaz). Yeni model eklemenin kod gerektirmediğini kanıtlar.

### Faz 6 — Cila ve yayın
**Süre:** 3–5 gün

- Erişilebilirlik denetimi (klavye, kontrast, ekran okuyucu).
- Linux udev sayfası: hazır kuralı kopyalat, "nasıl uygulanır" adımları.
- README + ekran görüntüleri + **`docs/SUPPORTED.md` manifest kataloğundan üretilir**
  (`node tools/gen-support.js` → tablo, elle yazılmaz).
- GitHub Pages, ilk etiket (`v0.1.0`), katkı rehberi (nasıl manifest yazılır → SCHEMA.md).

### Toplam
| Faz | Süre |
|---|---|
| 0 | 3–4 gün |
| 1 | 5–7 gün |
| 2 | 7–10 gün |
| 3 | 6–8 gün |
| 4 | 8–12 gün |
| 5 | 5–8 gün |
| 6 | 3–5 gün |
| **Toplam** | **~5–7 hafta** (tek geliştirici, tek donanım) |

---

## 13. Riskler

| # | Risk | Etki | Azaltma |
|---|---|---|---|
| 1 | WebHID yalnız Chromium masaüstü | Kullanıcı kitlesi daralır | Açık destek kontrolü + net mesaj; Electron sarmalayıcı yolu açık bırakıldı |
| 2 | Linux'ta udev kuralı yok | Cihaz listede görünür, açılmaz | Hazır kural + kopyala düğmesi + hata mesajında tam yol |
| 3 | Resmî sürücü açık | Bağlantı başarısız | "Diğer programı kapat" adımları; `exclusive` hatası yakalanır |
| 4 | Yanlış model tanıma | Bozuk ayar | Kimlik zorunlu, salt-okunur yedek, oku-değiştir-yaz, imza kontrolü |
| 5 | Yaz-only modeller (F75 Max) | Doğrulama yok | UI "gönderildi" der, "doğrulandı" demez |
| 6 | HE ailesi 4 alt protokol | Yanlış komut zararsız ama boşa deneme | Kimlik cevabı yoksa hiçbir şey yazılmaz |
| 7 | Protokol hatası donanımda ancak görülür | Sessiz yanlış davranış | Her aile için ayrı doğrulama listesi; C durumu kullanıcıya açık |
| 8 | Cihaz yazma sırasında çıkar | Yarım ayar | Kuyruk temizlenir; sonraki bağlantıda yeniden okunur; "kısmi yazıldı" loglanır |
| 9 | Tarayıcı sekmesi arka planda | Zamanlayıcı kısılır, gecikmeler kayar | `setTimeout` yerine paket başına ack beklenir; uzun işlemlerde `Worker` |
| 10 | Firma protokolü değiştirir | Model çalışmaz | Sürüm/firmware alanı gösterilir; "bilinmeyen davranış" raporlanabilir |
| 11 | Lisans uyumu | Proje yayınlanamaz | Kaynak başına lisans notu; GPL-3 seçildi; satıcı binary'si depoya girmez |
| 12 | Kapsam şişmesi | Bitmez | Fazlar dondurulmuş; kapsam dışı listesi planda sabit |
| 13 | JSON'un programlama diline dönüşmesi | Bakımı imkânsız "config DSL" | Manifest saf veri; koşul/döngü/formül yasak (SCHEMA §13). Karmaşık davranış koda, koda 1 kez |
| 14 | Manifest hatası yanlış modele yazar | Bozuk ayar | Doğrulama zorunlu, `verified` için `probe.strong`, paylaşılan VID:PID'de tekil `accept` (SCHEMA §9) |
| 15 | 36 manifestin bakımı | Tutarsız model listesi | `manifest.test.js` hepsini doğrular; `SUPPORTED.md` katalogdan üretilir |

---

## 14. Açık sorular

Bunlar yanıtlanmadan ilgili faz tamamlanamaz:

1. **Hangi fiziksel cihaz elinde var?** Doğrulama listesi buna göre önceliklenir.
2. **L99 ekranı iki yol.** `0C45:800A` üzerinden `0x72` mi, yoksa ayrı `EEEF:268A` COM cihazı mı?
   WebHID COM portuna erişemez — ikinci yol seçilirse L99 ekranı kapsam dışı kalır.
3. **AG60/AG75/NEOS/MINI/WIN60 gerçekten bytetech mi?** Hub'da aynı SDK görünüyor ama
   vendor sayfası ve UUID eşlemesi doğrulanmadı. Bir cihaz olmadan teyit edilemez.
4. **SC580 / S12 Pro / KUNLUN X5** protokolü hiçbir açık kaynakta yok. Resmî sürücü indirilip
   statik analiz mi yapılacak, yoksa "desteklenmiyor" mu kalacak?
5. **F98 Pro / F98X / S99 Pro** gerçekten aula04 mü? Önce salt-okunur snif.
6. **Klavye görselleri.** Kendi SVG'lerimizi mi çizeceğiz (telifsiz), yoksa basit şematik
   tuş ızgarasıyla mı başlayacağız? İkincisi daha hızlı ve hukuken temiz.

---

## 15. Çalışma kuralları

- **Build yok, bağımlılık yok.** Yeni paket eklemek gerekçe ister.
- **Protokol modülleri saf.** I/O, DOM, `window` yok; testleri donanımsız koşar.
- **Her yeni protokol kodu bir testle gelir.** Fixture kaynaktan alınır.
- **Bilinmeyen byte korunur.** "Şimdilik sıfırlayalım" yasak.
- **Hata mesajı kullanıcı dilinde**, eylem içerir.
- **Türkçe açıklama, İngilizce kod.** Tanımlayıcılar ve yorumlar İngilizce; README/PLAN Türkçe.
- **Manifest değişikliği kod değişikliği değildir.** Yeni model PR'ı yalnızca `.json` + layout içerir.
- **Manifest'e mantık sızmaz.** Koşul, formül, değişken, şablon yok. İhtiyaç doğarsa önce
  şemada yeni bir *veri* alanı olarak düşünülür; olmuyorsa aile kodu büyür (bir kez, tüm modeller için).
- **Commit:** tek konu, çalışan durum. `feat(protocol):`, `fix(g5kb):`, `feat(model):`, `docs:`.

---

## 16. İlk adım

Faz 0.1–0.7 sırayla:

1. Kabuk, tema token'ları, router, i18n (0.1–0.4).
2. **`core/manifest.js` doğrulayıcı** + `tests/manifest.test.js` (0.5) — 14 kural, bozuk örnek fixture'ları.
3. **`models/index.json`** + iki örnek manifest: `f75.json` (klavye) ve `sc620.json` (fare) (0.6).
4. **Sekmelerin manifest'ten üretilmesi** (0.7) — `?sim=f75` ile Lighting/Keys/Macros,
   `?sim=sc620` ile Mouse. Aynı view kodu, iki farklı arayüz.

0.7 bittiğinde mimarinin asıl iddiası kanıtlanmış olur: **model davranışı JSON'da, arayüz tek**.
Ondan sonra transport (0.8), registry (0.9), simülatör (0.10), log (0.11), dashboard (0.12).
F75 takılıysa 0.9'da kimlik okuması `03 00 00 00 00 cd` döndüğü an temel doğrulanmış olur.
