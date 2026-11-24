# Code Signing, Distribution & App Store

Certificates, provisioning profiles, App Store Connect, TestFlight, macOS notarization, privacy manifests, APNs auth.

## 1. Code signing mental model

```
Certificate (private key + public key + identity)
  + Provisioning Profile (cert + device UDIDs + App ID + entitlements)
  + Bundle ID (unique app identifier)
  + Team ID (developer account)
  = Signed app that can run on specific devices / be distributed
```

- **Certificate**: proves identity; contains public key + private key (private key stays on your Mac or CI).
- **Provisioning Profile**: links certificate + devices + App ID + entitlements; embedded in app.
- **App ID**: `TEAMID.com.example.app` — unique identifier for your app (bundle ID + team prefix).
- **Team ID**: 10-character Apple Developer Team identifier.
- **Bundle ID**: `com.example.app` — reverse-DNS, unique per app.

## 2. Certificate types

| Type | Purpose | Distribution |
|---|---|---|
| **Apple Development** | Development builds on registered devices | Development |
| **Apple Distribution** | App Store / TestFlight / Ad Hoc / Enterprise | Distribution |
| **Developer ID Application** | macOS apps outside App Store (direct distribution) | Developer ID |
| **Developer ID Installer** | macOS installer packages | Developer ID |
| **Mac App Distribution** | Mac App Store (legacy, now unified as Apple Distribution) | Mac App Store |
| **iOS Distribution** | Legacy name for Apple Distribution | App Store |
| **Apple Development** (unified) | All platforms development | Development |
| **Apple Distribution** (unified) | All platforms distribution | Distribution |
| **APNs Auth Key (.p8)** | Push notification token-based auth | APNs |
| **Pass Type ID** | Wallet passes | PassKit |
| **MDM Vendor Certificate** | MDM enrollment | MDM |

### Creating certificates
- **Xcode automatic**: Xcode > Settings > Accounts > Manage Certificates > + > Apple Development/Distribution.
- **Developer portal**: developer.apple.com > Certificates, Identifiers & Profiles > Certificates > +.
- **Fastlane match**: `fastlane match development`/`distribution` — shared certs via git repo.
- **CI**: App Store Connect API key (.p8) + `-allowProvisioningUpdates` for automatic signing.

## 3. Provisioning profiles

### Profile types
| Type | Devices | Distribution | Expiry |
|---|---|---|---|
| **Development** | Registered devices | Xcode debug builds | 1 year |
| **Ad Hoc** | Registered devices (up to 100 iOS / 100 Mac) | TestFlight? no — direct install | 1 year |
| **App Store** | No device limit | App Store / TestFlight | No expiry (managed by Apple) |
| **Enterprise** | No device limit | In-house enterprise distribution | 1 year |
| **Developer ID** | No device limit | macOS direct distribution | No expiry |
| **Mac App Store** | No device limit | Mac App Store | No expiry |
| **tvOS Profile** | Registered Apple TV devices | tvOS App Store | No expiry |
| **watchOS Profile** | Registered Apple Watch devices | watchOS App Store | No expiry |
| **visionOS Profile** | Registered Vision Pro devices | visionOS App Store | No expiry |

### Profile contents
- Certificate(s) — which certs can sign.
- App ID — which bundle ID.
- Devices — which UDIDs (development/ad hoc only).
- Entitlements — which capabilities enabled.
- Provisioning profile UUID — embedded in app.
- Expiration date.

### Xcode automatic vs manual signing
- **Automatic** (`CODE_SIGN_STYLE = Automatic`): Xcode manages certs/profiles; `-allowProvisioningUpdates` for CLI.
- **Manual** (`CODE_SIGN_STYLE = Manual`): specify `PROVISIONING_PROFILE_SPECIFIER` + `CODE_SIGN_IDENTITY`; more control, needed for CI/enterprise.

### Common signing errors
| Error | Cause | Fix |
|---|---|---|
| "No profiles for bundle ID" | Bundle ID not registered, wrong team | register App ID, check team, use automatic signing |
| "Provisioning profile doesn't include device" | Device UDID not in profile | register device, update profile, use development profile |
| "Certificate has expired" | Cert expired | renew cert, update profile |
| "Profile doesn't include entitlement" | Capability not enabled in App ID | enable capability in developer portal, update profile |
| "Ambiguous bundle IDs" | Multiple apps with same bundle ID | use unique bundle ID |
| "No signing certificate found" | Cert not installed or expired | install cert, check keychain |
| "Profile is invalid" | Profile expired or corrupted | regenerate profile |
| "Code signing identity not found" | `CODE_SIGN_IDENTITY` mismatch | check build settings, use automatic signing |

