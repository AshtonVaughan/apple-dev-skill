# Networking Reference

URLSession is the foundation; Alamofire wraps it; Network.framework for low-level. Auth, WebSockets, reachability, ATS, cert pinning, background transfers.

## 1. URLSession — the core

```swift
// Simple GET
let (data, response) = try await URLSession.shared.data(from: url)
guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else { throw ... }
let user = try JSONDecoder().decode(User.self, from: data)

// POST with JSON body
var request = URLRequest(url: url)
request.httpMethod = "POST"
request.setValue("application/json", forHTTPHeaderField: "Content-Type")
request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
request.httpBody = try JSONEncoder().encode(payload)
let (data, _) = try await URLSession.shared.data(for: request)

// Streaming bytes (large downloads, SSE)
let (bytes, response) = try await URLSession.shared.bytes(for: request)
for try await line in bytes.lines { print(line) }
// or: for try await byte in bytes { ... }

// Download to file (memory-efficient for large files)
let (downloadURL, response) = try await URLSession.shared.download(for: request)
try FileManager.default.moveItem(at: downloadURL, to: destination)

// Upload from file
let (data, _) = try await URLSession.shared.upload(for: request, fromFile: fileURL)
// or from Data: upload(for:from:)

// WebSocket
let task = URLSession.shared.webSocketTask(with: url)
task.resume()
try await task.send(.string("hello"))
let message = try await task.receive()   // .string/.data
// ping/pong: task.sendPing(pongReceiveHandler:)
```

- `URLSession.shared` — singleton, ephemeral-ish (has cache + cookie storage).
- `URLSessionConfiguration`:
  - `.default` — persistent (cache, cookies, credentials in keychain).
  - `.ephemeral` — no persistent storage (private browsing equivalent).
  - `.background(withIdentifier:)` — transfers continue when app backgrounds/suspends; completion handler in AppDelegate/SceneDelegate.
  - Properties: `timeoutIntervalForRequest` (default 60s), `timeoutIntervalForResource` (7 days), `waitsForConnectivity` (wait for network instead of failing), `networkServiceType` (`.default`, `.responsiveData` for latency-sensitive, `.responsiveDataConstrained`?), `allowsCellularAccess`, `allowsExpensiveNetworkAccess`, `allowsConstrainedNetworkAccess`, `httpCookieStorage`, `httpCookieAcceptPolicy`, `httpShouldSetCookies`, `urlCache`, `requestCachePolicy` (`.useProtocolCachePolicy`, `.reloadIgnoringLocalCacheData`, `.returnCacheDataElseLoad`), `httpAdditionalHeaders`, `tlsMinimumSupportedProtocol` (`.tlsProtocol12`, `.tlsProtocol13`), `multipathServiceType` (`.handover`, `.interactive`, `.aggregate` — WiFi+cellular bonding).
- `URLRequest` — `url`, `httpMethod` (GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS), `httpBody`/`httpBodyStream`, `httpBodyStream` for chunked upload without loading all into memory, `allHTTPHeaderFields`, `setValue(_:forHTTPHeaderField:)`, `timeoutInterval`, `cachePolicy`, `httpShouldHandleCookies`, `networkServiceType`, `assumesHTTP3Capable`? (HTTP/3 support).
- `URLResponse`/`HTTPURLResponse` — `statusCode`, `allHeaderFields`, `expectedContentLength`, `textEncodingName`, `suggestedFilename`, `url`, `MIMEType`.
- `URLSessionTask` states: `.running`, `.suspended`, `.canceling`, `.completed`; `suspend()`, `resume()`, `cancel()`.
- `URLSessionDownloadTask` — `downloadTask(with:fromResumeData:)`, progress via delegate `didWriteData`, resume with `resumeData` on failure.
- `URLSessionUploadTask` — `uploadTask(with:from:completionHandler:)`, progress via delegate.
- `URLSessionWebSocketTask` — `webSocketTask(with:)` or `webSocketTask(with:url:protocols:)`; `send(_:)`, `receive()`, `sendPing`, `cancel(with:reason:)`; `maximumMessageSize`, `maximumFrameSize`.
- `URLSessionStreamTask` — `readData`, `write(_:timeout:completionHandler:)`, `schedule` for streaming TCP.
- `URLSessionDataDelegate` — `urlSession(_:dataTask:didReceive:completionHandler:)` for auth challenges, redirects, incremental data (`didReceive data` chunks), `didBecomeDownloadTask`, `willCacheResponse`.

