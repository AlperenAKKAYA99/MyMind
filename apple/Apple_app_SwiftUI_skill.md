---
name: apple-iphone-device-compatibility
description: "Swift/SwiftUI/UIKit ile geliştirilen her tür iOS projesinin tüm iPhone modellerinde (Home Button, çentik, Dynamic Island, iPhone Air, katlanabilir iPhone Duo) eksiksiz çalışmasını sağlar. Safe Area, adaptif yerleşim, Dynamic Type, klavye, yatay yön ve karanlık mod hatalarını tespit eder, en küçük güvenli düzeltmeyi uygular, XCTest/XCUITest ile doğrular. Tetikleyiciler: iPhone uyumluluğu, ekran boyutu, Safe Area, çentik, Dynamic Island, taşan arayüz, kesilen metin, klavye kapatıyor, yatay mod, yeni iPhone desteği, cihaz matrisi."
---

# Apple iPhone Cihaz Uyumluluğu Skill'i (Swift · SwiftUI · UIKit)

- **Doğrulama tarihi:** 25 Ekim 2026
- **Kapsam:** iPhone (iPad ve diğer platformlar kapsam dışı; yazılan kod bunlarda da adaptif kalır)
- **Dil:** Türkçe talimat, İngilizce kod/API adları

---

## 0. Altın Kurallar (ajan her işlemden önce okur)

1. **Cihaz modeline değil, gerçek container boyutuna ve kullanılabilir alana göre tasarla.** Model adı, `UIDevice.current.model` veya ekran yüksekliği ile `if` kurma.
2. **Safe Area değerlerini asla koda yazma** (47, 59, 62 vb.). Sistemden oku: `safeAreaInsets`, `safeAreaInset(edge:)`, `safeAreaLayoutGuide`.
3. **`UIScreen.main.bounds` yasak.** iOS 16'dan beri kullanımdan kaldırıldı; Split View, sheet, çoklu ekran ve iPhone Duo'da yanlış sonuç verir. Gerçek `view.bounds` / `GeometryProxy` / size class kullan.
4. **Metin her zaman Dynamic Type ile ölçeklenir.** `.font(.system(size:))` yerine text style; sabit yükseklik yok.
5. **Önce oku, sonra değiştir.** Proje incelenmeden tek satır değiştirilmez. Çalışan özelliği bozma, kapsam dışı refactoring yapma.
6. **Derlenmek, uyumlu olmak değildir.** Görsel doğrulama (matris + yön + yazı boyutu + klavye + tema) yapılmadan iş bitmiş sayılmaz.
7. **Doğrulanamayan cihaz verisini gerçekmiş gibi sunma.** Matriste `M` (ölçülmeli) işaretli değerler simülatörde `layoutProbe` ile ölçülür.
8. **Her düzeltme en küçük güvenli değişiklik olmalı.** Mimariyi, isimlendirmeyi, bağımlılıkları değiştirme.

---

## 1. Amaç ve Kapsam

Bu skill, mevcut veya yeni bir Apple projesinde şunları yapar:

- Cihaz uyumluluğunu bozan kırılgan varsayımları (sabit ölçü, Safe Area hatası, model kontrolü) **tespit eder**.
- Ekran boyutuna, yöne, yazı boyutuna, klavyeye ve görünüme **adaptif** çözümler uygular.
- Sonucu cihaz matrisi üzerinde **otomatik ve görsel olarak doğrular**, raporlar.

Proje türünden bağımsızdır: form/CRUD, liste/feed, e-ticaret, sohbet, medya, kamera, oyun, harita, dashboard, onboarding/paywall, widget/Live Activity, WebView/hibrit (bkz. Bölüm 5).

**Kapsam dışı:** Özellik geliştirme, iş mantığı refactoring'i, tasarım sistemi değişikliği (kullanıcı istemedikçe), üçüncü taraf kütüphane ekleme.

---

## 2. Kullanım Koşulları ve Tetikleyiciler

Skill şu durumlarda devreye girer:

| Tetikleyici | Örnek |
|---|---|
| Uyumluluk talebi | "Tüm iPhone'larda düzgün çalışsın", "yeni iPhone'u destekle" |
| Görsel hata | Taşan/kesilen metin, butonun home indicator altında kalması, çentik/Island çakışması |
| Yön/klavye/yazı | Yatay modda bozulma, klavye alanı kapatıyor, büyük yazıda dağılma |
| Yeni cihaz | Yeni iPhone ailesi veya form faktörü duyuruldu |
| Kod incelemesi | PR'da `UIScreen.main`, sabit `frame`, model kontrolü görüldü |
| Yeni proje | Sıfırdan projeye adaptif iskelet kurma |

**Kullanma:** Sadece iş mantığı hatası, ağ/veri katmanı, App Store metadata işleri.

---

## 3. Teknik Ön Koşullar

Ajan, işe başlamadan şunları tespit edip rapora yazar:

```bash
# Xcode, SDK ve simülatörler
xcodebuild -version
xcodebuild -showsdks | grep -i iphone
xcrun simctl list runtimes
xcrun simctl list devices available | grep -i iphone

# Proje yapısı, hedefler, şemalar
xcodebuild -list
grep -R "IPHONEOS_DEPLOYMENT_TARGET" --include=project.pbxproj . | sort -u
grep -R "swift-tools-version" Package.swift 2>/dev/null
grep -R "SWIFT_VERSION" --include=project.pbxproj . | sort -u

# Arayüz teknolojisi
grep -RIl "import SwiftUI" --include='*.swift' . | wc -l
grep -RIl "import UIKit"  --include='*.swift' . | wc -l
find . -name '*.storyboard' -o -name '*.xib' | head
```

**Deployment target kararı:**

| Hedef | Yaklaşım |
|---|---|
| iOS 16+ | Bu skill'deki tüm SwiftUI örnekleri doğrudan kullanılır (`NavigationStack`, `ViewThatFits`, `Layout`, `AnyLayout`) |
| iOS 15 | Bölüm 6.9'daki geri uyumluluk tablosunu kullan |
| iOS 17+/18+/26+ | İlgili API'yi yalnızca `if #available` ile ekle; temel çözümü eski sürümle de çalışır tut |

**API'lerin minimum sürümü (bu skill'de kullanılanlar):**

| API | Min iOS |
|---|---|
| `safeAreaInset(edge:)`, `@Environment(\.dynamicTypeSize)` | 15 |
| `keyboardLayoutGuide` (UIKit) | 15 |
| `NavigationStack`, `ViewThatFits`, `Layout`, `AnyLayout`, `scrollDismissesKeyboard`, `presentationDetents`, `UIHostingConfiguration`, `UIHostingController.sizingOptions` | 16 |
| `UIHostingController.safeAreaRegions` | 16.4 |
| `containerRelativeFrame`, `safeAreaPadding`, `contentMargins`, `registerForTraitChanges` | 17 |
| `onGeometryChange` | 16 (geri uyumlu; Xcode 16+ SDK ile) |

> iOS 26 SDK ile derlenen uygulamalar sistem çubuklarında yeni cam (Liquid Glass) görünümünü alır. Özel bar arka planları ve sabit üst/alt boşluklar bu görünümle çakışabilir; çubuk arka planlarını sisteme bırak. iOS 26/27'ye özgü yeni API'leri kullanmadan önce güncel Apple belgesini doğrula (Bölüm 17).

---

## 4. Doğrulanmış iPhone Cihaz Matrisi

### 4.1 Doğrulama kodları ve okuma kuralları

| Kod | Anlamı |
|---|---|
| **A** | Apple resmî teknik özellik sayfası (piksel çözünürlüğü, ppi) — 17e ve 18 Pro/Pro Max için doğrulandı |
| **U** | Apple SDK ölçümlerini yayımlayan Use Your Loaf (Keith Harrison) — point, ölçek, Safe Area, boyut sınıfı |
| **T** | Üçüncü taraf türetim — Apple point değerini doğrudan yayımlamamış |
| **M** | Önceki SDK bilgisi veya beklenen değer — **simülatörde `layoutProbe` ile ölç** |

Kurallar:

