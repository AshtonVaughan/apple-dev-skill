# Persistence Advanced — Deep Reference

Beyond `06-data-persistence.md`: Core Data internals, SwiftData advanced, GRDB deep, Keychain deep, secure file protection, migration strategies.

## 1. Core Data deep

### Contexts & concurrency
```swift
let container = NSPersistentContainer(name: "Model")
container.loadPersistentStores { desc, error in ... }

// View context (main queue)
let viewContext = container.viewContext
viewContext.automaticallyMergesChangesFromParent = true
viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
viewContext.shouldDeleteInaccessibleFaults = true

// Background context (private queue)
let bgContext = container.newBackgroundContext()
bgContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
bgContext.perform {
  let object = bgContext.object(with: objectID)
  object?.setValue(newValue, forKey: "property")
  try? bgContext.save()
}
```

- `NSManagedObjectContext` — `viewContext` (main), `newBackgroundContext()` (private), `parentContext`, `childContext`, `perform(_:)`/`performAndWait(_:)` (thread-safe), `save()`, `rollback()`, `reset()`, `mergeChanges(fromContextDidSave:)`, `automaticallyMergesChangesFromParent`, `mergePolicy`, `shouldDeleteInaccessibleFaults`, `retainsRegisteredObjects`, `stalenessInterval`, `fetchRequestTemplatesByName`, `allowsExternalBinaryDataStorage`.
- `NSMergePolicy` — `NSMergeByPropertyObjectTrumpMergePolicy` (property-level), `NSMergeByPropertyStoreTrumpMergePolicy` (store-level), `NSOverwriteMergePolicy`, `NSRollbackMergePolicy`, `NSMergePolicyType`.
- `NSManagedObjectContext` `parentContext` — parent-child contexts for nested saves.
- `NSManagedObjectContext` `childContext` — `newChildContext(with:)` for private queue child.
- `NSManagedObjectContext` `perform`/`performAndWait` — thread-safe access.
- `NSManagedObjectContext` `save()` — save changes; `hasChanges`, `hasSaveErrors`? — check.
- `NSManagedObjectContext` `rollback()` — discard changes.
- `NSManagedObjectContext` `reset()` — clear context.
- `NSManagedObjectContext` `mergeChanges(fromContextDidSave:)` — merge remote changes.
- `NSManagedObjectContext` `automaticallyMergesChangesFromParent` — auto-merge parent changes.
- `NSManagedObjectContext` `mergePolicy` — merge policy.
- `NSManagedObjectContext` `shouldDeleteInaccessibleFaults` — delete inaccessible faults.
- `NSManagedObjectContext` `retainsRegisteredObjects` — retain registered objects.
- `NSManagedObjectContext` `stalenessInterval` — staleness interval.
- `NSManagedObjectContext` `fetchRequestTemplatesByName` — fetch request templates.
- `NSManagedObjectContext` `allowsExternalBinaryDataStorage` — external binary storage.

### Fetching & NSFetchedResultsController
```swift
let fetchRequest = NSFetchRequest<Note>(entityName: "Note")
fetchRequest.predicate = NSPredicate(format: "title CONTAINS %@", "swift")
fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
fetchRequest.fetchLimit = 20
fetchRequest.fetchOffset = 0
fetchRequest.relationshipKeyPathsForPrefetching = ["tags", "author"]
fetchRequest.propertiesToFetch = ["title", "createdAt"]
fetchRequest.includesPropertyValues = false
fetchRequest.resultType = .managedObjectResultType
let results = try context.fetch(fetchRequest)

// NSFetchedResultsController
let frc = NSFetchedResultsController(fetchRequest: fetchRequest, managedObjectContext: context,
  sectionNameKeyPath: nil, cacheName: "NotesCache")
frc.delegate = self
try frc.performFetch()
// frc.fetchedObjects, frc.sections, frc.indexPath(forObject:)
```