## 4. App Store Connect

### App setup
- **App Store Connect** (appstoreconnect.apple.com) — manage apps, builds, metadata, reviews.
- **Apps** — create app: name, bundle ID, SKU, primary language.
- **Versions** — create version: version string (CFBundleShortVersionString), build number (CFBundleVersion).
- **Builds** — upload via Xcode/altool/Transporter; processing takes 5-30 min.
- **TestFlight** — beta testing: internal (100 testers, no review), external (10,000 testers, beta review required).
- **App Review** — submit for review: metadata, screenshots, description, keywords, privacy, age rating.

### Build upload
```bash
# Xcode: Product > Archive > Distribute App > App Store Connect > Upload

# CLI: altool (deprecated but works)
xcrun altool --upload-app -f MyApp.ipa -t ios --apiKey KEY --apiIssuer ISSUER

# CLI: Transporter
xcrun iTMSTransporter -m upload -assetFile MyApp.ipa -apiKey KEY -apiIssuer ISSUER

# API: App Store Connect API
# POST /v1/builds with JWT auth
```

### Build processing
- Upload → processing (5-30 min) → ready for testing/submission.
- Processing includes: symbolication, encryption declaration, content rights, age rating.
- `ITSAppUsesNonExemptEncryption` — declare encryption usage in Info.plist or during processing.
- Export compliance: `com.apple.developer.encryption-info` entitlement or ASC questionnaire.

### TestFlight
- **Internal testers**: up to 100 team members, no review, immediate access.
- **External testers**: up to 10,000, requires beta review (~24h), what-to-test notes required.
- **Build expiration**: 90 days.
- **Install**: via TestFlight app, email invite, public link.
- **Beta app description**: required for external testing.
- **What to test**: required for external testing.
- **Export compliance**: encryption declaration required before external testing.
- **Crashes**: symbolicated in ASC; MetricKit for diagnostics.

### App Review
- **Guidelines**: 2.1 App Completeness, 2.2 Performance, 2.3 Metadata, 3.1 In-App Purchase, 3.1.1 Subscriptions, 3.1.2 Subscriptions (reader apps), 4.2 Minimum Functionality, 4.3 Spam, 4.4 Hardware Compatibility, 4.5 Software Requirements, 5.1 Privacy, 5.1.1 Data Collection, 5.1.2 Data Use, 5.1.3 Health/Financial, 5.2 Intellectual Property, 5.3 Gaming/Gambling, 5.4 VPN, 5.5 Developer Information.
- **Common rejections**:
  - 2.1: crashes, bugs, incomplete features, placeholder content.
  - 2.2: poor performance, battery drain, excessive memory.
  - 3.1: IAP issues, subscription problems, reader app restrictions.
  - 4.2: web wrapper, minimum functionality, template app.
  - 5.1: privacy policy missing, data collection without consent, ATT issues.
  - 5.1.1: data collection declaration mismatch.
- **Review timeline**: 24-48h typical; expedited review available for critical fixes.
- **Rejection handling**: read rejection reason, fix, resubmit; appeal if disagreement.
- **Phased release**: 7-day auto release for updates (can pause).

## 5. macOS distribution

### Mac App Store
- **Requirements**: sandboxed app, App Store review, sandbox entitlements.
- **Distribution**: Xcode > Archive > Distribute App > Mac App Store.
- **Receipt**: app receipt for purchase validation.
- **Prohibited**: non-sandboxed apps, certain entitlements.

### Developer ID (direct distribution)
- **Certificate**: Developer ID Application.
- **Notarization**: required for Gatekeeper (macOS 10.15+).
- **Distribution**: direct download, Setapp, Steam, etc.

#### Notarization
```bash
# Create zip/dmg
zip -r MyApp.zip MyApp.app
# or create dmg
hdiutil create -volname MyApp -srcfolder MyApp.app -ov -format UDZO MyApp.dmg

# Submit for notarization
xcrun notarytool submit MyApp.zip --keychain-profile AC_PROFILE --wait
# or with API key
xcrun notarytool submit MyApp.zip --key APP_STORE_CONNECT_API_KEY.p8 --key-id KEY_ID --issuer ISSUER_ID --wait

# Check status
xcrun notarytool history --keychain-profile AC_PROFILE
xcrun notarytool log <submission-id> --keychain-profile AC_PROFILE

# Staple ticket
xcrun stapler staple MyApp.app
xcrun stapler validate MyApp.app

# Verify
spctl -a -vv MyApp.app
# or
codesign -dvvv MyApp.app
```