- **Point = piksel ÷ ölçek.** Mantıksal çözünürlük `GeometryProxy.size` / `view.bounds` ile okunur; tablodaki değerler yalnızca test ve bilgi içindir, koda yazılmaz.
- **Safe Area değerleri sabit değildir.** Cihaz, yön, sistem arayüzü ve iOS sürümüne göre değişir. Yatayda sol/sağ inset aynı raporlanır (simetri için).
- Portrede tüm iPhone'larda genişlik sınıfı **Compact**, yükseklik sınıfı **Regular**'dır. Yatay sınıf tablodaki "Yatay SC" sütunundadır (C = Compact, R = Regular; `genişlik × yükseklik`).
- 8 Plus, 12 mini/13 mini ve bazı modellerde panel pikseli ile render pikseli farklıdır (downsampling). Tablo, UIKit'in raporladığı mantıksal değerleri esas alır.

### 4.2 Ana matris

| Grup | Modeller | Mantıksal (pt) | Fiziksel (px) | Ölçek | h/w oranı | Ekran tipi | Safe Area dikey (üst / alt) | Safe Area yatay (sol=sağ / üst / alt) | Yatay SC | Doğr. |
|---|---|---|---|---|---|---|---|---|---|---|
| Klasik 4.7" | iPhone 8, SE (2. nesil), SE (3. nesil) | 375×667 | 750×1334 | @2x | 1.78 | Home Button | 20 / 0 | 0 / 0 / 0 | C×C | U (pt), M (inset) |
| Klasik 5.5" | iPhone 8 Plus | 414×736 | 1242×2208 (render; panel 1080×1920) | @3x | 1.78 | Home Button | 20 / 0 | 0 / 0 / 0 | R×C | U (pt), M (inset) |
| Klasik düz kenar 6.1" | iPhone 16e, iPhone 17e | 390×844 | 1170×2532 | @3x | 2.16 | Çentikli | 47 / 34 | 47 / 0 / 21 | C×C | A (px, 17e), U (pt, 16e), M (inset) |
| Çentikli kompakt | iPhone X, XS, 11 Pro | 375×812 | 1125×2436 | @3x | 2.16 | Çentikli | 44 / 34 | 44 / 0 / 21 | C×C | U (pt), M (inset) |
| Çentikli kompakt | iPhone 12 mini, 13 mini | 375×812 | 1080×2340 (panel; render 1125×2436) | @3x | 2.16 | Çentikli | 50 / 34 | 50 / 0 / 21 | C×C | U |
| Çentikli büyük | iPhone XS Max, 11 Pro Max | 414×896 | 1242×2688 | @3x | 2.16 | Çentikli | 44 / 34 | 44 / 0 / 21 | R×C | U (pt), M (inset) |
| Çentikli standart | iPhone XR, 11 | 414×896 | 828×1792 | @2x | 2.16 | Çentikli | 48 / 34 | 48 / 0 / 21 | R×C | U (pt), M (inset) |
| Düz kenar standart | iPhone 12, 12 Pro, 13, 13 Pro, 14 | 390×844 | 1170×2532 | @3x | 2.16 | Çentikli | 47 / 34 | 47 / 0 / 21 | C×C | U (13), M (14) |
| Düz kenar büyük | iPhone 12 Pro Max, 13 Pro Max, 14 Plus | 428×926 | 1284×2778 | @3x | 2.16 | Çentikli | 47 / 34 | 47 / 0 / 21 | R×C | U (pt, 13 Pro Max), M (14 Plus inset) |
| Dynamic Island 6.1" | iPhone 14 Pro, 15, 15 Pro, 16 | 393×852 | 1179×2556 | @3x | 2.17 | Dynamic Island | 59 / 34 | 59 / 0 / 21 | C×C | U (16), M (14 Pro, 15 ailesi inset) |
| Dynamic Island 6.7" | iPhone 14 Pro Max, 15 Plus, 15 Pro Max, 16 Plus | 430×932 | 1290×2796 | @3x | 2.17 | Dynamic Island | 59 / 34 | 59 / 0 / 21 | R×C | U (16 Plus), M (diğerleri inset) |
| Dynamic Island 6.3" | iPhone 16 Pro | 402×874 | 1206×2622 | @3x | 2.17 | Dynamic Island | 62 / 34 | 62 / 0 / 21 | C×C | U |
| Dynamic Island 6.9" | iPhone 16 Pro Max | 440×956 | 1320×2868 | @3x | 2.17 | Dynamic Island | 62 / 34 | 62 / 0 / 21 | R×C | U |
| Yeni nesil 6.3" | iPhone 17, iPhone 17 Pro | 402×874 | 1206×2622 | @3x | 2.17 | Dynamic Island | 62 / 34 | 62 / **20** / **20** | C×C | U |
| Yeni nesil 6.3" | **iPhone 18 Pro** | 402×874 | 1206×2622 | @3x | 2.17 | Dynamic Island (küçültülmüş) | **M** | **M** | C×C (beklenen) | A (px); pt = px÷3; inset M |
| Yeni nesil 6.9" | iPhone 17 Pro Max | 440×956 | 1320×2868 | @3x | 2.17 | Dynamic Island | 62 / 34 | 62 / **20** / **20** | R×C | U |
| Yeni nesil 6.9" | **iPhone 18 Pro Max** | 440×956 | 1320×2868 | @3x | 2.17 | Dynamic Island (küçültülmüş) | **M** | **M** | R×C (beklenen) | A (px); pt = px÷3; inset M |
| Yeni nesil 6.5" | iPhone Air | 420×912 | 1260×2736 | @3x | 2.17 | Dynamic Island | **68** / 34 | 68 / **20** / **29** | R×C | U |

**Önemli değişiklikler (ajan bunları bilerek kod yazar):**

- **iPhone 17 ailesi ve Air'de yatay yönde üst inset artık 20 pt** (iPhone 16 ailesinde 0). Yatayda "üst inset sıfırdır" varsayımı yapan kod bozulur. Air'de yatay alt inset 29 pt'tir.
- Üst inset kademeleri: 47 → 59 → 62 → 68 pt. Bu değerlerin hiçbiri koda yazılmaz.
- **iPhone 18 Pro/Pro Max:** Apple, daha küçük Dynamic Island duyurdu ve üç eşzamanlı Live Activity desteğini belirtti. Ekran piksel çözünürlüğü 17 Pro/Pro Max ile aynıdır (A). Safe Area değerleri bu çalışmada doğrulanmadı → `M`. Dynamic Island boyutunu varsaymayan kod zaten uyumludur.
- **iPhone 17e:** 6.1", 2532×1170 piksel, 460 ppi (A). 16e ile aynı mantıksal boyut (390×844).

### 4.3 Alternatif ekran biçimi: katlanabilir iPhone Duo (ayrı kategori)

Apple, Eylül 2026 etkinliğinde katlanabilir **iPhone Duo**'yu duyurdu. İki ekranı vardır; uygulama ekranlar arasında geçiş yapar (scene/pencere modeli).

| Ekran | Fiziksel (px) | ppi | Mantıksal (pt, @3x) | Boyut sınıfı | Doğr. |
|---|---|---|---|---|---|
| Dış ekran 5.4" | 1398×2034 | 460 | **466×678** | Dikey C×R · Yatay C×C | T (pt türetilmiş) |
| İç ekran 7.6" | 1878×2670 | 430 | **669×951** | **R×R (her yönde)** | T (pt türetilmiş) |

- Apple'ın bu cihaz için doğrudan yayımladığı point değeri bu çalışmada bulunamadı; point değerleri App Store Connect ekran görüntüsü boyutlarından (dış 1398×2034, iç 2007×2853) @3x ile türetilmiştir → **koda yazma, runtime'da ölç.**
- İç ekranda yatay genişlik sınıfı da Regular'dır; yani iPad benzeri çok sütunlu düzen (`NavigationSplitView`) devreye girebilir. Dış ekranda çubuklar yana taşınabilir.
- Kural: **idiom'a (`.phone`/`.pad`) değil, size class ve gerçek container boyutuna göre dallan.** İç ekranın iPad idiom'u olduğunu varsayma.
- Yön kilidi (`supportedInterfaceOrientations`) iç ekranda farklı davranabilir; yön kilitleyen uygulamalarda iç ekran ayrıca test edilir.
- Simülatör: Xcode 27 ile "iPhone Duo" adlı cihaz bulunur (`xcrun simctl list devices available` ile doğrula).
- Duo'ya özgü API'ler (çoklu ekran/scene, kamera) için Apple'ın "Design for iPhone Duo" HIG bölümü ve Tech Talk'larını oku (Bölüm 17); API adlarını ezberden yazma.

### 4.4 Matrise eklenmeyecek modeller