## 2. URL building & components

```swift
var comps = URLComponents()
comps.scheme = "https"
comps.host = "api.example.com"
comps.path = "/v1/users"
comps.queryItems = [URLQueryItem(name: "limit", value: "10"), URLQueryItem(name: "filter", value: "active")]
let url = comps.url!

// Or URL(string:) + percent encoding
let encoded = "user name".addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)!
let url = URL(string: "https://api.example.com/search?q=\(encoded)")!
```

- `URLComponents` — proper way to build URLs with query params; handles encoding automatically.
- `URLQueryItem` — name/value pairs for query strings.
- `URL(string:)` returns nil for invalid URLs — always handle.
- `URL(fileURLWithPath:)` — file URLs; `url.path`, `url.lastPathComponent`, `url.deletingLastPathComponent()`, `url.appendingPathComponent(_:)`, `url.appending(queryItems:)`.
- `URLRequest` `setValue` for headers — case-insensitive names.

## 3. JSON & Codable networking

```swift
struct APIClient {
  let session: URLSession
  let decoder: JSONDecoder = {
    let d = JSONDecoder()
    d.keyDecodingStrategy = .convertFromSnakeCase
    d.dateDecodingStrategy = .iso8601
    return d
  }()

  func get<T: Decodable>(_ path: String, as type: T.Type) async throws -> T {
    let url = URL(string: "https://api.example.com\(path)")!
    let (data, response) = try await session.data(from: url)
    guard let http = response as? HTTPURLResponse else { throw APIError.invalidResponse }
    guard (200..<300).contains(http.statusCode) else { throw APIError.httpError(http.statusCode) }
    return try decoder.decode(T.self, from: data)
  }
}
```

- Standard pattern: `data(from:)`/`data(for:)` + status check + decode.
- Custom `DecodingError` handling: `DecodingError.keyNotFound`, `.typeMismatch`, `.valueNotFound`, `.dataCorrupted` — wrap in domain error for better diagnostics.
- Polymorphic responses: decode envelope `struct Envelope<T: Codable>: Codable { let data: T; let meta: Meta }`.
- Pagination: cursor-based (`meta.nextCursor`) vs offset (`page`, `perPage`) — handle both.
- Error responses: decode `APIErrorResponse { error: { code, message } }` for structured errors.
- `JSONDecoder` date strategies: `.iso8601` (no fractional seconds), `.formatted(ISO8601DateFormatter)` with `.withFractionalSeconds`, `.secondsSince1970`, `.custom`.
- `JSONEncoder` for requests: `.iso8601`, `.secondsSince1970`, `.formatted`, `.custom`.
- `PropertyListEncoder`/`Decoder` for plist APIs (rare).
- `XMLParser`/`XMLDocument` for XML APIs (legacy but still used).

## 4. Authentication

### OAuth 2.0 / OIDC
- **ASWebAuthenticationSession** — THE way on iOS/macOS:
  ```swift
  let session = ASWebAuthenticationSession(url: authURL, callbackURLScheme: "myapp") { callbackURL, error in
    guard let callbackURL else { return }
    // Parse tokens from callbackURL query or exchange code
  }
  session.presentationContextProvider = self   // for iPad anchor
  session.prefersEphemeralWebBrowserSession = false  // true for private mode (no shared cookies)
  session.start()
  ```
  - Returns callback URL with `code` (authorization code flow) or tokens (implicit — deprecated, use PKCE).
  - PKCE: generate `code_verifier` (random 43-128 chars), `code_challenge` = BASE64URL(SHA256(verifier)); include in auth request; exchange code + verifier at token endpoint.
  - Token storage: **Keychain** (never UserDefaults); refresh token rotation; expiry handling.
  - `prefersEphemeralWebBrowserSession = true` for privacy-sensitive auth (banking etc.).

