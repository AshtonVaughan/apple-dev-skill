# Data Persistence Reference

Where to put data on Apple platforms: UserDefaults, files, Keychain, SwiftData, Core Data, CloudKit, CryptoKit. Choose by sensitivity + query needs + sync requirements.

## 1. Decision matrix

| Need | Use | Why |
|---|---|---|
| Small settings/flags | `UserDefaults` / `@AppStorage` | plist-backed, fast, per-app or suite |
| Secrets (tokens, passwords) | **Keychain** | encrypted at rest, biometric-gated, survives reinstall |
| Structured app data, queries, relationships | **SwiftData** (new) or Core Data | ORM-like, observed queries, migrations |
| Simple document files | Files + FileManager / security-scoped bookmarks | full control, user-visible |
| Relational SQL, complex queries | GRDB.swift / SQLite.swift | raw SQLite power |
| Cloud sync | CloudKit (native) or your backend | server-authoritative sync |
| Key-value cache | `NSCache` (memory) or file cache | auto-eviction |
| Large binary blobs | File system + thumbnail cache | don't put blobs in DB |

## 2. Codable (JSON & plist)

```swift
struct User: Codable, Sendable {
  let id: UUID
  let name: String
  let email: String?
  let createdAt: Date
  let role: Role          // enum: Codable synthesizes raw-value or associated-value coding
}

let encoder = JSONEncoder()
encoder.outputFormatting = [.prettyPrinted, .sortedKeys, .withoutEscapingSlashes]
encoder.dateEncodingStrategy = .iso8601                    // or .secondsSince1970, .formatted(formatter), .custom
encoder.keyEncodingStrategy = .convertToSnakeCase          // name→name_snake? actually camel→snake for keys
encoder.dataEncodingStrategy = .base64                     // Data → base64 string

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
// .convertFromSnakeCase handles "user_name"→userName but NOT "URL"→url edge cases — custom if needed
decoder.dateDecodingStrategy = .iso8601
// ISO8601 with fractional seconds needs custom formatter:
let f = ISO8601DateFormatter(); f.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
decoder.dateDecodingStrategy = .custom { container in
  let s = try container.singleValueContainer().decode(String.self)
  guard let d = f.date(from: s) else { throw DecodingError.dataCorruptedError(...) }
  return d
}
```

- Synthesis rules: all stored props `Codable` → auto `Codable`; `let` with default values still must decode (unless custom init); optionals decode missing keys as nil.
- `CodingKeys` enum to rename/map keys; nested containers `container(keyedBy:)`, `nestedContainer(keyedBy:forKey:)`, `unkeyedContainer`.
- `singleValueContainer` for primitives; `superDecoder`/`superEncoder` for inheritance-like flattening.
- Polymorphic APIs: decode `[String: JSONValue]` first or use `AnyCodable` wrapper / custom `init(from:)` switching on `type` field.
- `PropertyListEncoder`/`PropertyListSerialization` — plist format (XML/binary) for config files; `NSKeyedArchiver` — legacy object graphs (avoid for new data; not forward-compatible).
- `JSONSerialization` — untyped `[String: Any]` JSON; prefer Codable.
- `XMLParser`/`XMLDocument` (FoundationXML on Linux) — XML parsing.
- `PropertyListDecoder`/`Encoder` for `.plist` files (NSUserDefaults format).

## 3. UserDefaults

```swift
UserDefaults.standard.set(true, forKey: "hasOnboarded")
UserDefaults.standard.bool(forKey: "hasOnboarded")        // returns false if missing
UserDefaults.standard.string(forKey: "name")
UserDefaults.standard.object(forKey: "key") as? [String: Any]
UserDefaults.standard.register(defaults: ["key": value])  // in-memory defaults (not persisted)
UserDefaults(suiteName: "group.com.app.shared")           // app groups / shared containers
UserDefaults.standard.removeObject(forKey: "key")
UserDefaults.standard.synchronize()                        // DEPRECATED — don't call; system syncs
```

- SwiftUI: `@AppStorage("key") var value = default` — two-way binding to UserDefaults.
- Limits: plist types only (String, Number, Bool, Data, Date, Array, Dictionary, URL); ~1MB practical; NOT for secrets (plaintext plist, readable via backup); NOT for large data.
- `volatileDomain` for session-only: `setVolatileDomain(_:forName:)`.
- Observation: KVO on `UserDefaults.didChangeNotification` or per-key; `@AppStorage` handles SwiftUI updates.
- App Groups: `UserDefaults(suiteName: "group.com.team.app")` shares between app+extensions — requires `com.apple.security.application-groups` entitlement.

