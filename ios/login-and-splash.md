# Login & Splash

Two screens set the tone for the whole app: the **Commercial Cat splash** and a **simple
login**. Both are deliberately sparse — a centered mascot, one form, nothing else. They sit
behind an auth gate driven by a Bearer token in the Keychain.

## The Commercial Cat Splash

Every app opens on the same brand splash: a full-white screen with the **Commercial Cat**
mascot centered. Nothing else — no spinner, no title, no version string. It shows for the
brief moment after login while shared managers spin up.

<p align="center">
  <img src="assets/commercial_cat.png" width="220" alt="Commercial Cat">
</p>

Ship [`assets/commercial_cat.png`](assets/commercial_cat.png) into the app's asset catalog as a
`commercial_cat` image set.

```swift
struct SplashScreen: View {
    var body: some View {
        ZStack {
            Color.white
                .ignoresSafeArea()

            Image("commercial_cat")
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(width: 256, height: 256)
        }
    }
}
```

Rules:

- **Pure white background**, edge to edge (`.ignoresSafeArea()`). Not the system background —
  white, so it reads the same in light and dark mode and matches the mascot art.
- **Centered, fixed-size mascot** (`256×256`). No text, no motion.
- **Status bar hidden** (set on the root; see the auth gate below).
- The splash is a *transition state*, not a timed screen. It's shown only while the app is
  genuinely doing startup work — never `sleep` to pad it out.

The same mascot reappears smaller (`120×120`) as the login logo, tying the two screens together.

## Simple Login

One screen, vertically centered by `Spacer()`s: mascot, app name, two fields, one button.
No "forgot password", no sign-up, no social buttons, no background art. The button stays
disabled until both fields have input, and errors appear inline in red.

```swift
/**
 Login view for authenticating with the server.

 Presents a username/password form. On success the auth token is stored in the
 Keychain and the app transitions to the main content view.
 */

import SwiftUI

struct LoginView: View {
    @Binding var isAuthenticated: Bool

    @State private var username = ""
    @State private var password = ""
    @State private var isLoading = false
    @State private var errorMessage: String?

    var body: some View {
        VStack(spacing: 32) {
            Spacer()

            Image("commercial_cat")
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(width: 120, height: 120)

            Text("Project")
                .font(.largeTitle)
                .fontWeight(.bold)

            VStack(spacing: 16) {
                TextField("Username", text: $username)
                    .textContentType(.username)
                    .textInputAutocapitalization(.never)
                    .autocorrectionDisabled()
                    .padding()
                    .background(.ultraThinMaterial)
                    .clipShape(RoundedRectangle(cornerRadius: 12))

                SecureField("Password", text: $password)
                    .textContentType(.password)
                    .padding()
                    .background(.ultraThinMaterial)
                    .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding(.horizontal, 32)

            if let error = errorMessage {
                Text(error)
                    .font(.callout)
                    .foregroundStyle(.red)
            }

            Button {
                Task { await login() }
            } label: {
                if isLoading {
                    ProgressView()
                        .tint(.white)
                        .frame(maxWidth: .infinity)
                        .padding()
                } else {
                    Text("Sign In")
                        .fontWeight(.semibold)
                        .frame(maxWidth: .infinity)
                        .padding()
                }
            }
            .background(Color.appAccent)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 12))
            .padding(.horizontal, 32)
            .disabled(isLoading || username.isEmpty || password.isEmpty)

            Spacer()
            Spacer()
        }
    }

    private func login() async {
        isLoading = true
        errorMessage = nil
        do {
            try await APIClient.shared.login(username: username, password: password)
            isAuthenticated = true
        } catch let error as APIError {
            if case .httpError(let code) = error, code == 401 {
                errorMessage = "Invalid username or password"
            } else {
                errorMessage = error.localizedDescription
            }
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }
}
```

Conventions:

- **Centered by `Spacer()`s.** A leading `Spacer()` and a double trailing `Spacer()` push the
  form to a natural optical center (slightly above true middle).
- **Fields** use `.ultraThinMaterial` backgrounds and radius-`12` corners. Set the right
  `.textContentType` (`.username`, `.password`) so the system offers autofill and saves
  credentials to the keychain. Disable autocapitalization and autocorrect on the username.
- **Button** is accent-filled, white text, radius `12`, full width within a `32pt` inset.
  It shows a white `ProgressView` in place of the label while the request is in flight, and is
  disabled until both fields are non-empty.
- **Errors are inline**, `.callout` in `.red`, above the button. Map `401` to a friendly
  "Invalid username or password"; fall back to the error's description otherwise. No alerts.

## The Auth Gate

`isAuthenticated` is derived from whether a token exists in the Keychain. The `App` swaps
between login, splash, and content based on it. Flipping the `@Binding` from `LoginView`
transitions the whole app.

```swift
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
    await setupManagers()   // splash is visible during this
    isReady = true
}
```

## Token Storage — Keychain, not UserDefaults

The Bearer token lives in the Keychain with `kSecAttrAccessibleAfterFirstUnlock`, so reads
never prompt and keep working for background tasks once the device has been unlocked after a
boot. `APIConfig` is the single source of truth for auth state.

```swift
nonisolated enum APIConfig {
    static let baseURL = URL(string: "https://api.example.com")!
    private static let tokenKey = "authToken"

    static var authToken: String? {
        get { KeychainStore.read(key: tokenKey) }
        set {
            if let newValue { KeychainStore.write(key: tokenKey, value: newValue) }
            else { KeychainStore.delete(key: tokenKey) }
        }
    }

    static var isAuthenticated: Bool { authToken != nil }

    static func logout() { authToken = nil }

    /// Attach the Bearer header to a request.
    static func authenticatedRequest(url: URL) -> URLRequest {
        var request = URLRequest(url: url)
        if let token = authToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        return request
    }
}
```

A minimal generic-password Keychain wrapper:

```swift
private nonisolated enum KeychainStore {
    private static let service = "com.author.Project"

    static func read(key: String) -> String? {
        var query = baseQuery(key: key)
        query[kSecReturnData as String] = true
        query[kSecMatchLimit as String] = kSecMatchLimitOne
        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        guard status == errSecSuccess, let data = result as? Data else { return nil }
        return String(data: data, encoding: .utf8)
    }

    static func write(key: String, value: String) {
        let data = Data(value.utf8)
        let status = SecItemUpdate(
            baseQuery(key: key) as CFDictionary,
            [kSecValueData as String: data] as CFDictionary
        )
        if status == errSecItemNotFound {
            var attributes = baseQuery(key: key)
            attributes[kSecValueData as String] = data
            attributes[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlock
            SecItemAdd(attributes as CFDictionary, nil)
        }
    }

    static func delete(key: String) {
        SecItemDelete(baseQuery(key: key) as CFDictionary)
    }

    private static func baseQuery(key: String) -> [String: Any] {
        [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
        ]
    }
}
```

On a `401`, call `APIConfig.logout()` in the response validator so an expired token drops the
user straight back to the login screen.
</content>