Resmî teknik belge yayımlanmayan modeller (ör. standart iPhone 18, iPhone 18e, iPhone Air 2) **matrise eklenmez ve mevcutmuş gibi anılmaz.** Yeni model duyurulduğunda Bölüm 4.5'teki prosedür uygulanır.

### 4.5 Yeni cihaz eklendiğinde prosedür

1. Apple'ın o modelin Teknik Özellikler sayfasından piksel çözünürlüğü ve ppi'yi al.
2. `mantıksal = piksel ÷ ölçek` (ölçek = Apple'ın SDK'da raporladığı değer; çoğu iPhone için 3).
3. Güncel Xcode'da simülatörü oluştur, `layoutProbe` (Bölüm 6.6) ile **gerçek** Safe Area ve boyut sınıflarını ölç.
4. Bu dosyadaki tabloya satırı `Doğr.` kodlarıyla ekle; ölçülemeyen alanı `M` bırak.
5. Yeni bir **ekran biçimi** (Island dışında cutout, farklı oran, katlanır) çıkarsa Bölüm 4.3 gibi ayrı kategori aç.

### 4.6 Minimum doğrulama seti (her değişiklikte)

Her benzersiz mantıksal boyut ve ekran tipini kapsayan en küçük set:

| Rol | Cihaz (simülatör adı) | Neden |
|---|---|---|
| En küçük / Home Button | iPhone SE (3rd generation) | 375×667, @2x, Safe Area üst 20, alt 0 |
| Kompakt çentik | iPhone 13 mini | 375×812, üst inset 50 |
| @2x çentik | iPhone 11 | 414×896, @2x |
| Standart çentik | iPhone 17e (veya 14) | 390×844 |
| Standart Island | iPhone 16 | 393×852, üst 59 |
| Yeni standart | iPhone 17 Pro (veya 18 Pro) | 402×874, yatay üst inset 20 |
| Yeni orta büyük | iPhone Air | 420×912, üst 68, yatay alt 29 |
| Büyük Island | iPhone 16 Plus | 430×932 |
| En büyük | iPhone 17 Pro Max (veya 18 Pro Max) | 440×956, yatay Regular genişlik |
| Katlanabilir (varsa) | iPhone Duo dış + iç | Farklı oran, R×R iç ekran |

Mevcut olmayan simülatör adı atlanır ve raporda belirtilir; en yakın benzersiz boyutlu cihazla ikame edilir.

---

## 5. Proje Türüne Göre Oyun Kitabı

Uyumluluk kuralları aynıdır; türe göre **kritik riskler** değişir.

| Proje türü | Kritik riskler | Yaklaşım |
|---|---|---|
| Form / CRUD / ayarlar | Klavye alanı kapatır, uzun etiketler, Dynamic Type | `ScrollView` + `scrollDismissesKeyboard` + alt aksiyonu `safeAreaInset(edge: .bottom)`; etiketleri `AnyLayout` ile dikeye çevir |
| Liste / feed / sosyal | Hücre yüksekliği sabit, büyük yazıda kesilme, büyük ekranda çok geniş satır | Self-sizing hücre/`List`, okunabilir genişlik sınırı, `LazyVGrid(.adaptive)` |
| E-ticaret / katalog | Ürün gridinde sabit sütun, görsel oranı, sticky "Sepete ekle" çubuğu | Adaptif grid, `aspectRatio`, sticky çubuk `safeAreaInset` ile |
| Sohbet / mesajlaşma | Klavye + giriş çubuğu + home indicator | `keyboardLayoutGuide` (UIKit) / otomatik klavye kaçınma + `safeAreaInset` giriş çubuğu |
| Medya / video oynatıcı | Tam ekran video ile kontrollerin çentik/Island/home indicator altında kalması | Video yüzeyi `ignoresSafeArea`, kontroller Safe Area içinde; `persistentSystemOverlays` ile home indicator yönetimi |
| Kamera | Önizleme oranı ≠ ekran oranı, Island çakışması | Önizleme `AVCaptureVideoPreviewLayer` + `videoGravity`, kontroller Safe Area içinde |
| Oyun (SpriteKit/Metal/SceneKit/RealityKit) | Sahne boyutu cihaza göre değişir, HUD çentik/Island altında kalır | Render yüzeyi tam ekran; HUD'ı Safe Area'ya sabitle; sahne ölçeğini `scaleMode`/viewport ile ayarla, sabit piksel kullanma |
| Harita | Kontrol/arama çubuğu Island altında, alt sheet | Harita tam ekran, kontroller `safeAreaInset`; sheet için `presentationDetents` |
| Dashboard / grafik | Küçük ekranda taşan grafikler, büyük ekranda boşluk | `ViewThatFits`, grafik minimum genişlik + yatay kaydırma, `containerRelativeFrame` |
| Onboarding / paywall | Tek sayfa sabit yükseklik, CTA home indicator altında | `ScrollView` + sabit alt CTA (`safeAreaInset`), küçük ekranda kaydırılabilir |
| Widget / Live Activity / Dynamic Island | Island boyutu varsayımı, 3 eşzamanlı aktivite | Sistem bölgeleri (compact/expanded/minimal) kullan; boyut varsayma; `#Preview(as: .dynamicIsland(...))` ile dene |
| WebView / hibrit | CSS safe-area, viewport | `viewport-fit=cover` + `env(safe-area-inset-*)`; `WKWebView` `contentInsetAdjustmentBehavior` |

---

## 6. SwiftUI Kuralları

### 6.1 Ekran iskeleti (okunabilir genişlik + klavye + alt aksiyon)

Deployment target: iOS 16+.

```swift
import SwiftUI

/// Kaydırılabilir, okunabilir genişlikte, klavye dostu, alt aksiyonlu ekran iskeleti.
/// Sabit cihaz ölçüsü içermez; 680 pt bir tasarım üst sınırıdır (cihaz modeli değil).
struct AdaptiveScreen<Content: View, Footer: View>: View {
    private let content: Content
    private let footer: Footer

    init(@ViewBuilder content: () -> Content,
         @ViewBuilder footer: () -> Footer) {
        self.content = content()
        self.footer = footer()
    }

    var body: some View {
        ScrollView {
            content
                .frame(maxWidth: 680, alignment: .leading)
                .frame(maxWidth: .infinity)
                .padding(.horizontal, 16)
                .padding(.vertical, 16)
        }
        .scrollDismissesKeyboard(.interactively)
        .safeAreaInset(edge: .bottom, spacing: 0) {
            footer
                .frame(maxWidth: 680)
                .frame(maxWidth: .infinity)
                .padding(.horizontal, 16)
                .padding(.vertical, 12)
                .background(.bar)
        }
    }
}

extension AdaptiveScreen where Footer == EmptyView {
    init(@ViewBuilder content: () -> Content) {
        self.init(content: content, footer: { EmptyView() })
    }
}
```

### 6.2 Dynamic Type'a duyarlı satır (yatay ↔ dikey)

```swift
import SwiftUI

struct AdaptiveRow<Leading: View, Trailing: View>: View {
    @Environment(\.dynamicTypeSize) private var typeSize
    private let leading: Leading
    private let trailing: Trailing

    init(@ViewBuilder leading: () -> Leading,
         @ViewBuilder trailing: () -> Trailing) {
        self.leading = leading()
        self.trailing = trailing()
    }

    var body: some View {
        let layout: AnyLayout = typeSize.isAccessibilitySize
            ? AnyLayout(VStackLayout(alignment: .leading, spacing: 8))
            : AnyLayout(HStackLayout(alignment: .firstTextBaseline, spacing: 12))

        layout {
            leading
            trailing
        }
    }
}
```

### 6.3 Sığmazsa alternatif düzen: `ViewThatFits`

```swift
ViewThatFits(in: .horizontal) {
    HStack(spacing: 12) { icon; title; Spacer(minLength: 0); actionButton }   // sığarsa
    VStack(alignment: .leading, spacing: 8) { HStack { icon; title }; actionButton } // sığmazsa
}
```

### 6.4 Adaptif grid ve ölçeklenen boyutlar

```swift
import SwiftUI

struct CardGrid<Item: Identifiable, Card: View>: View {
    let items: [Item]
    @ViewBuilder let card: (Item) -> Card
    @ScaledMetric(relativeTo: .body) private var minCardWidth: CGFloat = 160

    var body: some View {
        LazyVGrid(
            columns: [GridItem(.adaptive(minimum: minCardWidth, maximum: 360), spacing: 16)],
            spacing: 16
        ) {
            ForEach(items) { card($0) }
        }
    }
}
```