## 4. Keychain

```swift
import Security

// Add item
let query: [String: Any] = [
  kSecClass as String: kSecClassGenericPassword,
  kSecAttrAccount as String: "user@example.com",
  kSecAttrService as String: "com.example.app.auth",
  kSecValueData as String: token.data(using: .utf8)!,
  kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
]
SecItemAdd(query as CFDictionary, nil)

// Read
let readQuery: [String: Any] = [
  kSecClass as String: kSecClassGenericPassword,
  kSecAttrAccount as String: "user@example.com",
  kSecAttrService as String: "com.example.app.auth",
  kSecReturnData as String: true,
  kSecMatchLimit as String: kSecMatchLimitOne,
]
var result: AnyObject?
SecItemCopyMatching(readQuery as CFDictionary, &result)
let data = result as? Data

// Update / Delete
SecItemUpdate(query as CFDictionary, [kSecValueData as String: newData] as CFDictionary)
SecItemDelete(query as CFDictionary)
```

- Classes: `kSecClassGenericPassword`, `kSecClassInternetPassword`, `kSecClassCertificate`, `kSecClassKey`, `kSecClassIdentity`.
- Attributes: `kSecAttrAccount`, `kSecAttrService`, `kSecAttrAccessGroup`, `kSecAttrLabel`, `kSecAttrDescription`, `kSecAttrComment`, `kSecAttrCreationDate`, `kSecAttrModificationDate`, `kSecAttrAccessible` (protection class), `kSecAttrSynchronizable` (iCloud Keychain sync).
- **Accessibility** (`kSecAttrAccessible`):
  - `kSecAttrAccessibleWhenUnlocked` — default, device only
  - `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` — no iCloud sync/backup
  - `kSecAttrAccessibleAfterFirstUnlock` — background access after first unlock
  - `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`
  - `kSecAttrAccessibleAlways` — **deprecated/avoid** (background access anytime)
  - `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` — requires passcode
- **Access control** (`kSecAttrAccessControl`): `SecAccessControlCreateWithFlags` with `.userPresence` (biometry OR passcode), `.biometryAny`, `.biometryCurrentSet`, `.devicePasscode`, `.or`, `.and` — for biometric-gated secrets.
- **Access groups**: `kSecAttrAccessGroup` = `TEAMID.com.example.shared` — share keychain items between apps (requires `keychain-access-groups` entitlement).
- iCloud Keychain sync: `kSecAttrSynchronizable = kSecAttrSynchronizableAny` + iCloud Keychain capability.
- Errors: `errSecItemNotFound` (-25300), `errSecDuplicateItem` (-25299), `errSecAuthFailed` (-25293), `errSecInteractionNotAllowed` (-25308 — device locked), `errSecMissingEntitlement` (-34018).
- Libraries: `KeychainAccess` (kishikawakatsumi), `Valet` (Square) — simpler APIs.
- NEVER store secrets in UserDefaults, files, or code. Keychain persists across reinstalls (this is a feature AND a gotcha — clean up on logout).

## 5. File system

```swift
let fm = FileManager.default
let docs = fm.urls(for: .documentDirectory, in: .userDomainMask).first!
let caches = fm.urls(for: .cachesDirectory, in: .userDomainMask).first!
let tmp = fm.temporaryDirectory
let appSupport = try fm.url(for: .applicationSupportDirectory, in: .userDomainMask, appropriateFor: nil, create: true)

// Directory enumeration
let contents = try fm.contentsOfDirectory(at: docs, includingPropertiesForKeys: [.fileSizeKey, .creationDateKey])
let enumerator = fm.enumerator(at: docs, includingPropertiesForKeys: nil)

// File attributes
let attrs = try fm.attributesOfItem(atPath: path)   // [.size, .modificationDate, .creationDate]
let values = try url.resourceValues(forKeys: [.fileSizeKey, .isDirectoryKey])

// Copy/move/delete
try fm.copyItem(at: src, to: dst)
try fm.moveItem(at: src, to: dst)
try fm.removeItem(at: url)

// Security-scoped bookmarks (sandboxed macOS/iOS)
let bookmark = try url.bookmarkData(options: [.withSecurityScope], includingResourceValuesForKeys: nil, relativeTo: nil)
let resolved = try URL(resolvingBookmarkData: bookmark, options: [], relativeTo: nil, bookmarkDataIsStale: &stale)
let ok = resolved.startAccessingSecurityScopedResource()
defer { resolved.stopAccessingSecurityScopedResource() }

// Atomic writes
do { try data.write(to: url, options: .atomic) } catch { ... }
```

