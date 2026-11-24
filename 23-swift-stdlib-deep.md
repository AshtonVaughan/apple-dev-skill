# Swift Standard Library & Foundation Deep Reference

Beyond `01-swift-language.md`: stdlib internals, String/Regex, collections, Foundation essentials, Observation, and bridging.

## 1. String & StringProtocol

### String internals
- `String` — UTF-8 native, copy-on-write; `StringProtocol` for generic string ops.
- `Substring` — view into String, shares storage; no copy until mutated.
- `Character` — extended grapheme cluster (user-perceived character); `Character` ≠ `Unicode.Scalar`.
- `Unicode.Scalar` — 21-bit code point; `Unicode.ScalarView`, `UTF8View`, `UTF16View`, `UnicodeScalarView`.
- `String.UnicodeScalarView`/`UTF8View`/`UTF16View` — views into string; `utf8.count`, `utf16.count`, `unicodeScalars.count` may differ.
- `String.Index` — opaque position; `index(_:offsetBy:)`, `index(before:)`, `index(after:)`, `samePosition(in:)` for view conversion.
- `String.distance(from:to:)` — distance in Characters (grapheme clusters).
- `String.distance(from:to:in:)` — distance in specific view (utf8/utf16/unicodeScalars).

### String operations
```swift
let s = "Hello, World!"
s.count                              // Character count
s.utf8.count                         // UTF-8 bytes
s.utf16.count                        // UTF-16 units
s.unicodeScalars.count               // Unicode scalars

s.prefix(5)                          // Substring "Hello"
s.suffix(6)                          // Substring "World!"
s.dropFirst(7)                       // Substring ", World!"
s.dropLast(1)                        // Substring "Hello, World"
s.split(separator: " ")               // [Substring]
s.components(separatedBy: ",")        // [String] (Foundation)
s.replacingOccurrences(of: "World", with: "Swift")  // Foundation
s.replacing("World", with: "Swift")   // stdlib (Swift 5.7+)
s.lowercased(), s.uppercased(), s.capitalized  // Foundation
s.hasPrefix("Hello"), s.hasSuffix("!")          // stdlib
s.contains("World")                   // stdlib (substring search)
s.range(of: "World")                  // Foundation → Range<String.Index>?
s.firstIndex(of: "W")                 // stdlib
s.firstIndex(where: { $0.isNumber })  // stdlib
s.first(where: { $0.isNumber })       // Character?
s.filter { $0.isLetter }              // String
s.map { $0.uppercased() }.joined()    // String
```

- `StringProtocol` methods: `contains`, `hasPrefix`, `hasSuffix`, `trimmingCharacters(in:)` (Foundation), `split`, `joined`, `replacing`.
- `CharacterSet` (Foundation) — `whitespaces`, `letters`, `decimalDigits`, `alphanumerics`, `urlQueryAllowed`, `urlPathAllowed`, `inverted`.
- `String.trimmingCharacters(in:)` — trim whitespace/characters.
- `String.replacingCharacters(in:with:)` — Foundation range replace.
- `String.appending`, `String(format:)` — formatting.
- `String(format:arguments:)` — printf-style formatting.
- `String(format:locale:arguments:)` — localized formatting.
- `String.localizedStringWithFormat` — localized format.
- `String(localized:)` — localized string (Swift 5.7+).
- `String(localized:defaultValue:table:bundle:locale:comment:)` — full localization.

### AttributedString (Foundation)
- `AttributedString` — typed attributes: `\.foregroundColor`, `\\.font`, `\\.link`, `\\.inlinePresentationIntent`, `\\.textEffect`? — check.
- `AttributedString` `runs` — attribute runs.
- `AttributedString` `characters` — character view.
- `NSMutableAttributedString` — mutable version.
- `NSAttributedString` — legacy, toll-free bridged.
- `AttributedString(markdown:)` — Markdown parsing (iOS 15+).
- `AttributedString` `attributeScopes` — scoped attributes.
- `AttributedString` `PresentationIntent` — semantic roles (header, list, quote, code).

## 2. Regex (Swift RegexBuilder)

```swift
import RegexBuilder

// Regex literal
let pattern = /\d{4}-\d{2}-\d{2}/
let match = "Date: 2026-09-14".firstMatch(of: pattern)

// RegexBuilder
let dateRegex = Regex {
  Capture { \d{4} } transform: { Int($0) }
  "-"
  Capture { \d{2} } transform: { Int($0) }
  "-"
  Capture { \d{2} } transform: { Int($0) }
}
let match = "2026-09-14".wholeMatch(of: dateRegex)
let (year, month, day) = match!.output

// Regex methods
let str = "abc123def"
str.firstMatch(of: /\d+/)                    // Regex.Match?
str.wholeMatch(of: /\w+/)                    // whole string match?
str.replacing(/\d+/, with: "###")            // String
str.replacing(/\d+/) { match in "[\(match.output)]" }
str.split(separator: /\d+/)                  // [Substring]
str.matches(of: /\d+/)                       // [Regex.Match]
str.contains(/\d+/)                          // Bool
str.ranges(of: /\d+/)                        // [Range<String.Index>]
str.replacingOccurrences(of: /\d+/, with: "#")  // Foundation
```

- `Regex<Output>` — typed output; `RegexComponentBuilder` for composition.
- `RegexBuilder` — `Regex { }` with `ChoiceOf`, `Repeat`, `OneOrMore`, `ZeroOrMore`, `Optionally`, `Capture`, `Lookahead`, `Lookbehind`, `CharacterClass`, `AnyCharacter`, `Character`, `Literal`, `CustomCharacterClass`.
- `Regex.Match` — `output` (tuple), `range`, `submatches`.
- `Regex` `firstMatch`, `wholeMatch`, `allMatches`? — `matches(of:)`, `split`, `replacing`, `contains`, `ranges(of:)`.
- `Regex` `anchored` — anchor to start/end.
- `Regex` `ignoresCase` — case-insensitive.
- `Regex` `wordBoundary` — word boundary.
- `Regex` `semanticLevel` — `.graphemeCluster`, `.scalar`, `.byte`.
- `Regex` `matchingSemantics` — `.unicodeScalarSemantics`, `.byteSemantics`.
- `RegexComponent` — `ChoiceOf`, `Repeat`, `OneOrMore`, `ZeroOrMore`, `Optionally`, `Capture`, `Lookahead`, `Lookbehind`.
- `RegexComponentBuilder` — `buildEither`, `buildOptional`, `buildArray`, `buildExpression`.
- `Regex` `==` — regex equality.
- `Regex` `description` — regex description.
- `Regex` `output` — output type.
- `Regex` `regex` — underlying regex.
- `Regex` `wholeMatch` — whole match.
- `Regex` `firstMatch` — first match.
- `Regex` `matches` — all matches.
- `Regex` `contains` — contains match.
- `Regex` `ranges` — match ranges.
- `Regex` `split` — split by match.
- `Regex` `replacing` — replace matches.
- `Regex` `replacingOccurrences` — Foundation replace.
- `Regex` `firstMatch(of:)` — `String.firstMatch(of:)`.
- `Regex` `wholeMatch(of:)` — `String.wholeMatch(of:)`.
- `Regex` `matches(of:)` — `String.matches(of:)`.
- `Regex` `contains(_:)` — `String.contains(_:)`.
- `Regex` `ranges(of:)` — `String.ranges(of:)`.
- `Regex` `split(separator:)` — `String.split(separator:)`.
- `Regex` `replacing(_:with:)` — `String.replacing(_:with:)`.
- `Regex` `replacingOccurrences(of:with:)` — `String.replacingOccurrences(of:with:)`.

