# Spotify Clone

## 1. Overview

**Full-stack development:** A multi-module Android app (Kotlin, Jetpack Compose) communicating with a Spring Boot backend over REST.

**Design goals:**

- Separation of responsibilities across layers
- Reusable core modules shared by all features
- Centralized authentication and token handling
- Encrypted local session storage
- Automatic access-token refresh
- Lifecycle-aware Compose UI
- Adaptive navigation for the Home screen (in progress)

---

## 2. Technology Stack

**Android:** Kotlin, Jetpack Compose, Navigation Compose, Hilt (dependency injection), Kotlin Coroutines, Kotlin Flow, Proto DataStore, Android Keystore, Retrofit, OkHttp.

**Backend:** Java, Spring Boot, JWT authentication, PostgreSQL.

| Term | Why It's Used |
|---|---|
| Jetpack Compose | Declarative UI reduces boilerplate and keeps the UI in sync with state automatically, instead of manually updating views. |
| Hilt | Removes manual dependency wiring, making repositories and ViewModels easier to provide, swap, and test. |
| Coroutines / Flow | Simplifies asynchronous code compared to callbacks, and Flow allows UI to reactively observe data changes over time. |
| Proto DataStore | Type-safe, schema-based persistence that avoids the runtime errors and boilerplate of manual serialization. |
| Android Keystore | Keeps cryptographic keys isolated from the app process, protecting them even if the device or app is compromised. |
| Retrofit | Turns REST endpoints into simple Kotlin function calls, reducing manual HTTP handling and parsing. |
| OkHttp | Centralizes cross-cutting request logic (auth headers, retries, refresh) in one place instead of duplicating it per call. |
| JWT | Carries the user's identity and permissions directly in the token. |

---

## 3. Module Layout

```
app
│
├── core
│   ├── common
│   ├── data
│   ├── datastore
│   ├── datastore-proto
│   ├── datastore-test
│   ├── designsystem
│   ├── domain
│   ├── model
│   ├── navigation
│   ├── network
│   ├── screenshot-testing
│   ├── security
│   ├── testing
│   └── ui
│
└── feature
    ├── auth
    │   ├── login
    │   ├── magiclink
    │   ├── openmails
    │   ├── signup
    │   └── start
    ├── create
    ├── history
    ├── home
    ├── library
    ├── news
    ├── premium
    ├── search
    └── settings
```

| Module | Definition |
|---|---|
| core.data | Repository interfaces and implementations; the boundary between features and network. |
| core.datastore | Session persistence via Proto DataStore. |
| core.designsystem | Shared, reusable Compose UI components. |
| core.domain | Business rules and validation logic, independent of UI or data source. |
| core.model | Shared data models used across features and core modules. |
| core.navigation | A wrapper around `NavHostController` providing a consistent navigation API. |
| core.network | Retrofit service definitions, interceptors, and authenticators. |
| core.security | Cryptographic operations backed by the Android Keystore. |
| core.ui | Shared UI utilities and building blocks used across feature modules. |

---

## Module Reference

### 1. core.designsystem

A shared library of Compose components, previewed and verified in isolation before being consumed by feature modules.

<img width="1526" height="761" alt="Screenshot 2026-09-06 153039" src="https://github.com/user-attachments/assets/ba1808bf-edf1-4abb-829d-ab2b78096949" />

```
core.designsystem
       │
       ├── GreenSpotifyButton
       ├── WhiteSpotifyButton
       ├── SpotifyOutlinedTextButton
       ├── SpotifyTextButton
       ├── SpotifyCheckbox
       ├── SpotifyTextFieldm
       ├── SpotifyPasswordField
       ├── SpotifyAutoCompleteTextField
       ├── SpotifyCenterAlignedTopAppBar
       └── SpotifyDialog
```

---

### 2. core.network — API Definition

