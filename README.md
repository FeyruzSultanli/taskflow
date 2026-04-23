# TaskFlow — Kanban Proje Yönetim Tahtası

Tek bir `index.html` dosyasından oluşan, build adımı olmayan, Trello benzeri bir kanban uygulaması. Statik olarak Vercel / GitHub Pages / Netlify — herhangi bir yere sürükle-bırak ile deploy edilebilir.

> **Canlı demo:** `index.html`'i çift tıklayarak aç veya `npx serve` ile çalıştır.

---

## İçindekiler
- [Özellikler](#özellikler)
- [Teknoloji tercihleri ve gerekçeler](#teknoloji-tercihleri-ve-gerekçeler)
- [Veri modeli](#veri-modeli)
- [Sıralama algoritması (fractional indexing)](#sıralama-algoritması-fractional-indexing)
- [Neye öncelik verildi, neye verilmedi](#neye-öncelik-verildi-neye-verilmedi)
- [Klavye kısayolları](#klavye-kısayolları)
- [Deploy](#deploy)

---

## Özellikler

### Zorunlu beklentilerin tamamı
- Hesap oluşturma / giriş yapma (PBKDF2 ile hash'lenmiş şifre)
- Birden çok board, board başına sınırsız sütun ve kart
- Kartları sürükle-bırak ile sütunlar arasında ve sütun içinde taşıma
- **Sütunların kendisi de sürüklenerek yeniden sıralanabilir**
- Kart detayları: başlık + açıklama
- Sıralama sayfa yenilemede kalıcıdır (fractional index + `localStorage`)
- Tamamen statik → Vercel'da sorunsuz çalışır

### Üstüne eklenenler
- **Renkli etiketler** (6 etiket, açık/koyu temayla uyumlu)
- **Son teslim tarihi** — süresi geçen / yakın kartlarda renkli uyarı
- **Çoklu sorumlu** (aynı cihazda kayıtlı başka kullanıcılar atanabilir)
- **Alt görevler (checklist)** — kart önizlemesinde ilerleme çubuğu
- **Kart aktivite geçmişi** — taşıma ve düzenlemeler otomatik kayıt altına alınır
- **Etiket filtresi** ve **metin araması** (başlık + açıklama)
- **Çoklu sekme senkronizasyonu** — `BroadcastChannel` ile aynı hesabı iki sekmede açtığında değişiklikler anında yansır
- **Geri al (undo)** — kart / sütun silindiğinde 5 saniye boyunca toast üzerinden geri alınabilir
- **Karanlık/aydınlık tema** (sistem tercihine uyumlu)
- **JSON dışa / içe aktarma** (yedek + taşıma)
- **Klavye kısayolları** + `?` ile kısayol rehberi
- **Erişilebilirlik**: kartlar `tabindex` ile klavye ile gezilebilir ve Enter ile açılır; modal ESC ile kapanır; ARIA etiketleri.
- **Mobil uyumlu**: `scroll-snap` ile tek tek sütun kaydırma; dokunmada `delay` ile kazara sürüklemeyi önleme.

---

## Teknoloji tercihleri ve gerekçeler

### Sürükle-bırak: neden SortableJS?
Dört aday değerlendirildi:

| Kütüphane | Artı | Eksi |
|---|---|---|
| **dnd-kit** | Modern, React-first, en popüler seçim. | React gerektirir (bu projede React yok); runtime + peer-deps yük. |
| **@hello-pangea/dnd** | `react-beautiful-dnd` halefi, güzel animasyonlar. | React gerektirir; tek HTML hedefiyle uyuşmaz. |
| **Tarayıcı yerleşik HTML5 DnD** | Sıfır bağımlılık. | **Mobilde çalışmaz** (touch event'leri desteklemez); kötü UX; kendiniz touch shim yazmak zorundasınız. |
| **SortableJS** ✅ | Framework-agnostik, **masaüstü + dokunmatik** tek API, sütunlar arası `group` desteği, ~30 KB gzip. | Animasyon ince ayarı dnd-kit kadar zengin değil. |

Bu projenin en sert kısıtı **“tek HTML dosyası + mobil desteği”** olduğundan **SortableJS** doğal seçimdi.
`delay: 120ms, delayOnTouchOnly: true` kombinasyonu ile mobilde kazara sürüklemeyi, masaüstünde gecikmesiz tepkiyi aynı anda sağlıyoruz.

> Not: `react-beautiful-dnd` artık bakım almadığı için düşünülmedi bile.

### Auth
Tam anlamıyla bir backend yok (proje kapsamı: 48 saat + tek dosya), bu yüzden kullanıcılar `localStorage`'da tutulur. Ancak şifreler düz tutulmaz: **PBKDF2 (SHA-256, 120k iterasyon, kullanıcı başına rastgele salt)** ile hash'lenir. Bu tek dosya bir saldırganın eline geçse bile şifreler tersine döndürülemez.

Gerçek üretimde bu katman yerine Supabase/Firebase/Clerk gibi bir servis kullanılır; API aynı kalacak şekilde kolayca geçirilebilir (tüm veri erişimi `loadData/saveData` ardında).

---

## Veri modeli

```ts
User   { email, name, salt, hash, createdAt }
Board  { id, name, createdAt }
Column { id, boardId, name, order }            // order: fractional index string
Card   { id, columnId, title, desc, labels[],
         due, assignees[], checklist[], activity[], order, createdAt }
```

- Her kullanıcının verisi `tf.data.<email>` anahtarında ayrı tutulur → aynı tarayıcıda birden fazla hesap bağımsız çalışır.
- `BroadcastChannel('taskflow')` → aynı origin'deki sekmeler arasında “bir şey değişti” sinyali; karşı taraf `localStorage`'ı yeniden okur.

---

## Sıralama algoritması (fractional indexing)

**Problem:** Kartlar `order: 0,1,2,3,...` gibi sayılarla sıralansaydı, araya yeni kart eklendiğinde sonraki tüm kartların `order`'ını güncellemek (`O(n)`) ve her birini tekrar kaydetmek gerekirdi. Büyük board'da ağırlaşır ve eşzamanlı çakışmalara yol açar.

**Çözüm:** Her kart, **karakter dizisi** tabanlı bir sıralama anahtarı tutar. İki komşu `a < b` verildiğinde, `keyBetween(a, b)` aralarında yeni bir anahtar üretir:

```
a = "M",  b = "O"   → "N"
a = "M",  b = "N"   → "MV"     (araya girecek alan yoksa bir karakter daha ekle)
a = null, b = "A"   → "05"     (başa koyma)
a = "z",  b = null  → "zV"     (sona koyma)
```

Sonuç: **tek kart güncellenir (`O(1)`)**, yeniden numaralandırma asla gerekmez, sayfa yenileme sonrası sıralama bire bir korunur. Aynı teknik Figma, Linear ve Notion'da da kullanılır.

---

## Neye öncelik verildi, neye verilmedi

### 48 saatte yapılanlar (bilinçli seçimler)
- Sürükle-bırak **mükemmel** çalışsın — masaüstü ve mobil
- Sıralama kalıcılığı **sağlam** bir yöntemle çözülsün (fractional index)
- Kart detayları anlamlı olsun (etiket, tarih, sorumlu, alt görev, geçmiş)
- Erişilebilirlik + klavye desteği unutulmasın
- Karanlık tema + mobil layout **görünür** iş olarak dahil
- **Tek dosya** kısıtına uyumlu tutarlı mimari

### Yapılmadı / yarım bırakılmadı
- **Gerçek zamanlı paylaşım**: Tek HTML + backend yok = farklı cihazlardaki kullanıcılar arasında paylaşım mümkün değil. Aynı tarayıcıdaki çoklu sekme için `BroadcastChannel` ile simüle edildi; gerçek paylaşım için WebSocket/CRDT (Yjs, Automerge) katmanı gerekir. Yarı çalışan bir paylaşımın olmaması, çalışan bir simülasyondan daha dürüst bulundu.
- **Sanal liste (virtualization)**: 1000+ kart senaryosunda sürükle-bırak yavaşlayabilir. Normal kullanım (board başına < 300 kart) için gereksiz; eklemek SortableJS ile non-trivial olduğundan dışarıda bırakıldı.
- **Çoklu fotoğraf ekleri**: `localStorage` 5–10 MB ile sınırlı, base64 görseller bu kotayı hızla doldurur. Doğru çözüm S3/Cloudinary + URL → backend gerekli.

Kısa versiyon: **“çalışan az özellik > yarım çok özellik”** kuralına sadık kalındı.

---

## Klavye kısayolları

| Tuş | Eylem |
|---|---|
| `/` | Arama kutusuna odaklan |
| `N` | İlk sütuna yeni kart ekle |
| `?` | Kısayol rehberini aç |
| `Esc` | Açık modal'ı kapat |
| `Enter` | Kart açıklamasında yeni satır, başlıkta kaydet |

---

## Deploy

```bash
# Vercel
npx vercel --prod            # root'tan; sıfır config

# ya da:
# - GitHub Pages: repo'ya push + Pages aç
# - Netlify: index.html'i drag-drop
```

Herhangi bir build adımı, env değişkeni, veya backend yok.

---

## Dosya yapısı

```
.
├── index.html     # tüm uygulama (HTML + CSS + JS)
└── README.md      # bu dosya
```

Tek harici bağımlılık: `SortableJS` (jsDelivr CDN üzerinden; offline çalışma gerekirse dosyayı yerelleştirmek kolay).