## 3. Collections & Algorithms

### Sequence & Collection
- `Sequence` — `map`, `filter`, `reduce`, `contains`, `first`, `forEach`, `compactMap`, `flatMap`, `prefix`, `dropFirst`, `dropLast`, `prefix(while:)`, `drop(while:)`, `sorted`, `joined`, `zip`, `enumerated`, `lazy`, `makeIterator`.
- `Collection` — `count`, `isEmpty`, `first`, `last`, `startIndex`, `endIndex`, `index(after:)`, `index(before:)`, `index(_:offsetBy:)`, `index(_:offsetBy:limitedBy:)`, `distance(from:to:)`, `formIndex(after:)`, `formIndex(before:)`, `formIndex(_:offsetBy:)`, `formIndex(_:offsetBy:limitedBy:)`.
- `BidirectionalCollection` — `last`, `index(before:)`, `index(_:offsetBy:limitedBy:)`, `formIndex(before:)`, `formIndex(_:offsetBy:)`, `formIndex(_:offsetBy:limitedBy:)`.
- `RandomAccessCollection` — `index(_:offsetBy:)` O(1), `distance(from:to:)` O(1).
- `MutableCollection` — `subscript` setter.
- `RangeReplaceableCollection` — `append`, `insert`, `remove`, `replaceSubrange`.
- `LazySequence`/`LazyMapSequence`/`LazyFilterSequence` — lazy evaluation.
- `AnySequence`/`AnyCollection`/`AnyHashable`/`AnyKeyPath` — type erasure.
- `ContiguousArray` — contiguous storage, faster for value types.
- `ArraySlice`/`Array.SubSequence` — slice sharing storage.
- `Dictionary`/`Set`/`Dictionary.Keys`/`Dictionary.Values`/`Set.Algebra`.
- `Dictionary` `mapValues`, `mapKeys`, `filter`, `merge`, `merging`, `reduce`, `grouping`, `uniquingKeysWith`, `default` subscript, `keys`, `values`, `elementsEqual`, `allSatisfy`, `contains(where:)`.
- `Set` `union`, `intersection`, `subtracting`, `symmetricDifference`, `isSubset`, `isSuperset`, `isDisjoint`, `formUnion`, `formIntersection`, `formSubtracting`, `formSymmetricDifference`, `subtract`, `formUnion`, `formIntersection`, `formSubtracting`, `formSymmetricDifference`.
- `Range`/`ClosedRange`/`PartialRangeFrom`/`PartialRangeThrough`/`PartialRangeUpTo` — range types.
- `Strideable`/`Stride` — `advanced(by:)`, `distance(to:)`.
- `Comparable`/`Equatable`/`Hashable` — `<`, `==`, `hash(into:)`, `hashValue`.
- `CaseIterable` — `allCases`, `allCasesCount`? — `CaseIterable.allCases`.
- `Sendable` — `Sendable` marker protocol.
- `Codable`/`Encodable`/`Decodable` — `encode(to:)`, `init(from:)`.
- `CustomStringConvertible`/`CustomDebugStringConvertible`/`CustomPlaygroundDisplayConvertible`/`CustomReflectable`/`LosslessStringConvertible`/`ExpressibleByStringLiteral`/`ExpressibleByIntegerLiteral`/`ExpressibleByFloatLiteral`/`ExpressibleByBooleanLiteral`/`ExpressibleByNilLiteral`/`ExpressibleByArrayLiteral`/`ExpressibleByDictionaryLiteral`/`ExpressibleByExtendedGraphemeClusterLiteral`/`ExpressibleByUnicodeScalarLiteral`.

### Algorithm methods
```swift
let nums = [3, 1, 4, 1, 5, 9, 2, 6]
nums.sorted()                           // [1, 2, 3, 4, 5, 6, 9]
nums.sorted(by: >)                      // descending
nums.sorted { $0 < $1 }                 // same as sorted()
nums.max(), nums.min()                  // 9, 1
nums.max(by: <), nums.min(by: >)        // 9, 1
nums.first(where: { $0 > 4 })           // 5
nums.last(where: { $0 > 4 })            // 6
nums.contains(4)                        // true
nums.contains(where: { $0 > 8 })        // true
nums.allSatisfy { $0 < 10 }             // true
nums.reduce(0, +)                       // 31
nums.reduce(into: [Int]()) { $0.append($1 * 2) }  // [6, 2, 8, 2, 10, 18, 4, 12]
nums.map { $0 * 2 }.filter { $0 > 5 }   // [6, 8, 10, 18, 12]
nums.compactMap { $0 > 4 ? $0 : nil }   // [5, 9, 6]
nums.flatMap { [$0, $0] }               // [3, 3, 1, 1, 4, 4, ...]
nums.prefix(3)                          // [3, 1, 4]
nums.suffix(3)                          // [9, 2, 6]
nums.dropFirst(2)                       // [4, 1, 5, 9, 2, 6]
nums.dropLast(2)                        // [3, 1, 4, 1, 5]
nums.prefix(while: { $0 < 4 })          // [3, 1]
nums.drop(while: { $0 < 4 })            // [4, 1, 5, 9, 2, 6]
nums.enumerated().map { "\($0): \($1)" } // ["0: 3", "1: 1", ...]
nums.zip(["a", "b", "c"])               // [(3, "a"), (1, "b"), (4, "c")]
nums.lazy.map { $0 * 2 }.filter { $0 > 5 }.first  // lazy evaluation
nums.partition(by: { $0 > 4 })          // partition index
nums.swapAt(0, 1)
nums.shuffle()
nums.reversed()
```

