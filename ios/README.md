# iOS Styleguide

Structure and design conventions for iOS apps. Use these documents as context when generating new iOS projects. The reference implementation is a native SwiftUI app: minimal, sparse, and built almost entirely from stock system components.

<p align="center">
  <img src="assets/commercial_cat.png" width="200" alt="Commercial Cat splash mascot">
</p>

## Documents

| File | Description |
|------|-------------|
| [project-structure.md](project-structure.md) | Xcode project layout, flat file organization, app entry point, SwiftData, managers |
| [design.md](design.md) | The minimalist aesthetic: color, materials, corner radii, SF Symbols, native components |
| [login-and-splash.md](login-and-splash.md) | Simple login screen, the Commercial Cat splash, and the Keychain auth gate |
| [code-style.md](code-style.md) | Swift/SwiftUI conventions: file headers, `MARK`, async/await, singletons, naming |

## Quick Reference

### Stack

- **UI**: SwiftUI only (no UIKit view controllers, no Storyboards, no XIBs)
- **Persistence**: SwiftData (`@Model`, `ModelContainer`, `@Query`)
- **Networking**: `URLSession` + `async`/`await`, `Codable` DTOs
- **Auth**: Bearer token in the Keychain
- **Icons**: SF Symbols (`Image(systemName:)`)
- **Min target**: latest iOS (the app tracks the current OS, not back-compat)

### New Project Checklist

```
Project/
├── Project.xcodeproj
├── Project/                    # All source in one flat directory
│   ├── ProjectApp.swift        # @main App: model container, auth gate, SplashScreen
│   ├── ContentView.swift       # Root TabView
│   ├── LoginView.swift         # Username/password form
│   ├── Models.swift            # @Model types + Codable API DTOs
│   ├── APIClient.swift         # Networking actor + APIConfig + Keychain
│   ├── <Feature>View.swift     # One file per screen
│   ├── <Thing>Manager.swift    # Singleton services (ObservableObject)
│   ├── SharedUI.swift          # Shared layout helpers, nav values
│   ├── Assets.xcassets/        # AccentColor, AppIcon, commercial_cat, placeholders
│   └── Info.plist
├── ProjectTests/
├── ProjectUITests/
├── README.md
├── USAGE.md
├── DEPLOYMENT.md               # Device build/install commands
└── LICENSE                     # CC0
```

### Key Conventions

| Aspect | Convention |
|--------|------------|
| UI framework | SwiftUI, declarative, one `View` file per screen |
| File layout | Flat directory, no nested `Views/`/`Models/` groups |
| State | `@State` local, `@Query` for SwiftData, `.shared` `ObservableObject` managers |
| Networking | `actor APIClient`, `async`/`await`, typed `APIError: LocalizedError` |
| Auth token | Keychain (`kSecAttrAccessibleAfterFirstUnlock`), never `UserDefaults` |
| Accent color | One soft accent (`Color.appAccent`), applied app-wide via `.tint()` |
| Empty states | `ContentUnavailableView`, never a hand-rolled placeholder |
| Splash | Full-white screen, centered Commercial Cat, status bar hidden |
| License | CC0 |
| Author | Philip Orange \<git@philiporange.com> |

### Design Tokens

| Token | Value |
|-------|-------|
| Accent | `Color(red: 0.4, green: 0.7, blue: 1.0)` — soft sky blue |
| Field / button radius | `12` |
| Cover radius (list / grid) | `6` / `8` |
| Field background | `.ultraThinMaterial` |
| Section spacing | `32` |
| Screen padding | `16`–`32` |
| Empty-state placeholder | Seeded pale pastel + faint icon |

See [design.md](design.md) for the full system and [login-and-splash.md](login-and-splash.md) for the splash and login screens.
</content>
</invoke>