- **Directory purposes** (`FileManager.SearchPathDirectory`): `.documentDirectory` (user-visible, backed up, iTunes-visible), `.cachesDirectory` (purgeable, NOT backed up), `.applicationSupportDirectory` (supporting files, backed up), `.libraryDirectory`, `.temporaryDirectory` (purgeable anytime), `.downloadsDirectory`.
- iOS: app sandbox container — `NSHomeDirectory()` = app container; `Documents/`, `Library/`, `tmp/`.
- macOS sandbox: container at `~/Library/Containers/com.app.id`; security-scoped bookmarks for user files outside container.
- Backup rules: exclude regenerable data from backup via `NSURLIsExcludedFromBackupKey` resource value or `.cachesDirectory` (auto-excluded).
- File coordination: `NSFileCoordinator`/`NSFilePresenter` for coordinated reads/writes (iCloud Drive, shared docs).
- `FileHandle` — low-level read/write: `FileHandle(forReadingFrom:)`, `seekToEnd`, `write`, `readData(ofLength:)`, `availableData`, `synchronizeFile`.
- `OutputStream`/`InputStream` — stream APIs.
- `NSData`/`NSDataWritingOptions` — `.atomic`, `.withoutOverwriting`.
- **Background URLSession downloads** — see `07-networking.md`.

## 6. SwiftData (iOS 17+/macOS 14+)

```swift
import SwiftData

@Model
final class Note {
  var title: String = ""
  var body: String = ""
  var createdAt: Date = Date()
  var tags: [Tag]? = []                    // to-many relationship
  @Attribute(.unique) var slug: String?   // unique constraint
  init(title: String, body: String) { self.title = title; self.body = body }
}

@Model
final class Tag {
  var name: String = ""
  @Relationship(inverse: \Note.tags) var notes: [Note]? = []
  @Relationship(deleteRule: .cascade) var subNotes: [Note]?  // cascade delete
}

// App setup
@main struct MyApp: App {
  var body: some Scene {
    WindowGroup { ContentView() }
      .modelContainer(for: [Note.self, Tag.self])   // or custom ModelContainer
  }
}

// In view
struct ContentView: View {
  @Query(sort: \Note.createdAt, order: .reverse) private var notes: [Note]
  @Query(filter: #Predicate<Note> { $0.title.contains("swift") }) private var swiftNotes: [Note]
  @Environment(\.modelContext) private var context
  var body: some View { List(notes) { note in Text(note.title) } }
}
```

- `@Model` macro — turns class into persistent model: generates schema, `PersistentModel` conformance, observation tracking.
- `ModelContainer` — database container: `ModelContainer(for:schema:isStoredInMemoryOnly:cloudKitDatabase:)`; `.modelContainer(for:)` on Scene.
- `ModelContext` — unit-of-work: `context.insert(note)`, `context.delete(note)`, `context.save()`, `context.rollback()`, `context.fetch(FetchDescriptor<Note>())`.
- `@Query` — view-bound fetch: `sort:`, `order:`, `filter:` (`#Predicate` macro or `Predicate`), `animation:`; auto-updates when model changes.
- `#Predicate` macro — type-safe predicate builder → `NSPredicate`/SQL; supports comparisons, `contains`, `hasPrefix`, `&&`, `||`, `!`, `if/switch` in builder; NOT all NSPredicate features.
- `FetchDescriptor` — programmatic fetch: `predicate:`, `sortBy: [SortDescriptor]`, `fetchLimit`, `fetchOffset`, `relationshipKeyPathsForPrefetching`.
- Relationships: `@Relationship` — `deleteRule` (.cascade/.nullify/.deny/.noAction), `inverse:`, optional arrays.
- `@Attribute` — `.unique`, `.externalStorage` (large blobs to disk), `.preserveValueOnDeletion`, `.spotlight` (Core Spotlight indexing), `.ephemeral` (non-persisted).
- Migrations: `VersionedSchema` + `SchemaMigrationPlan` — `SchemaV1`, `SchemaV2` with `MigrationStage.lightweight`/`custom`; `willMigrate`/`didMigrate`.
- CloudKit sync: `ModelConfiguration(cloudKitDatabase: .private("iCloud.com.app"))` — automatic sync via CloudKit private DB; requires iCloud capability + CloudKit container.
- **Limitations**: no batch updates/deletes (must fetch+delete), no derived attributes, limited predicate expressiveness vs Core Data, no fetch request templates, no `NSFetchedResultsController` equivalent (use `@Query` or `ModelContext.didSave` notifications), enum storage as raw values, no compound predicates beyond `&&`/`||`.
- Performance: `@Query` on main context is fine for UI; background contexts via `ModelContext(container)` on background queues; prefetch relationships via `relationshipKeyPathsForPrefetching`.
- Testing: `ModelConfiguration(isStoredInMemoryOnly: true)` for unit tests.

