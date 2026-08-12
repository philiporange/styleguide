# iOS Design

Native SwiftUI apps with a modern, minimalist, sparse aesthetic. The design comes almost
entirely from stock system components — the work is in *restraint*, not in custom styling.

## Design Philosophy

**Modernism and minimalism are paramount.** Interfaces should be clean, focused, and
uncluttered. Let the content — covers, titles, artwork — carry the screen.

- **Negative space.** Use generous padding and let elements breathe. Big `Spacer()`s,
  centered content, and empty margins are deliberate. White space creates hierarchy and
  reduces cognitive load.
- **Sparse UI.** Show only what's necessary. A screen is often one list or one grid and
  nothing else — no toolbars full of buttons, no chrome, no decoration.
- **Lean on the system.** Prefer stock SwiftUI components and system materials over custom
  drawing. `ContentUnavailableView`, `List`, `TabView`, SF Symbols, `.ultraThinMaterial`.
  The platform already looks good; don't fight it.
- **One accent, used sparingly.** A single soft accent color marks actions and progress.
  Everything else is `.primary` / `.secondary` and system backgrounds.
- **Typography over decoration.** Let type weight and size and spacing build hierarchy
  instead of borders, boxes, and shadows. Flat surfaces, no gradients-as-ornament, no
  drop shadows for their own sake.
- **Content-forward.** Artwork and covers are the brightest thing on screen. UI recedes.

## Color

One accent color, defined once and applied app-wide. Everything else is semantic system color.

```swift
extension Color {
    static let appAccent = Color(red: 0.4, green: 0.7, blue: 1.0)  // soft sky blue
}
```

Apply the tint once at the root so every control inherits it:

```swift
ContentView()
    .tint(.appAccent)
```

| Role | Color |
|------|-------|
| Accent (actions, progress, selection) | `Color.appAccent` |
| Primary text | `.primary` |
| Secondary text, captions, icons | `.secondary` |
| Backgrounds | System (`Color(.systemBackground)`, `.ultraThinMaterial`) |
| Destructive | `.red` |

Rules:

- **Never hardcode grays.** Use `.primary` / `.secondary` so light and dark mode both work.
- **Accent is for meaning**, not flavor — a percentage, a selected tab, a primary button.
  If everything is accented, nothing is.
- Tint the accent onto SF Symbols and text with `.foregroundStyle(Color.appAccent)`.

## Materials & Surfaces

Use system materials instead of solid custom fills. They adapt to context and keep the UI
flat and light.

```swift
TextField("Username", text: $username)
    .padding()
    .background(.ultraThinMaterial)
    .clipShape(RoundedRectangle(cornerRadius: 12))
```

- Input fields and inset surfaces: `.ultraThinMaterial`.
- No manual drop shadows or borders on cards. If separation is needed, use spacing or a
  material, not a stroke.

## Shape & Spacing

Consistent, generous, rounded.

| Element | Corner radius |
|---------|---------------|
| Text fields, buttons | `12` |
| List cover thumbnails | `6` |
| Grid / detail covers | `8` |
| Large floating art (mini-player) | `16` |

| Spacing | Value |
|---------|-------|
| Between major sections | `24`–`32` |
| Between fields in a group | `16` |
| Screen horizontal padding | `16`–`32` |
| Row internal spacing | `12` |

Always clip to a `RoundedRectangle`, never `.cornerRadius()` on the whole view stack:

```swift
.clipShape(RoundedRectangle(cornerRadius: 12))
```

## SF Symbols

All icons are SF Symbols — no icon fonts, no bundled PNGs for UI glyphs.

```swift
Image(systemName: "books.vertical")
Label("Delete", systemImage: "trash")
Image(systemName: "xmark.circle.fill")
    .symbolRenderingMode(.hierarchical)
    .foregroundStyle(.secondary)
```

- Use `.hierarchical` rendering for filled multi-tone glyphs.
- Size symbols with the text scale (`.font(.title3)`), not fixed frames, so they respect
  Dynamic Type.

## Native Components

Reach for the system component first. It's less code and looks right by default.

### Empty States

Always `ContentUnavailableView` — never a hand-built "nothing here" view.

```swift
ContentUnavailableView {
    Label("No Audiobooks", systemImage: "books.vertical")
} description: {
    Text("Add audiobooks from the Browse tab")
}

// For empty search results:
ContentUnavailableView.search(text: searchText)
```

### Lists

Plain style, hidden separators, custom row insets. Let rows be simple `HStack`s.