- `notarytool` — modern notarization CLI; `altool` deprecated.
- `--wait` — wait for completion; returns status.
- `stapler` — attach notarization ticket to app/dmg.
- `spctl -a -vv` — verify Gatekeeper assessment.
- `xcrun stapler validate` — verify staple.
- Notarization ticket: stapled to app or checked online (requires internet).

### Sparkle (auto-updates)
- Third-party framework for macOS auto-updates.
- EdDSA signing: generate keys, `SUPublicEDKey` in Info.plist.
- `SUUpdater`/`SPUStandardUpdaterController` — check/update UI.
- Appcast feed: XML with update info, signatures.
- Alternatives: custom update checker, `Squirrel.Mac` (legacy).

## 6. Privacy manifests & nutrition labels

### PrivacyInfo.xcprivacy
- Required for apps using required reason APIs or collecting data.
- Declare: `NSPrivacyTracking`, `NSPrivacyTrackingDomains`, `NSPrivacyCollectedDataTypes`, `NSPrivacyAccessedAPITypes`.
- Third-party SDKs must include their own manifests — check dependencies.
- Xcode generates warnings for missing manifests.

### App Store nutrition labels
- ASC questionnaire: what data collected, how used, linked to user, tracking.
- Must match privacy manifest.
- Categories: Contact Info, Health, Financial Info, Location, Sensitive Info, Contacts, User Content, Browsing History, Search History, Identifiers, Usage Data, Diagnostics, Other Data.
- Purposes: App Functionality, Analytics, Product Personalization, Developer Advertising, Third Party Advertising, Other.

### Required reason APIs
- `NSUserDefaults` — `CA92.1`, `1C8F.1`, `C56D.1`.
- `FileManager` timestamps — `C617.1`, `0A2A.1`.
- `SystemUptime` — `35F9.1`, `8FFB.1`.
- `DiskSpace` — `E174.1`, `7D9E.1`.
- `ActiveKeyboard` — `3B52.1`, `54BD.1`.
- `SystemBootTime` — `DDA9.1`, `AC6B.1`.
- `FileTimestamp` — `C617.1`, `0A2A.1`.

## 7. APNs authentication

### Token-based (.p8)
```bash
# Generate Auth Key in developer portal: Certificates, Identifiers & Profiles > Keys > + > Apple Push Notifications service (APNs)
# Download AuthKey_XXXXXXXXXX.p8
# Note Key ID and Team ID

# JWT provider token (ES256)
# Header: { "alg": "ES256", "kid": "KEY_ID" }
# Payload: { "iss": "TEAM_ID", "iat": <timestamp> }
# Sign with .p8 private key
```

- `.p8` key — never expires, works for all apps in team.
- JWT provider token — refresh every 20-60 min (max 1 hour).
- APNs endpoints: `api.push.apple.com` (prod), `api.sandbox.push.apple.com` (sandbox).
- Headers: `apns-topic` = bundle ID (+ `.push-type.liveactivity` for Live Activities), `apns-push-type`, `apns-priority`, `apns-collapse-id`, `apns-expiration`, `apns-id`.
- Payload: `aps` dictionary — `alert`, `badge`, `sound`, `content-available`, `mutable-content`, `category`, `thread-id`, `interruption-level`, `relevance-score`, `target-content-id`.

### Certificate-based (legacy)
- `.p12` certificate + password — per-app, expires yearly.
- Prefer token-based (.p8).

## 8. Export compliance & encryption

- `ITSAppUsesNonExemptEncryption` — declare in Info.plist:
  ```xml
  <key>ITSAppUsesNonExemptEncryption</key>
  <false/>  <!-- if only using exempt encryption (HTTPS, etc.) -->
  <!-- or -->
  <true/>   <!-- if using non-exempt encryption (custom crypto) -->
  ```
- Exempt encryption: HTTPS/TLS, Apple-provided crypto, standard algorithms.
- Non-exempt: custom crypto, proprietary encryption, certain VPN/DRM.
- Annual self-classification report required for non-exempt.
- `com.apple.developer.encryption-info` entitlement for specific exemptions.