## 7. Core Data (legacy but still used)

```swift
let container = NSPersistentContainer(name: "Model")
container.loadPersistentStores { desc, error in ... }
let context = container.viewContext          // main-queue context
let bgContext = container.newBackgroundContext()  // background queue

let fetch = NSFetchRequest<Note>(entityName: "Note")
fetch.predicate = NSPredicate(format: "title CONTAINS %@", "swift")
fetch.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
fetch.fetchLimit = 20
fetch.relationshipKeyPathsForPrefetching = ["tags"]
let results = try context.fetch(fetch)

// Batch operations (efficient for bulk)
let batchDelete = NSBatchDeleteRequest(fetchRequest: fetch)
try context.execute(batchDelete)
context.reset()   // batch ops bypass context — must reset

// NSFetchedResultsController — efficient table/collection updates
let frc = NSFetchedResultsController(fetchRequest: fetch, managedObjectContext: context,
  sectionNameKeyPath: nil, cacheName: "NotesCache")
frc.delegate = self
try frc.performFetch()
```

- Stack: `NSPersistentContainer` → `NSManagedObjectContext` (scratchpad) → `NSManagedObject` (entity instances) → `NSPersistentStoreCoordinator` → SQLite/CloudKit.
- Contexts: `viewContext` (main), `newBackgroundContext()` (private queue); `perform`/`performAndWait` for thread-safe access; `mergeChanges`/`mergePolicy` for cross-context sync.
- `NSManagedObjectContext` `automaticallyMergesChangesFromParent` for background→main merge.
- CloudKit mirroring: `NSPersistentCloudKitContainer` (iOS 13+) — automatic CloudKit sync; requires CloudKit capability + container.
- Migrations: lightweight (automatic for additive changes) vs staged/custom (`NSMappingModel`, `NSMigrationManager`).
- `NSFetchedResultsController` — efficient UI updates with section support; delegate callbacks for incremental updates.
- `NSBatchInsertRequest`/`NSBatchUpdateRequest`/`NSBatchDeleteRequest` — bypass context for bulk ops.
- **When to choose Core Data over SwiftData**: existing Core Data codebase, need `NSFetchedResultsController`/batch ops/complex migrations, CloudKit mirroring maturity, or SwiftData limitations block you.

## 8. GRDB.swift / SQLite.swift / Realm

### GRDB.swift (recommended for raw SQLite)
```swift
import GRDB
let dbQueue = try DatabaseQueue(path: "db.sqlite")
try dbQueue.write { db in
  try db.create(table: "players", ifNotExists: true) { t in
    t.autoIncrementedPrimaryKey("id")
    t.column("name", .text).notNull()
    t.column("score", .integer).notNull()
  }
  try db.execute(sql: "INSERT INTO players (name, score) VALUES (?, ?)", arguments: ["Alice", 100])
}
let players = try dbQueue.read { db in
  try Player.fetchAll(db)   // or Row.fetchAll(db)
}
```
- Type-safe queries via `SQLRequest`/`SQLInterpolation`; `FetchableRecord`/`PersistableRecord` protocols; migrations via `DatabaseMigrator`; observation via `ValueObservation`/`SharedValueObservation`.
- WAL mode, FTS5, JSON1, custom functions — full SQLite power.

### SQLite.swift
```swift
import SQLite
let db = try Connection("path/to/db.sqlite3")
try db.run("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
try db.run("INSERT INTO users (name) VALUES (?)", "Alice")
for user in try db.prepare("SELECT * FROM users") { print(user["name"] as String) }
```
- Lightweight wrapper; good for simple use cases; GRDB preferred for serious work.

