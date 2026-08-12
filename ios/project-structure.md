# iOS Project Structure

Native SwiftUI apps. One Xcode project, one app target, all source in a single flat directory.

## Directory Layout

```
Project/
├── Project.xcodeproj
├── Project/                    # App target — all Swift source lives here, flat
│   ├── ProjectApp.swift        # @main App entry: model container, auth gate, splash
│   ├── ContentView.swift       # Root navigation (TabView)
│   ├── LoginView.swift         # Login screen
│   ├── Models.swift            # SwiftData @Model types + Codable API DTOs
│   ├── APIClient.swift         # Networking actor, APIConfig, Keychain wrapper
│   ├── LibraryView.swift       # One file per screen
│   ├── BrowseView.swift
│   ├── DetailView.swift
│   ├── PlayerView.swift
│   ├── CoverView.swift         # Reusable image/cover component
│   ├── SharedUI.swift          # Shared layout helpers + navigation values
│   ├── DownloadManager.swift   # Singleton services (one responsibility each)
│   ├── AudioPlayerManager.swift
│   ├── Assets.xcassets/
│   │   ├── AccentColor.colorset
│   │   ├── AppIcon.appiconset
│   │   ├── commercial_cat.imageset   # Splash / brand mascot
│   │   └── placeholder.imageset
│   └── Info.plist
├── ProjectTests/
├── ProjectUITests/
├── README.md
├── USAGE.md
├── DEPLOYMENT.md
└── LICENSE
```

## Structure Rules

1. **Flat source directory.** No `Views/`, `Models/`, `Services/` subfolders. The target
   directory holds every `.swift` file side by side. File names carry the organization.

2. **One screen per file.** Each top-level screen is its own `SomethingView.swift`. Row
   views, cells, and small subviews used by that screen live in the same file, below a
   `// MARK: -` divider.

3. **One responsibility per file.** `APIClient.swift` is networking. `DownloadManager.swift`
   is downloads. `Models.swift` holds every model and DTO. Don't split a single concern
   across files or merge two concerns into one.

4. **SwiftUI only.** No UIKit view controllers, Storyboards, or XIBs. Drop to `UIKit` types
   (`UIImage`, `UIImpactFeedbackGenerator`) only for things SwiftUI doesn't cover.

5. **Track the current OS.** Target the latest iOS. Use the newest APIs (`ContentUnavailableView`,
   the value-based `Tab`, SwiftData) directly — no availability shims or back-compat branches.

## App Entry Point

The `@main App` owns the `ModelContainer` and gates the whole UI on auth state. Three
states, in order: not logged in → login; logged in but managers still spinning up → splash;
ready → main content. See [login-and-splash.md](login-and-splash.md) for the splash and login.

```swift
/**
 Main app entry point.

 Configures the SwiftData model container and initializes shared managers.
 Shows a login screen when not authenticated, then transitions to the main app.
 */

import SwiftUI
import SwiftData

extension Color {
    static let appAccent = Color(red: 0.4, green: 0.7, blue: 1.0)
}

@main
struct ProjectApp: App {
    @Environment(\.scenePhase) private var scenePhase

    @State private var isReady = false
    @State private var isAuthenticated = APIConfig.isAuthenticated

    var sharedModelContainer: ModelContainer = {
        let schema = Schema([LibraryItem.self, PlaybackState.self])
        let config = ModelConfiguration(schema: schema, isStoredInMemoryOnly: false)
        do {
            return try ModelContainer(for: schema, configurations: [config])
        } catch {
            fatalError("Could not create ModelContainer: \(error)")
        }
    }()

    var body: some Scene {
        WindowGroup {
            Group {
                if !isAuthenticated {
                    LoginView(isAuthenticated: $isAuthenticated)
                } else if isReady {
                    ContentView()
                        .tint(.appAccent)
                } else {
                    SplashScreen()
                }
            }
            .statusBarHidden()
            .task(id: isAuthenticated) {
                guard isAuthenticated else { return }
                await setupManagers()
                isReady = true
            }
        }
        .modelContainer(sharedModelContainer)
    }

    @MainActor
    private func setupManagers() async {
        let context = sharedModelContainer.mainContext
        SomeManager.shared.setModelContext(context)
        // Kick off network-dependent work, but never hold the splash on it.
    }
}
```

## Root Navigation

A single `TabView` using the value-based `Tab` API and SF Symbols for tab icons. Apply the
app tint once at the root; every control inherits it.

```swift
struct ContentView: View {
    @State private var selectedTab = 0

    var body: some View {
        TabView(selection: $selectedTab) {
            Tab("Library", systemImage: "books.vertical", value: 0) {
                LibraryView()
            }
            Tab("Browse", systemImage: "magnifyingglass", value: 1) {
                BrowseView()
            }
        }
        .tint(.appAccent)
    }
}
```

## Managers as Singletons

Cross-screen services are singletons exposed as `.shared`, conforming to `ObservableObject`
so views observe them with `@ObservedObject`. Each has a single responsibility and receives
the SwiftData context after launch.

```swift
@MainActor
final class DownloadManager: ObservableObject {
    static let shared = DownloadManager()
    private init() {}

    @Published private(set) var activeDownloads: [Int: DownloadProgress] = [:]

    private var modelContext: ModelContext?
    func setModelContext(_ context: ModelContext) { modelContext = context }
}
```

Views observe them directly:

```swift
struct LibraryView: View {
    @ObservedObject private var downloadManager = DownloadManager.shared
    @Query(sort: \LibraryItem.addedDate, order: .reverse) private var items: [LibraryItem]
    // ...
}
```

## SwiftData

Persistence is SwiftData. Models are `@Model` classes; screens read them with `@Query` and
mutate through the `@Environment(\.modelContext)`.

```swift
import SwiftData

@Model
final class LibraryItem {
    var itemId: Int
    var title: String
    var addedDate: Date
    var coverImageData: Data?

    init(itemId: Int, title: String, addedDate: Date = .now) {
        self.itemId = itemId
        self.title = title
        self.addedDate = addedDate
    }
}
```

## Info.plist

Keep it minimal. Portrait-only, launch screen empty (the app draws its own splash), and only
the capabilities the app actually uses.

```xml
<key>UILaunchScreen</key>
<dict/>
<key>UISupportedInterfaceOrientations</key>
<array>
    <string>UIInterfaceOrientationPortrait</string>
</array>
```

## Deployment

Ship a `DEPLOYMENT.md` with the exact device build/install/launch commands:

```bash
# Build for a connected device
xcodebuild -scheme Project -destination 'id=DEVICE_UDID' -configuration Debug build

# Install
xcrun devicectl device install app --device DEVICE_UDID \
  ~/Library/Developer/Xcode/DerivedData/Project-*/Build/Products/Debug-iphoneos/Project.app

# Launch
xcrun devicectl device process launch --device DEVICE_UDID com.author.Project
```

If code signing fails with `errSecInternalComponent`, the login keychain is locked. Unlock it
interactively before building:

```bash
security unlock-keychain ~/Library/Keychains/login.keychain-db
```
</content>
