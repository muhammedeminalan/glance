# Başarısız deneme fotoğrafları — Uygulama planı

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. (Bu oturumda kullanıcı isteğiyle satır içi uygulanıyor.)

**Goal:** Başarısız Glance kilit açma denemelerinde (tanınmayan / sahte yüz) fotoğraf ve saat kaydetmek; Ayarlar'da "Denemeler" sekmesi ve menü çubuğunda yeni deneme sayısı göstermek.

**Architecture:** `AttemptLog` (yeni, arayüzden bağımsız) JPEG'leri `~/Library/Application Support/glance/attempts/` altına tarih+tür içeren adlarla yazar/okur/siler. `FaceUnlockCoordinator` tarama sonucuyla birlikte kanıt karesini taşır ve başarısızlıkta `AttemptLog.record`'u arka planda çağırır. Ayarlar sayfası ve menü çubuğu `.attemptLogDidChange` bildirimiyle güncellenir.

**Tech Stack:** Swift 5 dil modu, SwiftUI + AppKit, ImageIO, UniformTypeIdentifiers. Test hedefi yok; `tools/` altındaki `swiftc` self-test kalıbı kullanılır.

**Spec:** `docs/superpowers/specs/2026-11-25-attempt-photos-design.md`

## Global Constraints
- macOS dağıtım hedefi 15.0; yeni bağımlılık yok.
- Varsayılan: `saveAttemptPhotos = true`; sınır **50** deneme; JPEG en uzun kenar **1280 px**, kalite **0.7**.
- Dosya adı: `yyyy-MM-dd'T'HH-mm-ss-SSS'Z'_<unrecognized|spoof>.jpg` (UTC, `en_US_POSIX`).
- Kayıt hatası kilit açma akışını asla etkilemez.
- Kullanıcıya görünen her yeni metin `String(localized:)` / SwiftUI literal; Türkçe çevirisi `glance/Localizable.xcstrings`'e eklenir.
- `glance.xcodeproj/project.pbxproj` yerelde `skip-worktree` (kullanıcının imza ayarları); bu plan o dosyaya dokunmaz. `glance/` altındaki yeni dosyalar senkronize grup sayesinde hedefe kendiliğinden girer.

---

### Task 1: `AttemptLog` + self-test

**Files:**
- Create: `glance/AttemptLog.swift`
- Create: `tools/attempt_log_selftest.swift`

**Interfaces:**
- Produces: `Notification.Name.attemptLogDidChange`; `AttemptLog.Kind` (`.unrecognized`, `.spoof`); `AttemptLog.Attempt { url: URL, date: Date, kind: Kind }`; `AttemptLog.record(_ image: CGImage, kind:, date: = Date(), in: = defaultDirectory, limit: = 50) throws -> Attempt`; `AttemptLog.list(in:) -> [Attempt]` (yeniden eskiye); `AttemptLog.unseenCount(since: Date?, in:) -> Int`; `AttemptLog.delete(_:)`; `AttemptLog.deleteAll(in:)`.

- [ ] **Step 1: Self-test'i yaz** — `tools/attempt_log_selftest.swift`:

```swift
//
//  attempt_log_selftest.swift
//  glance (tools)
//
//  Offline test for AttemptLog's file naming, ordering, limit and deletion.
//  This project has no test target, so this compiles as a standalone script:
//
//      swiftc -parse-as-library -o /tmp/attempt_log_selftest \
//        glance/AttemptLog.swift tools/attempt_log_selftest.swift && /tmp/attempt_log_selftest
//

import Foundation
import CoreGraphics
import ImageIO

@main
enum AttemptLogSelfTest {
    static var failures = 0

    static func check(_ condition: Bool, _ message: String) {
        if condition { print("ok   \(message)") } else { failures += 1; print("FAIL \(message)") }
    }

    static func makeImage(width: Int, height: Int) -> CGImage {
        let context = CGContext(
            data: nil, width: width, height: height, bitsPerComponent: 8, bytesPerRow: 0,
            space: CGColorSpaceCreateDeviceRGB(), bitmapInfo: CGImageAlphaInfo.noneSkipLast.rawValue
        )!
        context.setFillColor(CGColor(red: 0.2, green: 0.5, blue: 0.8, alpha: 1))
        context.fill(CGRect(x: 0, y: 0, width: width, height: height))
        return context.makeImage()!
    }

    static func names(_ attempts: [AttemptLog.Attempt]) -> [String] {
        attempts.map(\.url.lastPathComponent)
    }

    static func main() throws {
        let dir = FileManager.default.temporaryDirectory
            .appendingPathComponent("attempt-log-selftest-\(UUID().uuidString)", isDirectory: true)
        defer { try? FileManager.default.removeItem(at: dir) }
        let image = makeImage(width: 1920, height: 1080)
        let base = Date(timeIntervalSince1970: 1_800_000_000) // 2027-01-15 08:00:00 UTC

        let first = try AttemptLog.record(image, kind: .unrecognized, date: base, in: dir)
        check(FileManager.default.fileExists(atPath: first.url.path), "record writes a file")
        check(first.url.lastPathComponent == "2027-01-15T08-00-00-000Z_unrecognized.jpg",
              "file name encodes UTC date and kind (\(first.url.lastPathComponent))")
        check(names(AttemptLog.list(in: dir)) == [first.url.lastPathComponent], "list returns the recorded attempt")
        check(AttemptLog.list(in: dir).first?.date == base, "parsed date round-trips")

        if let source = CGImageSourceCreateWithURL(first.url as CFURL, nil),
           let props = CGImageSourceCopyPropertiesAtIndex(source, 0, nil) as? [CFString: Any],
           let width = props[kCGImagePropertyPixelWidth] as? Int, let height = props[kCGImagePropertyPixelHeight] as? Int {
            check(max(width, height) <= 1280, "JPEG longest edge <= 1280 (got \(width)x\(height))")
        } else {
            check(false, "JPEG is readable")
        }

        let second = try AttemptLog.record(image, kind: .spoof, date: base.addingTimeInterval(60), in: dir)
        check(AttemptLog.list(in: dir).map(\.kind) == [.spoof, .unrecognized], "list is newest first, kinds preserved")

        check(AttemptLog.unseenCount(since: nil, in: dir) == 2, "no last-seen mark means every attempt is unseen")
        check(AttemptLog.unseenCount(since: base.addingTimeInterval(30), in: dir) == 1, "only attempts after the mark are unseen")

        try Data("x".utf8).write(to: dir.appendingPathComponent("notes.txt"))
        try Data("x".utf8).write(to: dir.appendingPathComponent("bad_name.jpg"))
        check(AttemptLog.list(in: dir).count == 2, "unknown file names are skipped")

        AttemptLog.delete(second)
        check(names(AttemptLog.list(in: dir)) == [first.url.lastPathComponent], "delete removes only that attempt")

        for offset in 0..<5 {
            try AttemptLog.record(image, kind: .unrecognized, date: base.addingTimeInterval(Double(100 + offset)), in: dir, limit: 3)
        }
        let limited = AttemptLog.list(in: dir)
        check(limited.count == 3, "limit keeps only the newest N (got \(limited.count))")
        check(limited.first?.date == base.addingTimeInterval(104), "newest attempt survives the trim")

        AttemptLog.deleteAll(in: dir)
        check(AttemptLog.list(in: dir).isEmpty, "deleteAll removes every attempt")
        check(FileManager.default.fileExists(atPath: dir.appendingPathComponent("notes.txt").path),
              "deleteAll leaves unrelated files alone")

        print(failures == 0 ? "ALL PASSED" : "\(failures) FAILED")
        exit(failures == 0 ? 0 : 1)
    }
}
```

- [ ] **Step 2: Çalıştır, derlenmediğini gör** — `swiftc -parse-as-library -o /tmp/attempt_log_selftest glance/AttemptLog.swift tools/attempt_log_selftest.swift` → Beklenen: `glance/AttemptLog.swift` bulunamadı hatası.

- [ ] **Step 3: `glance/AttemptLog.swift`'i yaz**