### 6.5 Safe Area kullanımı

| Durum | Doğru kullanım |
|---|---|
| Arka plan/görsel kenardan kenara | `.background { Color.x.ignoresSafeArea() }` — içerik Safe Area içinde kalır |
| Tam ekran yüzey (video, oyun, harita) | Yüzeye `.ignoresSafeArea()`; kontrolleri ayrı katmanda Safe Area içinde tut |
| Alt sabit çubuk / CTA | `.safeAreaInset(edge: .bottom)` (home indicator ve klavyeyle birlikte hareket eder) |
| Üst sabit çubuk | `.safeAreaInset(edge: .top)` |
| Klavye kaçınmayı kapatma (nadir) | `.ignoresSafeArea(.keyboard)` yalnızca tam ekran yüzeyde |
| Home indicator'ı gizleme | `.persistentSystemOverlays(.hidden)` (iOS 16) — yalnızca gerekçeli (oyun/video) |
| Sheet / modal | Kendi Safe Area'sı vardır; `presentationDetents` ile boyutlandır, içeriği `ScrollView`'a al |

**Yasak:** `ignoresSafeArea()` uygulanmış metin/buton; `.padding(.top, 47)` gibi sabit inset.

### 6.6 Teşhis aracı: `layoutProbe` (yalnızca DEBUG)

Gerçek Safe Area ve boyut sınıflarını UI testlerinin okuyabileceği biçimde dışa vurur. Matristeki `M` değerlerini ölçmek için kullanılır.

```swift
import SwiftUI

#if DEBUG
struct LayoutProbe: ViewModifier {
    @Environment(\.horizontalSizeClass) private var hSize
    @Environment(\.verticalSizeClass) private var vSize

    func body(content: Content) -> some View {
        content.overlay(alignment: .topLeading) {
            GeometryReader { proxy in
                Color.clear
                    .accessibilityElement()
                    .accessibilityIdentifier("layout.probe")
                    .accessibilityLabel("layout probe")
                    .accessibilityValue(Self.describe(proxy, hSize, vSize))
            }
            .allowsHitTesting(false)
        }
    }

    private static func name(_ c: UserInterfaceSizeClass?) -> String {
        switch c {
        case .compact: return "compact"
        case .regular: return "regular"
        default: return "unspecified"
        }
    }

    private static func describe(_ p: GeometryProxy,
                                 _ h: UserInterfaceSizeClass?,
                                 _ v: UserInterfaceSizeClass?) -> String {
        let i = p.safeAreaInsets
        return String(
            format: "w=%.1f;h=%.1f;top=%.1f;leading=%.1f;bottom=%.1f;trailing=%.1f;hsc=%@;vsc=%@",
            p.size.width, p.size.height, i.top, i.leading, i.bottom, i.trailing,
            name(h), name(v)
        )
    }
}
#endif

extension View {
    /// DEBUG derlemesinde teşhis bilgisi ekler; Release'te hiçbir şey yapmaz.
    @ViewBuilder func layoutProbe() -> some View {
        #if DEBUG
        modifier(LayoutProbe())
        #else
        self
        #endif
    }
}
```

Kullanım: kök görünüme bir kez `.layoutProbe()` ekle (yalnızca UI test hedefi kullanıyorsa `ProcessInfo.processInfo.arguments.contains("-UITestLayoutProbe")` ile koşullandır).

### 6.7 `GeometryReader` kuralları

- Yaprak düzeyde, **gerçekten container boyutuna ihtiyaç duyan** görsel öğede kullan. Tüm ekranı `GeometryReader` ile sarma (kaydırma ve boyutlandırma bozulur).
- iOS 17+: önce `containerRelativeFrame` düşün. iOS 16+ (Xcode 16 SDK): `onGeometryChange` ile boyutu state'e al.
- `GeometryReader` içinde `UIScreen.main` karıştırma.

### 6.8 Yönlendirme ve boyut sınıfı

```swift
@Environment(\.horizontalSizeClass) private var hSize
@Environment(\.verticalSizeClass) private var vSize

var body: some View {
    if hSize == .regular {
        NavigationSplitView { sidebar } detail: { detail }   // Plus/Max yatay, iPhone Duo iç ekran
    } else {
        NavigationStack { sidebar }
    }
}
```

- "Yatay" tespiti için `vSize == .compact` kullan; `UIDevice.current.orientation` kullanma.
- Yön kilidi gerekiyorsa gerekçelendir ve hedef iOS sürümünde API'yi doğrula; varsayılan: telefonda portre + yatay destekle.

### 6.9 iOS 15 geri uyumluluk

| iOS 16+ API | iOS 15 alternatifi |
|---|---|
| `NavigationStack` | `NavigationView` + `.navigationViewStyle(.stack)` |
| `ViewThatFits` | `GeometryReader` ile ölç, `if` ile dallan (yaprak düzeyde) |
| `AnyLayout` | `if/else` ile iki ayrı düzen |
| `scrollDismissesKeyboard` | `UIScrollView.appearance().keyboardDismissMode = .interactive` (kapsamı belgele) |
| `presentationDetents` | Tam ekran sheet + `ScrollView` |

---

## 7. UIKit Kuralları

Deployment target: iOS 15+ (`registerForTraitChanges` iOS 17+).

```swift
import UIKit

final class AdaptiveViewController: UIViewController {
    private let scrollView = UIScrollView()
    private let contentStack = UIStackView()
    private let titleLabel = UILabel()
    private let detailLabel = UILabel()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground

        titleLabel.font = .preferredFont(forTextStyle: .title2)
        titleLabel.adjustsFontForContentSizeCategory = true
        titleLabel.numberOfLines = 0
        detailLabel.font = .preferredFont(forTextStyle: .body)
        detailLabel.adjustsFontForContentSizeCategory = true
        detailLabel.numberOfLines = 0

        contentStack.axis = .vertical
        contentStack.spacing = 16
        contentStack.isLayoutMarginsRelativeArrangement = true
        contentStack.insetsLayoutMarginsFromSafeArea = false   // Safe Area'yı scroll view yönetir
        contentStack.addArrangedSubview(titleLabel)
        contentStack.addArrangedSubview(detailLabel)

        scrollView.keyboardDismissMode = .interactive
        scrollView.translatesAutoresizingMaskIntoConstraints = false
        contentStack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(scrollView)
        scrollView.addSubview(contentStack)

        NSLayoutConstraint.activate([
            // Klavye açıkken alt kenar klavyenin üstüne, kapalıyken Safe Area alt kenarına oturur.
            scrollView.topAnchor.constraint(equalTo: view.topAnchor),
            scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            scrollView.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor),

            contentStack.topAnchor.constraint(equalTo: scrollView.contentLayoutGuide.topAnchor),
            contentStack.bottomAnchor.constraint(equalTo: scrollView.contentLayoutGuide.bottomAnchor),
            contentStack.leadingAnchor.constraint(equalTo: scrollView.contentLayoutGuide.leadingAnchor),
            contentStack.trailingAnchor.constraint(equalTo: scrollView.contentLayoutGuide.trailingAnchor),
            contentStack.widthAnchor.constraint(equalTo: scrollView.frameLayoutGuide.widthAnchor)
        ])

        if #available(iOS 17.0, *) {
            registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) {
                (self: Self, _: UITraitCollection) in
                self.view.setNeedsLayout()
            }
        }
    }

    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        // Okunabilir genişlik sınırı: geniş container'da yanlardan eşit boşluk.
        let side = max(16, (view.bounds.width - 680) / 2)
        contentStack.directionalLayoutMargins = NSDirectionalEdgeInsets(
            top: 16, leading: side, bottom: 16, trailing: side
        )
        // Çentik/Island yatayda yanlardan girer: Safe Area'yı yatay marja ekle.
        let safe = view.safeAreaInsets
        contentStack.directionalLayoutMargins.leading += safe.left
        contentStack.directionalLayoutMargins.trailing += safe.right
    }

    @available(iOS, deprecated: 17.0, message: "iOS 17+ için registerForTraitChanges kullanılır")
    override func traitCollectionDidChange(_ previous: UITraitCollection?) {
        super.traitCollectionDidChange(previous)
        if previous?.preferredContentSizeCategory != traitCollection.preferredContentSizeCategory {
            view.setNeedsLayout()
        }
    }
}
```

**UIKit kuralları:**