- `NSFetchRequest` — `entity`, `entityName`, `predicate`, `sortDescriptors`, `fetchLimit`, `fetchOffset`, `fetchBatchSize`, `relationshipKeyPathsForPrefetching`, `propertiesToFetch`, `includesPropertyValues`, `resultType`, `includesSubentities`, `returnsObjectsAsFaults`, `propertiesToGroupBy`, `havingProperties`, `fetchRequestTemplatesByName`? — check.
- `NSPredicate` — `format:`, `predicateWithFormat:`, `predicateWithSubstitutionVariables:`, `compoundPredicateWithType:subpredicates:`, `predicateWithBlock:`, `predicateWithValue:`, `predicateWithComparisonPredicateWithBlock:`.
- `NSSortDescriptor` — `key`, `ascending`, `selector`, `reversed`.
- `NSFetchedResultsController` — `delegate`, `fetchRequest`, `managedObjectContext`, `sectionNameKeyPath`, `cacheName`, `fetchedObjects`, `sections`, `indexPath(forObject:)`, `object(at:)`, `section(for:)`, `indexPath(forSection:)`, `performFetch()`, `sections`.
- `NSFetchedResultsControllerDelegate` — `controllerWillChangeContent`, `controller(_:didChange:at:for:newIndexPath:)`, `controller(_:didChange:at:for:with:)`, `controllerDidChangeContent`, `controller(_:sectionIndexTitle:for:)`, `controller(_:sectionIndexTitleForSectionAt:)`, `controller(_:didChangeAnObject:at:for:newIndexPath:)`.
- `NSFetchedResultsSectionInfo` — `name`, `indexTitle`, `objects`, `numberOfObjects`.
- `NSBatchDeleteRequest` — `fetchRequest`, `resultType`; `NSBatchUpdateRequest` — `fetchRequest`, `propertiesToUpdate`, `resultType`; `NSBatchInsertRequest` — `entity`, `objects`, `resultType`.
- `NSManagedObjectContext` `execute(_:)` — execute batch requests.

### Migrations
```swift
// Lightweight migration (automatic for additive changes)
let container = NSPersistentContainer(name: "Model")
container.persistentStoreDescriptions.first?.shouldMigrateStoreAutomatically = true
container.persistentStoreDescriptions.first?.shouldInferMappingModelAutomatically = true

// Staged migration
let migrationManager = NSMigrationManager(sourceModel: oldModel, destinationModel: newModel)
try migrationManager.migrateStore(from: oldURL, to: newURL, mappingModel: mappingModel,
  progressHandler: { request in ... })
```

- Lightweight migration — automatic for additive changes (new entity, new attribute, new relationship).
- Staged migration — `NSMappingModel` for complex changes (renames, type changes, splits).
- `NSMigrationManager` — `migrateStore(from:to:mappingModel:progressHandler:)`.
- `NSMappingModel` — `mappingModelWithClassName:`, `init(fromXML:)`, `entityMappingByName`.
- `NSEntityMapping` — `sourceEntityName`, `destinationEntityName`, `mappingType` (`.custom`, `.add`, `.copy`, `.remove`, `.transform`, `.value`, `.subset`), `attributeMappings`, `relationshipMappings`, `userInfo`.
- `NSAttributeMapping` — `sourceAttributeName`, `destinationAttributeName`, `mappingType`, `valueExpression`, `userInfo`.
- `NSRelationshipMapping` — `sourceRelationshipName`, `destinationRelationshipName`, `mappingType`, `valueExpression`, `userInfo`.
- `NSExpressionDescription` — `expression`, `expressionResultType`, `expressionResultClassName`.
- `NSMigrationManager` `sourceModel`, `destinationModel`.
- `NSPersistentStoreCoordinator` — `addPersistentStore`, `removePersistentStore`, `replacePersistentStore`, `destroyPersistentStore`, `stores`, `managedObjectModel`, `metadata(forPersistentStoreOfType:at:)`.
- `NSPersistentStoreDescription` — `type` (`.sqlite`, `.binary`, `.xml`), `url`, `options`, `shouldMigrateStoreAutomatically`, `shouldInferMappingModelAutomatically`.