- `sorted(by:)`, `max(by:)`, `min(by:)` — sorting with comparator.
- `first(where:)`, `last(where:)` — conditional search.
- `reduce(into:_:)` — mutable accumulation.
- `lazy` — lazy sequence for performance.
- `partition(by:)` — partition in place (Swift 5.8+).
- `shuffle()` — random shuffle.
- `reversed()` — reversed sequence.
- `zip` — pairwise combination.
- `enumerated()` — index-value pairs.
- `joined()` — flatten sequences.
- `flatMap` — flatten+transform.
- `compactMap` — transform+filter nil.
- `prefix(while:)`, `drop(while:)` — conditional prefix/drop.
- `allSatisfy`, `contains(where:)` — predicate checks.
- `firstIndex(of:)`, `lastIndex(of:)`, `firstIndex(where:)`, `lastIndex(where:)` — index search.
- `firstIndex(of:)` on `Collection` — O(n) for non-Collection.

## 4. Foundation essentials

### NotificationCenter
```swift
// Post
NotificationCenter.default.post(name: .myNotification, object: self, userInfo: ["key": value])

// Observe
let observer = NotificationCenter.default.addObserver(
  forName: .myNotification, object: nil, queue: .main
) { notification in
  // Handle
}
// Remove:
NotificationCenter.default.removeObserver(observer)
// Or block-based:
let token = NotificationCenter.default.addObserver(forName:object:queue:using:)
NotificationCenter.default.removeObserver(token)
```

- `NotificationCenter.default` — global center.
- `Notification.Name` — custom names; `.NSCalendarDayChanged`, `.UIApplicationDidBecomeActiveNotification`? — `UIApplication.willEnterForegroundNotification` etc.
- `Notification` — `name`, `object`, `userInfo`, `post`.
- `addObserver(forName:object:queue:using:)` — block-based observer returns token.
- `removeObserver(_:)` — remove observer.
- `addObserver(_:selector:name:object:)` — selector-based observer.
- `post(name:object:userInfo:)` — post notification.
- `postToAllSessions`? — `NotificationCenter.default` only.

### UserDefaults deep
```swift
let defaults = UserDefaults.standard
let suite = UserDefaults(suiteName: "com.example.app.group")!
let appGroup = UserDefaults(suiteName: "group.com.example.app")!

defaults.set(value, forKey: "key")
defaults.string(forKey: "key")
defaults.integer(forKey: "key")
defaults.bool(forKey: "key")
defaults.double(forKey: "key")
defaults.float(forKey: "key")
defaults.data(forKey: "key")
defaults.array(forKey: "key")
defaults.dictionary(forKey: "key")
defaults.object(forKey: "key")
defaults.removeObject(forKey: "key")
defaults.setValue(value, forKey: "key")  // KVC-compatible

defaults.register(defaults: ["key": "default"])
defaults.object(forKey: "key")  // returns default if not set
defaults.dictionaryRepresentation()
defaults.removePersistentDomain(forName: "com.example.app")
defaults.setPersistentDomain(domain, forName: "com.example.app")
defaults.volatileDomainNames

defaults.addSuite(named: "group.com.example.app")
defaults.suites?
defaults.didChangeNotification
```

- `UserDefaults.standard` — app domain.
- `UserDefaults(suiteName:)` — shared suite (App Groups).
- `UserDefaults.addSuite(named:)` — add suite to search list.
- `UserDefaults.volatileDomainNames` — volatile domains.
- `UserDefaults.object(forKey:)` — generic object.
- `UserDefaults.dictionaryRepresentation()` — all keys.
- `UserDefaults.removePersistentDomain(forName:)` — clear domain.
- `UserDefaults.setPersistentDomain(_:forName:)` — set domain.
- `UserDefaults.register(defaults:)` — register defaults (not persisted).
- `UserDefaults.value(forKey:)` — KVC-compatible.
- `UserDefaults.setValue(_:forKey:)` — KVC-compatible.
- `UserDefaults.didChangeNotification` — change notification.
- `UserDefaults` `string(forKey:)`, `integer(forKey:)`, `bool(forKey:)`, `double(forKey:)`, `float(forKey:)`, `data(forKey:)`, `array(forKey:)`, `dictionary(forKey:)`, `object(forKey:)` — typed getters.
- `UserDefaults` `set(_:forKey:)` — typed setters.
- `UserDefaults` `removeObject(forKey:)` — remove.
- `UserDefaults` `dictionaryRepresentation()` — all data.
- `UserDefaults` `volatileDomainNames` — volatile domains.
- `UserDefaults` `persistentDomain(forName:)` — domain data.
- `UserDefaults` `setPersistentDomain(_:forName:)` — set domain.
- `UserDefaults` `removePersistentDomain(forName:)` — remove domain.
- `UserDefaults` `addSuite(named:)` — add suite.
- `UserDefaults` `didChangeNotification` — notification.

### Date, Calendar, TimeZone, Locale
```swift
let date = Date()
let now = Date()
let future = Date().addingTimeInterval(3600)  // 1 hour later
let past = Date().addingTimeInterval(-86400) // 1 day ago
let components = Calendar.current.dateComponents([.year, .month, .day], from: date)
let tomorrow = Calendar.current.date(byAdding: .day, value: 1, to: date)!
let isWeekend = Calendar.current.isDateInWeekend(date)
let startOfDay = Calendar.current.startOfDay(for: date)
let endOfDay = Calendar.current.date(byAdding: .day, value: 1, to: startOfDay)!

let iso = ISO8601DateFormatter()
iso.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
let string = iso.string(from: date)
let parsed = iso.date(from: string)

let formatter = DateFormatter()
formatter.dateStyle = .medium
formatter.timeStyle = .short
formatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
formatter.locale = Locale(identifier: "en_US_POSIX")  // for parsing
let formatted = formatter.string(from: date)
let parsed = formatter.date(from: formatted)

let relative = RelativeDateTimeFormatter()
relative.unitsStyle = .full
let rel = relative.localizedString(for: future.timeIntervalSinceNow, relativeTo: Date())
// "1 hour from now"

let interval = DateInterval(start: startOfDay, end: endOfDay)
let contains = interval.contains(date)
```