1. Kısıtları `view.safeAreaLayoutGuide`, `view.readableContentGuide`, `view.keyboardLayoutGuide` (iOS 15+) üzerinden kur.
2. Sabit `width/height` kısıtı yerine `>=`/`<=` ve düşük öncelikli tercih kısıtı kullan; hücreler self-sizing olsun.
3. `UIFont.preferredFont(forTextStyle:)` + `adjustsFontForContentSizeCategory = true`; özel font için `UIFontMetrics`.
4. `traitCollection.horizontalSizeClass`/`verticalSizeClass` ile düzen değiştir; `view.window?.windowScene?.screen` yalnızca gerçekten ekran bilgisi gerekirse.
5. `viewDidLayoutSubviews` içinde frame hesaplama yerine Auto Layout; kaçınılmazsa `view.bounds` ve `view.safeAreaInsets` kullan.
6. `UIApplication.shared.keyWindow`, `statusBarFrame`, `UIScreen.main` kullanma.
7. `UIButton.Configuration` (iOS 15+) ile Dynamic Type'a uyumlu, çok satırlı butonlar kur.

---

## 8. UIKit ↔ SwiftUI Birlikte Çalışabilirlik

```swift
// SwiftUI içinde UIKit
struct LegacyScreen: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> AdaptiveViewController { AdaptiveViewController() }
    func updateUIViewController(_ vc: AdaptiveViewController, context: Context) {}
}

// UIKit içinde SwiftUI
final class HostViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let host = UIHostingController(rootView: AdaptiveScreen { Text("Merhaba") })
        addChild(host)
        host.view.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(host.view)
        // Kenarlara sabitle: SwiftUI kendi Safe Area'sını hesaplar (çift inset olmasın).
        NSLayoutConstraint.activate([
            host.view.topAnchor.constraint(equalTo: view.topAnchor),
            host.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            host.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            host.view.trailingAnchor.constraint(equalTo: view.trailingAnchor)
        ])
        host.didMove(toParent: self)
    }
}
```

Kurallar:

- Hosting view'ı `safeAreaLayoutGuide`'a sabitliyorsan çift inset oluşabilir; kenarlara sabitle veya iOS 16.4+'ta `host.safeAreaRegions = []` ayarla.
- Hücre içinde SwiftUI: `UIHostingConfiguration` (iOS 16+). Stack içinde: `host.sizingOptions = .intrinsicContentSize` (iOS 16+).
- Klavye ve Safe Area yönetimini **tek katmanda** yap (ya UIKit ya SwiftUI); ikisinde birden çözme.

---

## 9. Safe Area ve Adaptif Layout Standartları

### 9.1 Yasak ↔ Doğru

| Yasak | Sorun | Doğrusu |
|---|---|---|
| `UIScreen.main.bounds` / `.size` | Deprecated (iOS 16); Split View, sheet, Duo'da yanlış | `GeometryProxy`, `view.bounds`, size class |
| `UIDevice.current.model/name` ile düzen | Yeni cihazda kırılır | Size class, `ViewThatFits`, container boyutu |
| `if height == 812` / `screenHeight > 800` | Her yeni modelde güncelleme gerekir | Kullanılabilir alana göre adaptif düzen |
| `.padding(.top, 47)`, `constant: 59` | Safe Area sabit değildir (47/59/62/68) | `safeAreaInset`, `safeAreaLayoutGuide` |
| `.frame(width: 375)` / `width: 390` | Küçük ekranda taşar, büyükte boşluk | `maxWidth`, `minWidth`, `.infinity` |
| `.offset(x:y:)` ile konumlandırma | Layout'u bozmadan kaydırır, çakışma yaratır | `alignment`, `Spacer`, `safeAreaInset` |
| Root'ta tek `GeometryReader` | Boyut/kaydırma sorunları | Yaprakta, ihtiyaca göre |
| `.font(.system(size: 17))` | Dynamic Type'ı yok sayar | `.font(.body)`, `@ScaledMetric` |
| `lineLimit(1)` + sabit yükseklik (kritik metin) | Büyük yazıda kesilir | `lineLimit(nil)` / `.minimumScaleFactor` yalnızca ikincil metinde |
| Her yerde `ignoresSafeArea()` | İçerik çentik/home indicator altında kalır | Yalnızca arka plan/yüzey |
| `UIApplication.shared.keyWindow`, `statusBarFrame` | Deprecated, çoklu pencerede yanlış | Pencere sahnesi / `view.window` |
| Yatay yönde "üst inset = 0" varsayımı | iPhone 17 ailesi/Air yatay üst inset 20 | Daima Safe Area'dan oku |
| Yön kilidi + sabit düzen | Duo iç ekran ve yatay kullanımda bozulur | Adaptif düzen, gerekçeli kilit |

### 9.2 Boşluk ve dokunma standartları

- Minimum dokunma hedefi **44×44 pt** (kenar boşlukları 8 pt ızgarasına oturt: 8/16/24/32).
- Kenar boşluğu: telefonda 16–20 pt; geniş container'da içerik genişliğini ~680 pt ile sınırla.
- Metin bloklarında `readableContentGuide` (UIKit) veya `frame(maxWidth:)` (SwiftUI).
- Renkler anlamsal olsun (`.primary`, `Color(.systemBackground)`); karanlık modda ayrı test et.

### 9.3 Klavye

- SwiftUI: `ScrollView` + `scrollDismissesKeyboard`; alttaki sabit alanı `safeAreaInset(edge: .bottom)` ile ver.
- UIKit: `view.keyboardLayoutGuide` (iOS 15+); manuel `keyboardWillShow` hesabından kaçın.
- Odaklı alanın klavyenin üstünde göründüğünü **UI testiyle** doğrula (Bölüm 13).

### 9.4 Yönlendirme

- Portre ve yatayı birlikte tasarla; yatayda içerik yanlardan Safe Area kadar girer (47–68 pt).
- Plus/Max ve Duo iç ekranda yatay genişlik sınıfı Regular'dır; iki sütunlu düzen mümkün.

---

## 10. Hata Tespit Prosedürü

### 10.1 Statik tarama (salt okunur) — `scripts/layout_audit.sh`

```bash
#!/usr/bin/env bash
# Statik düzen taraması. Dosya değiştirmez. Çıkış kodu: HIGH bulgu varsa 1, yoksa 0.
set -uo pipefail
ROOT="${1:-.}"
EXCL=(--exclude-dir=.git --exclude-dir=.build --exclude-dir=Pods --exclude-dir=Carthage \
      --exclude-dir=DerivedData --exclude-dir=SourcePackages)
high=0
total=0

scan() {
  local sev="$1" label="$2" re="$3" hits n
  hits=$(grep -RInE "${EXCL[@]}" --include='*.swift' -e "$re" "$ROOT" 2>/dev/null || true)
  if [ -n "$hits" ]; then
    n=$(printf '%s\n' "$hits" | wc -l | tr -d ' ')
    total=$((total + n))
    if [ "$sev" = "HIGH" ]; then high=$((high + n)); fi
    printf '\n[%s] %s (%s bulgu)\n%s\n' "$sev" "$label" "$n" "$hits"
  fi
}

scan HIGH "UIScreen.main kullanımı (deprecated, yanlış container)"        'UIScreen\.main'
scan HIGH "keyWindow / statusBarFrame kullanımı (deprecated)"             'UIApplication\.shared\.(keyWindow|statusBarFrame)'
scan HIGH "Cihaz modeli/adı ile düzen kararı"                             'UIDevice\.current\.(model|name)'
scan HIGH "Bilinen ekran boyutuna karşı sabit karşılaştırma"              '(width|height|Width|Height)[^;]{0,24}(==|>=|<=|>|<)[[:space:]]*(568|667|736|812|844|852|874|896|912|926|932|956|375|390|393|402|414|420|428|430|440)([^0-9.]|$)'
scan MED  "Sabit Safe Area değeri olası (üst/alt padding sayıları)"       '\.padding\(\.(top|bottom)[^)]*,[[:space:]]*(20|34|44|47|48|50|59|62|68)\)'
scan MED  "Sabit geniş frame (>=200 pt)"                                  '\.frame\((width|height):[[:space:]]*[0-9]{3,}'
scan MED  "Sabit offset (>=10 pt)"                                        '\.offset\((x|y):[[:space:]]*-?[0-9]{2,}'
scan MED  "Sabit CGRect (genişlik >=100 pt)"                              'CGRect\(x:[[:space:]]*[0-9.]+,[[:space:]]*y:[[:space:]]*[0-9.]+,[[:space:]]*width:[[:space:]]*[0-9]{3,}'
scan MED  "UIApplication.shared.windows kullanımı"                        'UIApplication\.shared\.windows'
scan LOW  "Dynamic Type'ı yok sayan sabit font"                           '\.font\(\.system\(size:'
scan LOW  "ignoresSafeArea kullanımı (içerik mi arka plan mı kontrol et)" 'ignoresSafeArea'
scan LOW  "GeometryReader kullanımı (yaprak düzeyde mi kontrol et)"       'GeometryReader'
scan LOW  "lineLimit(1) (kritik metinde kesilme riski)"                   'lineLimit\(1\)'

printf '\nToplam bulgu: %s (HIGH: %s)\n' "$total" "$high"
[ "$high" -eq 0 ]
```