### CloudKit mirroring
```swift
let container = NSPersistentCloudKitContainer(name: "Model")
container.loadPersistentStores { desc, error in ... }
// Automatic CloudKit sync — requires CloudKit capability + container
```

- `NSPersistentCloudKitContainer` — `options` (`NSPersistentCloudKitContainerOptions`), `initializeCloudKitSchema`, `setupRemoteChangeNotifications`.
- `NSPersistentCloudKitContainerOptions` — `databaseScope` (`.private`, `.shared`, `.public`), `containerIdentifier`.
- `NSPersistentHistoryToken` — `fetchHistory`, `historyFetchRequest`.
- `NSPersistentHistoryChangeRequest` — `resultType` (`.dictionaryResultType`, `.managedObjectResultType`), `transactionIDs`.
- `NSPersistentHistoryResult` — `resultType`, `transactionID`, `storeUUID`.
- `NSPersistentHistoryChange` — `changeType` (`.insert`, `.update`, `.delete`), `changedObjectID`, `changedObjectType`, `storeUUID`, `transactionID`, `queryGeneration`.
- `NSPersistentHistoryToken` — `fetchHistory` token.
- `NSPersistentHistoryChangeRequest` — `from`, `to`, `resultType`, `transactionIDs`.
- `NSPersistentHistoryResult` — `resultType`, `transactionID`, `storeUUID`.
- `NSPersistentHistoryChange` — `changeType`, `changedObjectID`, `changedObjectType`, `storeUUID`, `transactionID`, `queryGeneration`.

## 2. SwiftData advanced

### Migrations
```swift
enum SchemaV1: VersionedSchema {
  static var versionIdentifier = Schema.Version(1, 0, 0)
  static var models: [any PersistentModel.Type] { [Note.self] }

  @Model final class Note {
    var title: String = ""
    var body: String = ""
  }
}

enum SchemaV2: VersionedSchema {
  static var versionIdentifier = Schema.Version(2, 0, 0)
  static var models: [any PersistentModel.Type] { [Note.self] }

  @Model final class Note {
    var title: String = ""
    var body: String = ""
    var tags: [Tag]? = []
  }
}

enum MigrationPlan: SchemaMigrationPlan {
  static var schemas: [any VersionedSchema.Type] { [SchemaV1.self, SchemaV2.self] }
  static var stages: [MigrationStage] { [migrateV1toV2] }
  static let migrateV1toV2 = MigrationStage.lightweight(fromVersion: SchemaV1.self, toVersion: SchemaV2.self)
}

let container = try ModelContainer(for: SchemaV2.self, migrationPlan: MigrationPlan.self)
```

- `VersionedSchema` — `versionIdentifier`, `models`.
- `SchemaMigrationPlan` — `schemas`, `stages`, `MigrationStage` (`.lightweight`, `.custom`).
- `MigrationStage.lightweight` — automatic migration.
- `MigrationStage.custom` — custom migration with `willMigrate`/`didMigrate`.
- `ModelConfiguration` — `url`, `cloudKitDatabase`, `allowsSave`, `isStoredInMemoryOnly`, `groupContainer`.

### Predicates deep
```swift
// #Predicate macro — type-safe
let predicate = #Predicate<Note> {
  $0.title.contains("swift") && $0.createdAt > Date().addingTimeInterval(-86400)
}
// Supported: ==, !=, <, <=, >, >=, &&, ||, !, contains, hasPrefix, hasSuffix, localizedStandardContains, localizedCaseInsensitiveContains
// NOT supported: complex NSPredicate features, compound predicates beyond &&/||, custom functions

// Predicate (non-macro)
let predicate = Predicate<Note> { note in
  note.title.contains("swift")
}
```