```swift
List {
    ForEach(items) { item in
        ItemRow(item: item)
            .listRowInsets(EdgeInsets(top: 12, leading: 16, bottom: 12, trailing: 16))
            .listRowSeparator(.hidden)
            .swipeActions(edge: .trailing, allowsFullSwipe: true) {
                Button(role: .destructive) { delete(item) } label: {
                    Label("Delete", systemImage: "trash")
                }
            }
    }
}
.listStyle(.plain)
.refreshable { await refresh() }
.navigationTitle("Library")
.navigationBarTitleDisplayMode(.large)
```

Large navigation titles are the default. Keep toolbars nearly empty — an edit toggle at most.

### Grids

`LazyVGrid` with adaptive columns so it fills any width without hardcoded counts.

```swift
private let columns = [GridItem(.adaptive(minimum: 150, maximum: 180), spacing: 16)]

ScrollView {
    LazyVGrid(columns: columns, spacing: 20) {
        ForEach(items) { item in
            ItemCell(item: item)
        }
    }
    .padding()
}
```

### Rows

A cover thumbnail, a title/subtitle stack, and a trailing accent detail. Nothing more.

```swift
HStack(spacing: 12) {
    CoverView(imageData: item.coverImageData, seed: item.id, cornerRadius: 6)
        .frame(width: 70, height: 70)

    VStack(alignment: .leading, spacing: 4) {
        Text(item.title)
            .font(.body).fontWeight(.medium)
            .lineLimit(2)
        Text(item.subtitle)
            .font(.subheadline)
            .foregroundStyle(.secondary)
            .lineLimit(1)
    }

    Spacer()

    if let percent = item.progressPercent {
        Text("\(percent)%")
            .font(.subheadline).fontWeight(.medium)
            .foregroundStyle(Color.appAccent)
            .monospacedDigit()          // numbers don't jitter as they update
    }
}
```

## Covers & Placeholders

Content art is the visual anchor. When there's no image yet, generate a calm, deterministic
placeholder instead of showing a gray box — a pale pastel derived from the item's id, with a
faint icon on top. Same item → same color, every launch.

```swift
/// Deterministic pale pastel from a seed value.
struct SeededPaleColor {
    static func color(for seed: Int) -> Color {
        var rng = SeededRandomGenerator(seed: UInt64(bitPattern: Int64(seed)))
        let hue = Double.random(in: 0...1, using: &rng)
        let saturation = Double.random(in: 0.15...0.35, using: &rng)   // muted
        let brightness = Double.random(in: 0.85...0.95, using: &rng)   // pale
        return Color(hue: hue, saturation: saturation, brightness: brightness)
    }
}

struct ItemPlaceholder: View {
    let seed: Int
    var body: some View {
        Rectangle()
            .fill(SeededPaleColor.color(for: seed))
            .overlay {
                Image("placeholder-icon")
                    .resizable().aspectRatio(contentMode: .fit)
                    .frame(width: 40, height: 40)
                    .opacity(0.6)
            }
    }
}
```

Wrap all image display in one reusable `CoverView` that handles: local data, remote load with
a placeholder, and failure fallback — with a soft cross-fade in.

```swift
.animation(.easeInOut(duration: 0.3), value: loadedImage != nil)
```

## Motion & Feedback

- **Subtle, short animations.** `.easeInOut(duration: 0.2–0.3)`. Motion confirms a change;
  it never performs.
- **Gradient fades, not hard edges.** Where content scrolls under a floating bar, mask it with
  a `clear → black` `LinearGradient` so it dissolves rather than clipping.

  ```swift
  .mask(
      VStack(spacing: 0) {
          LinearGradient(colors: [.clear, .black], startPoint: .top, endPoint: .bottom)
              .frame(height: 60)
          Color.black
      }
  )
  ```

- **Light haptics on destructive or committing taps.**

  ```swift
  UIImpactFeedbackGenerator(style: .light).impactOccurred()
  ```

## Full-Screen Presentation

Immersive screens (a player, a reader) present with `.fullScreenCover`, not a partial sheet.
Detail pages that can be dismissed present as sheets owning their own `NavigationStack` with a
single `xmark` toolbar button.

```swift
.fullScreenCover(item: $selectedItem) { item in
    PlayerView(item: item)
}
```

## Checklist

- [ ] One accent color, applied via `.tint()` at the root
- [ ] Text is `.primary` / `.secondary` — no hardcoded grays
- [ ] `.ultraThinMaterial` for input surfaces; no manual shadows or borders
- [ ] Corner radii from the token table (`12` controls, `6`/`8` covers)
- [ ] All icons are SF Symbols
- [ ] Empty states use `ContentUnavailableView`
- [ ] Lists are `.plain` with hidden separators and large titles
- [ ] Missing images fall back to a seeded pastel placeholder, not a gray box
- [ ] Animations are short `.easeInOut`; light haptics on committing actions
- [ ] Status bar hidden where the design calls for a clean canvas
</content>