Tarama yalnızca **aday** üretir; her bulguyu bağlamıyla oku (ör. `ignoresSafeArea` arka planda meşrudur).

### 10.2 Manuel inceleme listesi

- Kök görünümler, ortak bileşenler (buton, kart, hücre, çubuk), navigasyon kabuğu (`NavigationStack`, `TabView`, sheet), klavye alanları, tam ekran yüzeyler.
- Storyboard/XIB: sabit boyut kısıtları, "Safe Area Layout Guide" kapalı olan görünümler, Size Class varyasyonları.
- `Info.plist`: desteklenen yönler, `UIRequiresFullScreen`, `UILaunchScreen`/Launch Storyboard (eski launch screen sabit boyutlu görsel kullanıyorsa letterbox oluşur).
- Asset Catalog: `@2x/@3x` kaynakları, `Preserve Vector Data`, tam ekran görseller.

### 10.3 Önem derecesi

| Seviye | Tanım | Örnek | Aksiyon |
|---|---|---|---|
| **P0 — Engelleyici** | Kritik içerik/aksiyona erişilemiyor | CTA home indicator/klavye altında, buton ekran dışı, çökme | Hemen düzelt |
| **P1 — Yüksek** | Okunamaz/çakışan arayüz | Metin kesik, Island üstüne binen başlık, yatay mod bozuk | Aynı iş içinde düzelt |
| **P2 — Orta** | Tutarsız ama kullanılabilir | Büyük ekranda orantısız boşluk, AX yazı boyutunda dağınıklık | Planla |
| **P3 — Düşük** | Kozmetik/öneri | Hizalama, boşluk ızgarası | Raporla |

---

## 11. Düzeltme Reçeteleri

| Sorun | Kök neden | Minimal düzeltme |
|---|---|---|
| Küçük ekranda buton/kart taşıyor | Sabit genişlik | `.frame(maxWidth: .infinity)` + `minWidth`; içeriği `ScrollView`'a al |
| Büyük ekranda orantısız geniş içerik | Sınırsız genişlik | `.frame(maxWidth: 680)` + ortala (Bölüm 6.1) |
| Alt buton home indicator ile çakışıyor | Sabit alt padding / `ignoresSafeArea` | `safeAreaInset(edge: .bottom)` |
| Başlık Island/çentik altında kalıyor | Sabit üst padding veya `ignoresSafeArea` | Sabit değeri kaldır; Safe Area'ya bırak |
| Yatayda içerik çentiğe giriyor | Yatay Safe Area yok sayıldı | `ignoresSafeArea` kaldır; UIKit'te `safeAreaLayoutGuide` |
| Klavye alanı kapatıyor | Sabit konum/`offset`, `ignoresSafeArea(.keyboard)` | `ScrollView` + `safeAreaInset` / `keyboardLayoutGuide` |
| Büyük yazıda metin kesiliyor | `lineLimit(1)`, sabit yükseklik | `lineLimit(nil)`, `fixedSize(horizontal: false, vertical: true)`, `AnyLayout` ile dikey düzen |
| Hücre yüksekliği sabit | Sabit `rowHeight` | Self-sizing (`UITableView.automaticDimension`, SwiftUI `List`) |
| Tam ekran görsel kesiliyor/letterbox | Eski launch screen veya `aspectRatio(.fit)` | `scaledToFill` + `clipped` + `ignoresSafeArea` (yalnızca görsel katmanı) |
| `TabView`/sheet içinde alt boşluk fazla | Çift Safe Area | İç `ScrollView`'a `safeAreaInset` ekleme, dışarıda yönet |
| Yatayda üst boşluk eksik/fazla | "Üst inset 0" varsayımı | Safe Area'dan oku (17 ailesinde yatay üst 20) |
| Screenshot'larda cihazdan cihaza fark | Sabit ölçü, farklı boyut sınıfı | Matris testi + snapshot baseline (Bölüm 13) |

**Düzeltme kuralları:** Bir seferde tek kök neden; değişikliği `git diff` ile küçük tut; her düzeltmeden sonra matris testini tekrar çalıştır; düzeltme başka bir cihaz grubunu bozuyorsa geri al ve alternatif ara.

---

## 12. Skill İş Akışı (Ajan Adımları)

1. **Analiz:** Bölüm 3 komutlarını çalıştır. Swift sürümü, deployment target, UI teknolojisi (SwiftUI/UIKit/karma) ve simülatör envanterini kaydet.
2. **Envanter:** Ekranları, ortak bileşenleri, navigasyon kabuğunu ve layout yapılarını listele (`find`, `grep`, dosya okuma).
3. **Tespit:** `scripts/layout_audit.sh` çalıştır, ardından Bölüm 10.2'yi manuel uygula. Gerekirse `layoutProbe` ekleyip matris cihazlarında gerçek Safe Area'yı ölç.
4. **Sınıflandır:** Bulguları P0–P3 olarak sırala; her biri için dosya:satır, kök neden, etkilenen cihaz grubu yaz.
5. **Düzelt:** P0 → P1 → P2 sırasıyla, en küçük değişiklikle (Bölüm 11). Mimariyi bozma, kapsam dışına çıkma, bağımlılık ekleme.
6. **Doğrula:** Minimum cihaz setinde (Bölüm 4.6) portre + yatay + Dynamic Type + klavye + açık/koyu tema çalıştır (Bölüm 13).
7. **Test üret:** Eksik XCTest/XCUITest senaryolarını ekle (kritik akışlar, taşma/Safe Area/klavye kontrolleri).
8. **Raporla:** Bölüm 15 şablonuyla değişen dosyaları, kalan riskleri ve test sonuçlarını bildir.

**Akış kuralı:** 1–3. adımlar bitmeden 5. adıma geçilmez. Doğrulama başarısızsa 5'e dön.

---

## 13. Test ve Doğrulama Sistemi

### 13.1 Kapsam matrisi

| Eksen | Değerler |
|---|---|
| Cihaz | Bölüm 4.6 seti |
| Yön | Portre, yatay (sol) — yatay sağ en az bir çentikli/Island cihazda |
| Yazı boyutu | Varsayılan, en küçük (`XS`), `AccessibilityXXXL` |
| Klavye | Kapalı, açık (her metin alanı akışı) |
| Tema | Açık, koyu |
| İçerik | Uzun metin, boş veri, çok öğe, tek öğe, hata durumu |
| iOS sürümü | Desteklenen minimum + güncel |
| Alternatif ekran | iPhone Duo dış + iç (Xcode destekliyorsa) |

### 13.2 Matris çalıştırıcı — `scripts/run_device_matrix.sh`