```kotlin
interface SpotifyAuthNetwork {
    suspend fun isEmailAvailable(email: String): Result<Boolean, DataError.Network>
    suspend fun register(registerRequest: NetworkRegisterRequest): Result<NetworkAuthSession, DataError.Network>
    suspend fun login(loginRequest: NetworkLoginRequest): Result<NetworkAuthSession, DataError.Network>
    suspend fun requestMagicLink(request: NetworkMagicLinkRequest): Result<Unit, DataError.Network>
    suspend fun verifyMagicLink(token: String): Result<NetworkAuthSession, DataError.Network>
    suspend fun logout(): Result<Unit, DataError.Network>
}
```

Retrofit executes the HTTP requests defined by this interface. Cross-cutting authentication behavior (token attachment, retry, refresh) is implemented via three OkHttp components, described below.

| Term | Definition |
|---|---|
| Result<T, E> | A sealed type representing either a successful value of type `T` or an error of type `E`, used instead of exceptions for expected failure cases. |
| DataError.Network | An enumeration of network-related failure categories (e.g., no internet, server error, timeout). |

---

#### 2.1 Auth Interceptor

Determines, per request, whether an access token must be attached.

```
Request
  ↓
AuthInterceptor
  ↓
Is endpoint public?
  ├── Yes → send unchanged
  └── No  → attach "Authorization: Bearer <token>" → send
```

Public/protected status is determined by a custom annotation read from Retrofit's method metadata:

```kotlin
val invocation = request.tag(Invocation::class.java)
val isPublicEndpoint = invocation?.method()?.getAnnotation(PublicRoute::class.java) != null
```

| Endpoint category | Examples |
|---|---|
| Public (no token required) | Register, Login, Request magic link, Verify magic link |
| Protected (token required) | User-specific data, authenticated features, user profile, protected content |

---

#### 2.2 Retry Interceptor

Automatically retries a request when the failure looks temporary rather than permanent — for example, a `503` caused by the server being briefly overloaded is worth retrying, since the same request will likely succeed a moment later. A `404`, by contrast, is not retried, since retrying it will never succeed no matter how many attempts are made.

**Retried conditions:** HTTP status codes `500`, `502`, `503`, `504`, and `IOException`.

**Configuration:**

```
Maximum attempts: 3
Base delay:       300 ms
Maximum delay:    10 seconds
Jitter:           0–150 ms
```

```
Attempt 1 → Failure → delay → Attempt 2 → Failure → longer delay → Attempt 3
```

| Term | Definition |
|---|---|
| Exponential backoff | A retry strategy in which the delay between attempts increases exponentially with each failure. |
| Jitter | A small random delay added to backoff intervals to prevent multiple clients from retrying in synchrony. |

---

#### 2.3 Token Authenticator

Handles `401 Unauthorized` responses by refreshing the access token.

```
Protected request → 401 → TokenAuthenticator
   ↓
Check current access token
   ↓
Already refreshed by another request?
  ├── Yes → reuse newer token
  └── No  → fetch refresh token → refresh → store new tokens → retry original request
```

The method is `synchronized` to prevent concurrent refresh attempts.

**Concurrency case:** if three requests fail near-simultaneously —

```
Request A → 401
Request B → 401
Request C → 401
```

— only one should trigger a refresh. This is enforced by comparing the currently stored access token against the token the failed request originally used:

```
Current stored access token   vs.   token used by the failed request
```

If they differ, another request has already completed a refresh; the current request reuses that token instead of issuing a redundant refresh call.

**Refresh failure:**

```
Refresh failed → clear session → user is no longer authenticated
```

The authenticator returns `null`, allowing the original request to fail rather than retry indefinitely.

---

### 3. core.data — Repository Layer

Features depend on repository interfaces, not on Retrofit directly:

```
LoginViewModel → AuthRepository → Network implementation → Spotify Backend
```

`AuthRepository` is the single boundary between the app's features and everything network-related. A ViewModel calls a method like `login(...)` and gets back a `Result`, without knowing (or needing to know) whether that call went through Retrofit, hit a cache, or came from somewhere else entirely.

`AuthRepository` exposes:

- Register
- Check email availability
- Login
- Logout
- Request magic link
- Verify magic link
- Refresh authentication session

This separation matters for two practical reasons:

- **Swappable implementation.** The `Network` box in the diagram above is just one implementation of `AuthRepository`. If the backend, the serialization format, or even the entire networking library changed tomorrow, only that implementation would need to change — every ViewModel that depends on `AuthRepository` stays untouched.
- **Testability.** Because features depend on an interface rather than a concrete `Retrofit` client, tests can substitute a fake `AuthRepository` that returns canned results, without spinning up any real network calls.

---

### 4. core.datastore — Session Interface

```kotlin
interface SessionHandler {
    val isAuthenticated: Flow<Boolean>
    suspend fun getRefreshToken(): String?
    suspend fun getAccessToken(): String?
    suspend fun setAuthSession(authSession: AuthSession)
    fun getAuthSession(): Flow<AuthSession?>
    suspend fun clear()
}
```

| Term | Definition |
|---|---|
| SessionHandler | An interface abstracting read, write, and clear operations on the authentication session, independent of the underlying storage mechanism. |
| AuthSession | A data object holding the access token, refresh token. |

Session data is not stored as plain Proto DataStore bytes; it passes through an encryption step before being written.

**Write path:**

```
AuthSession → Proto serialization → ByteArray → Encryption → Encrypted ByteArray → Proto DataStore
```

```kotlin
override suspend fun writeTo(t: Session, output: OutputStream) {
    val bytes = t.toByteArray()
    val encryptedBytes = cryptoManager.encrypt(bytes)
    withContext(Dispatchers.IO) {
        output.write(encryptedBytes)
    }
}

override suspend fun readFrom(input: InputStream): Session =
    try {
        val bytes = input.readBytes()
        val decryptedBytes = cryptoManager.decrypt(bytes)
        Session.parseFrom(decryptedBytes)
    } catch (exception: InvalidProtocolBufferException) {
        throw CorruptionException("Cannot read proto.", exception)
    }
```

---

### 5. core.security — Cryptography

`CryptoManager` performs encryption/decryption using a key held in the Android Keystore; the key itself is never exposed to application code.

**Configuration:**

```
Algorithm:     AES
Block mode:    CBC
Padding:       PKCS7
Key storage:   AndroidKeyStore
Key alias:     secret
```

```kotlin
fun encrypt(bytes: ByteArray): ByteArray {
    val cipher = Cipher.getInstance(TRANSFORMATION)
    cipher.init(Cipher.ENCRYPT_MODE, getKey())
    val iv = cipher.iv
    val encrypted = cipher.doFinal(bytes)
    return iv + encrypted
}

fun decrypt(bytes: ByteArray): ByteArray {
    val cipher = Cipher.getInstance(TRANSFORMATION)
    val iv = bytes.copyOfRange(0, cipher.blockSize)
    val data = bytes.copyOfRange(cipher.blockSize, bytes.size)
    cipher.init(Cipher.DECRYPT_MODE, getKey(), IvParameterSpec(iv))
    return cipher.doFinal(data)
}
```

Each encryption operation generates a random IV, stored as `IV + ciphertext`. Decryption extracts the IV from the start of the byte array before decrypting the remainder.

```
Android Keystore → AES SecretKey → CryptoManager → encrypt/decrypt → SessionSerializer → Proto DataStore
```

| Term | Definition |
|---|---|
| AES | Advanced Encryption Standard; a symmetric-key block cipher. |
| CBC | Cipher Block Chaining; a block cipher mode in which each ciphertext block depends on the previous one. |
| PKCS7 padding | A padding scheme that fills the final block to a fixed block size before encryption. |
| IV (Initialization Vector) | A random value used to ensure that encrypting the same plaintext twice produces different ciphertext. |
| KeyGenParameterSpec | An Android API object specifying the properties (algorithm, purpose, block mode) of a key generated in the Keystore. |

---

### 6. core.domain — Validation

Validation is implemented as standalone classes independent of any Compose screen, which allows unit testing without a UI.