- `#Predicate` macro — builds `Predicate`/`NSPredicate`; limitations: no compound OR beyond simple, no custom functions, no `IN` queries? — check.
- `Predicate` — type-safe predicate builder; `PredicateExpressions` protocol.
- `FetchDescriptor` — `predicate:`, `sortBy: [SortDescriptor]`, `fetchLimit`, `fetchOffset`, `relationshipKeyPathsForPrefetching`, `includesPropertyValues`? — check.
- `SortDescriptor` — `keyPath`, `order` (`.forward`, `.reverse`).
- `ModelContext` — `insert`, `delete`, `save`, `rollback`, `fetch`, `fetchCount`, `processPendingChanges`, `didSave` notification.
- `ModelContext` `didSave` — `NotificationCenter` notification for changes.
- `ModelContext` `perform`/`performAndWait`? — check.
- `ModelContext` `autosaveEnabled` — auto-save.
- `ModelContext` `container` — parent container.
- `ModelContext` `insert`/`delete` — mark for save.
- `ModelContext` `save()` — persist changes.
- `ModelContext` `rollback()` — discard changes.
- `ModelContext` `fetch(_:)` — fetch with descriptor.
- `ModelContext` `fetchCount(_:)` — count fetch.
- `ModelContext` `processPendingChanges` — process pending changes.
- `ModelContext` `didSave` — notification.

### Relationships & performance
- `@Relationship` — `deleteRule` (`.cascade`, `.nullify`, `.deny`, `.noAction`), `inverse:`, `optional`.
- `@Relationship(inverse:)` — inverse relationship.
- `@Relationship(deleteRule:)` — delete rule.
- `@Attribute` — `.unique`, `.externalStorage`, `.preserveValueOnDeletion`, `.spotlight`, `.ephemeral`.
- `@Attribute(.externalStorage)` — large blobs to disk (threshold ~128KB? — check).
- `@Attribute(.spotlight)` — Core Spotlight indexing.
- `@Attribute(.ephemeral)` — non-persisted.
- `@Attribute(.unique)` — unique constraint.
- `@Attribute(.preserveValueOnDeletion)` — preserve value on delete.
- `@Transient` — non-persisted computed properties.
- `@Transient` macro — non-persisted properties.
- `@Model` — generates `PersistentModel` conformance, observation tracking.
- `PersistentModel` — `persistentModelID`, `modelContext`, `isDeleted`, `isFault`, `modelContext`.
- `ModelContext` `insert`/`delete` — mark for save.
- `ModelContext` `save()` — persist.
- `ModelContext` `rollback()` — discard.
- `ModelContext` `fetch`/`fetchCount` — fetch.
- `ModelContext` `processPendingChanges` — process pending.
- `ModelContext` `didSave` — notification.
- `ModelContext` `autosaveEnabled` — auto-save.
- `ModelContext` `container` — parent.
- `ModelContext` `perform`/`performAndWait`? — check.

## 3. GRDB deep

```swift
import GRDB

// DatabaseQueue (single writer)
let dbQueue = try DatabaseQueue(path: "db.sqlite")
try dbQueue.write { db in
  try db.create(table: "players", ifNotExists: true) { t in
    t.autoIncrementedPrimaryKey("id")
    t.column("name", .text).notNull()
    t.column("score", .integer).notNull()
    t.column("created_at", .datetime).notNull()
  }
}

// DatabasePool (concurrent reads + single writer)
let dbPool = try DatabasePool(path: "db.sqlite")
try await dbPool.write { db in ... }
let players = try await dbPool.read { db in
  try Player.fetchAll(db)
}

// Migrations
var migrator = DatabaseMigrator()
migrator.registerMigration("v1") { db in
  try db.create(table: "players") { t in ... }
}
migrator.registerMigration("v2") { db in
  try db.alter(table: "players") { t in t.add(column: "email", .text) }
}
try migrator.migrate(dbQueue)

// FetchableRecord / PersistableRecord
struct Player: FetchableRecord, PersistableRecord {
  var id: Int64
  var name: String
  var score: Int
  var createdAt: Date
  static let databaseTableName = "players"
  enum Columns: String, ColumnExpression {
    case id, name, score, createdAt
  }
}

// QueryInterfaceRequest
let request = Player.filter(Column("score") > 100)
  .order(Column("score").desc)
  .limit(10)
let topPlayers = try await dbPool.read { db in
  try Player.fetchAll(db, request)
}

// ValueObservation
let observation = ValueObservation.tracking { db in
  try Player.fetchAll(db).count
}
let cancellable = observation.start(in: dbPool) { error in
  print("Observation error: \(error)")
} onChange: { count in
  print("Count changed: \(count)")
}
// Cancel: cancellable.cancel()
```

