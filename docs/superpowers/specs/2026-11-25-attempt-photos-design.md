# Başarısız deneme fotoğrafları — Tasarım

Tarih: 2026-11-25 · Dal: `attempt-photos`

## Amaç
Biri Mac'in kilidini Glance ile açmaya çalışır ve başarısız olursa, o anın fotoğrafı ve saati kaydedilir. Kullanıcı Ayarlar'daki **Denemeler** sekmesinde "kim denedi?" listesini görür; menü çubuğu yeni denemeleri haber verir.

## Kararlar (kullanıcıyla netleştirildi)
| Konu | Karar |
|---|---|
| Ne kaydedilir | Tanınmayan yüz (`consistentlyWrongFace`) **ve** sahte yüz denemesi (`spoofSuspected`). "Yüz yok" (`noResolution`) kaydedilmez. |
| Liste yeri | Yeni **Denemeler** sekmesi (Tanıma ile Hakkında arasında). |
| Varsayılan | **Açık**; sekmedeki anahtarla kapatılabilir. |
| Saklama | **Son 50 deneme**; 51. gelince en eskisi silinir. Tek tek ve toptan silme var. |
| Haber verme | **Menü çubuğu**: simgenin yanında yeni deneme sayısı + menüde "N yeni deneme" satırı. |
| Depolama | **Şifresiz JPEG** dosyaları (kullanıcı tercihi). Sonuç: sekmeyi görmek için Touch ID gerekmez; bu macOS kullanıcısına erişen herkes dosyaları Finder'da görebilir. |

## Bölüm 1 — Kayıt akışı

### Kanca: `FaceUnlockCoordinator`
- `observeScanWindow` tarama boyunca **kanıt karesini** tutar: yüz algılanıp eşleşmeyen ya da canlılık reddi tetikleyen en son karenin `CGImage`'ı.
- `ScanOutcome.consistentlyWrongFace` ve `.spoofSuspected` bu kareyi taşır (`evidence: CGImage?`).
- `runScanCycle` bu iki sonuçta `recordAttempt(evidence:kind:)` çağırır:
  - `GlanceSettings.shared.saveAttemptPhotos` kapalıysa veya kare yoksa hiçbir şey yapmaz.
  - Kayıt `Task.detached(priority: .utility)` ile arka planda yapılır; başarısızlık animasyonu ve otomatik yeniden deneme beklemez.
- Değişiklik bu dosyada en küçük hâlde tutulur; mevcut kilit açma mantığına dokunulmaz.

### Yeni dosya: `glance/AttemptLog.swift`
`nonisolated enum AttemptLog` — arayüzden bağımsız, test edilebilir:
- `enum Kind: String { case unrecognized, spoof }`
- `struct Attempt: Identifiable, Hashable { url, date, kind }`
- `static var defaultDirectory: URL` → `~/Library/Application Support/glance/attempts/`
- `static func record(_ image: CGImage, kind:, date: = .now, in directory: = defaultDirectory, limit: = 50) throws -> Attempt`
  - Klasörü gerekirse oluşturur.
  - ImageIO ile JPEG yazar: en uzun kenar ≤ 1280 px, kalite 0,7.
  - Dosya adı: `yyyy-MM-dd'T'HH-mm-ss-SSS'Z'_<kind>.jpg` (UTC, `en_US_POSIX`).
  - Ardından `trim(to: limit)`; sonra ana iş parçacığında `.attemptLogDidChange` bildirimi yayınlar.
- `static func list(in directory:) -> [Attempt]` — dosya adlarını çözer, yeniden eskiye sıralar, tanınmayan adları atlar.
- `static func delete(_ attempt:)`, `static func deleteAll(in directory:)` — silip bildirim yayınlar.
- `static func unseenCount(since: Date?, in directory:) -> Int`

### Hata durumları
Disk/kodlama hatası kilit açma akışını asla etkilemez (`try?`, sessiz geçiş). Bozuk veya yabancı dosya adları listede atlanır.

## Bölüm 2 — Arayüz

### Ayarlar
- `GlanceSettings.saveAttemptPhotos: Bool` (varsayılan `true`) ve `attemptsLastSeenAt: Date?`. İkincisi değişince `.attemptLogDidChange` yayınlanır.
- `SettingsTab.attempts`: başlık "Attempts" (tr: "Denemeler"), simge `person.crop.circle.badge.exclamationmark`.

### `glance/Settings/Pages/AttemptsSettingsPage.swift`
- **Grup:** "Save photos of failed attempts" anahtarı + açıklama ("Fotoğraflar sadece bu Mac'te saklanır").
- **Bölüm başlığı:** "Recent attempts".
- **Satırlar:** küçük resim (64×48, yuvarlatılmış), tür başlığı ("Unrecognized face" / "Spoof attempt"), tarih (`Date.formatted(date: .abbreviated, time: .shortened)` — dile göre biçimlenir), sağda çöp kutusu düğmesi. Küçük resme tıklayınca açılır pencerede (popover) büyük fotoğraf.
- **Boş durum:** "No failed attempts yet." açıklaması.
- **Alt satır:** "Delete all attempts" + mevcut `HoldToConfirmButton` ("Remove").
- `onAppear`: listeyi okur, `attemptsLastSeenAt = .now` yapar. `.attemptLogDidChange` gelince listeyi yeniler.

### Sekmeye doğrudan açma
- `AppEnvironment.pendingSettingsTab: SettingsTab?`.
- `SettingsWindowView`: `onAppear` ve `onChange(of: pendingSettingsTab)` ile varsa seçimi o sekmeye alır, sonra `nil` yapar.

### Menü çubuğu (`glanceApp.swift`)
- Yeni menü öğesi en üstte: "%lld new attempts" (tr: "%lld yeni deneme"), simge `exclamationmark.triangle.fill`; yalnızca yeni deneme varken görünür. Tıklanınca `pendingSettingsTab = .attempts` + Ayarlar açılır.
- Durum simgesinin yanında sayı: yeni deneme varsa `button.title = " N"` ve `imagePosition = .imageLeading` (uzunluk değişken); yoksa başlık boş.
- `menuNeedsUpdate` ve `.attemptLogDidChange` bildiriminde güncellenir.

### Yerelleştirme
Yeni metinler `String(localized:)` / SwiftUI literal olarak yazılır; derlemeden sonra `xcstringstool sync` ile kataloğa alınıp Türkçe çevirileri eklenir.

## Test
- **Otomatik:** `AttemptLog.swift` küçük bir komut satırı test programıyla (`swiftc`) geçici klasörde denenir: kayıt + dosya adı çözme, sıralama, 50 sınırı, tek/toplu silme, `unseenCount`.
- **Derleme:** Debug derlemesi hatasız; katalog senkronunda eksik çeviri sıfır.
- **Elle (kullanıcı):** kilit ekranında başka bir yüz ve telefondaki bir fotoğrafla deneme → Denemeler sekmesi ve menü çubuğu sayısı.

## Kapsam dışı
Bildirim Merkezi bildirimi, şifreleme, fotoğrafı Finder'da gösterme, iCloud eşitleme.