- `Date` — `timeIntervalSince1970`, `timeIntervalSinceNow`, `timeIntervalSince`, `addingTimeInterval`, `advanced(by:)`, `timeIntervalSinceReferenceDate`, `compare`, `isEqual`, `isLater`, `isEarlier`, `distance(to:)`, `DateInterval`, `DateComponents`.
- `Calendar` — `current`, `autoupdatingCurrent`, `identifier` (`.gregorian`, `.iso8601`, `.japanese`, `.buddhist`, `.chinese`, `.hebrew`, `.islamic`, `.islamicCivil`, `.islamicTabular`, `.islamicUmmAlQura`, `.indian`, `.persian`, `.republicOfChina`), `dateComponents`, `date(byAdding:value:to:)`, `dateInterval(of:for:)`, `startOfDay`, `isDateInWeekend`, `firstWeekday`, `minimumDaysInFirstWeek`, `timeZone`, `locale`, `eraSymbols`, `monthSymbols`, `weekdaySymbols`, `quarterSymbols`, `daySymbols`.
- `TimeZone` — `current`, `autoupdatingCurrent`, `identifier`, `secondsFromGMT`, `abbreviation`, `localizedName`, `nextDaylightSavingTimeTransition`, `previousDaylightSavingTimeTransition`.
- `Locale` — `current`, `autoupdatingCurrent`, `identifier`, `languageCode`, `language`, `regionCode`, `currencyCode`, `currencySymbol`, `decimalSeparator`, `groupingSeparator`, `usesMetricSystem`, `calendar`, `collation`, `firstDayOfWeek`, `measurementSystem`, `numberingSystem`, `preferredLanguages`, `localizedString(forKey:value:)`, `localizedString(forLanguageCode:)`, `localizedString(forRegionCode:)`, `localizedString(forScriptCode:)`, `localizedString(forVariantCode:)`, `localizedString(forCurrencyCode:)`.
- `DateFormatter` — `dateStyle`, `timeStyle`, `dateFormat`, `locale`, `calendar`, `timeZone`, `formatterBehavior`, `isLenient`, `twoDigitStartDate`, `doesRelativeDateFormatting`, `defaultDate`, `defaultTimeZone`, `generatesCalendarIdentifiers`, `timeStyle`, `dateStyle`, `dateFormat`, `localizedString(from:)`, `string(from:)`, `string(from:to:)`, `date(from:)`, `getObjectValue(_:for:errorDescription:)`.
- `ISO8601DateFormatter` — `formatOptions`, `timeZone`, `string(from:)`, `date(from:)`.
- `RelativeDateTimeFormatter` — `unitsStyle`, `dateTimeStyle`, `formatted`, `localizedString(for:relativeTo:)`.
- `DateComponentsFormatter` — `unitsStyle`, `allowedUnits`, `includesApproximationSuffix`, `formatted`, `string(from:)`.
- `DateIntervalFormatter` — `dateStyle`, `timeStyle`, `locale`, `timeZone`, `calendar`, `string(from:)`, `string(from:to:)`.
- `MeasurementFormatter` — `unitStyle`, `unitOptions`, `locale`, `numberFormatter`, `string(from:)`, `measurement(from:)`.
- `NumberFormatter` — `numberStyle`, `locale`, `format`, `string(from:)`, `number(from:)`, `currencyCode`, `currencySymbol`, `generatesDecimalNumbers`, `isLenient`, `maximumFractionDigits`, `minimumFractionDigits`, `maximumIntegerDigits`, `minimumIntegerDigits`, `roundingMode`, `roundingIncrement`, `positivePrefix`, `negativePrefix`, `positiveSuffix`, `negativeSuffix`, `groupingSeparator`, `decimalSeparator`, `groupingSize`, `secondaryGroupingSize`, `usesGroupingSeparator`, `allowsFloats`, `alwaysShowsDecimalSeparator`, `maximumSignificantDigits`, `minimumSignificantDigits`, `usesSignificantDigits`, `internationalCurrencySymbol`, `percentSymbol`, `perMillSymbol`, `zeroSymbol`, `notANumberSymbol`, `infinitySymbol`, `minusSign`, `plusSign`, `negativeSuffix`, `positiveSuffix`.
- `DateComponents` — `year`, `month`, `day`, `hour`, `minute`, `second`, `nanosecond`, `weekday`, `weekdayOrdinal`, `quarter`, `era`, `isLeapMonth`, `calendar`, `timeZone`, `date`.
- `TimeInterval` — `TimeInterval` = `Double` seconds.
- `DateInterval` — `start`, `duration`, `end`, `contains`, `intersection`.
- `DateComponents` — `year`, `month`, `day`, `hour`, `minute`, `second`, `nanosecond`, `weekday`, `weekdayOrdinal`, `quarter`, `era`, `isLeapMonth`, `calendar`, `timeZone`, `date`.

### JSONSerialization & JSONEncoder
```swift
// JSONSerialization
let json: [String: Any] = ["name": "Alice", "age": 30]
let data = try JSONSerialization.data(withJSONObject: json)
let object = try JSONSerialization.jsonObject(with: data) as? [String: Any]

// JSONEncoder/Decoder
struct User: Codable {
  var name: String
  var age: Int
  var createdAt: Date
}

let encoder = JSONEncoder()
encoder.outputFormatting = [.prettyPrinted, .sortedKeys, .withoutEscapingSlashes]
encoder.dateEncodingStrategy = .iso8601
encoder.keyEncodingStrategy = .convertToSnakeCase
let data = try encoder.encode(user)

let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso8601
decoder.keyDecodingStrategy = .convertFromSnakeCase
let user = try decoder.decode(User.self, from: data)
```