```swift
//
//  AttemptLog.swift
//  glance
//
//  Photos of failed unlock attempts (an unrecognized face, or a spoof caught by liveness), stored as plain JPEGs under
//  Application Support. The date and kind live in each file name, so there is no separate index to keep in sync.
//

import Foundation
import CoreGraphics
import ImageIO
import UniformTypeIdentifiers

extension Notification.Name {
    /// Posted on the main queue whenever attempts are added or removed, and when the "last seen" mark moves.
    nonisolated static let attemptLogDidChange = Notification.Name("AttemptLog.didChange")
}

nonisolated enum AttemptLog {
    enum Kind: String, CaseIterable, Sendable {
        case unrecognized
        case spoof
    }

    struct Attempt: Identifiable, Hashable, Sendable {
        let url: URL
        let date: Date
        let kind: Kind
        var id: URL { url }
    }

    static let defaultLimit = 50
    static let maxPixelSize = 1280
    static let jpegQuality = 0.7

    static var defaultDirectory: URL {
        FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("glance", isDirectory: true)
            .appendingPathComponent("attempts", isDirectory: true)
    }

    /// Fresh per call — `DateFormatter` isn't Sendable, and at most a few dozen files are ever parsed at once.
    private static func makeFormatter() -> DateFormatter {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "en_US_POSIX")
        formatter.timeZone = TimeZone(identifier: "UTC")
        formatter.dateFormat = "yyyy-MM-dd'T'HH-mm-ss-SSS'Z'"
        return formatter
    }

    private static func parse(_ fileName: String, formatter: DateFormatter) -> (date: Date, kind: Kind)? {
        guard fileName.hasSuffix(".jpg") else { return nil }
        let stem = fileName.dropLast(4)
        guard let separator = stem.lastIndex(of: "_"),
              let kind = Kind(rawValue: String(stem[stem.index(after: separator)...])),
              let date = formatter.date(from: String(stem[..<separator])) else { return nil }
        return (date, kind)
    }

    /// Writes `image` as a downscaled JPEG, then drops the oldest attempts beyond `limit`.
    @discardableResult
    static func record(
        _ image: CGImage,
        kind: Kind,
        date: Date = Date(),
        in directory: URL = defaultDirectory,
        limit: Int = defaultLimit
    ) throws -> Attempt {
        try FileManager.default.createDirectory(at: directory, withIntermediateDirectories: true)
        let formatter = makeFormatter()
        let url = directory.appendingPathComponent("\(formatter.string(from: date))_\(kind.rawValue).jpg")
        guard let destination = CGImageDestinationCreateWithURL(url as CFURL, UTType.jpeg.identifier as CFString, 1, nil) else {
            throw CocoaError(.fileWriteUnknown)
        }
        let options: [CFString: Any] = [
            kCGImageDestinationLossyCompressionQuality: jpegQuality,
            kCGImageDestinationImageMaxPixelSize: maxPixelSize,
        ]
        CGImageDestinationAddImage(destination, image, options as CFDictionary)
        guard CGImageDestinationFinalize(destination) else { throw CocoaError(.fileWriteUnknown) }

        for stale in list(in: directory).dropFirst(limit) {
            try? FileManager.default.removeItem(at: stale.url)
        }
        postDidChange()
        // The file name keeps millisecond precision, so report the date exactly as `list` will.
        return Attempt(url: url, date: parse(url.lastPathComponent, formatter: formatter)?.date ?? date, kind: kind)
    }

    /// Newest first. Files whose names don't parse are ignored, so stray files in the folder are harmless.
    static func list(in directory: URL = defaultDirectory) -> [Attempt] {
        let urls = (try? FileManager.default.contentsOfDirectory(at: directory, includingPropertiesForKeys: nil)) ?? []
        let formatter = makeFormatter()
        return urls
            .compactMap { url in
                parse(url.lastPathComponent, formatter: formatter).map { Attempt(url: url, date: $0.date, kind: $0.kind) }
            }
            .sorted { $0.date > $1.date }
    }

    /// Attempts newer than `lastSeen`; with no mark yet, every attempt counts.
    static func unseenCount(since lastSeen: Date?, in directory: URL = defaultDirectory) -> Int {
        let attempts = list(in: directory)
        guard let lastSeen else { return attempts.count }
        return attempts.filter { $0.date > lastSeen }.count
    }

    static func delete(_ attempt: Attempt) {
        try? FileManager.default.removeItem(at: attempt.url)
        postDidChange()
    }

    /// Removes recognized attempt files only; anything else in the folder is left alone.
    static func deleteAll(in directory: URL = defaultDirectory) {
        for attempt in list(in: directory) {
            try? FileManager.default.removeItem(at: attempt.url)
        }
        postDidChange()
    }

    private static func postDidChange() {
        DispatchQueue.main.async {
            NotificationCenter.default.post(name: .attemptLogDidChange, object: nil)
        }
    }
}
```