### API keys / bearer tokens
- Store in Keychain; attach via `Authorization: Bearer <token>` header.
- Refresh logic: intercept 401 → refresh token → retry original request; use actor/serial queue to prevent thundering herd.
- JWT: decode payload (base64) for claims (exp, sub, roles) — NEVER trust client-side for authz decisions; verify signature server-side.

### Certificate pinning
```swift
// URLSessionDelegate — evaluate server trust
func urlSession(_ session: URLSession, didReceive challenge: URLAuthenticationChallenge,
  completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
  guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
        let trust = challenge.protectionSpace.serverTrust else {
    completionHandler(.performDefaultHandling, nil); return
  }
  // Evaluate trust chain
  var error: CFError?
  let status = SecTrustEvaluateWithError(trust, &error)
  guard status, let chain = SecTrustCopyCertificateChain(trust) as? [SecCertificate] else {
    completionHandler(.cancelAuthenticationChallenge, nil); return
  }
  // Pin: check if any cert in chain matches pinned SPKI hash
  let pinnedHashes: Set<String> = ["sha256/base64-of-spki-hash"]
  let leafCert = chain[0]
  guard let publicKey = SecCertificateCopyKey(leafCert),
        let publicKeyData = SecKeyCopyExternalRepresentation(publicKey, nil) as Data? else {
    completionHandler(.cancelAuthenticationChallenge, nil); return
  }
  let hash = SHA256.hash(data: publicKeyData)
  let hashString = Data(hash).base64EncodedString()
  if pinnedHashes.contains(hashString) {
    completionHandler(.useCredential, URLCredential(trust: trust))
  } else {
    completionHandler(.cancelAuthenticationChallenge, nil)
  }
}
```
- Pin SPKI (Subject Public Key Info) hash, not cert hash — survives cert rotation.
- Include backup pins; pinning breaks when server rotates certs — have rotation plan.
- `URLSessionDelegate` must be retained (session holds weak ref? — actually session retains delegate until invalidated; keep strong ref to be safe).

## 5. WebSockets

```swift
let task = URLSession.shared.webSocketTask(with: url)
task.maximumMessageSize = 1_000_000   // 1MB
// task.maximumFrameSize = ...         // for fragmented messages
task.resume()

// Send
try await task.send(.string("{\"type\":\"ping\"}"))
try await task.send(.data(binaryData))

// Receive loop
while true {
  let message = try await task.receive()   // throws on close/error
  switch message {
  case .string(let text): handle(text)
  case .data(let data): handle(data)
  @unknown default: break
  }
}

// Ping/pong keepalive
task.sendPing { error in ... }

// Close
task.cancel(with: .normalClosure, reason: "bye".data(using: .utf8))
```
- `URLSessionWebSocketTask` — full-duplex over WS/WSS; `receive()` throws `NSError` on close.
- Close codes: `.normalClosure` (1000), `.goingAway` (1001), `.protocolError` (1002), `.unsupportedData` (1003), `.noStatusReceived` (1005), `.abnormalClosure` (1006), `.invalidFramePayloadData` (1007), `.policyViolation` (1008), `.messageTooBig` (1009), `.mandatoryExtensionMissing` (1010), `.internalServerError` (1011), `.tlsHandshakeFailure` (1015).
- Reconnection: exponential backoff with jitter; track last message ID for resume; handle `URLSessionWebSocketTask` close codes.
- Alternatives: `Starscream` (popular WS lib), `Network.framework NWConnection` for raw TCP/WebSocket.

