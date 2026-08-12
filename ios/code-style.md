# Swift Code Style

Conventions for SwiftUI app code. Modern Swift concurrency, declarative views, minimal
ceremony.

## File Headers

Every file opens with a `/** ... */` block describing what it does and any non-obvious design
decisions. One paragraph of intent, then details.

```swift
/**
 API client for communicating with the backend.

 Handles all network requests using async/await. Requests are authenticated
 via a Bearer token obtained from the login endpoint and stored in the Keychain.
 */

import Foundation
```

## MARK Organization

Divide files with `// MARK: -`. One screen file typically reads: the main `View`, then its row
/ cell subviews, then small helpers — each behind a divider.

```swift
struct LibraryView: View {
    // ...
}

// MARK: - Library Row

struct LibraryRow: View {
    // ...
}

// MARK: - Empty State

private struct LibraryEmptyState: View {
    // ...
}
```

Group methods inside a type the same way: `// MARK: - Auth`, `// MARK: - Search`, etc.

## Views

- **Small, composable views.** A screen is a `View`; its rows and sections are their own
  `View`s in the same file. If a subview is only used here, mark it `private`.
- **`private` all local state.** `@State`, `@Query`, and helper methods are `private`.
- **Computed properties for derived UI values**, not inline expressions in `body`.

```swift
struct LibraryView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \LibraryItem.addedDate, order: .reverse) private var items: [LibraryItem]
    @State private var selectedItem: LibraryItem?

    private var hasContent: Bool { !items.isEmpty }

    var body: some View {
        NavigationStack {
            Group {
                if !hasContent {
                    ContentUnavailableView {
                        Label("No Items", systemImage: "tray")
                    } description: {
                        Text("Add items from the Browse tab")
                    }
                } else {
                    list
                }
            }
            .navigationTitle("Library")
        }
    }

    private var list: some View {
        List(items) { item in
            LibraryRow(item: item)
        }
        .listStyle(.plain)
    }
}
```

## Concurrency

- **`async`/`await` everywhere.** No completion handlers, no Combine for networking.
- Kick off async work from views with `.task { }` (cancels automatically) or
  `Button { Task { await ... } }`.
- Network layers are `actor`s; shared mutable managers are `@MainActor final class`.
- Mark value types and functions that touch no actor state `nonisolated`.

```swift
actor APIClient {
    static let shared = APIClient()

    func login(username: String, password: String) async throws {
        let url = baseURL.appending(path: "/api/login")
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONEncoder().encode(["username": username, "password": password])

        let (data, response) = try await session.data(for: request)
        try validateResponse(response)

        let decoded = try JSONDecoder().decode(LoginResponse.self, from: data)
        APIConfig.authToken = decoded.token
    }
}
```

## Error Handling

Define one typed error enum per domain, conforming to `LocalizedError`, and surface
`errorDescription` to the UI.

```swift
enum APIError: LocalizedError {
    case invalidResponse
    case httpError(statusCode: Int)
    case authenticationRequired

    var errorDescription: String? {
        switch self {
        case .invalidResponse:      return "Invalid server response"
        case .httpError(let code):  return "Server error: \(code)"
        case .authenticationRequired: return "Authentication required"
        }
    }
}
```

Validate responses in one place and log the user out on `401`:

```swift
private func validateResponse(_ response: URLResponse) throws {
    guard let http = response as? HTTPURLResponse else { throw APIError.invalidResponse }
    if http.statusCode == 401 {
        APIConfig.logout()
        throw APIError.authenticationRequired
    }
    guard (200...299).contains(http.statusCode) else {
        throw APIError.httpError(statusCode: http.statusCode)
    }
}
```

In views, catch and set an `errorMessage` string for inline display — don't pop alerts for
network failures.

## Models & DTOs

- SwiftData persistence types are `@Model final class`.
- API response types are `struct`s conforming to `Decodable`. Match the server's JSON keys
  directly (`is_admin`, `poster_path`) rather than adding `CodingKeys` for cosmetics.

```swift
struct LoginResponse: Decodable {
    let token: String
    let username: String
    let is_admin: Bool
}
```

## Naming

```swift
// Types: PascalCase. Screens end in View, services in Manager.
struct BrowseView: View {}
final class DownloadManager: ObservableObject {}

// Properties / functions: camelCase.
private var isLoading = false
func searchItems(query: String) async throws -> [Item] {}

// Constants: camelCase (Swift, not SCREAMING_CASE).
private let pageSize = 50
private let coverSize: CGFloat = 70

// Booleans read as assertions.
var isAuthenticated: Bool
var hasContent: Bool
```

Name design constants instead of scattering magic numbers:

```swift
private let coverSize: CGFloat = 70
private let cornerRadius: CGFloat = 12
```

## Avoid

1. **No UIKit view controllers, Storyboards, or XIBs** — SwiftUI only.
2. **No completion-handler APIs** — use `async`/`await`.
3. **No back-compat branches or `if #available` shims** — target the current OS.
4. **No hardcoded grays** — use `.primary` / `.secondary`.
5. **No token in `UserDefaults`** — Keychain only.
6. **No magic numbers** — name spacing, sizes, and radii.
7. **No commented-out code** — delete it; git remembers.
8. **No force-unwraps on remote data** — decode into optionals and handle absence.
</content>