```bash
#!/usr/bin/env bash
# Kullanım: SCHEME=MyApp WORKSPACE=MyApp.xcworkspace ./scripts/run_device_matrix.sh
#       veya SCHEME=MyApp PROJECT=MyApp.xcodeproj  ./scripts/run_device_matrix.sh
# İsteğe bağlı: UI_TEST_TARGET=MyAppUITests
set -uo pipefail

: "${SCHEME:?SCHEME gerekli}"
if   [ -n "${WORKSPACE:-}" ]; then CONTAINER=(-workspace "$WORKSPACE")
elif [ -n "${PROJECT:-}" ];   then CONTAINER=(-project "$PROJECT")
else echo "WORKSPACE veya PROJECT gerekli" >&2; exit 2; fi

DEVICES=(
  "iPhone SE (3rd generation)"
  "iPhone 13 mini"
  "iPhone 11"
  "iPhone 17e"
  "iPhone 16"
  "iPhone 17 Pro"
  "iPhone Air"
  "iPhone 16 Plus"
  "iPhone 17 Pro Max"
  "iPhone Duo"
)

AVAILABLE="$(xcrun simctl list devices available)"
OUT="build/device-matrix"
mkdir -p "$OUT"
failed=(); skipped=(); passed=()

for name in "${DEVICES[@]}"; do
  if ! printf '%s\n' "$AVAILABLE" | grep -qF "    ${name} ("; then
    skipped+=("$name"); echo "ATLANDI (simülatör yok): $name"; continue
  fi
  slug="$(printf '%s' "$name" | tr -c 'A-Za-z0-9' '_')"
  bundle="$OUT/${slug}.xcresult"
  rm -rf "$bundle"
  only=()
  if [ -n "${UI_TEST_TARGET:-}" ]; then only=(-only-testing:"$UI_TEST_TARGET"); fi

  echo "==> $name"
  if xcodebuild test "${CONTAINER[@]}" -scheme "$SCHEME" \
        -destination "platform=iOS Simulator,name=${name},OS=latest" \
        -resultBundlePath "$bundle" "${only[@]}" >"$OUT/${slug}.log" 2>&1; then
    passed+=("$name")
  else
    failed+=("$name")
  fi
done

echo; echo "GEÇTİ  : ${#passed[@]}";  printf '  %s\n' "${passed[@]:-}"
echo "KALDI  : ${#failed[@]}";  printf '  %s\n' "${failed[@]:-}"
echo "ATLANDI: ${#skipped[@]}"; printf '  %s\n' "${skipped[@]:-}"
echo "Sonuç paketleri: $OUT/*.xcresult (ekran görüntüleri test eklerindedir)"
[ "${#failed[@]}" -eq 0 ]
```

### 13.3 XCUITest örneği (taşma, Safe Area, klavye, ekran görüntüsü)

Kritik görünümlere `accessibilityIdentifier` ver (ör. `"primary.cta"`, `"field.email"`). Kök görünüme `.layoutProbe()` ekle.

```swift
import XCTest

final class DeviceLayoutUITests: XCTestCase {
    private var app: XCUIApplication!

    @MainActor
    override func setUp() async throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments += ["-UITestLayoutProbe", "1"]
    }

    // MARK: Senaryolar

    @MainActor func test_portrait_default() {
        XCUIDevice.shared.orientation = .portrait
        app.launch()
        assertCriticalElementsRespectSafeArea(["primary.cta", "screen.title"])
        attach("portrait-default")
    }

    @MainActor func test_landscape_default() {
        XCUIDevice.shared.orientation = .landscapeLeft
        app.launch()
        assertCriticalElementsRespectSafeArea(["primary.cta", "screen.title"])
        attach("landscape-default")
    }

    @MainActor func test_portrait_accessibilityXXXL() {
        XCUIDevice.shared.orientation = .portrait
        app.launchArguments += ["-UIPreferredContentSizeCategoryName",
                                "UICTContentSizeCategoryAccessibilityXXXL"]
        app.launch()
        // Büyük yazıda içerik kayabilir; kritik aksiyon yine erişilebilir olmalı.
        let cta = app.descendants(matching: .any)["primary.cta"]
        XCTAssertTrue(cta.waitForExistence(timeout: 5))
        if !cta.isHittable { app.swipeUp() }
        XCTAssertTrue(cta.isHittable, "AX XXXL yazıda CTA erişilemiyor")
        attach("portrait-axxxl")
    }

    @MainActor func test_dark_appearance() {
        app.launchArguments += ["-AppleInterfaceStyle", "Dark"]
        app.launch()
        assertCriticalElementsRespectSafeArea(["primary.cta"])
        attach("dark")
    }

    @MainActor func test_keyboard_doesNotCoverFocusedField() {
        app.launch()
        let field = app.textFields["field.email"]
        XCTAssertTrue(field.waitForExistence(timeout: 5))
        field.tap()
        let keyboard = app.keyboards.firstMatch
        XCTAssertTrue(keyboard.waitForExistence(timeout: 5))
        XCTAssertLessThanOrEqual(field.frame.maxY, keyboard.frame.minY + 0.5,
                                 "Klavye odaklı alanı kapatıyor")
        attach("keyboard-open")
    }

    // MARK: Yardımcılar

    @MainActor
    private func assertCriticalElementsRespectSafeArea(
        _ ids: [String], file: StaticString = #filePath, line: UInt = #line
    ) {
        let window = app.windows.firstMatch
        XCTAssertTrue(window.waitForExistence(timeout: 5), file: file, line: line)
        let inset = probeInsets()

        for id in ids {
            let el = app.descendants(matching: .any)[id]
            XCTAssertTrue(el.waitForExistence(timeout: 5), "\(id) bulunamadı", file: file, line: line)
            let f = el.frame
            XCTAssertGreaterThanOrEqual(f.minX, window.frame.minX + inset.leading - 0.5,
                                        "\(id) sol Safe Area dışında", file: file, line: line)
            XCTAssertLessThanOrEqual(f.maxX, window.frame.maxX - inset.trailing + 0.5,
                                     "\(id) sağ Safe Area dışında", file: file, line: line)
            XCTAssertGreaterThanOrEqual(f.minY, window.frame.minY + inset.top - 0.5,
                                        "\(id) üst Safe Area dışında", file: file, line: line)
            XCTAssertLessThanOrEqual(f.maxY, window.frame.maxY - inset.bottom + 0.5,
                                     "\(id) alt Safe Area dışında", file: file, line: line)
        }
    }

    @MainActor
    private func probeInsets() -> (top: CGFloat, leading: CGFloat, bottom: CGFloat, trailing: CGFloat) {
        let probe = app.descendants(matching: .any)["layout.probe"]
        guard probe.waitForExistence(timeout: 5), let raw = probe.value as? String else {
            XCTFail("layout.probe bulunamadı; kök görünüme .layoutProbe() ekle")
            return (0, 0, 0, 0)
        }
        var map: [String: CGFloat] = [:]
        for pair in raw.split(separator: ";") {
            let kv = pair.split(separator: "=")
            if kv.count == 2, let v = Double(kv[1]) { map[String(kv[0])] = CGFloat(v) }
        }
        return (map["top"] ?? 0, map["leading"] ?? 0, map["bottom"] ?? 0, map["trailing"] ?? 0)
    }

    @MainActor
    private func attach(_ name: String) {
        let a = XCTAttachment(screenshot: app.screenshot())
        a.name = name
        a.lifetime = .keepAlways
        add(a)
    }
}
```

Notlar:

- "Safe Area dışında" kontrolü **arka plan/tam ekran yüzey** için uygulanmaz; yalnızca etkileşimli/okunur içerik kimliklerine uygula.
- Ekran görüntüleri `.xcresult` içinde cihaz başına saklanır; baseline karşılaştırması için (a) projede zaten kullanılan snapshot aracını kullan veya (b) görüntüleri `xcresulttool` ile dışa aktarıp önceki koşuyla karşılaştır. Yeni üçüncü taraf bağımlılık yalnızca kullanıcı onayıyla eklenir.
- Dinamik içerik testleri: uzun metin, boş liste ve çok öğeli veriyi launch argümanı veya test fixture'ı ile besle (`-UITestFixture long|empty|many`).

### 13.4 SwiftUI önizleme matrisi (hızlı geri bildirim; nihai doğrulama değildir)

```swift
#Preview("Portrait – küçük (375×667)")  { MyScreen().frame(width: 375, height: 667) }
#Preview("Portrait – büyük (440×956)")  { MyScreen().frame(width: 440, height: 956) }
#Preview("AX XXXL", traits: .sizeThatFitsLayout) {
    MyScreen().dynamicTypeSize(.accessibility5)
}
#Preview("Yatay", traits: .landscapeLeft) { MyScreen() }
#Preview("Koyu") { MyScreen().preferredColorScheme(.dark) }
```

Önizlemedeki `frame` değerleri **yalnızca simülasyondur**; üretim koduna girmez. Gerçek Safe Area için simülatör zorunludur.

### 13.5 Elle kontrol komutları

```bash
xcrun simctl boot "iPhone 17 Pro" 2>/dev/null; open -a Simulator
xcrun simctl ui booted appearance dark
xcrun simctl ui booted content_size accessibility-extra-extra-extra-large
xcrun simctl io booted screenshot ./shots/iphone17pro-dark-axxxl.png
```