- `DatabaseQueue` — single connection, serialized writes; `read`, `write`, `barrierWriteWithoutTransaction`.
- `DatabasePool` — connection pool, concurrent reads + single writer; `read`, `write`, `barrierWriteWithoutTransaction`.
- `DatabaseRegion` — `observe`, `suspend`, `resume`.
- `DatabaseMigrator` — `registerMigration`, `migrate`, `eraseDatabaseOnSchemaChange`.
- `FetchableRecord` — `fetchAll`, `fetchOne`, `fetchCount`, `fetchCursor`.
- `PersistableRecord` — `insert`, `update`, `save`, `delete`.
- `TableRecord` — `databaseTableName`, `databaseSelection`, `databaseEncodingUserInfo`, `databaseDecodingUserInfo`.
- `ColumnExpression` — `Column`, `SQL`, `SQLInterpolation`.
- `QueryInterfaceRequest` — `filter`, `order`, `limit`, `offset`, `group`, `having`, `select`, `distinct`, `joins`.
- `SQLRequest`/`SQLInterpolation` — raw SQL with parameters.
- `ValueObservation` — `tracking`, `fetch`, `eraseToAnyValueObservation`, `start`.
- `SharedValueObservation` — shared observation for multiple subscribers.
- `DatabaseFunction` — custom SQL functions.
- `DatabaseCollation` — custom collations.
- `DatabaseDateComponents` — date components storage.
- `DatabaseValue` — type-safe database values.
- `Row` — `Row.fetchAll`, `Row.fetchOne`, `Row.fetchCursor`, `Row.subscript`.
- `Statement` — `Row` iteration, `SQL` execution.
- `Database` — `execute`, `execute(sql:)`, `execute(literal:)`, `create(table:)`, `alter(table:)`, `drop(table:)`, `rename(table:)`, `foreignKeyViolations`, `inTransaction`, `transaction`, `savepoint`, `rollback`, `commit`.
- `DatabaseQueue`/`DatabasePool` — `path`, `configuration`, `read`, `write`, `barrierWriteWithoutTransaction`, `inDatabase`, `asyncRead`, `asyncWrite`, `asyncBarrierWriteWithoutTransaction`.
- `Configuration` — `journalMode` (`.wal`, `.delete`, `.truncate`, `.persist`, `.memory`, `.off`), `synchronous` (`.off`, `.normal`, `.full`, `.extra`), `foreignKeysEnabled`, `caseSensitiveLike`, `busyMode` (`.immediateError`, `.timeout`), `label`, `tempStore` (`.file`, `.memory`), `cacheSizeInPages`, `prepareDatabase`.
- `DatabaseQueue`/`DatabasePool` — `path`, `configuration`, `read`, `write`, `barrierWriteWithoutTransaction`, `inDatabase`, `asyncRead`, `asyncWrite`, `asyncBarrierWriteWithoutTransaction`.
- `Configuration` — `journalMode`, `synchronous`, `foreignKeysEnabled`, `caseSensitiveLike`, `busyMode`, `label`, `tempStore`, `cacheSizeInPages`, `prepareDatabase`.
- `DatabaseQueue`/`DatabasePool` — `path`, `configuration`, `read`, `write`, `barrierWriteWithoutTransaction`, `inDatabase`, `asyncRead`, `asyncWrite`, `asyncBarrierWriteWithoutTransaction`.
- `Configuration` — `journalMode`, `synchronous`, `foreignKeysEnabled`, `caseSensitiveLike`, `busyMode`, `label`, `tempStore`, `cacheSizeInPages`, `prepareDatabase`.