## 6. Reachability & Network.framework

### NWPathMonitor (reachability)
```swift
import Network
let monitor = NWPathMonitor()
monitor.pathUpdateHandler = { path in
  print("Available: \(path.status == .satisfied)")
  print("Expensive: \(path.isExpensive)")      // cellular
  print("Constrained: \(path.isConstrained)")  // low data mode
  print("Interfaces: \(path.availableInterfaces.map { $0.type })")  // .wifi, .cellular, .wiredEthernet, .loopback
}
monitor.start(queue: .main)
```
- `NWPath.Status`: `.satisfied`, `.unsatisfied`, `.requiresConnection`.
- `NWPath` properties: `isExpensive` (cellular/hotspot), `isConstrained` (Low Data Mode), `usesInterfaceType`, `availableInterfaces` (`NWInterface` with `.type`, `.name`, `.index`).
- `NWInterface.InterfaceType`: `.other`, `.wifi`, `.cellular`, `.loopback`, `.wiredEthernet`, `.thunderbolt`.

### NWConnection (raw TCP/UDP/TLS)
```swift
let connection = NWConnection(host: NWEndpoint.Host("example.com"), port: 443, using: .tls)
connection.stateUpdateHandler = { state in ... }
connection.start(queue: .main)
connection.send(content: data, completion: .contentProcessed { error in })
connection.receive(minimumIncompleteLength: 1, maximumLength: 65536) { data, context, isComplete, error in }
```
- `NWConnection` — TCP/UDP with TLS; `state` (.setup, .waiting, .preparing, .ready, .failed, .cancelled); `send`, `receive`, `cancel`.
- `NWListener` — accept incoming connections: `NWListener(using:on:)` or `NWListener(using: .tls, on: port)`; `newConnectionHandler`.
- `NWBrowser`/`NWBrowser.Result` — Bonjour/mDNS service discovery: `NWBrowser(for: .bonjour(type: "_http._tcp", domain: nil))`.
- `NWParameters` — `tcp`, `udp`, `tls`, `quic`? (`NWParameters.tls`, `.dtls`? — QUIC via `NWProtocolQUIC`? not public API yet); `requiredInterfaceType`, `prohibitExpensivePaths`, `prohibitConstrainedPaths`, `allowLocalEndpointReuse`.
- `NWPathMonitor` is for reachability; `NWConnection`/`NWListener` for actual connections — use for custom protocols, Bonjour, local network.

## 7. Background transfers

```swift
// AppDelegate or SceneDelegate
func application(_ application: UIApplication, handleEventsForBackgroundURLSession identifier: String,
  completionHandler: @escaping () -> Void) {
  backgroundCompletionHandler = completionHandler
  // Recreate session with same identifier — system calls delegate when done
}

// Session config
let config = URLSessionConfiguration.background(withIdentifier: "com.example.downloads")
config.isDiscretionary = true          // let system schedule (power-aware)
config.sessionSendsLaunchEvents = true // wake app if needed
let session = URLSession(configuration: config, delegate: self, delegateQueue: nil)

// Download task
let task = session.downloadTask(with: url)
task.taskDescription = "filename.jpg"   // for resume
session.startTasks(withCompletionHandler: { ... })

// Delegate callbacks
func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didFinishDownloadingTo location: URL)
func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?)
func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didWriteData bytesWritten: Int64,
  totalBytesWritten: Int64, totalBytesExpectedToWrite: Int64)
func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask, didResumeAtOffset fileOffset: Int64,
  expectedTotalBytes: Int64)
```