- [ ] **Step 4: Self-test'i çalıştır** — Step 2'deki komut `&& /tmp/attempt_log_selftest` ile. Beklenen: her satır `ok`, son satır `ALL PASSED`.

- [ ] **Step 5: Commit** — `git add glance/AttemptLog.swift tools/attempt_log_selftest.swift && git commit -m "Add AttemptLog for failed-attempt photos"`

---

### Task 2: Ayar anahtarları + tarama kancası

**Files:**
- Modify: `glance/Settings/GlanceSettings.swift` (Key enum, iki özellik, init)
- Modify: `glance/FaceUnlockCoordinator.swift` (`ScanOutcome`, `observeScanWindow` iki `return`, `runScanCycle` iki `case`, yeni `recordAttempt`)

**Interfaces:**
- Consumes: `AttemptLog.record`, `AttemptLog.Kind`, `.attemptLogDidChange`.
- Produces: `GlanceSettings.shared.saveAttemptPhotos: Bool`, `GlanceSettings.shared.attemptsLastSeenAt: Date?`.

- [ ] **Step 1: `GlanceSettings`** — `Key` enum'a:

```swift
        static let saveAttemptPhotos = "GlanceSettings.saveAttemptPhotos"
        static let attemptsLastSeenAt = "GlanceSettings.attemptsLastSeenAt"
```

`hasAcknowledgedSecurityNotice` özelliğinin altına:

```swift
    /// Off means failed attempts leave no photo — see `AttemptLog`.
    var saveAttemptPhotos: Bool {
        didSet { defaults.set(saveAttemptPhotos, forKey: Key.saveAttemptPhotos) }
    }
    /// When the Attempts tab was last opened; attempts newer than this count as unseen in the menu bar.
    var attemptsLastSeenAt: Date? {
        didSet {
            defaults.set(attemptsLastSeenAt, forKey: Key.attemptsLastSeenAt)
            NotificationCenter.default.post(name: .attemptLogDidChange, object: nil)
        }
    }
```

`init` içinde `hasAcknowledgedSecurityNotice = …` satırının altına:

```swift
        saveAttemptPhotos = defaults.object(forKey: Key.saveAttemptPhotos) as? Bool ?? true
        attemptsLastSeenAt = defaults.object(forKey: Key.attemptsLastSeenAt) as? Date
```

- [ ] **Step 2: `ScanOutcome`** — iki durum kanıt karesini taşır:

```swift
    private enum ScanOutcome {
        case matched
        /// `evidence` is the last frame showing the unmatched face, for `AttemptLog`.
        case consistentlyWrongFace(evidence: CGImage?)
        /// A deny cue (glare, device rectangle) fired — actively rejected as a spoof regardless of match. Same failure path as `.consistentlyWrongFace`.
        case spoofSuspected(evidence: CGImage?)
        case noResolution
    }
```

`observeScanWindow` içinde: `return .spoofSuspected` → `return .spoofSuspected(evidence: frame.image)`; `return .consistentlyWrongFace` → `return .consistentlyWrongFace(evidence: frame.image)`.

- [ ] **Step 3: `runScanCycle`** — `case .consistentlyWrongFace:` → `case .consistentlyWrongFace(let evidence):` ve ilk satır olarak `recordAttempt(evidence, kind: .unrecognized)`; `case .spoofSuspected:` → `case .spoofSuspected(let evidence):` ve ilk satır `recordAttempt(evidence, kind: .spoof)`. `ScanOutcome` tanımının üstüne:

