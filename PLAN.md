# AulaBridge — Geliştirme Planı

Tarayıcıdan çalışan, kurulum gerektirmeyen AULA klavye ve fare yapılandırıcısı.
Tek sekmede model tanıma, ışık, tuş atama, makro, DPI, ekran, profil ve paket günlüğü.

> Resmî AULA ürünü değildir. AULA markası ve ürün adları sahiplerine aittir.
> Protokol bilgisi geri mühendislikle elde edilmiştir; kullanım riski kullanıcıya aittir.

Protokol bilgisi kaynaktan izlenir: her modelin `driver.js` başında uyarlanabilir kaynak depo ve
lisans notu bulunur; ayrı bir protokol belgesi tutulmaz.

---

## İçindekiler

1. [Ürün tanımı](#1-ürün-tanımı)
2. [Kararlar](#2-kararlar)
3. [Destek matrisi](#3-destek-matrisi)
4. [Mimari](#4-mimari)
5. [Sürücü sözleşmeleri](#5-sürücü-sözleşmeleri)
6. [Sürücü kataloğu](#6-sürücü-kataloğu) → [docs/SCHEMA.md](docs/SCHEMA.md)
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

### Her model kendi sürücüsü
AULA her model için ayrı bir sürücü yayınlıyor; bu proje de öyle davranır. Model sayısı kadar
sürücü vardır, hiçbiri diğerinin yerine geçmez. Aynı VID:PID'yi paylaşan modeller (ör.
`258A:010C` üzerinde 8 klavye) birbirine karışmaz: her sürücü önce **kendi kimlik sorgusunu**
yapar, eşleşmezse o modelde durmaz.

Bir modelin sürücüsü = `models/<model>/driver.json` (bildirim) + `models/<model>/driver.js`
(kimlik, komut kodlayıcıları, okuma-yazma akışı). `driver.json` taşıdığı alanları beyan eder;
`driver.js` yalnızca o modelin protokolünü uygular. Var olan bir sürücüyü başka modele
"uyarlamak" diye bir şey yoktur — yeni model, yeni dosya.

Ortak olan tek şey, arayüzün sürücü bildirimini nasıl okuduğu ve cihaz taşımasının (WebHID)
nasıl yapıldığıdır. Şema: [docs/SCHEMA.md](docs/SCHEMA.md)

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
Her satır bir sürücü klasörüdür: `models/<id>/`. Bayt düzeni o klasörün `driver.js` başında,
kaynak depo ve lisans notuyla birlikte.

**Kural: her sürücü en az iki bağımsız kaynaktan yazılır.** Kaynaklar uyuşmuyorsa fark
`driver.json`'daki `divergence[]` alanına yazılır — hangi alan, hangi kaynak ne diyor, ne
seçildi, neden. Tek kaynaklı sürücü `verified` olamaz. Ayrıntı: [SCHEMA §2a](docs/SCHEMA.md#2a-kaynak-çapraz-kontrolü).

Aşağıdaki tabloda "Kaynak" sütunu tek isim gösteriyor; uygulamada her sürücüde en az ikisi
kullanılır (ör. F75 → open-aula + AulaSofwareLinux + AulaToolkit).

### Klavyeler — 24 sürücü

| Model (`models/…`) | Kaynak | Kablo / dongle | Durum | Öne çıkan |
|---|---|---|---|---|
| `f75` | open-aula | 258A:010C / 3554:FA09 | **V** | 18 efekt, per-key, makro |
| `f75pro` | open-aula | aynı | C | kimlik parolası farklı |
| `f87` | open-aula | aynı | C | 87 tuş |
| `f87pro` | open-aula | aynı | C | 87 tuş |
| `f99` | AulaSofwareLinux | aynı | C | 96% düzen |
| `f99pro` | AulaSofwareLinux | aynı | C | 96% düzen |
| `f108` | AulaSofwareLinux | aynı | C | tam boy |
| `h108` | AulaSofwareLinux | aynı | C | tam boy |
| `f87pro-sonix` | open-aula | 38A6:271x / 0C45:FEFC | C | 7×7 LED matris |
| `s75pro` | open-aula | 0C45:800A / 0C45:FEFE, 05AC:024F | **V** | 19 efekt, TFT 135×240, yan ışık |
| `l99` | AulaToolkit | 0C45:800A / 05AC:024F | C | 84 tuş, TFT 320×480 |
| `f108pro` | parsiya/f108-pro | 0C45:800A / 05AC:024F | C | 20 efekt, 3 ışık bölgesi |
| `f75max` | FaizAsta | 0C45:800A / 05AC:024F | C | **yaz-only** |
| `f98pro` · `s99pro` | — | 0C45:800A | ? | salt-okunur keşif |
| `s98pro` | — | 0C45:800A | ? | önce snif |
| `f2088` | Dahuum | ? | C | yalnız ışık |
| `f75he` | FaizAsta | 38A6:2729 / 0C45:FEFE | C | manyetik, per-key |
| `hero68he` | shizunavn/OpenAula | 372E:103E / 372E:1000, 106C | **V** | AP/RT, SOCD, DKS |
| `hero84he` | shizunavn/OpenAula | aynı | **V** | aynı çerçeve |
| `hero75he` · `hero99he` | shizunavn/OpenAula | aynı | C | kendi UUID'leri |
| `hero68ultra` | OpenAula | 1CA5:0407 | C | report ID 3 |
| `win68hemax` | OpenAula | 1CA2:1901, 1C4F:EE88 | **V** | µm birimi, 4 slot |
| `ag60` · `ag75` · `neos75` · `mini60he` · `win60he` | hub | 372E / hub | ? | kimlik sorgusuyla teyit |

### Fareler — 12 sürücü

| Model (`models/…`) | Kaynak | USB ID | Durum |
|---|---|---|---|
| `sc620` | open-aula | A8A4:2255 / A8A5:2255 | buton haritası **V** |
| `sc900` | AulaToolkit | 093A / 248A / 379A | **V** |
| `sc390` | AulaToolkit | 32C2:0024 | C |
| `v9` | shDavlatbek/aulaar | 320F:22B7/22B8 | C |
| `x9` | shDavlatbek/aulaar | 320F:22B6 | C |
| `sc800max` | OpenAula | 38A6:28FC/28FE/2F09 | C (kısmi) |
| `sc580` · `kunlunx5` | — | ? | tanınır, sekme kapalı |

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
│ registry.js     katalog; VID:PID → aday sürücüler; her adayın identify()'ı
├──────────────────────────────────────────────────────────┤
│ models/<id>/    driver.json (veri) + driver.js (o modelin protokolü)
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
│  ├─ SCHEMA.md               sürücü bildirimi sözleşmesi
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
│  ├─ driver.js               driver.json şema doğrulama + çapraz referans kontrolü
│  ├─ registry.js             katalog → aday sürücüler → her adayın identify()'ı
│  ├─ log.js
│  ├─ persist.js              localStorage, profil içe/dışa aktarma
│  └─ sim.js                  simülatör: driver.json + driver.js ile sahte cihaz
├─ models/
│  ├─ index.json              katalog: id → klasör, durum
│  ├─ f75/
│  │  ├─ driver.json          USB ID, kimlik, caps, efektler, ayarlar, sınırlar
│  │  ├─ driver.js            kimlik sorgusu + komut kodlayıcıları + okuma/yazma akışı
│  │  └─ layout.json          tuş yerleşimi
│  ├─ f75pro/  f87/  f87pro/  f99/  f99pro/  f108/  h108/
│  ├─ s75pro/  l99/  f108pro/  f75max/  f98pro/  s99pro/  s98pro/  f2088/
│  ├─ f75he/  hero68he/  hero84he/  hero75he/  hero99he/
│  ├─ win68hemax/  hero68ultra/  mini60he/  win60he/  ag60/  ag75/  neos75/
│  ├─ sc620/  sc900/  sc390/  v9/  x9/  sc800max/  sc580/  kunlunx5/
│  └─ f87pro-sonix/
├─ shared/
│  └─ util.js                 hex, tag(AA55), sum8, chunk — sürücülerin ortak yardımcıları
├─ assets/
│  ├─ devices/                model fotoğrafı (koyu zemin, 3/4 açı)
│  └─ icons/                  efekt ve işlev ikonları
├─ tests/
│  ├─ f75.test.js  s75pro.test.js  hero68he.test.js   sürücü başına bir test
│  ├─ sc620.test.js  sc900.test.js  …
│  ├─ driver.test.js          her driver.json doğrulanır; çapraz referanslar; caps ↔ fonksiyon
│  ├─ registry.test.js i18n.test.js
│  └─ fixtures/               gerçek paket baytları + bozuk bildirim örnekleri
└─ .github/workflows/pages.yml
```

---

## 5. Sürücü sözleşmeleri

Her model kendi sürücüsüne sahiptir: `models/<id>/driver.json` + `models/<id>/driver.js`.
Aile/paylaşılan protokol katmanı yoktur. İki model aynı firmware'i kullanıyor olsa bile
birbirinden bağımsız iki dosyadır; birini değiştirmek diğerini etkilemez.

### `models/<id>/driver.json`

```json
{
  "schema": 1,
  "id": "f75",
  "name": "AULA F75",
  "kind": "keyboard",
  "usb": [
    { "vid": "258a", "pid": "010c", "role": "usb",    "usagePage": 65280 },
    { "vid": "3554", "pid": "fa09", "role": "dongle", "usagePage": 65280 }
  ],
  "probe": { "accept": ["0300000000cd"] },
  "status": "verified",
  "caps": { "lighting": true, "perKey": true, "keymap": true, "fnLayer": true,
            "macros": true, "settings": true, "battery": true, "readBack": true },
  "matrix": { "len": 128 },
  "light": { ... },
  "perKey": { "leds": 90 },
  "macros": { "maxBytes": 3584, "nameMax": 30, "playModes": ["repeatN", "hold", "untilAny"] },
  "settings": [ ... ],
  "battery": { "level": true, "charging": false }
}
```

Şema ve doğrulama kuralları: [docs/SCHEMA.md](docs/SCHEMA.md).

### `models/<id>/driver.js`

Yalnızca o modelin protokolü. Saf kodlayıcı/çözücü; I/O yok — `session` gönderir.
Dosyanın başında kaynak listesi ve lisans notu **zorunlu**:

```js
// AULA F75 — BYCOMBO4 520-byte feature reports, report id 6.
// Sources (cross-checked, see driver.json source[]/divergence[]):
//   paulcoding810/open-aula   web/PROTOCOL.md §F75, web/protocols/g5kb.js   (no licence stated)
//   yhuikzdtguioaert/AulaSofwareLinux  src/proto.rs                        (MIT)
//   livvaa/AulaToolkit        src/shared/protocols/g5kb.ts                 (GPL-3.0)

export const reportId = 6;                       // varsa; bazı sürücülerde aramayla bulunur
export const identify   = (io) => string | null; // io feature oku/yaz verir; accept ile eşleşir
export const readLight  = (block) => {...};
export const writeLight = (state, block) => Uint8Array[];
export const writePerKey= (colors) => Uint8Array[];
export const writeKeymap= (layer, actions) => Uint8Array[];
export const writeMacros= (list) => Uint8Array[];
export const writeSettings = (values, block) => Uint8Array[];
export const queryBattery = () => Uint8Array;
export const parseBattery = (bytes) => number | null;
```

- Sürücü, `driver.json`'daki `caps`'te `true` olan fonksiyonları **sağlamak zorunda**;
  olmayanları hiç yazmaz. `driver.test.js` bu eşleşmeyi zorunlu tutar.
- Fonksiyonlar paket üretir, göndermez. Adım listesinde `{wait: 35}` belirteci olabilir.
- Okuma fonksiyonları bozuk/eksik veride `null` döner; tahmin etmez.
- `identify` cihazın cevabını döner; `probe.accept` listesiyle `registry` eşleştirir.
- Başlık yorumu kaynakları ve **ayrışmayı** listeler. Ayrışma `driver.json`'daki
  `divergence[]` ile aynı olmalı; `driver.test.js` ikisini karşılaştırır.

### `tools/diff-sources.js`

İki kaynak repoyu yan yana okumaya yardım eden betik (Faz 0'da basit hâli):

```
node tools/diff-sources.js f75
  → models/f75/driver.json içindeki source[] listesini okur
  → her kaynak repodaki ilgili dosyayı açar
  → aynı kavramın (ör. efekt tablosu, kimlik parolası) farklı yazıldığı yerleri listeler
```

Kod karşılaştırması yapmaz; **veri** karşılaştırır (sabit tablolar, ID'ler, sınırlar).
Amaç: üç kaynaktan hangisinin ne dediğini tek ekranda görmek, `divergence[]`'ı elle doldurmak.

### `shared/util.js`

Tüm sürücülerin ortak yardımcıları: `hex`, `tag(buf, off)` (AA55), `sum8`, `chunk`.
Protokol mantığı içermez.

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

## 6. Sürücü kataloğu

Her model `models/<id>/` klasöründe kendi bildirimi ve kendi koduyla durur. Ayrıntılı şema:
**[docs/SCHEMA.md](docs/SCHEMA.md)**

### Neyin bildirim, neyin sürücü kodu olduğu

| `driver.json` (veri) | `driver.js` (o modelin protokolü) |
|---|---|
| Model adı, VID:PID, kimlik baytları | Kimlik sorgusu paketleri |
| Yetenekler, efekt tablosu, tuş yerleşimi | Çerçeve, checksum, parçalama |
| Ayar seçenekleri, sınırlar, pil bayrakları | Komut sırası, oku-değiştir-yaz |
| DPI kademeleri, yoklama listesi, buton slotları | — |

İkisi de modele ait. **Ortak protokol katmanı yok**: yeni model = yeni klasör
(`driver.json` + `driver.js` + `layout.json` + test). Aynı firmware'i paylaşan bir model
eklenirken ilgili kaynak repo okunur, ama kod o modelin klasöründe yazılır.

### Katalog

`models/index.json` — hangi sürücü var, hangi durumda:

```json
{
  "schema": 1,
  "models": [
    { "id": "f75",   "status": "verified",     "dir": "f75" },
    { "id": "s75pro","status": "verified",     "dir": "s75pro" },
    { "id": "sc580", "status": "exploring",    "dir": "sc580" }
  ]
}
```

Uygulama açılışta yalnızca kataloğu indirir. Cihaz bağlanınca o VID:PID'ye uyan sürücüleri
tembel yükler — 36 klasörü baştan indirmez.

### Çözüm sırası (`registry.js`)

```
1. Cihazın tüm HID koleksiyonlarını topla (aynı VID:PID altında birden çok olabilir)
2. Katalogdan bu VID:PID'ye uyan sürücüleri yükle ve doğrula (§SCHEMA 9)
3. Aday yoksa            → "tanınmıyor" kartı, hiçbir yazma yok
4. Aday tekse            → o sürücünün identify() fonksiyonunu çağır;
                            dönen değer probe.accept ile eşleşirse bağlan, yoksa "tanınmıyor"
5. Aday birden çoksa     → her adayın KENDİ identify() fonksiyonu sırayla denenir:
                            258A:010C → 8 sürücü, her biri kendi parolasını bekler
                            0C45:800A → 7 sürücü, her biri kendi cevabını bekler
                            eşleşen model bağlanır, diğerleri reddedilir
6. Kimlik çözüldü        → driver.json yüklendi, sekmeler caps'ten üretilir
7. Çözülemedi            → fallback işaretli sürücü varsa SALT OKUNABİLİR aç:
                            sekmeler görünür, yazma düğmeleri kilitli
                            Yoksa "tanınmıyor"
```

Kimlik okuması **her bağlantıda** yapılır, önbelleğe alınmaz. USB portu değişebilir.
Aynı VID:PID'deki her sürücü **farklı** `probe.accept` taşımak zorunda (`driver.test.js`).

### Bildirimin sürdüğü şeyler

`driver.json` değişince arayüz değişir; view koduna dokunulmaz:

| `driver.json` | Arayüzde karşılığı |
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

Aynı VID:PID altındaki her sürücü **farklı `probe.accept` değeri** taşımalı; `driver.test.js`
bunu zorunlu kılar (bkz. SCHEMA §9 kural 14). İki sürücü aynı cevabı bekliyorsa yanlış
modelin yazması riski var — test bunu yakalar.

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
6. **Her sürücü yalnız kendi işlevlerini üretir.** `driver.js` fabrika sıfırlama, firmware
   veya test komutu içermez; sıfırlama gerekiyorsa ayrı ve açık isimli bir fonksiyondur,
   arayüzde iki adım onay ister.
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
| `f75.test.js` | 520-byte başlık, parça bölme, tuş kodlama, LED bloğu read-modify-write, makro olay kodlama, `identify()` → `0300000000cd` |
| `s75pro.test.js` | 64-byte komut, `AA55` etiketi, 9 paketlik tuş/per-key bloğu, makro `count` formülü, ekran başlığı ve RGB565, 32-byte dongle checksum |
| `f87pro-sonix.test.js` | `AA cmd len …` başlığı, yankı bekleme sırası, offset'li blok okuma |
| `sc620.test.js` | `55 cmd flag chk …` checksum, parametre bloğu 46 byte, buton girişi kodlama |
| `sc900.test.js` | 9-byte çerçeve, alt-komut yankısı, yoklama tablosu, LOD eşlemesi |
| `sc390.test.js` | nibble maskesi, bölüm 0x40 byte 14 bit alanları |
| `v9.test.js` | report ID 4, blok transferi, configId → model, 128-byte blok offset'leri |
| `f75he.test.js` | 5 komutluk beyaz liste (başka komut üretilemez), kimlik doğrulama |
| `hero68he.test.js` | 63-byte payload başlığı, `cmd\|0x80`, 56-byte veri, birim dönüşümü (mm → 0.01 mm) |
| `win68hemax.test.js` | `0x5C` çerçevesi, `CK` formülü, **iki istisna** (`OP<0x80` → CK 0, `OP=0x30` → `frame[63]`), µm birimi |
| **`driver.test.js`** | **Depodaki tüm sürücüler**: her `driver.json` 18 kurala uyuyor mu, `caps`'teki her yeteneğin karşılığı `driver.js`'te var mı, layout'ta `idx`/`light` sınırları, paylaşılan VID:PID'de tekil `accept`, `verified` için `source.length >= 2`, `divergence[].chosen` gerçek değerle aynı, `driver.js` başlığındaki kaynak listesi `driver.json` ile aynı |
| `feature.test.js` | Yeni özellik eklendiğinde: alan varsa kontrol görünür, **alan silinince kontrol yok ve sürücü hata vermiyor** (SCHEMA §10.6) |
| `registry.test.js` | Paylaşılan VID:PID'lerde aday listesi doğru; her adayın kendi `identify()`'ı sırayla; `fallback` yalnız kimlik okunamayınca; bilinmeyen ID'de boş |
| `i18n.test.js` | Tüm diller `en.json` ile aynı anahtar kümesinde |
| `tabs.test.js` | `caps` → beklenen sekme seti; eksik bölüm → sekme yok; `exploring` → tüm yazma kilitli |

Fixture'lar kaynaklardaki gerçek kaptür baytlarından gelir; "benim kodum kendini doğrulasın"
değil, "satıcının paketiyle aynı baytı üretsin" test edilir.

### Elle donanım testi
`docs/HARDWARE-TEST.md` — her model için:
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

### Faz 0 — İskelet + sürücü motoru
**Süre:** 3–4 gün · **Bağımlılık:** yok

| # | İş | Kabul ölçütü |
|---|---|---|
| 0.1 | `index.html`, `styles/`, tema token'ları, açık/koyu | Tema değişiyor, `prefers-color-scheme` çalışıyor |
| 0.2 | `store.js` + subscribe + localStorage | Yenilemede ui tercihleri korunuyor |
| 0.3 | Hash router, sekme iskeletleri | Sekmeler gezilebiliyor |
| 0.4 | `i18n` + `en.json` + `tr.json` | Dil değişiyor, test geçiyor |
| 0.5 | **`core/driver.js`**: `driver.json` şema doğrulaması + 14 çapraz kural | Geçerli bildirimler yükleniyor, `tests/fixtures/bad-drivers/` hepsi reddediliyor |
| 0.6 | **`models/index.json`** katalog + tembel yükleme | Katalog → klasör → `driver.json` + `driver.js` zinciri çalışıyor |
| 0.7 | **Sekmelerin `driver.json`'dan üretilmesi** | Aynı sitede iki farklı sürücü → iki farklı sekme seti, view koduna dokunmadan |
| 0.8 | `transport.js` — istek, aç, feature oku/yaz, zaman aşımı | Simülatörle uçtan uca |
| 0.9 | `registry.js` — aday listesi, her adayın `identify()`'ı sırayla, salt-okunur yedek | Paylaşılan VID:PID'de doğru sürücü seçiliyor |
| 0.10 | `sim.js` — `driver.json` + `driver.js` ile sahte cihaz | Donanımsız her sürücü gezilebiliyor, demo hiç yazmıyor |
| 0.11 | `log.js` + Log sekmesi | Her çağrı hex olarak görünüyor, indirilebiliyor |
| 0.12 | Dashboard + boş durumlar | "Connect" akışı çalışıyor |
| 0.13 | `tests/` koşucusu + CI + Pages | `node --test` yeşil, site yayında |

**Faz 0 çıkışı:** İki örnek sürücü elle yazılır: `models/f75/` (klavye) ve `models/sc620/` (fare).
Tarayıcıda `?sim=f75` açıldığında Lighting/Keys/Macros sekmeleri, `?sim=sc620` açıldığında
Mouse sekmesi görünür — arayüz kodu tek, davranış sürücü bildiriminden.
F75 takılıysa Connect → Log'da `06 82 01 00 …` → cevap `03 00 00 00 00 cd`;
`f75` sürücüsü seçilir, `f87`/`f99`/… reddedilir.

### Faz 1 — klavyeler, tek tek sürücüler
**Süre:** 6–9 gün · **Bağımlılık:** Faz 0

| # | İş | Kabul ölçütü |
|---|---|---|
| 1.1 | `tools/diff-sources.js`: iki repo aç, veri farklarını listele | F75 için open-aula / AulaSofwareLinux / AulaToolkit efekt tabloları yan yana |
| 1.2 | `models/f75/`: üç kaynağı karşılaştır, `divergence[]`'ı doldur | Her ayrışan alan gerekçeli; `driver.test.js` kural 17 geçiyor |
| 1.3 | `models/f75/driver.js`: kimlik, başlık, parçalama, okuma/yazma | Birim testleri fixture ile eşleşiyor |
| 1.4 | `identify()` → 6-byte parola → `driver.json`'daki `accept` | F75 takılınca yalnız `f75` sürücüsü seçiliyor |
| 1.5 | LED bloğu read-modify-write; `light.modes[].hw` eşlemesi | Parlaklık 1 artır → yalnız 1 byte değişiyor |
| 1.6 | Işık sekmesi bildirimden: efektler, renk/hız/yön görünürlüğü | Cihazda efekt değişiyor; `color:false` efektte renk gizli |
| 1.7 | Per-key: `perKey.leds` kadar LED, fırça, paletler | 3 tuş boya → cihazda görünüyor |
| 1.8 | Tuş haritası: base + Fn, atama editörü | 1 tuş ata → test et → geri al |
| 1.9 | Makrolar: kodlayıcı + editör + `macros.maxBytes` sınırı | 3 adımlık makro çalışıyor; dolunca yazma reddediliyor |
| 1.10 | Ayarlar: `settings[]` → kontrol üretimi | Yeniden bağlanınca kalıcı |
| 1.11 | Pil sorgusu (`battery.level`) | Yüzde görünüyor |
| 1.12 | `feature.test.js`: her `settings` alanı için "alan silinince kontrol yok" | Devamsızlık testi geçiyor |
| 1.13 | **7 modele kopyala**, her birinde kaynakları ayrıca oku ve kimliği değiştir: f75pro, f87, f87pro, f99, f99pro, f108, h108 | 8 model listeleniyor; her sürücüde en az iki kaynak ve doldurulmuş `divergence` |

**Çıkış:** En az bir fiziksel F75'te ışık + tuş + makro çalışıyor; `f75` `verified` (üç kaynak),
diğer 7 sürücü `experimental` — her birinin kendi `source[]` ve `divergence[]`'ı var.

### Faz 2 — ekranlı klavyeler, tek tek sürücüler
**Süre:** 7–10 gün · **Bağımlılık:** Faz 1 (bileşenler hazır)

| # | İş | Kabul ölçütü |
|---|---|---|
| 2.1 | `models/s75pro/`: USB çerçevesi, ack, yazma dizisi | Birim testleri geçiyor |
| 2.2 | 2.4 GHz çerçevesi + checksum | Dongle üzerinden ışık değişiyor |
| 2.3 | Işık: 19 efekt, yön, yan ışık | S75 Pro'da doğrulandı |
| 2.4 | Per-key: 9×64 blok, oku/yaz | Gerçek renkler okunup geri yazılıyor |
| 2.5 | Tuş + Fn | Atama çalışıyor |
| 2.6 | Makrolar (`count` formülü, `AA55` sonu) | Çalışıyor |
| 2.7 | Ayarlar | Cihazda kalıcı |
| 2.8 | Saat senkronu (`caps.clock`) | Cihaz ekranında doğru saat |
| 2.9 | **Ekran sekmesi**: `screen.width/height` bildirimden → kırpma + GIF → RGB565 → 4096'lık parçalar | S75 Pro'da GIF oynuyor |
| 2.10 | `models/l99/`: aynı akış, ayar byte'ları kaymış (4/3/2), ekran 320×480 | Kendi `driver.js`'i; S75 Pro dosyasına dokunulmadı |
| 2.11 | `models/f108pro/`: 20 efekt, 3 ışık bölgesi | Kendi dosyası |
| 2.12 | `models/f75max/`: `readBack:false`, yaz-only | UI "doğrulandı" demiyor |
| 2.13 | `models/f98pro/`, `models/s99pro/`, `models/s98pro/`: `status:"exploring"` | Salt-okunur; Log'da anlamlı cevap var mı not edildi |

**Çıkış:** S75 Pro tam (ekran dahil) `verified`; diğer 6 sürücü kendi klasöründe, kendi kimlik
sorgusuyla. Hiçbiri ortak bir "aula04 katmanı" paylaşmıyor.

### Faz 3 — fareler, tek tek sürücüler
**Süre:** 6–8 gün · **Bağımlılık:** Faz 0

| # | İş | Kabul ölçütü |
|---|---|---|
| 3.1 | `models/sc620/` (modele özel) | DPI, yoklama, buton, pil |
| 3.2 | `models/sc900/` | 6 DPI kademesi, 8K yoklama, sensör |
| 3.3 | `models/sc390/` | Yoklama + LOD + anahtarlar |
| 3.4 | `models/v9/` | `identify()` → configId |
| 3.5 | `models/x9/` — V9'dan ayrı dosya; X9'da gövde ışığı yok | Kendi `driver.js`'i |
| 3.6 | `models/sc800max/` | Tanıma + pil (kısmi) |
| 3.7 | **Mouse sekmesi tamamen bildirimden**: `mouse.dpi`, `.polling`, `.sensor`, `.power` | Eksik alt anahtar → o kontrol gizli |
| 3.8 | Buton slotları `mouse.buttons.slots` → fare görseli | `locked` slotlar düzenlenemez |
| 3.9 | `models/sc580/`, `models/kunlunx5/`: `status:"exploring"` | Tanınır, yazma kilitli |

**Çıkış:** SC900 ve SC620 fizikselde doğrulandı; her fare kendi sürücü klasöründe.

### Faz 4 — HE / manyetik, tek tek sürücüler
**Süre:** 8–12 gün · **Bağımlılık:** Faz 1–2 · **En riskli faz**

| # | İş | Kabul ölçütü |
|---|---|---|
| 4.1 | `models/hero68he/`: çerçeve, CRC, `0x82` UUID kimlik | HERO 68 tanınıyor |
| 4.2 | Aktüasyon okuma/yazma (11 kayıt/paket) | 1 tuş AP değişiyor |
| 4.3 | Rapid Trigger + deadzone | Çalışıyor |
| 4.4 | Canlı Hall telemetrisi | Basınca derinlik grafiği doluyor |
| 4.5 | HE sekmesi | Tüm kontroller `hall` bölümünden |
| 4.6 | `models/hero84he/`, `models/hero75he/`, `models/hero99he/`: ayrı dosyalar | Her biri kendi UUID'siyle tanınıyor |
| 4.7 | `models/win68hemax/` — µm birimi, `0x5C` çerçevesi | Çerçeve testleri + okuma |
| 4.8 | `models/f75he/` — beyaz liste | Yalnız 5 komut; okuma çalışıyor |
| 4.9 | `models/hero68ultra/` — report ID 3 | Tanıma |
| 4.10 | SOCD / DKS / Mod Tap editörü (HERO 68) | Bir senaryo uçtan uca |
| 4.11 | `models/ag60/`, `ag75/`, `neos75/`, `mini60he/`, `win60he/`: kimlik teyidi | Tanındı veya `exploring` |

**Çıkış:** HERO 68/84 ve WIN 68 HE MAX `verified`; F75 HE `experimental`.
Aktüasyon birimi her sürücünün bildiriminde (`"unit": "0.01mm"` / `"1um"`), paylaşılan sabit yok.

### Faz 5 — Derinlik
**Süre:** 5–8 gün

- Ekran editörü: kırpma arayüzü, kare kare gezinme, önizleme, kare gecikmesi.
- Profil yönetimi: kopyala, paylaşım bağlantısı (sıkıştırılmış JSON, URL fragment).
- Pil geçmişi (oturum içi, kalıcı değil).
- Ritim/müzik senkronu — yalnız canlı per-key akışı olan sürücülerde.
- Tanılama paketi: log + ortam + model, cihaz verisi içermez.
- **Sürücü karşılaştırma aracı**: iki sürücünün `driver.js`'ini yan yana koyup farkı göster
  (kopyala-türet sırasında sürüklenmeyi yakalar). Yeni model eklemenin maliyetini düşürür.

### Faz 6 — Cila ve yayın
**Süre:** 3–5 gün

- Erişilebilirlik denetimi (klavye, kontrast, ekran okuyucu).
- Linux udev sayfası: hazır kuralı kopyalat, "nasıl uygulanır" adımları.
- README + ekran görüntüleri + **`docs/SUPPORTED.md` katalogdan üretilir**
  (`node tools/gen-support.js` → tablo, elle yazılmaz).
- GitHub Pages, ilk etiket (`v0.1.0`), katkı rehberi (nasıl sürücü yazılır → SCHEMA.md).

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
| 6 | HE klavyeleri 4 farklı çerçeve kullanıyor | Yanlış komut zararsız ama boşa deneme | Her sürücü kendi kimliğini bekler; eşleşmezse hiçbir şey yazılmaz |
| 7 | Protokol hatası donanımda ancak görülür | Sessiz yanlış davranış | Her model için ayrı doğrulama listesi; durum kullanıcıya açık |
| 8 | Cihaz yazma sırasında çıkar | Yarım ayar | Kuyruk temizlenir; sonraki bağlantıda yeniden okunur; "kısmi yazıldı" loglanır |
| 9 | Tarayıcı sekmesi arka planda | Zamanlayıcı kısılır, gecikmeler kayar | `setTimeout` yerine paket başına ack beklenir; uzun işlemlerde `Worker` |
| 10 | Firma protokolü değiştirir | Model çalışmaz | Sürüm/firmware alanı gösterilir; "bilinmeyen davranış" raporlanabilir |
| 11 | Lisans uyumu | Proje yayınlanamaz | Kaynak başına lisans notu; GPL-3 seçildi; satıcı binary'si depoya girmez |
| 12 | Kapsam şişmesi | Bitmez | Fazlar dondurulmuş; kapsam dışı listesi planda sabit |
| 13 | Bildirimin programlama diline dönüşmesi | Bakımı imkânsız "config DSL" | `driver.json` saf veri; koşul/döngü/formül yasak (SCHEMA §14). Karmaşık davranış `driver.js`'e |
| 14 | Bildirim hatası yanlış modele yazar | Bozuk ayar | Doğrulama zorunlu, paylaşılan VID:PID'de tekil `accept`, `verified` için fiziksel test (SCHEMA §9) |
| 15 | 36 sürücü arasında kopyala-türet sürüklenmesi | Bir modelde düzeltilen hata diğerinde kalır; sessiz tutarsızlık | Her model kendi testini taşır; Faz 5'te sürücü karşılaştırma aracı; ortak `shared/util.js` yalnız yardımcı |
| 16 | 36 sürücü bildiriminin bakımı | Tutarsız model listesi | `driver.test.js` hepsini doğrular; `SUPPORTED.md` katalogdan üretilir |
| 17 | Tek kaynağa güvenmek | O kaynağın hatası doğru sanılır, cihaza yanlış yazılır | En az iki kaynak (`source.length >= 2`), ayrışma `divergence[]`'a yazılır, tek kaynaklı sürücü `verified` olamaz (SCHEMA §2a) |
| 18 | Yeni özellik arayüze/model özel `if` olarak sızması | Model eklemek yine kod değişikliği gerektirir; mimari çöker | Kontrol JSON'da alan varsa görünür; `feature.test.js` alan silinince kontrolün kaybolduğunu doğrular (SCHEMA §10.6) |
| 19 | İki kaynak ayrışıyor, kimse fark etmiyor | Sessiz yanlış seçim | `divergence[]` zorunlu alan; `driver.test.js` `chosen` ile gerçek değeri karşılaştırır; aynı ayrışma çok modelde görünürse kaynak hatasıdır |

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
- **Sürücü modülleri saf.** I/O, DOM, `window` yok; testleri donanımsız koşar.
- **Her sürücü kendi dosyası.** Bir model için düzeltme yaparken başka modelin `driver.js`'i
  değişmez. Ortak kod yalnızca `shared/util.js` ve `core/` katmanıdır.
- **Her yeni sürücü kodu bir testle gelir.** Fixture kaynaktan alınır.
- **Bilinmeyen byte korunur.** "Şimdilik sıfırlayalım" yasak.
- **Hata mesajı kullanıcı dilinde**, eylem içerir.
- **Türkçe açıklama, İngilizce kod.** Tanımlayıcılar ve yorumlar İngilizce; README/PLAN Türkçe.
- **Bildirim değişikliği ile sürücü kodu ayrı commit.** `driver.json`'da yanlış bir efekt değeri
  kodu ilgilendirmez; `driver.js`'te bir çerçeve düzeltmesi bildirimi ilgilendirmez.
- **Bildirime mantık sızmaz.** Koşul, formül, değişken, şablon yok. İhtiyaç doğarsa önce
  şemada yeni bir *veri* alanı olarak düşünülür; olmuyorsa o modelin `driver.js`'i büyür.
- **Tek kaynak yeterli değil.** Her sürücü en az iki bağımsız kaynaktan yazılır; ayrışma
  `divergence[]`'a yazılır. Kaynaklar uyuşmuyorsa donanım testi olmadan `verified` yok.
- **Yeni özellik = şema → arayüz → sürücü → bildirim → test.** Ters sıra yasak. Kontrol
  modele özel `if` ile değil, JSON'da alan varsa görünür; `feature.test.js` bunu doğrular.
- **Commit:** tek konu, çalışan durum. `feat(driver):`, `fix(f75):`, `feat(model):`, `docs:`.

---

## 16. İlk adım

Faz 0.1–0.7 sırayla:

1. Kabuk, tema token'ları, router, i18n (0.1–0.4).
2. **`core/driver.js` doğrulayıcı** + `tests/driver.test.js` (0.5) — 14 kural, bozuk örnek fixture'ları.
3. **`models/index.json`** + iki örnek sürücü: `models/f75/` (klavye) ve `models/sc620/` (fare) (0.6).
4. **Sekmelerin `driver.json`'dan üretilmesi** (0.7) — `?sim=f75` ile Lighting/Keys/Macros,
   `?sim=sc620` ile Mouse. Aynı view kodu, iki farklı arayüz.

0.7 bittiğinde mimarinin asıl iddiası kanıtlanmış olur: **her modelin davranışı kendi
klasöründe, arayüz tek**.
Ondan sonra transport (0.8), registry (0.9), simülatör (0.10), log (0.11), dashboard (0.12).
F75 takılıysa 0.9'da `models/f75/driver.js`'in kimlik sorgusu `03 00 00 00 00 cd` döndüğü an
temel doğrulanmış olur; `f87`, `f99` ve diğer adaylar reddedilir.