- Background session survives app suspension; system wakes app to deliver completion.
- `isDiscretionary = true` — system may delay for power; `false` for user-initiated urgent downloads.
- `sessionSendsLaunchEvents` — app launched in background to handle events.
- `getAllTasks`/`getTasksWithCompletionHandler` — list pending tasks on relaunch.
- `handleEventsForBackgroundURLSession` — MUST be implemented in AppDelegate/SceneDelegate; store completion handler, call when all tasks done.
- Progress: `task.countOfBytesExpectedToReceive`/`countOfBytesReceived` for data/upload; delegate for download.
- Resume: on failure, `error.userInfo[NSURLSessionDownloadTaskResumeData]` contains resume data — save and use `downloadTask(withResumeData:)`.

## 8. App Transport Security (ATS)

- ATS is ON by default — requires HTTPS with TLS 1.2+, forward secrecy, valid cert.
- Exceptions via `NSAppTransportSecurity` in Info.plist:
  - `NSAllowsArbitraryLoads` = true — **AVOID** (App Review rejects unless justified); disables ATS entirely.
  - `NSAllowsArbitraryLoadsForMedia` = true — allows HTTP for media only (AVFoundation).
  - `NSAllowsArbitraryLoadsInWebContent` = true — allows HTTP in WKWebView only.
  - `NSExceptionDomains` — per-domain exceptions:
    ```xml
    <key>NSAppTransportSecurity</key>
    <dict>
      <key>NSExceptionDomains</key>
      <dict>
        <key>example.com</key>
        <dict>
          <key>NSExceptionAllowsInsecureHTTPLoads</key><true/>
          <key>NSExceptionMinimumTLSVersion</key><string>TLSv1.1</string>
          <key>NSIncludesSubdomains</key><true/>
          <key>NSThirdPartyExceptionRequiresForwardSecrecy</key><false/>
        </dict>
      </dict>
    </dict>
    ```
  - `NSRequiresCertificateTransparency`? — not a key; CT is automatic.
- HTTP/3: `assumesHTTP3Capable` on URLRequest for QUIC/HTTP3 hints.
- Local network: `NSLocalNetworkUsageDescription` + `NSBonjourServices` for Bonjour; `NWBrowser`/`NWListener` for mDNS.

## 9. Alamofire

```swift
import Alamofire

AF.request("https://api.example.com/users", method: .get, parameters: ["page": 1])
  .validate(statusCode: 200..<300)
  .responseDecodable(of: UsersResponse.self) { response in
    switch response.result {
    case .success(let users): ...
    case .failure(let error): ...
    }
  }

// Async/await (Alamofire 5.7+)
let users = try await AF.request(url).validate().serializingDecodable(UsersResponse.self).value

// Upload
AF.upload(multipartFormData: { formData in
  formData.append(imageData, withName: "photo", fileName: "photo.jpg", mimeType: "image/jpeg")
  formData.append("value", withName: "field")
}, to: url).responseDecodable(of: Response.self) { ... }

// Download
AF.download(url).downloadProgress { progress in ... }.response { response in ... }

// Interceptors (auth refresh, logging)
class AuthInterceptor: RequestInterceptor {
  func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
    var req = urlRequest
    req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    completion(.success(req))
  }
  func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
    // On 401: refresh token, retry with new token
  }
}
let session = Session(interceptor: AuthInterceptor())
```

- `Session` — Alamofire's URLSession wrapper; default `Session.default`.
- `Request` — chainable: `.validate()`, `.response()`, `.responseData`, `.responseString`, `.responseJSON`, `.responseDecodable`, `.downloadProgress`, `.uploadProgress`.
- `AF.upload` — multipart, file upload, resumable.
- `AF.download` — file download with progress, destination closure.
- `RequestInterceptor` — `adapt` (modify request) + `retry` (retry logic) — for auth, logging, retry.
- `ServerTrustManager` — cert pinning: `ServerTrustManager(evaluators: ["example.com": .publicKeys])` or `.pinnedCertificates`.
- `EventMonitor` — request lifecycle events for logging/metrics.
- Async/await: `.serializingDecodable`, `.serializingData`, `.serializingString`, `.serializingDownload`.
- **When to use Alamofire vs URLSession**: URLSession is sufficient for most; Alamofire adds interceptors, retry, validation, multipart helpers, nicer async API. For new projects, URLSession+async/await is often enough; Alamofire for complex auth/retry/multipart needs.