```kotlin
class EmailValidator : Validator<String> {
    override fun execute(data: String): ValidationResult {
        if (data.isBlank() || data.isEmpty()) {
            return ValidationResult(
                successful = false,
                errorMessage = UiText.DynamicString("The email can't be blank")
            )
        }

        if (!EMAIL_REGEX.matches(data)) {
            return ValidationResult(
                successful = false,
                errorMessage = UiText.DynamicString(
                    "This email is invalid. Make sure it's written like example@email.com"
                )
            )
        }

        return ValidationResult(successful = true)
    }
}
```

**Data flow:**

```
Text Field → Validator → ValidationResult → UI displays error
```

| Term | Definition |
|---|---|
| Validator | An interface exposing `execute(data)`, returning a `ValidationResult`; each form field type (email, password, etc.) has its own implementation. |
| ValidationResult | A data object containing a boolean success flag and an optional error message. |
| UiText | A wrapper type that defers string resolution (e.g., from a string resource or a literal) until the UI layer renders it. |

**Registration flow:** each field of the sign-up form lives on its own page inside a `HorizontalPager` — Email, Password, Gender, Date of Birth, and Name. The user cannot advance to the next page until the current page's validator returns a successful `ValidationResult`.

```
Page 1: Email → validate → Page 2: Password → validate → Page 3: Gender → validate → Page 4: Date of Birth → validate → Page 5: Name → validate → Submit
```

Each field has its own validator with its own rules:

| Field | Validation rules |
|---|---|
| Email | Must not be blank; must match a valid email pattern (e.g. `example@email.com`). |
| Password | At least 8 characters; at least one uppercase letter; at least one digit. |
| Gender | Must be one of the predefined options (e.g. Male, Female); cannot be left unselected. |
| Date of Birth | Must not be blank; must represent a valid, real date; user must meet the minimum required age. |
| Name | Must not be blank; must not contain digits or special characters. |

---

### 7. core.navigation

```kotlin
navigator.navigate(route)
navigator.popBackStack()
navigator.navigate(destination = destination, popUpToRoute = route)
```

Top-level destinations use state-preserving navigation:

```kotlin
navController.navigate(route) {
    popUpTo(navController.graph.startDestinationId) {
        saveState = true
    }
    launchSingleTop = true
    restoreState = true
}
```

| Term | Definition |
|---|---|
| Navigator | A wrapper class around `NavHostController`, centralizing navigation calls so screens do not manipulate the controller directly. |
| popUpTo | A navigation option that removes destinations from the back stack up to a specified route. |
| launchSingleTop | A navigation option preventing a duplicate instance of a destination from being placed on top of itself. |
| restoreState / saveState | Options controlling whether a destination's UI and back-stack state are preserved when navigating away and returning. |

---

### 8. Login Feature

#### 8.1 Presentation flow

```
LoginScreen → LoginViewModel → AuthRepository → Network
```

State is collected with lifecycle awareness:

```kotlin
val state by viewModel.state.collectAsStateWithLifecycle()
```

Navigation events are one-time and delivered via a `Channel`, collected in a `LaunchedEffect`, rather than stored in `StateFlow`:

```
Login success → Channel → NavigateToHome → UI → Navigator
```

| Term | Definition |
|---|---|
| collectAsStateWithLifecycle | A Compose function that collects a Flow only while the UI is in an active lifecycle state, avoiding work when the screen is not visible. |
| Channel | A Kotlin coroutines construct for sending discrete events exactly once to a single collector, distinct from Flow's continuously-observed state. |
| One-time event | An event (e.g., navigation) that should be consumed exactly once and not re-triggered by state re-collection, such as after a configuration change. |

#### 8.2 UI events and state

```kotlin
internal sealed interface LoginUiEvent {
    data class EmailChanged(val email: TextFieldValue) : LoginUiEvent
    data class PasswordChanged(val password: TextFieldValue) : LoginUiEvent
    data object BackPressed : LoginUiEvent
    data object Login : LoginUiEvent
    data object RequestMagicLink : LoginUiEvent
}
```