## 4. Keychain deep

### Access control
```swift
import Security

// Biometric-gated access
var error: Unmanaged<CFError>?
let accessControl = SecAccessControlCreateWithFlags(nil,
  kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
  [.userPresence],  // biometry OR passcode
  &error)!

let query: [String: Any] = [
  kSecClass as String: kSecClassGenericPassword,
  kSecAttrAccount as String: "user@example.com",
  kSecAttrService as String: "com.example.app.auth",
  kSecValueData as String: token.data(using: .utf8)!,
  kSecAttrAccessControl as String: accessControl,
]
SecItemAdd(query as CFDictionary, nil)
```

- `SecAccessControlCreateWithFlags` — `kSecAttrAccessible` + `SecAccessControlCreateFlags`: `.userPresence` (biometry OR passcode), `.biometryAny` (any enrolled biometry), `.biometryCurrentSet` (current biometry set), `.devicePasscode`, `.or`, `.and`, `.watch`? — check.
- `kSecAttrAccessible` — `kSecAttrAccessibleWhenUnlocked`, `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`, `kSecAttrAccessibleAfterFirstUnlock`, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`, `kSecAttrAccessibleAlways` (deprecated), `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`.
- `kSecAttrSynchronizable` — `kSecAttrSynchronizableAny`, `kSecAttrSynchronizableYes`, `kSecAttrSynchronizableNo` — iCloud Keychain sync.
- `kSecAttrAccessGroup` — `TEAMID.com.example.shared` — share between apps.
- `kSecAttrLabel`, `kSecAttrDescription`, `kSecAttrComment`, `kSecAttrCreator`, `kSecAttrType`, `kSecAttrScriptCode`? — check.
- `kSecAttrGeneric`, `kSecAttrAccount`, `kSecAttrService` — item identifiers.
- `kSecReturnData`, `kSecReturnAttributes`, `kSecReturnRef`, `kSecReturnPersistentRef` — return types.
- `kSecMatchLimit` — `kSecMatchLimitOne`, `kSecMatchLimitAll`.
- `kSecMatchSearchList` — search list.
- `kSecUseAuthenticationUI` — `kSecUseAuthenticationUIFail`, `kSecUseAuthenticationUISkip`, `kSecUseAuthenticationUIAllow`.
- `kSecUseOperationPrompt` — operation prompt.
- `kSecAttrAccessControl` — access control.
- `kSecAttrAccessible` — accessibility.
- `kSecAttrSynchronizable` — sync.
- `kSecAttrAccessGroup` — access group.
- `kSecAttrLabel`, `kSecAttrDescription`, `kSecAttrComment`, `kSecAttrCreator`, `kSecAttrType` — metadata.
- `kSecAttrGeneric`, `kSecAttrAccount`, `kSecAttrService` — identifiers.
- `kSecReturnData`, `kSecReturnAttributes`, `kSecReturnRef`, `kSecReturnPersistentRef` — returns.
- `kSecMatchLimit` — match limit.
- `kSecMatchSearchList` — search list.
- `kSecUseAuthenticationUI` — auth UI.
- `kSecUseOperationPrompt` — prompt.
- `SecItemAdd`, `SecItemCopyMatching`, `SecItemUpdate`, `SecItemDelete`.
- `SecItemCopyMatching` — `kSecReturnData` returns `Data`.
- `SecItemAdd` — add item.
- `SecItemUpdate` — update item.
- `SecItemDelete` — delete item.
- `SecItemCopyMatching` — `kSecMatchLimitAll` returns array.
- `SecItemUpdate` — `kSecValueData` update.
- `SecItemDelete` — delete matching items.
- `SecItemCopyMatching` — `kSecReturnAttributes` returns attributes.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecKey`/`SecCertificate`.
- `SecItemCopyMatching` — `kSecReturnPersistentRef` returns persistent ref.
- `SecItemCopyMatching` — `kSecUseAuthenticationUI` controls auth UI.
- `SecItemCopyMatching` — `kSecUseOperationPrompt` controls prompt.