```swift
    /// Saves the failed attempt's frame off the main actor, so the failure UI and auto-retry never wait on disk I/O.
    private func recordAttempt(_ evidence: CGImage?, kind: AttemptLog.Kind) {
        guard GlanceSettings.shared.saveAttemptPhotos, let evidence else { return }
        Task.detached(priority: .utility) {
            _ = try? AttemptLog.record(evidence, kind: kind)
        }
    }
```

- [ ] **Step 4: Derle** — `xcodebuild -project glance.xcodeproj -scheme glance -configuration Debug CODE_SIGNING_ALLOWED=NO build` → `BUILD SUCCEEDED`.

- [ ] **Step 5: Commit** — `git add glance/Settings/GlanceSettings.swift glance/FaceUnlockCoordinator.swift && git commit -m "Record a photo when a scan ends on an unrecognized or spoofed face"`

---

### Task 3: Denemeler sekmesi

**Files:**
- Create: `glance/Settings/Pages/AttemptsSettingsPage.swift`
- Modify: `glance/Settings/SettingsTab.swift` (yeni `case attempts`, başlık, simge)
- Modify: `glance/AppEnvironment.swift` (`pendingSettingsTab`)
- Modify: `glance/Settings/SettingsWindowView.swift` (sayfa `case`, bekleyen sekmeyi uygulama)

**Interfaces:**
- Consumes: `AttemptLog.list/delete/deleteAll`, `.attemptLogDidChange`, `GlanceSettings.saveAttemptPhotos/attemptsLastSeenAt`.
- Produces: `SettingsTab.attempts`, `AppEnvironment.pendingSettingsTab: SettingsTab?`.

- [ ] **Step 1: `SettingsTab`** — `case recognition` ile `case about` arasına `case attempts`; `title`'a `case .attempts: return String(localized: "Attempts")`; `icon`'a `case .attempts: return .system("person.crop.circle.badge.exclamationmark")`.

- [ ] **Step 2: `AppEnvironment`** — `isDebugSectionRevealed`'in altına:

```swift
    /// Set before opening Settings to land on a specific tab (e.g. the menu bar's "new attempts" row); the window consumes it.
    var pendingSettingsTab: SettingsTab?
```

- [ ] **Step 3: `SettingsWindowView`** — `pageBody` switch'ine `case .attempts: AttemptsSettingsPage()`; `settingsContent` zincirinde `.onDisappear { selection = .general }` satırının altına:

```swift
        .onAppear { applyPendingTab() }
        .onChange(of: environment.pendingSettingsTab) { _, _ in applyPendingTab() }
```

ve `selectAdjacentTab`'ın üstüne:

```swift
    private func applyPendingTab() {
        guard let tab = environment.pendingSettingsTab else { return }
        selection = tab
        environment.pendingSettingsTab = nil
    }
```

- [ ] **Step 4: `AttemptsSettingsPage.swift`'i yaz**