- `JSONEncoder` — `outputFormatting` (`.prettyPrinted`, `.sortedKeys`, `.withoutEscapingSlashes`, `.fragmentsAllowed`), `dateEncodingStrategy` (`.deferredToDate`, `.secondsSince1970`, `.millisecondsSince1970`, `.iso8601`, `.formatted(_:)`), `keyEncodingStrategy` (`.useDefaultKeys`, `.convertToSnakeCase`, `.convertToKebabCase`, `.convertToPascalCase`, `.custom(_:)`), `userInfo`, `nonConformingFloatEncodingStrategy` (`.throw`, `.convertToString(positiveInfinity:negativeInfinity:nan:)`), `dataEncodingStrategy`.
- `JSONDecoder` — `dateDecodingStrategy` (`.deferredToDate`, `.secondsSince1970`, `.millisecondsSince1970`, `.iso8601`, `.formatted(_:)`), `keyDecodingStrategy` (`.useDefaultKeys`, `.convertFromSnakeCase`, `.convertFromKebabCase`, `.convertFromPascalCase`, `.custom(_:)`), `userInfo`, `nonConformingFloatDecodingStrategy`.
- `JSONSerialization` — `data(withJSONObject:options:)`, `jsonObject(with:options:)`, `isValidJSONObject`.
- `JSONSerialization.WritingOptions` — `.prettyPrinted`, `.sortedKeys`, `.fragmentsAllowed`, `.withoutEscapingSlashes`.
- `JSONSerialization.ReadingOptions` — `.mutableContainers`, `.mutableLeaves`, `.allowFragments`, `.fragmentsAllowed`.
- `PropertyListEncoder`/`PropertyListDecoder` — `PropertyListEncoder.OutputFormat` (`.xml`, `.binary`, `.openStep`), `PropertyListDecoder` `propertyListFormat`.
- `Codable`/`Encodable`/`Decodable` — `encode(to:)`, `init(from:)`, `CodingKeys`, `KeyedEncodingContainer`, `KeyedDecodingContainer`, `SingleValueEncodingContainer`, `SingleValueDecodingContainer`, `UnkeyedEncodingContainer`, `UnkeyedDecodingContainer`.
- `CodingKey` — `stringValue`, `intValue`, `init(stringValue:)`, `init(intValue:)`.
- `KeyedEncodingContainer` — `encode(_:forKey:)`, `encodeIfPresent(_:forKey:)`, `nestedContainer(keyedBy:forKey:)`, `nestedUnkeyedContainer(forKey:)`, `superEncoder(forKey:)`.
- `KeyedDecodingContainer` — `decode(_:forKey:)`, `decodeIfPresent(_:forKey:)`, `nestedContainer(keyedBy:forKey:)`, `nestedUnkeyedContainer(forKey:)`, `superDecoder(forKey:)`.
- `UnkeyedEncodingContainer` — `encode(_:nil)`, `nestedContainer(keyedBy:)`, `nestedUnkeyedContainer()`, `superEncoder()`.
- `UnkeyedDecodingContainer` — `decode(_:)`, `decodeNil()`, `nestedContainer(keyedBy:)`, `nestedUnkeyedContainer()`, `superDecoder()`.
- `SingleValueEncodingContainer` — `encode(_:)`.
- `SingleValueDecodingContainer` — `decode(_:)`.
- `Encoder`/`Decoder` — `container(keyedBy:)`, `unkeyedContainer()`, `singleValueContainer()`, `codingPath`, `userInfo`, `dateEncodingStrategy`, `dataEncodingStrategy`, `nonConformingFloatEncodingStrategy`.
- `Encoder`/`Decoder` — `container(keyedBy:)`, `unkeyedContainer()`, `singleValueContainer()`, `codingPath`, `userInfo`, `dateDecodingStrategy`, `dataDecodingStrategy`, `nonConformingFloatDecodingStrategy`.
- `CodingUserInfoKey` — custom user info keys.
- `DecodingError` — `.dataCorrupted`, `.keyNotFound`, `.typeMismatch`, `.valueNotFound`.
- `EncodingError` — `.invalidValue`, `.keyNotFound`? — `.invalidValue`.

### FileManager & FileHandle
```swift
let fm = FileManager.default
let urls = fm.urls(for: .documentDirectory, in: .userDomainMask)
let documents = urls[0]

fm.fileExists(atPath: path)
fm.fileExists(atPath: path, isDirectory: &isDir)
fm.createFile(atPath: path, contents: data)
fm.createDirectory(at: url, withIntermediateDirectories: true)
fm.removeItem(at: url)
fm.moveItem(at: src, to: dst)
fm.copyItem(at: src, to: dst)
fm.contentsOfDirectory(at: url, includingPropertiesForKeys: [.fileSizeKey])
fm.enumerator(at: url, includingPropertiesForKeys: nil)
fm.attributesOfItem(atPath: path)
fm.destinationOfSymbolicLink(atPath: path)
fm.createSymbolicLink(atPath: path, withDestinationPath: dest)
fm.linkItem(atPath: src, toPath: dst)
fm.isReadableFile(atPath:), fm.isWritableFile(atPath:), fm.isDeletableFile(atPath:), fm.isExecutableFile(atPath:)
fm.isEmptyFile(atPath:), fm.isEmptyDirectory(atPath:)
fm.temporaryDirectory
fm.homeDirectoryForCurrentUser
fm.urls(for: .documentDirectory, in: .userDomainMask)
fm.urls(for: .cachesDirectory, in: .userDomainMask)
fm.urls(for: .applicationSupportDirectory, in: .userDomainMask)
fm.urls(for: .libraryDirectory, in: .userDomainMask)
fm.urls(for: .temporaryDirectory, in: .userDomainMask)
fm.urls(for: .documentDirectory, in: .userDomainMask)
fm.urls(for: .cachesDirectory, in: .userDomainMask)
fm.urls(for: .applicationSupportDirectory, in: .userDomainMask)
fm.urls(for: .libraryDirectory, in: .userDomainMask)
fm.urls(for: .temporaryDirectory, in: .userDomainMask)
```