## 10. GraphQL

- **Apollo iOS** — standard GraphQL client: `ApolloClient`, codegen from `.graphql` schema + operations, normalized cache (`InMemoryNormalizedCache`), `fetch`, `watch`, `mutate` queries; supports SwiftUI integration.
- `swift-graphql`? — less common; Apollo dominates.
- For simple GraphQL: POST JSON `{"query": "...", "variables": {...}}` to endpoint, decode `data`/`errors` response.

## 11. Common networking bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| ATS error -1022 | HTTP blocked by ATS | use HTTPS, or add exception domain (justify) |
| Timeout on slow network | `timeoutIntervalForRequest` too low | increase, or `waitsForConnectivity = true` |
| Background download not completing | missing `handleEventsForBackgroundURLSession` | implement in AppDelegate/SceneDelegate, store completion handler |
| WebSocket disconnects | no keepalive/ping | send `sendPing` periodically, implement reconnect with backoff |
| Auth token expired mid-request | no refresh handling | interceptor retry on 401, refresh token flow |
| JSON decode fails | snake_case keys, date format mismatch | `keyDecodingStrategy`, custom `dateDecodingStrategy` |
| Cert pinning fails after rotation | pinned old cert | pin SPKI hash, include backup pins, plan rotation |
| Memory spike on large download | `data(from:)` loads all into memory | use `download(for:)` or `bytes(for:)` streaming |
| Duplicate requests on retry | no idempotency | use idempotency keys, dedupe in-flight requests |
| iOS 14+ local network denied | missing `NSLocalNetworkUsageDescription` + Bonjour services | add both, request permission via `NWBrowser`/`NWListener` |
| HTTP/3 not used | server/client negotiation | set `assumesHTTP3Capable` on request, ensure server supports |

## 12. Performance & best practices

- Reuse `URLSession` instances (don't create per-request); configure once.
- `URLCache` for HTTP caching: `session.configuration.urlCache` — respects `Cache-Control` headers; `.useProtocolCachePolicy` default.
- ETag/conditional requests: `If-None-Match` header → 304 Not Modified saves bandwidth.
- Compression: `Accept-Encoding: gzip, deflate, br` — URLSession handles automatically.
- Connection reuse: HTTP/1.1 keep-alive and HTTP/2 multiplexing handled by URLSession.
- Request coalescing: dedupe identical in-flight GETs (Alamofire does this? — implement if needed).
- Rate limiting: exponential backoff with jitter; respect `Retry-After` header.
- Idempotency: `Idempotency-Key` header for POST retries; safe retries only for GET/PUT/DELETE.
- Logging: `os_log`/`Logger` for request/response metadata (URL, method, status, duration) — never log tokens/bodies with PII.
- Metrics: `URLSessionTaskMetrics` via delegate `didFinishCollecting` — timing, protocol, connection info.
- Testing: `URLProtocol` subclass for mocking; inject `URLSession`/`URLSessionProtocol` for testability.

## 13. Privacy & security checklist

- [ ] HTTPS everywhere (ATS enforced); certificate pinning for sensitive APIs.
- [ ] Tokens in Keychain, not UserDefaults/files.
- [ ] Certificate transparency validation (automatic with ATS).
- [ ] App Transport Security exceptions documented + minimal.
- [ ] Local network permission requested only when needed.
- [ ] Privacy manifest declares network data collection if applicable.
- [ ] No sensitive data in URLs (use headers/body); URLs may be logged.
- [ ] Certificate pinning has backup pins + rotation plan.
- [ ] Background session identifier unique per app group.
- [ ] WebSocket uses WSS (TLS) not WS.
- [ ] OAuth uses PKCE + ASWebAuthenticationSession.
- [ ] JWT signature verified server-side; client only reads claims.
