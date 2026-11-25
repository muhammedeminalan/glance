# Uygulama Kilidi — Tasarım

Tarih: 2026-11-25 · Dal: `app-lock`

## Amaç
Kullanıcının seçtiği uygulamalar (WhatsApp, Notlar, Fotoğraflar…) öne geldiğinde ekran bulanık bir örtüyle kapanır; yüz doğrulanınca (ya da Touch ID / Mac parolasıyla) uygulama açılır. Bu, meraklı gözlere karşı bir kilittir; ayrı bir macOS kullanıcı hesabının yerini tutmaz.

## Kararlar (kullanıcıyla netleştirildi)
| Konu | Karar |
|---|---|
| Tekrar kilitleme | Ayarlar'dan seçilir: **Uygulama kapanınca (varsayılan)**, Başka uygulamaya geçince 5 dk sonra, Her öne gelişte. Mac kilitlenince/uyuyunca her modda hepsi kilitlenir. |
| Yedek yöntem | **Touch ID / Mac parolası** (`LAContext`, `.deviceOwnerAuthentication`). |
| Kilit görünümü | **Bulanık tam ekran örtü + yüz simgesi**; ortada uygulama simgesi ve "X kilitli". Vazgeçilirse uygulama gizlenir. |
| Yaklaşım | Kilit ekranı kodu (`FaceUnlockCoordinator`) değişmez. Ayrı `FaceVerifier`, tanıma modelini paylaşır ama **kendi `CameraManager`'ını** kullanır (paylaşılan kameraya gelen geç bir `stop()` kilit ekranı taramasını bozabilir). |
| Kapsam dışı | Glance'in kendisi kilitlenemez; bildirim içerikleri ve Spotlight önizlemeleri gizlenmez. |

## Bölüm 1 — İzleme ve doğrulama
- **`AppLockState`** (saf, test edilebilir): açılmış uygulamaları `bundleID → (pid, açılma zamanı, arka plana geçiş zamanı)` olarak tutar. `needsVerification(bundleID:pid:policy:now:)` kararı verir:
  - Kayıt yoksa ya da pid farklıysa (uygulama yeniden başlamış) → doğrulama gerekir.
  - Açıldıktan sonraki 2 sn içinde → gerekmez (kilit açılınca uygulamanın yeniden öne gelmesi döngü yaratmasın).
  - `onQuit` → gerekmez. `afterFiveMinutesAway` → arka planda 5 dk'dan uzun kalmışsa gerekir. `everyActivation` → her zaman gerekir.
  - `markUnlocked`, `markActivated` (5 dk sayacını sıfırlar), `markDeactivated`, `markTerminated`, `lockAll`.
- **`AppLockMonitor`** (`NSObject`, `@MainActor`): `NSWorkspace` öne geldi / arka plana geçti / kapandı bildirimleri, `com.apple.screenIsLocked` ve uyku bildirimi. Kendi uygulamamızın öne gelmesi yok sayılır. Doğrulama sürerken başka bir uygulama öne gelirse doğrulama iptal edilir ve kilitli uygulama gizlenir. Glance açılırken öndeki uygulama da kontrol edilir.
- **`FaceVerifier`** (`@MainActor`): oturum kilitliyse ya da kayıtlı yüz yoksa `.unavailable`; değilse kamerayı açar, ≤5 sn boyunca tanıma + canlılık döngüsü (`GlanceSettings` eşik/canlılık ayarlarıyla) → `.matched` / `.notRecognized`.
- **`AppLockOverlayController`**: her ekrana `.floating` seviyesinde (Touch ID / parola penceresi üstünde görünebilsin diye), tüm Spaces'te görünen, kenarlıksız bir pencere. Denetimler ana ekranda; diğer ekranlarda yalnız bulanıklık. Durumlar: tarıyor, tanınmadı, doğrulanıyor, Touch ID gerekli. Düğmeler: **Touch ID ile aç**, **Yeniden dene** (tanınmadıysa), **Vazgeç**.
- Akış: eşleşirse kilit açılır → örtü kapanır → uygulama öne getirilir. `.unavailable` → doğrudan Touch ID istenir. Touch ID başarısızsa "tanınmadı" durumuna döner.

## Bölüm 2 — Ayarlar, metinler, test
- **`GlanceSettings`**: `lockedAppBundleIDs: [String]` (varsayılan boş), `appLockRelockPolicy: AppLockRelockPolicy` (varsayılan `.onQuit`).
- **`SettingsTab.appLock`**: "App Lock" (tr: "Uygulama Kilidi"), simge `lock.rectangle.stack.fill`, Denemeler ile Hakkında arasında.
- **`AppLockSettingsPage`**:
  - Üstte açıklama: ne işe yaradığı ve meraklı göz kilidi olduğu.
  - "Lock again" satırı + açılır seçici (üç seçenek).
  - "Locked apps" başlığı; boşsa "No apps are locked yet."; doluysa satırlar: uygulama simgesi, adı, çöp kutusu.
  - "Add app…" satırı: `NSOpenPanel` (`/Applications`, `.application`, çoklu seçim). Glance'in kendisi ve zaten listede olanlar eklenmez.
- **`AppEnvironment`**: `appLockMonitor` (kilit ekranının `pipeline`'ını paylaşır).
- **Metinler**: yeni metinler `String(localized:)` / SwiftUI literal; Türkçeleri katalogda.
- **Test**:
  - `tools/app_lock_selftest.swift` (`swiftc`): `AppLockState`'in üç politikası, 2 sn payı, pid değişimi, 5 dk sınırı, `lockAll`, `markTerminated`.
  - Debug derlemesi uyarısız; katalog senkronunda eksik çeviri 0.
  - Kullanıcı: Developer ID ile imzalı test kurulumu → Notlar'ı kilitle → aç → örtü → yüz / Touch ID / Vazgeç.