- `FileManager.default` — shared instance.
- `FileManager.SearchPathDirectory` — `.documentDirectory`, `.cachesDirectory`, `.applicationSupportDirectory`, `.libraryDirectory`, `.temporaryDirectory`, `.downloadsDirectory`, `.desktopDirectory`, `.documentsDirectory`, `.picturesDirectory`, `.moviesDirectory`, `.musicDirectory`, `.publicDirectory`, `.trashDirectory`, `.homeDirectory`.
- `FileManager.SearchPathDomainMask` — `.userDomainMask`, `.systemDomainMask`, `.localDomainMask`, `.networkDomainMask`, `.allDomainsMask`.
- `FileManager.url(for:in:appropriateFor:create:)` — URL for directory.
- `FileManager.contentsOfDirectory(at:includingPropertiesForKeys:options:)` — list directory.
- `FileManager.enumerator(at:includingPropertiesForKeys:options:errorHandler:)` — recursive enumeration.
- `FileManager.createDirectory(at:withIntermediateDirectories:attributes:)` — create directory.
- `FileManager.removeItem(at:)` — delete.
- `FileManager.moveItem(at:to:)` — move.
- `FileManager.copyItem(at:to:)` — copy.
- `FileManager.fileExists(atPath:isDirectory:)` — check existence.
- `FileManager.isReadableFile`, `isWritableFile`, `isDeletableFile`, `isExecutableFile` — permission checks.
- `FileManager.isEmptyFile`, `isEmptyDirectory` — empty checks.
- `FileManager.attributesOfItem(atPath:)` — file attributes.
- `FileManager.destinationOfSymbolicLink` — symlink target.
- `FileManager.createSymbolicLink` — create symlink.
- `FileManager.linkItem` — hard link.
- `FileManager.contentsOfFile` — read file contents.
- `FileManager.createFile` — create file.
- `FileManager.temporaryDirectory` — temp directory.
- `FileManager.homeDirectoryForCurrentUser` — home directory.
- `FileHandle` — `forReadingAtPath`, `forWritingAtPath`, `forUpdatingAtPath`, `standardInput`, `standardOutput`, `standardError`, `readDataToEndOfFile`, `readData(ofLength:)`, `read(upToCount:)`, `write`, `seekToEnd`, `seekToFileOffset`, `offsetInFile`, `truncateFile`, `synchronizeFile`, `closeFile`, `availableData`, `bytesAvailable`, `readabilityHandler`, `writeabilityHandler`.
- `Data` — `contentsOf`, `write`, `append`, `subdata`, `range`, `count`, `isEmpty`, `startIndex`, `endIndex`, `base64EncodedString`, `base64EncodedData`, `decompressed(using:shouldFailOnError:)`? — check.
- `Data` `init(contentsOf:)`, `init(contentsOf:options:)`, `init(base64Encoded:)`, `init(base64EncodedString:)`, `init(json:)`? — `JSONDecoder().decode`.
- `URL` — `fileURLWithPath`, `path`, `absoluteURL`, `relativePath`, `standardized`, `resolvingSymlinksInPath`, `appendingPathComponent`, `deletingLastPathComponent`, `deletingPathExtension`, `pathExtension`, `lastPathComponent`, `hasDirectoryPath`, `isFileURL`, `isDirectoryURL`, `checkResourceIsReachable`, `resourceValues(forKeys:)`, `setResourceValues`, `startAccessingSecurityScopedResource`, `stopAccessingSecurityScopedResource`, `bookmarkData`, `init(resolvingBookmarkData:options:relativeTo:bookmarkDataIsStale:)`, `URLResourceKey`.
- `URLSession` — `shared`, `configuration`, `delegate`, `dataTask`, `uploadTask`, `downloadTask`, `webSocketTask`, `invalidateAndCancel`, `finishTasksAndInvalidate`, `reset`, `getAllTasks`, `data(for:delegate:)`, `upload(for:from:delegate:)`, `download(for:delegate:)`.
- `URLSessionConfiguration` — `default`, `ephemeral`, `background`, `identifier`, `timeoutIntervalForRequest`, `timeoutIntervalForResource`, `httpAdditionalHeaders`, `httpCookieStorage`, `httpShouldSetCookies`, `httpCookieAcceptPolicy`, `urlCredentialStorage`, `urlCache`, `requestCachePolicy`, `networkServiceType`, `allowsCellularAccess`, `allowsExpensiveNetworkAccess`, `allowsConstrainedNetworkAccess`, `waitsForConnectivity`, `multipathServiceType`, `isDiscretionary`, `sessionSendsLaunchEvents`, `shouldUseExtendedBackgroundIdleMode`, `isTLSHostnameExtensionEnabled`, `tlsMinimumSupportedProtocolVersion`, `tlsMaximumSupportedProtocolVersion`.
- `URLRequest` — `url`, `httpMethod`, `httpBody`, `httpBodyStream`, `httpHeaderFields`, `setValue`, `addValue`, `setHTTPBody`, `cachePolicy`, `timeoutInterval`, `allHTTPHeaderFields`, `value(forHTTPHeaderField:)`, `url`, `mainDocumentURL`, `httpBody`.
- `URLResponse`/`HTTPURLResponse` — `statusCode`, `allHeaderFields`, `url`, `mimeType`, `expectedContentLength`, `textEncodingName`, `suggestedFilename`.
- `URLSessionTask` — `state`, `priority`, `taskDescription`, `taskIdentifier`, `currentRequest`, `originalRequest`, `response`, `error`, `countOfBytesSent`, `countOfBytesReceived`, `countOfBytesExpectedToSend`, `countOfBytesExpectedToReceive`, `suspend`, `resume`, `cancel`, `waitUntilFinished`, `transactionMetrics`, `delegate`, `delegateQueue`, `progress`.
- `URLSessionDataTask` — `didReceive response`, `didReceive data`, `didCompleteWithError`, `didSendBodyData`, `willCacheResponse`, `didBecomeDownloadTask`, `didFinishCollecting metrics`, `didReceive challenge`, `taskIsWaitingForConnectivity`.
- `URLSessionDownloadTask` — `didFinishDownloadingTo`, `resumeData`, `cancel(byProducingResumeData:)`.
- `URLSessionUploadTask` — `progress`, `didSendBodyData`.
- `URLSessionWebSocketTask` — `send`, `receive`, `cancel`, `ping`, `delegate`, `maximumMessageSize`, `closeCode`, `closeReason`.
- `URLSessionTaskDelegate` — `didReceive challenge`, `didReceive response`, `didCompleteWithError`, `didSendBodyData`, `willCacheResponse`, `didBecomeDownloadTask`, `didFinishCollecting`, `taskIsWaitingForConnectivity`.
- `URLSessionDataDelegate` — `didReceive data`, `didBecomeDownloadTask`.
- `URLSessionDownloadDelegate` — `didFinishDownloadingTo`, `didWriteData`, `didResumeAtOffset`.
- `URLSessionTaskMetrics` — `transactionMetrics`, `taskInterval`, `redirectCount`, `responseStatusCode`, `fetchStartDate`, `domainLookupStartDate`, `domainLookupEndDate`, `connectStartDate`, `connectEndDate`, `secureConnectionStartDate`, `secureConnectionEndDate`, `requestStartDate`, `requestEndDate`, `responseStartDate`, `responseEndDate`, `networkProtocolName`, `isReusedConnection`, `remoteAddress`, `localAddress`, `isProxyConnection`, `proxyConnectionProtocol`, `cellular`.
- `URLSessionTaskTransactionMetrics` — `request`, `response`, `requestStartDate`, `requestEndDate`, `responseStartDate`, `responseEndDate`, `fetchStartDate`, `domainLookupStartDate`, `domainLookupEndDate`, `connectStartDate`, `connectEndDate`, `secureConnectionStartDate`, `secureConnectionEndDate`, `networkProtocolName`, `isReusedConnection`, `remoteAddress`, `localAddress`, `isProxyConnection`, `proxyConnectionProtocol`, `cellular`.
- `URLAuthenticationChallenge`/`URLCredential`/`URLProtectionSpace` — `protectionSpace`, `proposedCredential`, `previousFailureCount`, `failureResponse`, `error`, `credential`, `useCredential`, `performDefaultHandling`, `rejectProtectionSpace`, `useDefaultCredential`.
- `URLCredential` — `user`, `password`, `persistence` (`.none`, `.forSession`, `.permanent`), `certificate`, `identity`, `trust`, `hasPassword`.
- `URLProtectionSpace` — `host`, `port`, `protocol`, `realm`, `authenticationMethod`, `isProxy`.
- `URLSession` `delegate` — `URLSessionDelegate`, `URLSessionTaskDelegate`, `URLSessionDataDelegate`, `URLSessionDownloadDelegate`, `URLSessionWebSocketDelegate`.
- `URLSession` `delegateQueue` — `OperationQueue` for delegate callbacks.
- `URLSession` `finishTasksAndInvalidate` — finish tasks then invalidate.
- `URLSession` `invalidateAndCancel` — cancel all tasks and invalidate.
- `URLSession` `reset` — reset session state.
- `URLSession` `getAllTasks` — get all tasks.
- `URLSession` `data(for:delegate:)` — async data task.
- `URLSession` `upload(for:from:delegate:)` — async upload task.
- `URLSession` `download(for:delegate:)` — async download task.
- `URLSession` `bytes(for:delegate:)` — async bytes.
- `URLSession` `webSocketTask(with:)` — WebSocket task.