```swift
//
//  AttemptsSettingsPage.swift
//  glance
//
//  "Who tried?" — photos of failed unlock attempts from `AttemptLog`, newest first.
//

import SwiftUI
import AppKit

struct AttemptsSettingsPage: View {
    @Bindable private var settings = GlanceSettings.shared
    @State private var attempts: [AttemptLog.Attempt] = []

    var body: some View {
        VStack(alignment: .leading, spacing: SettingsMetrics.rowSpacing) {
            SettingsGroup {
                SettingsRowContent(
                    title: String(localized: "Save photos of failed attempts"),
                    subtitle: String(localized: "Takes a photo when a face that isn't enrolled, or a photo of one, tries to unlock your Mac. Photos stay on this Mac."),
                    subtitleMaxWidth: SettingsMetrics.rowSubtitleMaxWidth
                ) {
                    GlanceToggle(isOn: $settings.saveAttemptPhotos)
                }
            }

            VStack(alignment: .leading, spacing: 8) {
                SettingsSectionTitle(text: String(localized: "Recent attempts"))
                if attempts.isEmpty {
                    SettingsCaption(text: String(localized: "No failed attempts yet."))
                        .padding(.leading, SettingsMetrics.sectionTitleHorizontalInset)
                } else {
                    SettingsGroup {
                        ForEach(Array(attempts.enumerated()), id: \.element.id) { index, attempt in
                            if index > 0 {
                                SettingsGroupDivider()
                            }
                            AttemptRow(attempt: attempt) { AttemptLog.delete(attempt) }
                        }
                    }
                    SettingsGroup {
                        SettingsRowContent(title: String(localized: "Delete all attempts")) {
                            HoldToConfirmButton(title: String(localized: "Remove")) { AttemptLog.deleteAll() }
                        }
                    }
                }
            }
        }
        .onAppear {
            reload()
            settings.attemptsLastSeenAt = Date()
        }
        .onReceive(NotificationCenter.default.publisher(for: .attemptLogDidChange)) { _ in reload() }
    }

    private func reload() {
        attempts = AttemptLog.list()
    }
}

/// Thumbnail (click for a larger popover), kind and time, and a delete button.
private struct AttemptRow: View {
    let attempt: AttemptLog.Attempt
    let onDelete: () -> Void
    @State private var image: NSImage?
    @State private var isPreviewing = false

    var body: some View {
        HStack(spacing: 12) {
            Button { isPreviewing = true } label: { thumbnail }
                .buttonStyle(.plain)
                .help(String(localized: "Show photo"))
                .popover(isPresented: $isPreviewing, arrowEdge: .trailing) {
                    if let image {
                        Image(nsImage: image)
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(width: 480)
                            .padding(8)
                    }
                }

            VStack(alignment: .leading, spacing: 2) {
                Text(title)
                    .font(SettingsMetrics.rowFont)
                    .foregroundStyle(SettingsMetrics.textPrimary)
                Text(attempt.date.formatted(date: .abbreviated, time: .shortened))
                    .font(.system(size: 11))
                    .foregroundStyle(SettingsMetrics.textTertiary)
            }

            Spacer(minLength: 8)

            Button(action: onDelete) {
                Image(systemName: "trash")
                    .font(.system(size: 12))
                    .foregroundStyle(SettingsMetrics.textPrimary)
                    .frame(width: 28, height: 28)
                    .background(SettingsMetrics.neutralButtonFill)
                    .clipShape(Circle())
            }
            .buttonStyle(.plain)
            .help(String(localized: "Delete this attempt"))
        }
        .padding(.horizontal, SettingsMetrics.rowHorizontalInset)
        .padding(.vertical, 8)
        .task(id: attempt.url) {
            image = NSImage(contentsOf: attempt.url)
        }
    }

    private var title: String {
        switch attempt.kind {
        case .unrecognized: return String(localized: "Unrecognized face")
        case .spoof: return String(localized: "Spoof attempt")
        }
    }

    private var thumbnail: some View {
        ZStack {
            SettingsMetrics.optionPreviewFill
            if let image {
                Image(nsImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            }
        }
        .frame(width: 64, height: 48)
        .clipShape(RoundedRectangle(cornerRadius: 8, style: .continuous))
    }
}
```

- [ ] **Step 5: Derle** — Task 2 Step 4'teki komut → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit** — `git add glance/Settings/Pages/AttemptsSettingsPage.swift glance/Settings/SettingsTab.swift glance/AppEnvironment.swift glance/Settings/SettingsWindowView.swift && git commit -m "Add Attempts settings tab"`

---

### Task 4: Menü çubuğu işareti

**Files:**
- Modify: `glance/glanceApp.swift` (`AppDelegate`)

**Interfaces:**
- Consumes: `AttemptLog.unseenCount`, `.attemptLogDidChange`, `GlanceSettings.attemptsLastSeenAt`, `AppEnvironment.pendingSettingsTab`, `SettingsTab.attempts`.

- [ ] **Step 1:** `private var sessionMenuItem: NSMenuItem?` altına `private var attemptsMenuItem: NSMenuItem?`.