### 13.6 Başarı ölçütleri

- Derleme başarılı **ve** matris testleri geçti **ve** P0/P1 bulgu kalmadı.
- Her kritik akış (giriş, ana liste, detay, ödeme/CTA, ayarlar) portre + yatay + AX XXXL + klavye ile görsel olarak gözden geçirildi.
- Sadece derleme/test geçmesi uyumluluk kanıtı sayılmaz; ekran görüntüleri incelenir.

---

## 14. AI Ajanı Çalışma Kuralları

1. **Okuma önceliği:** Dosyayı okumadan düzenleme. Değişiklik öncesi `git status` temiz olmalı; değilse kullanıcıya bildir.
2. **Kapsam disiplini:** Yalnızca cihaz uyumluluğuyla ilgili satırları değiştir. İsim değiştirme, dosya taşıma, stil temizliği yapma.
3. **Koruma:** Çalışan davranışı bozma. Her düzeltmeden sonra derle ve ilgili testi çalıştır.
4. **Sabit sayı yasağı:** Yeni koda cihaz boyutu, Safe Area, status bar/home indicator yüksekliği sabiti yazma. Tasarım boşluğu (8/16/24) ve 44 pt dokunma hedefi serbesttir.
5. **Doğrulanabilirlik:** Cihaz bilgisi verirken kaynağı (A/U/T/M) belirt. Doğrulayamadığını "bilinmiyor/ölçülmeli" olarak yaz; tahmin uydurma.
6. **API güvenliği:** Deprecated API'yi varsayılan çözüm yapma. Yeni API'yi `if #available` ile ekle, eski sürümde çalışan yol bırak.
7. **Bağımlılık:** Üçüncü taraf kütüphane ekleme (kullanıcı isteyene kadar). Vanilla Swift/Apple framework'leri kullan.
8. **Tasarım dili:** Projenin mevcut tasarım diline uy. Yeni görsel dil dayatma.
9. **Küçük adım:** Bir düzeltme = bir kök neden = küçük diff. Büyük refactoring gerekiyorsa uygulamadan önce kullanıcıdan onay iste.
10. **Raporlama dürüstlüğü:** Çalıştırılmayan testi "geçti" diye yazma. Çalıştırılamayanı ve nedenini belirt (simülatör yok, imzalama vb.).
11. **Gizlilik/güvenlik:** `layoutProbe` ve test argümanları yalnızca DEBUG/Test hedefinde; Release'e sızmasın.
12. **Uzun görevler:** Her aşamada özet çıkar (bulgu sayısı, düzeltilen, kalan); yarım bırakılan iş varsa durumunu yaz.

---

## 15. Rapor Şablonu

```markdown
# iPhone Uyumluluk Raporu — <Proje> — <Tarih>

## Ortam
Xcode: … · Swift: … · Deployment target: … · UI: SwiftUI/UIKit/Karma · Simülatörler: …

## Bulgular
| # | Seviye | Dosya:Satır | Sorun | Kök neden | Etkilenen gruplar |
|---|---|---|---|---|---|

## Yapılan değişiklikler
| Dosya | Değişiklik | Neden |
|---|---|---|

## Test sonuçları
| Cihaz | Yön | Yazı | Klavye | Tema | Sonuç | Not |
|---|---|---|---|---|---|---|

## Kalan riskler / çalıştırılamayanlar
- …

## Sonraki adımlar
- …
```

---

## 16. Kabul Kriterleri ve Tamamlanma Kontrol Listesi

**Kod**
- [ ] `layout_audit.sh` çıktısında HIGH bulgu yok; MED/LOW bulgular gerekçelendirildi.
- [ ] Yeni/değişen kodda cihaz modeli kontrolü, `UIScreen.main`, sabit Safe Area değeri yok.
- [ ] Tüm metinler Dynamic Type ile ölçekleniyor; AX XXXL'de kritik aksiyon erişilebilir.
- [ ] Tam ekran yüzeyler dışında içerik Safe Area içinde; alt çubuklar `safeAreaInset`/`safeAreaLayoutGuide` ile.
- [ ] Klavye açıkken odaklı alan ve alt aksiyon görünür.
- [ ] Portre ve yatay çalışıyor; yatayda 17 ailesi/Air üst inset (20) ve Air alt inset (29) bozmuyor.
- [ ] Geniş container'da içerik genişliği sınırlı veya adaptif (grid/split view).
- [ ] Yeni API'ler `#available` ile; minimum iOS'ta derleniyor ve çalışıyor.
- [ ] Yeni bağımlılık eklenmedi (veya kullanıcı onayladı).

**Test**
- [ ] Bölüm 4.6 cihaz setinde matris testi çalıştı (atlananlar raporlandı).
- [ ] Portre/yatay, varsayılan/AX XXXL, klavye açık/kapalı, açık/koyu kapsandı.
- [ ] Uzun metin, boş veri ve çok öğeli içerik kontrol edildi.
- [ ] Desteklenen minimum ve güncel iOS sürümünde çalıştırıldı (veya yapılamadığı belirtildi).
- [ ] Ekran görüntüleri gözden geçirildi (yalnızca derleme başarısı kabul edilmez).
- [ ] Duo veya yeni form faktörü destekleniyorsa dış + iç ekran test edildi.

**Rapor**
- [ ] Bölüm 15 şablonu dolduruldu; değişen dosyalar, kalan riskler, çalıştırılamayan testler açıkça yazıldı.
- [ ] Cihaz verileri için kaynak kodları (A/U/T/M) belirtildi; ölçülen yeni değerler Bölüm 4 tablosuna işlendi.

---

## 17. Güvenilir Teknik Kaynaklar ve Doğrulama Notları

**Resmî (öncelikli):**

- Apple HIG — Layout / Layout and organization: https://developer.apple.com/design/human-interface-guidelines/layout
- Apple HIG — iPhone Duo tasarım bölümü ve Tech Talk'lar: "Design for iPhone Duo", "Leverage multiple displays and scenes on iPhone Duo", "Build a great camera experience for iPhone Duo" (https://developer.apple.com/videos/play/tech-talks/111465)
- Apple iPhone teknik özellikleri: https://www.apple.com/iphone-18-pro/specs/ · https://www.apple.com/iphone-17e/specs/ (her model için `apple.com/<model>/specs/`)
- Apple Developer Documentation: SwiftUI (`GeometryProxy`, `safeAreaInset`, `ViewThatFits`, `Layout`, `AnyLayout`), UIKit (`safeAreaLayoutGuide`, `keyboardLayoutGuide`, `UITraitCollection`), XCTest/XCUITest
- Xcode simülatör envanteri: `xcrun simctl list devices available`

**Topluluk ölçümleri (SDK ölçümlerinden):**

- Use Your Loaf — iPhone 17 / 16 / 13 Screen Sizes: https://useyourloaf.com/blog/iphone-17-screen-sizes/ · https://useyourloaf.com/blog/iphone-16-screen-sizes/ · https://useyourloaf.com/blog/iphone-13-screen-sizes/

**Doğrulama günlüğü (25 Ekim 2026) — açık belirsizlikler:**

| Konu | Durum |
|---|---|
| iPhone 17e, 18 Pro, 18 Pro Max piksel çözünürlüğü | Apple teknik özellik sayfalarından doğrulandı |
| 17/Air/16/13 ailesi point, ölçek, Safe Area, boyut sınıfı | Use Your Loaf ölçümlerinden doğrulandı |
| X/XS/XR/11, SE/8, 14 ve 15 ailesi Safe Area; 16e/17e inset | Doğrudan kaynakla bu çalışmada doğrulanmadı → `M`, `layoutProbe` ile ölç |
| iPhone 18 Pro/Pro Max Safe Area (küçültülmüş Dynamic Island) | Bu çalışmada bulunamadı → `M` |
| iPhone Duo point değerleri (466×678, 669×951) | Apple doğrudan yayımlamadı; App Store Connect ekran görüntüsü boyutlarından türetilmiş → runtime'da ölç |
| iOS 26/27 yeni layout API'leri ve Liquid Glass opt-out anahtarları | Kullanmadan önce güncel Apple belgesinden doğrula |
| Standart iPhone 18, iPhone 18e, iPhone Air 2 | Resmî belge yok → matriste yok |

**Bakım:** Her yeni iPhone duyurusunda Bölüm 4.5 prosedürünü uygula, bu tabloyu ve Bölüm 4'ü güncelle, tarihi değiştir.