```kotlin
internal data class LoginUiState(
    val email: TextFieldValue = TextFieldValue(""),
    val password: TextFieldValue = TextFieldValue(""),
    val errorMessage: UiText? = null,
    val isLoading: Boolean = false,
    val isOnline: Boolean = true
) {
    val isLoginButtonEnabled: Boolean =
        email.text.isNotEmpty() &&
        password.text.isNotEmpty() &&
        errorMessage == null &&
        isOnline &&
        !isLoading
}
```

#### 8.3 ViewModel

```kotlin
val state: StateFlow<LoginUiState> = combine(
    _state,
    networkMonitor.isOnline
) { uiState, online ->
    uiState.copy(
        isOnline = online)
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5_000),
    initialValue = LoginUiState()
)
```

```kotlin
private fun login() = viewModelScope.launch {
    _state.update { it.copy(isLoading = true) }

    val email = _state.value.email.text
    val password = _state.value.password.text
    val loginResult = authRepository.login(LoginData(email, password))

    _state.update { it.copy(isLoading = false) }

    when (loginResult) {
        is Result.Success -> _uiEvent.send(LoginNavigationEvents.NavigateToHome)
        is Result.Failure -> _state.update {
            it.copy(errorMessage = loginResult.error.asUiText())
        }
    }
}
```


---

### 9. feature.home — In Progress

`feature.home` is currently being built end to end — backend entities first, then the Compose frontend. Work is starting with Podcasts.

#### 9.1 Backend — Podcast & Episode entities

```java
@Entity
@Table(name = "podcast")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Podcast {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID podcastId;

    private String description;

    private Image image;

    @ElementCollection
    @CollectionTable(
            name = "podcast_languages",
            joinColumns = @JoinColumn(name = "id")
    )
    private List<String> languages;

    private String mediaType;

    private String name;

    private String publisher;

    private int totalEpisodes;

    @OneToMany(mappedBy = "podcast", cascade = CascadeType.ALL)
    private List<Episode> episodes;
}
```

```java
@Entity
@Table(name = "episodes")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Episode {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID episodeId;

    private String audioPreviewUrl;

    private String description;

    private int durationMs;

    private boolean explicit;

    @ElementCollection
    @CollectionTable(name = "episode_images", joinColumns = @JoinColumn(name = "episode_id"))
    private List<Image> images;

    @ElementCollection
    @CollectionTable(
            name = "episode_languages",
            joinColumns = @JoinColumn(name = "episode_id")
    )
    private List<String> languages;

    private String name;

    private String releaseDate;

    private String mediaType;

    private String publisher;

    @ManyToOne
    @JoinColumn(name = "podcast_id")
    private Podcast podcast;
}
```

`Episode` holds a many-to-one back-reference to `Podcast`, while `Podcast` holds the corresponding one-to-many collection — a podcast can have many episodes, and each episode belongs to exactly one podcast.

#### 9.2 Frontend — Adaptive layout

The Home screen's Compose layout is being built with adaptive layout in mind: the same screen reflows across compact and expanded window sizes rather than assuming a single fixed layout.

<table>
  <tr>
          <td align="center">
      <h3>Phone</h3>
      <img width="300" height="700" alt="Screenshot 2026-09-06 165126" src="https://github.com/user-attachments/assets/8580cace-7794-42ef-b99a-1803c7a06c32" />
    </td>
    <td align="center">
      <h3>Foldable</h3>
      <img width="500" height="450" alt="Screenshot 2026-09-06 165106" src="https://github.com/user-attachments/assets/cefacbb1-27b9-420b-aa49-1e5678c68fef" />
    </td>
    <td align="center">
      <h3>Tablet</h3>
      <img width="700" height="620" alt="Screenshot 2026-09-06 165156" src="https://github.com/user-attachments/assets/c0338a83-150e-42aa-800e-9f96e2186288" />
    </td>
  </tr>
</table>