- [ ] **Step 2:** `applicationDidFinishLaunching` içinde `windowWillClose` gözlemcisinin altına:

```swift
        NotificationCenter.default.addObserver(
            self, selector: #selector(attemptLogDidChange(_:)),
            name: .attemptLogDidChange, object: nil
        )
```

- [ ] **Step 3:** `installStatusItemIfNeeded` içinde `sessionItem` oluşturulmadan önce:

```swift
        let attemptsItem = NSMenuItem(title: "", action: #selector(openAttempts), keyEquivalent: "")
        attemptsItem.target = self
        attemptsItem.image = NSImage(systemSymbolName: "exclamationmark.triangle.fill", accessibilityDescription: nil)
        attemptsItem.isHidden = true
        menu.addItem(attemptsItem)
        attemptsMenuItem = attemptsItem
```

ve sondaki `updateSessionMenuItem()` çağrısının altına `updateAttemptsIndicator()`. `setMenuBarVisibility`'de `sessionMenuItem = nil` altına `attemptsMenuItem = nil`. `menuNeedsUpdate`'e `updateAttemptsIndicator()`.

- [ ] **Step 4:** `updateSessionMenuItem`'ın altına:

```swift
    @objc private func attemptLogDidChange(_ notification: Notification) {
        updateAttemptsIndicator()
    }

    /// Failed attempts since the Attempts tab was last opened — shown next to the icon and as a menu row that opens the tab.
    private func updateAttemptsIndicator() {
        let unseen = AttemptLog.unseenCount(since: GlanceSettings.shared.attemptsLastSeenAt)
        attemptsMenuItem?.isHidden = unseen == 0
        attemptsMenuItem?.title = String(localized: "\(unseen) new attempts")
        guard let statusItem, let button = statusItem.button else { return }
        button.title = unseen == 0 ? "" : " \(unseen)"
        button.imagePosition = unseen == 0 ? .imageOnly : .imageLeading
        statusItem.length = unseen == 0 ? NSStatusItem.squareLength : NSStatusItem.variableLength
    }

    @objc private func openAttempts() {
        environment.pendingSettingsTab = .attempts
        openSettingsWindow()
    }
```

- [ ] **Step 5: Derle** → `BUILD SUCCEEDED`.

- [ ] **Step 6: Commit** — `git add glance/glanceApp.swift && git commit -m "Show unseen failed attempts in the menu bar"`

---

### Task 5: Türkçe + son doğrulama

**Files:**
- Modify: `glance/Localizable.xcstrings`

- [ ] **Step 1:** Derlemenin `.stringsdata` dosyalarıyla `xcrun xcstringstool sync glance/Localizable.xcstrings --stringsdata …` (katalog adı `Localizable.xcstrings` kalmalı; sync tablo adını dosya adından alır).
- [ ] **Step 2:** Türkçe değerler:

| Anahtar | tr |
|---|---|
| Attempts | Denemeler |
| Save photos of failed attempts | Başarısız denemelerin fotoğrafını kaydet |
| Takes a photo when a face that isn't enrolled, or a photo of one, tries to unlock your Mac. Photos stay on this Mac. | Kayıtlı olmayan bir yüz ya da bir yüzün fotoğrafı Mac'inizin kilidini açmaya çalışırsa fotoğraf çekilir. Fotoğraflar sadece bu Mac'te saklanır. |
| Recent attempts | Son denemeler |
| No failed attempts yet. | Henüz başarısız deneme yok. |
| Delete all attempts | Tüm denemeleri sil |
| Show photo | Fotoğrafı göster |
| Delete this attempt | Bu denemeyi sil |
| Unrecognized face | Tanınmayan yüz |
| Spoof attempt | Sahte yüz denemesi |
| %lld new attempts | %lld yeni deneme (en: one → "%lld new attempt", other → "%lld new attempts") |

- [ ] **Step 3:** Yeniden derle; sync tekrarında yeni anahtar ve eksik çeviri **0**; self-test `ALL PASSED`.
- [ ] **Step 4: Commit** — `git add glance/Localizable.xcstrings && git commit -m "Translate failed-attempt strings to Turkish"`