## 9. Versioning & build numbers

- `CFBundleShortVersionString` — marketing version (e.g. "1.2.3").
- `CFBundleVersion` — build number (e.g. "42"); must increment for each upload.
- `CFBundleVersion` must be unique per version string? — no, must be unique per upload; can reset per version.
- Build number: auto-increment via `agvtool` or build script.
- Version format: `major.minor.patch` (semver) or `major.minor.build`.
- `MARKETING_VERSION`/`CURRENT_PROJECT_VERSION` build settings.
- `agvtool new-version -all 42` — set build number.
- `agvtool next-version -all` — increment build number.

## 10. Screenshots & metadata

### Required screenshots
- iPhone: 6.7" (iPhone 15 Pro Max/16 Pro Max), 6.5" (iPhone 14 Plus/15 Plus/16 Plus), 5.5" (iPhone 8 Plus) — at least one required.
- iPad: 12.9" (iPad Pro), 11" (iPad Pro) — required if iPad supported.
- Mac: 1280x800, 1440x900, 2560x1600 — required if Mac supported.
- Apple Watch: 41mm, 45mm — required if Watch supported.
- Apple TV: 1920x1080 — required if TV supported.
- Vision Pro: required if visionOS supported.

### App previews
- Video previews: up to 30s, 30fps, H.264.
- Required for some categories; optional for others.

### Metadata
- App name (30 chars), subtitle (30 chars), description (4000 chars), keywords (100 chars), promotional text (170 chars), support URL, marketing URL.
- Age rating questionnaire: violence, sexual content, profanity, etc.
- Content rights: declare if using third-party content.
- Privacy policy URL: required.

## 11. Common distribution bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Archive fails signing | cert/profile mismatch, expired cert | check `CODE_SIGN_IDENTITY`, `PROVISIONING_PROFILE_SPECIFIER`, renew certs |
| Upload fails ITMS-9000 | invalid bundle, missing icon, wrong format | check bundle structure, icon, `altool --validate-app` |
| Build processing stuck | missing compliance, invalid binary | check encryption declaration, `xcrun altool --validate-app` |
| TestFlight install fails | build expired, device not registered, beta review pending | check build status, register device, wait for review |
| App Review rejection 2.1 | crashes, bugs, incomplete | fix issues, resubmit |
| App Review rejection 5.1 | privacy issues, ATT, data collection | fix privacy, add manifest, resubmit |
| Notarization fails | missing hardened runtime, invalid signature | check entitlements, `codesign -d --entitlements`, `spctl -a -vv` |
| Gatekeeper blocks app | not notarized, quarantined | notarize, staple, or user right-click > Open |
| Sparkle update fails | signature mismatch, appcast error | check EdDSA keys, appcast format, `SUPublicEDKey` |
| APNs push fails | wrong topic, expired token, sandbox/prod mismatch | check bundle ID, token validity, environment |
| Privacy manifest missing | required reason APIs used | add `PrivacyInfo.xcprivacy`, declare APIs |
| Version conflict | build number already used | increment `CFBundleVersion`, resubmit |

## 12. Distribution checklist

### Before archive
- [ ] Version/build numbers correct and incremented.
- [ ] All targets have correct bundle IDs and team.
- [ ] Entitlements match capabilities (push, iCloud, app groups, etc.).
- [ ] Privacy manifest included and accurate.
- [ ] Info.plist usage descriptions for all permissions.
- [ ] App icon complete (all sizes, dark/tinted variants).
- [ ] Launch screen configured.
- [ ] No debug code, print statements, or test data.
- [ ] Third-party licenses included if required.
- [ ] Crash reporting configured (Crashlytics/Sentry).
- [ ] Analytics configured (if used).

### Before upload
- [ ] Archive succeeds with correct signing.
- [ ] `altool --validate-app` passes.
- [ ] Export compliance declared.
- [ ] Screenshots for all required device sizes.
- [ ] Metadata complete (description, keywords, URLs).
- [ ] Privacy policy URL valid.
- [ ] Age rating accurate.
- [ ] TestFlight beta tested (if applicable).
- [ ] App Review guidelines reviewed.

### After upload
- [ ] Build processing completes.
- [ ] TestFlight internal testing (if applicable).
- [ ] External beta review submitted (if applicable).
- [ ] App Review submitted.
- [ ] Phased release configured (if update).
- [ ] Monitor crashes/feedback post-release.