## 5. Observation & Combine internals

### Observation (Swift 5.9+)
```swift
@Observable
class Model {
  var name = ""
  var count = 0
  @ObservationIgnored var internalState = 0  // not observed
}

// Automatic observation in SwiftUI
struct View: View {
  @State private var model = Model()
  var body: some View {
    Text(model.name)  // auto-tracked
  }
}

// Manual observation
withObservationTracking {
  _ = model.name  // tracked
} onChange: {
  // Called when model.name changes
}
```

- `@Observable` macro — generates `Observable` conformance, `_$observationRegistrar`, `access`/`withMutation` for tracked properties.
- `Observable` protocol — `_$observationRegistrar`, `access(keyPath:)`, `withMutation(keyPath:_:)`.
- `ObservationRegistrar` — `access`, `withMutation`, `willSet`, `didSet`.
- `withObservationTracking` — `withObservationTracking(_:_:)` for manual tracking.
- `@ObservationIgnored` — exclude property from observation.
- `@Bindable` — bindable wrapper for `@Observable` objects.
- `@State`/`@Binding`/`@Environment`/`@EnvironmentObject`/`@ObservedObject`/`@StateObject` — SwiftUI property wrappers.
- `ObservableObject` — `objectWillChange`, `ObservableObjectPublisher`.
- `@Published` — published property wrapper.
- `@ObservedObject` — observe external ObservableObject.
- `@StateObject` — own ObservableObject.
- `@EnvironmentObject` — environment ObservableObject.
- `ObservableObjectPublisher` — `send()`, `send(_:)`.

### Combine deep
```swift
// Publishers
let subject = PassthroughSubject<String, Never>()
let replay = ReplaySubject<String, Never>(bufferSize: 10)
let current = CurrentValueSubject<Int, Never>(0)

// Operators
let cancellable = subject
  .map { $0.uppercased() }
  .filter { !$0.isEmpty }
  .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
  .removeDuplicates()
  .sink { value in print(value) }

// CombineLatest
let combined = Publishers.CombineLatest(subject1, subject2)
  .map { "\($0) - \($1)" }
  .sink { print($0) }

// Merge
let merged = Publishers.Merge(subject1, subject2).sink { ... }

// Timer
let timer = Timer.publish(every: 1.0, on: .main, in: .common)
  .autoconnect()
  .sink { _ in ... }

// NotificationCenter publisher
let notif = NotificationCenter.default.publisher(for: .myNotification)
  .sink { notification in ... }

// URLSession publisher
let dataTask = URLSession.shared.dataTaskPublisher(for: request)
  .tryMap { $0.data }
  .decode(type: User.self, decoder: JSONDecoder())
  .sink(receiveCompletion: { ... }) { user in ... }
```