### Realm (MongoDB)
- `let realm = try Realm()` — object database; `@Persisted` property wrapper; `Results<T>` live-updating collections; `Realm(configuration:)` for multiple realms; `writeAsync` for background writes; sync via Atlas Device Sync (MongoDB cloud).
- Note: Realm's future under MongoDB ownership is uncertain — check current status before adopting for new projects.

## 9. CloudKit

```swift
import CloudKit
let container = CKContainer(identifier: "iCloud.com.example.app")
let db = container.privateCloudDatabase   // or .publicCloudDatabase, .sharedCloudDatabase

// Save record
let record = CKRecord(recordType: "Note", recordID: CKRecord.ID(recordName: UUID().uuidString))
record["title"] = "My Note" as CKRecordValue
_ = try await db.save(record)

// Query
let predicate = NSPredicate(format: "title CONTAINS %@", "swift")
let query = CKQuery(recordType: "Note", predicate: predicate)
let (results, cursor) = try await db.records(matching: query, resultsLimit: 50)
for await (recordID, result) in results { ... }

// Subscriptions (push notifications for changes)
let subscription = CKDatabaseSubscription(subscriptionID: "allNotes")
let notificationInfo = CKSubscription.NotificationInfo()
notificationInfo.shouldSendContentAvailable = true
subscription.notificationInfo = notificationInfo
_ = try await db.save(subscription)

// Zones for atomic batch operations
let zone = CKRecordZone(zoneName: "NotesZone")
_ = try await db.modifyRecordZones(saving: [zone], deleting: [])
```