## 5. Secure file protection

- `NSFileProtection` — `complete`, `completeUnlessOpen`, `completeUntilFirstUserAuthentication`, `none`.
- `FileProtectionType` — `.complete`, `.completeUnlessOpen`, `.completeUntilFirstUserAuthentication`, `.none`.
- `Data.WritingOptions` — `.completeFileProtection`, `.completeFileProtectionUnlessOpen`, `.fileProtectionCompleteUntilFirstUserAuthentication`, `.noFileProtection`.
- `FileManager` `setAttributes` — `NSFileProtectionKey`.
- `URLResourceValues` — `fileProtectionKey`.
- `FileProtectionType` — `.complete`, `.completeUnlessOpen`, `.completeUntilFirstUserAuthentication`, `.none`.
- `Data.WritingOptions` — `.completeFileProtection`, `.completeFileProtectionUnlessOpen`, `.fileProtectionCompleteUntilFirstUserAuthentication`, `.noFileProtection`.
- `FileManager` `setAttributes` — `NSFileProtectionKey`.
- `URLResourceValues` — `fileProtectionKey`.

## 6. Common persistence bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Core Data crash on save | merge conflict, wrong context | use merge policies, `perform`/`performAndWait` |
| SwiftData query not updating | `@Query` on wrong context or model not observed | ensure `ModelContainer` injected, model properties are `var` |
| SwiftData migration crash | schema change without migration plan | add `VersionedSchema` + `SchemaMigrationPlan` |
| GRDB SQLITE_BUSY | concurrent writes | use `DatabaseQueue` for writes, `DatabasePool` for concurrent reads |
| Keychain errSecMissingEntitlement | missing keychain-access-groups or wrong access group | add entitlement, use `TEAMID.group` format |
| Keychain item persists after logout | `kSecAttrSynchronizable` or accessible class too permissive | use `WhenUnlockedThisDeviceOnly`, delete on logout |
| File write fails sandboxed | missing entitlement or security-scoped access | add entitlement, use bookmark for user files |
| CloudKit sync conflicts | last-write-wins by default | use `CKSyncEngine` for proper conflict resolution |
| Core Data merge conflicts | two contexts editing same object | use merge policies, `NSMergeByPropertyObjectTrumpMergePolicy` |
| Memory spike loading JSON | decoding huge array into memory | stream with `JSONDecoder` + `AsyncSequence`, or paginate |
| GRDB SQLITE_BUSY | concurrent writes | use single `DatabaseQueue` for writes, `DatabasePool` for concurrent reads |
| iCloud file not downloading | ubiquity container not configured | check iCloud capability, `NSMetadataQuery` for download status |
| File protection too strict | `NSFileProtectionComplete` blocks background access | use `CompleteUntilFirstUserAuthentication` for background access |
| SwiftData predicate fails | unsupported predicate feature | simplify predicate, use `NSPredicate` directly, or fetch+filter |
| Core Data fault crash | accessing deleted object | check `isDeleted`, `isFault`, use `refresh` |
| GRDB migration fails | schema change without migration | add `DatabaseMigrator` migration |
| Keychain biometric prompt every time | `kSecAttrAccessible` too permissive | use `WhenUnlockedThisDeviceOnly`, access control for biometric |