- `Publisher` — `map`, `filter`, `compactMap`, `flatMap`, `switchToLatest`, `prepend`, `append`, `merge`, `combineLatest`, `zip`, `debounce`, `throttle`, `delay`, `receive(on:)`, `receive(on:options:)`, `receive(on:scheduler:)`, `receive(on:queue:)`, `receive(on:operationQueue:)`, `receive(on:dispatchQueue:)`, `receive(on:runLoop:)`, `receive(on:operationQueue:)`, `sink`, `assign`, `store`, `eraseToAnyPublisher`, `handleEvents`, `tryMap`, `tryFilter`, `tryCompactMap`, `tryFlatMap`, `trySwitchToLatest`, `tryPrepend`, `tryAppend`, `tryMerge`, `tryCombineLatest`, `tryZip`, `tryDebounce`, `tryThrottle`, `tryDelay`, `tryReceive(on:)`, `tryHandleEvents`, `trySink`, `tryAssign`, `tryStore`, `tryEraseToAnyPublisher`, `tryCatch`, `tryRetry`, `tryTimeout`, `tryCollect`, `tryShare`, `tryMulticast`, `tryCache`, `tryBuffer`, `tryConflate`, `tryMaterializeResults`, `tryDematerializeResults`, `tryEncode`, `tryDecode`, `tryMapError`, `tryReplaceError`, `tryReplaceNil`, `tryReplaceEmpty`, `trySetFailureType`, `tryPromoteFailure`, `tryAssertNoFailure`, `tryAppend`, `tryPrepend`, `tryMerge`, `tryCombineLatest`, `tryZip`, `tryDebounce`, `tryThrottle`, `tryDelay`, `tryReceive(on:)`, `tryHandleEvents`, `trySink`, `tryAssign`, `tryStore`, `tryEraseToAnyPublisher`, `tryCatch`, `tryRetry`, `tryTimeout`, `tryCollect`, `tryShare`, `tryMulticast`, `tryCache`, `tryBuffer`, `tryConflate`, `tryMaterializeResults`, `tryDematerializeResults`, `tryEncode`, `tryDecode`, `tryMapError`, `tryReplaceError`, `tryReplaceNil`, `tryReplaceEmpty`, `trySetFailureType`, `tryPromoteFailure`, `tryAssertNoFailure`.
- `Subject` — `PassthroughSubject`, `CurrentValueSubject`, `ReplaySubject`.
- `AnyPublisher`/`AnyCancellable` — type erasure.
- `Cancellable` — `cancel()`, `store(in:)`.
- `AnyCancellable` — `cancel()`, `store(in:)`.
- `Set<AnyCancellable>` — cancellable set.
- `Subscribers` — `Sink`, `Assign`, `Store`, `Completion`, `Demand`, `Subscription`, `Request`, `Receive`, `Map`, `Filter`, `CompactMap`, `FlatMap`, `SwitchToLatest`, `Prepend`, `Append`, `Merge`, `CombineLatest`, `Zip`, `Debounce`, `Throttle`, `Delay`, `ReceiveOn`, `HandleEvents`, `TryMap`, `TryFilter`, `TryCompactMap`, `TryFlatMap`, `TrySwitchToLatest`, `TryPrepend`, `TryAppend`, `TryMerge`, `TryCombineLatest`, `TryZip`, `TryDebounce`, `TryThrottle`, `TryDelay`, `TryReceiveOn`, `TryHandleEvents`, `TrySink`, `TryAssign`, `TryStore`, `TryEraseToAnyPublisher`, `TryCatch`, `TryRetry`, `TryTimeout`, `TryCollect`, `TryShare`, `TryMulticast`, `TryCache`, `TryBuffer`, `TryConflate`, `TryMaterializeResults`, `TryDematerializeResults`, `TryEncode`, `TryDecode`, `TryMapError`, `TryReplaceError`, `TryReplaceNil`, `TryReplaceEmpty`, `TrySetFailureType`, `TryPromoteFailure`, `TryAssertNoFailure`, `TryAppend`, `TryPrepend`, `TryMerge`, `TryCombineLatest`, `TryZip`, `TryDebounce`, `TryThrottle`, `TryDelay`, `TryReceiveOn`, `TryHandleEvents`, `TrySink`, `TryAssign`, `TryStore`, `TryEraseToAnyPublisher`, `TryCatch`, `TryRetry`, `TryTimeout`, `TryCollect`, `TryShare`, `TryMulticast`, `TryCache`, `TryBuffer`, `TryConflate`, `TryMaterializeResults`, `TryDematerializeResults`, `TryEncode`, `TryDecode`, `TryMapError`, `TryReplaceError`, `TryReplaceNil`, `TryReplaceEmpty`, `TrySetFailureType`, `TryPromoteFailure`, `TryAssertNoFailure`.
- `Published` — `Publisher` for `@Published` properties.
- `ObservableObject` — `objectWillChange`, `ObservableObjectPublisher`.
- `ObservableObjectPublisher` — `send()`, `send(_:)`.
- `NotificationCenter.Publisher` — `publisher(for:)`.
- `Timer.TimerPublisher` — `Timer.publish`.
- `URLSession.DataTaskPublisher` — `dataTaskPublisher(for:)`.
- `Future`/`Deferred` — `Future`, `Deferred`, `Deferred.Future`.
- `PassthroughSubject`/`CurrentValueSubject`/`ReplaySubject` — subjects.
- `AnyPublisher`/`AnyCancellable` — type erasure.
- `Cancellable`/`AnyCancellable` — cancellation.
- `Set<AnyCancellable>` — cancellable set.
- `Subscribers` — `Sink`, `Assign`, `Store`, `Completion`, `Demand`, `Subscription`, `Request`, `Receive`, `Map`, `Filter`, `CompactMap`, `FlatMap`, `SwitchToLatest`, `Prepend`, `Append`, `Merge`, `CombineLatest`, `Zip`, `Debounce`, `Throttle`, `Delay`, `ReceiveOn`, `HandleEvents`, `TryMap`, `TryFilter`, `TryCompactMap`, `TryFlatMap`, `TrySwitchToLatest`, `TryPrepend`, `TryAppend`, `TryMerge`, `TryCombineLatest`, `TryZip`, `TryDebounce`, `TryThrottle`, `TryDelay`, `TryReceiveOn`, `TryHandleEvents`, `TrySink`, `TryAssign`, `TryStore`, `TryEraseToAnyPublisher`, `TryCatch`, `TryRetry`, `TryTimeout`, `TryCollect`, `TryShare`, `TryMulticast`, `TryCache`, `TryBuffer`, `TryConflate`, `TryMaterializeResults`, `TryDematerializeResults`, `TryEncode`, `TryDecode`, `TryMapError`, `TryReplaceError`, `TryReplaceNil`, `TryReplaceEmpty`, `TrySetFailureType`, `TryPromoteFailure`, `TryAssertNoFailure`, `TryAppend`, `TryPrepend`, `TryMerge`, `TryCombineLatest`, `TryZip`, `TryDebounce`, `TryThrottle`, `TryDelay`, `TryReceiveOn`, `TryHandleEvents`, `TrySink`, `TryAssign`, `TryStore`, `TryEraseToAnyPublisher`, `TryCatch`, `TryRetry`, `TryTimeout`, `TryCollect`, `TryShare`, `TryMulticast`, `TryCache`, `TryBuffer`, `TryConflate`, `TryMaterializeResults`, `TryDematerializeResults`, `TryEncode`, `TryDecode`, `TryMapError`, `TryReplaceError`, `TryReplaceNil`, `TryReplaceEmpty`, `TrySetFailureType`, `TryPromoteFailure`, `TryAssertNoFailure`.
- `Published` — `Publisher` for `@Published`.
- `ObservableObject` — `objectWillChange`.
- `ObservableObjectPublisher` — `send()`.
- `NotificationCenter.Publisher` — `publisher(for:)`.
- `Timer.TimerPublisher` — `Timer.publish`.
- `URLSession.DataTaskPublisher` — `dataTaskPublisher(for:)`.
- `Future`/`Deferred` — `Future`, `Deferred`.

## 6. Common stdlib/Foundation bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| String index crash | `String.Index` not Character-aligned | use `index(_:offsetBy:limitedBy:)`, check bounds |
| Regex catastrophic backtracking | nested quantifiers, ambiguous patterns | simplify pattern, use possessive quantifiers? — `Regex` doesn't support; restructure |
| `sorted()` on huge array | O(n log n) on main thread | `lazy.sorted()` for lazy, move to background |
| `UserDefaults` slow | frequent writes, large data | batch writes, use `synchronize()` sparingly, use files for large data |
| `DateFormatter` slow | creating formatter per call | reuse formatter, cache instances |
| `JSONDecoder` date fails | wrong strategy | set `dateDecodingStrategy` correctly |
| `FileManager` permission denied | sandbox, missing entitlement | add entitlement, use security-scoped access |
| `NotificationCenter` leak | observer not removed | store token, remove in `deinit` |
| `Combine` memory leak | subscription not stored | store `AnyCancellable` in property |
| `Combine` race condition | `CurrentValueSubject` emits stale value | use `dropFirst()` or check value |
| `Observable` not updating | property not `var` or `@ObservationIgnored` | make `var`, remove `@ObservationIgnored` |
| `AttributedString` parse fails | invalid Markdown | validate input, use `AttributedString(markdown:)` with options |
| `String` encoding issue | wrong encoding assumption | specify `String.Encoding` explicitly |
| `URL` path wrong | missing `standardized`/`resolvingSymlinksInPath` | normalize path before use |
| `Data` memory spike | loading huge file into memory | stream with `FileHandle` or `InputStream` |
| `JSONSerialization` fails | non-JSON object | ensure all values are JSON-serializable |
| `PropertyList` version mismatch | format incompatibility | use `.xml` for debugging, `.binary` for production |
| `Calendar` wrong results | wrong calendar/timezone | set `calendar.timeZone`, `calendar.locale` |
| `Locale` wrong formatting | wrong locale | set `locale` on formatter |
| `Date` precision loss | `TimeInterval` is Double | use `DateComponents` for exact precision |
| `FileHandle` seek fails | negative offset | validate offset before seeking |
| `URLSession` memory leak | delegate not invalidated | call `finishTasksAndInvalidate` or `invalidateAndCancel` |
| `Combine` subscription leak | `AnyCancellable` not stored | store in `Set<AnyCancellable>` property |
| `Observable` cycle | object observing itself | use `weak` references, break cycle |