- Databases: `privateCloudDatabase` (user's private iCloud), `publicCloudDatabase` (app-wide shared), `sharedCloudDatabase` (shared records between users).
- Record types: user-defined schemas in CloudKit Dashboard; `CKRecord` key-value storage; `CKRecord.Reference` for relationships.
- Queries: `CKQuery` with `NSPredicate` (limited predicate syntax — no compound OR beyond simple cases, no CONTAINS on all fields); `querySortDescriptors`; `resultsLimit`; cursor pagination via `CKQueryOperation`/`records(matching:resultsLimit:desiredKeys:)`.
- Subscriptions: `CKDatabaseSubscription` (all changes), `CKRecordZoneSubscription` (zone changes), `CKQuerySubscription` (matching query changes) → silent pushes.
- Zones: `CKRecordZone` for atomic batch saves/deletes; custom zones for user-scoped data.
- Assets: `CKAsset` for large files (photos, videos) — uploaded/downloaded separately.
- Sync engines: `CKSyncEngine` (iOS 17+) — Apple-provided sync engine for record zones; handles conflict resolution, batching, subscriptions automatically. **Use this for new CloudKit sync** instead of hand-rolled.
- Entitlements: `com.apple.developer.icloud-container-identifiers`, `com.apple.developer.icloud-services` (CloudKit, CloudDocuments, key-value-store).
- Cost/quotas: CloudKit has operation limits (reads/writes per second, storage per user) — monitor in dashboard; use zones + subscriptions efficiently.

## 10. CryptoKit

```swift
import CryptoKit

// Symmetric encryption (AES-GCM)
let key = SymmetricKey(size: .bits256)              // or .bits128, .bits192
let nonce = AES.GCM.Nonce()                          // random 96-bit
let plaintext = "secret".data(using: .utf8)!
let ciphertext = try AES.GCM.seal(plaintext, using: key, nonce: nonce)
let combined = ciphertext.combined!                   // nonce+ciphertext+tag
let box = try AES.GCM.SealedBox(combined: combined)
let decrypted = try AES.GCM.open(box, using: key)

// Key derivation
let derivedKey = HKDF<SHA256>.deriveKey(
  inputKeyMaterial: passwordData,
  salt: salt,
  info: "app-auth".data(using: .utf8)!,
  outputByteCount: 32)

// Hashing
let hash = SHA256.hash(data: data)
let hashString = hash.map { String(format: "%02x", $0) }.joined()

// Key agreement (Curve25519)
let privateKey = Curve25519.KeyAgreement.PrivateKey()
let publicKey = privateKey.publicKey
let sharedSecret = try privateKey.sharedSecretFromKeyAgreement(with: peerPublicKey)
let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(using: SHA256.self, salt: salt, sharedInfo: info, outputByteCount: 32)

// Signing (P256)
let signingKey = P256.Signing.PrivateKey()
let signature = try signingKey.signature(for: data)
let isValid = signingKey.publicKey.isValidSignature(signature, for: data)
```

- Algorithms: `AES.GCM` (authenticated encryption), `ChaChaPoly` (authenticated encryption, faster on some ARM), `SHA256`/`SHA384`/`SHA512`/`SHA512_256`/`SHA3_256`/`SHA3_384`/`SHA3_512`/`SHAKE128`/`SHAKE256` (hashing), `Curve25519` (key agreement), `P256`/`P384`/`P521`/`Curve25519.Signing` (signatures), `HKDF`/`PBKDF2` (key derivation).
- `SymmetricKey(size:)` — random symmetric key; `SymmetricKey(data:)` — from existing data.
- `AES.GCM.Nonce()` — random nonce; `AES.GCM.SealedBox(nonce:ciphertext:tag:)` — manual construction.
- `SecureEnclave` — hardware-backed keys: `SecureEnclave.P256.Signing.PrivateKey()` — keys never leave enclave; `SecKeyCreateRandomKey` with `kSecAttrTokenIDSecureEnclave`.
- CommonCrypto (`CCCrypt`, `CC_SHA256`) — legacy C API, still works but CryptoKit preferred.
- NEVER roll your own crypto — use CryptoKit/Security framework primitives.

## 11. App Groups & shared containers

- `com.apple.security.application-groups` entitlement: `["group.com.example.shared"]`.
- Shared container URL: `FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.example.shared")`.
- Shared UserDefaults: `UserDefaults(suiteName: "group.com.example.shared")`.
- Shared Keychain: access group `TEAMID.group.com.example.shared` + `keychain-access-groups` entitlement.
- Use for: app + extensions (widgets, share extensions, watch apps) sharing data.

## 12. Privacy manifest requirements

- `PrivacyInfo.xcprivacy` file required for apps using "required reason APIs" (file timestamp, disk space, system boot time, UserDefaults, active keyboard) or collecting data.
- Required reason APIs declare WHY you use them: `NSPrivacyAccessedAPICategoryFileTimestamp` (reason: `C617.1` app functionality, `0A2A.1` third-party SDK), `NSPrivacyAccessedAPICategorySystemBootTime`, `NSPrivacyAccessedAPICategoryDiskSpace`, `NSPrivacyAccessedAPICategoryUserDefaults` (reason `CA92.1` app functionality, `1C8F.1` third-party SDK, `C56D.1` app group sharing), `NSPrivacyAccessedAPICategoryActiveKeyboard`.
- Collected data types: `NSPrivacyCollectedDataType` — declare what you collect (email, name, photos, location, usage data, diagnostics, etc.) with purpose (app functionality, analytics, advertising, product personalization, developer advertising).
- Tracking domains: `NSPrivacyTrackingDomains` — domains used for tracking (ATT).
- Third-party SDKs must include their own privacy manifests — check dependencies.

## 13. Common data bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| SwiftData query not updating | `@Query` on wrong context or model not observed | ensure `ModelContainer` injected, model properties are `var` not `let` |
| Keychain errSecMissingEntitlement | missing keychain-access-groups or wrong access group | add entitlement, use `TEAMID.group` format |
| Keychain item persists after logout | `kSecAttrSynchronizable` or accessible class too permissive | use `WhenUnlockedThisDeviceOnly`, delete on logout |
| UserDefaults not persisting | calling on wrong suite, or exceeding size | check suite name, keep data small |
| File write fails sandboxed | missing entitlement or security-scoped access | add entitlement, use bookmark for user files |
| CloudKit sync conflicts | last-write-wins by default | use `CKSyncEngine` for proper conflict resolution |
| SwiftData migration crash | schema change without migration plan | add `VersionedSchema` + `SchemaMigrationPlan` |
| Core Data merge conflicts | two contexts editing same object | use merge policies, `NSMergeByPropertyObjectTrumpMergePolicy` |
| Memory spike loading JSON | decoding huge array into memory | stream with `JSONDecoder` + `AsyncSequence`, or paginate |
| GRDB SQLITE_BUSY | concurrent writes | use single `DatabaseQueue` for writes, `DatabasePool` for concurrent reads |
| iCloud file not downloading | ubiquity container not configured | check iCloud capability, `NSMetadataQuery` for download status |
