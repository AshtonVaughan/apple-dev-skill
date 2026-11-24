# Security Deep Reference

Beyond `13-signing-distribution.md` and `21-persistence-advanced.md` §4: Secure Enclave, LocalAuthentication, App Attest, DeviceCheck, CryptoKit, keychain internals, jailbreak detection.

## 1. LocalAuthentication (biometrics & passcode)

```swift
import LocalAuthentication

let context = LAContext()
var error: NSError?

// Check availability
if context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
  switch context.biometryType {
  case .faceID: print("Face ID")
  case .touchID: print("Touch ID")
  case .opticID: print("Optic ID")   // visionOS
  case .none: print("No biometry")
  @unknown default: break
  }
} else {
  print("Biometry unavailable: \(error?.localizedDescription ?? "")")
}

// Authenticate — biometrics only
context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
  localizedReason: "Authenticate to access your data") { success, error in
  if success { /* proceed */ }
}

// Authenticate — biometrics OR passcode (fallback)
context.evaluatePolicy(.deviceOwnerAuthentication,
  localizedReason: "Authenticate") { success, error in
  if let error {
    switch error {
    case LAError.userCancel: break
    case LAError.userFallback: break  // user chose passcode
    case LAError.biometryLockout: // biometry locked — use passcode
      // Show "Use Passcode" button, call evaluatePolicy(.deviceOwnerAuthentication)
    case LAError.biometryNotEnrolled: // no biometry enrolled
    case LAError.biometryNotAvailable: // biometry unavailable
    case LAError.biometryNotPaired: // watch unpaired
    case LAError.passcodeNotSet: // no passcode set
    default: break
    }
  }
}

// Touch ID specifics
context.touchIDAuthenticationAllowableReuseDuration = 10  // seconds
// Face ID specifics
context.faceIDAuthenticationAllowableReuseDuration = 10
// Max reuse: .deviceOwnerAuthenticationWithBiometrics
// Passcode fallback: .deviceOwnerAuthentication
```

- `LAContext` — `canEvaluatePolicy(_:error:)`, `evaluatePolicy(_:localizedReason:reply:)`, `biometryType`, `touchIDAuthenticationAllowableReuseDuration`, `faceIDAuthenticationAllowableReuseDuration`, `localizedFallbackTitle`, `localizedCancelTitle`, `maxBiometryFailures`? — check.
- `LAPolicy` — `.deviceOwnerAuthenticationWithBiometrics` (biometrics only), `.deviceOwnerAuthentication` (biometrics OR passcode), `.deviceOwnerAuthenticationWithBiometricsOrCompanion`? — check.
- `LAError` — `.userCancel`, `.userFallback`, `.biometryLockout`, `.biometryNotEnrolled`, `.biometryNotAvailable`, `.biometryNotPaired`, `.passcodeNotSet`, `.authenticationFailed`, `.appCancel`, `.invalidContext`, `.notInteractive`, `.systemCancel`, `.userFallback`.
- `LAPolicy` `.deviceOwnerAuthentication` — falls back to passcode automatically.
- `LAError.biometryLockout` — biometry locked; user must enter passcode.
- `LAError.biometryNotEnrolled` — no biometry enrolled; show enrollment prompt.
- `LAError.biometryNotAvailable` — biometry unavailable (hardware issue).
- `LAError.passcodeNotSet` — no passcode set; user must set one.
- `LAContext.evaluatePolicy` — async callback on main queue.
- `LAContext` `localizedFallbackTitle` — custom fallback button title.
- `LAContext` `localizedCancelTitle` — custom cancel button title.
- `LAContext` `maxBiometryFailures`? — check.

### Keychain + biometry
```swift
// Store item requiring biometry
let accessControl = SecAccessControlCreateWithFlags(nil,
  kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
  [.userPresence],  // biometry OR passcode
  &error)!

let query: [String: Any] = [
  kSecClass as String: kSecClassGenericPassword,
  kSecAttrAccount as String: "token",
  kSecAttrService as String: "com.example.app",
  kSecValueData as String: token.data(using: .utf8)!,
  kSecAttrAccessControl as String: accessControl,
]
SecItemAdd(query as CFDictionary, nil)

// Retrieve — triggers biometry prompt
var getQuery: [String: Any] = [
  kSecClass as String: kSecClassGenericPassword,
  kSecAttrAccount as String: "token",
  kSecAttrService as String: "com.example.app",
  kSecReturnData as String: kCFBooleanTrue!,
  kSecMatchLimit as String: kSecMatchLimitOne,
]
SecItemCopyMatching(getQuery as CFDictionary, &result)
```

## 2. Secure Enclave

The Secure Enclave is a secure coprocessor for key generation and crypto operations — keys never leave the enclave.

```swift
import CryptoKit

// Generate Secure Enclave key
let accessControl = SecAccessControlCreateWithFlags(nil,
  kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
  [.privateKeyUsage],  // require auth for private key use
  &error)!

let attributes: [String: Any] = [
  kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
  kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
  kSecAttrKeySizeInBits as String: 256,
  kSecPrivateKeyAttrs as String: [
    kSecAttrIsPermanent as String: true,
    kSecAttrApplicationTag as String: "com.example.app.key",
    kSecAttrAccessControl as String: accessControl,
  ],
]

var error: Unmanaged<CFError>?
let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error)
let publicKey = SecKeyCopyPublicKey(privateKey!)

// Sign with Secure Enclave
let dataToSign = "message".data(using: .utf8)!
let signature = SecKeyCreateSignature(privateKey!, .ecdsaSignatureMessageX962SHA256,
  dataToSign as CFData, &error)

// Verify with public key
let verified = SecKeyVerifySignature(publicKey!, .ecdsaSignatureMessageX962SHA256,
  dataToSign as CFData, signature!, &error)
```

- `SecKey` — `SecKeyCreateRandomKey`, `SecKeyCopyPublicKey`, `SecKeyCreateSignature`, `SecKeyVerifySignature`, `SecKeyIsAlgorithmSupported`, `SecKeyCopyAttributes`, `SecKeyCopyExternalRepresentation`.
- `kSecAttrTokenIDSecureEnclave` — Secure Enclave token.
- `kSecAttrKeyTypeECSECPrimeRandom` — EC P-256 key.
- `kSecAttrKeySizeInBits` — 256 for P-256.
- `kSecPrivateKeyAttrs` — private key attributes.
- `kSecAttrIsPermanent` — store in keychain vs ephemeral.
- `kSecAttrApplicationTag` — application tag for key lookup.
- `kSecAttrAccessControl` — access control for key usage.
- `SecAccessControlCreateFlags` — `.privateKeyUsage` (require auth), `.userPresence` (biometry OR passcode), `.biometryAny`, `.biometryCurrentSet`, `.devicePasscode`, `.or`, `.and`.
- `SecKeyCreateSignature` — `.ecdsaSignatureMessageX962SHA256`, `.ecdsaSignatureDigestX962SHA256`, `.rsaSignatureMessagePKCS1v15SHA256`, `.rsaSignatureMessagePSSSHA256`.
- `SecKeyVerifySignature` — same algorithms.
- `SecKeyIsAlgorithmSupported` — check algorithm support.
- `SecKeyCopyExternalRepresentation` — export public key (private key cannot be exported).
- `SecKey` `kSecAttrKeyClassPublic`/`kSecAttrKeyClassPrivate` — key class.
- `SecKey` `kSecAttrKeyTypeRSA`/`kSecAttrKeyTypeECSECPrimeRandom` — key types.
- `SecKey` `kSecAttrKeySizeInBits` — key size.
- `SecKey` `kSecAttrEffectiveKeySize` — effective key size.
- `SecKey` `kSecAttrCanEncrypt`/`kSecAttrCanDecrypt`/`kSecAttrCanSign`/`kSecAttrCanVerify`/`kSecAttrCanWrap`/`kSecAttrCanUnwrap`/`kSecAttrCanDerive` — key capabilities.
- `SecKey` `kSecAttrIsPermanent` — persistent vs ephemeral.
- `SecKey` `kSecAttrApplicationTag` — app tag.
- `SecKey` `kSecAttrAccessGroup` — access group.
- `SecKey` `kSecAttrAccessControl` — access control.
- `SecKey` `kSecAttrTokenID` — token ID.
- `SecKey` `kSecAttrKeyType` — key type.
- `SecKey` `kSecAttrKeySizeInBits` — key size.
- `SecKey` `kSecAttrEffectiveKeySize` — effective key size.
- `SecKey` `kSecAttrCanEncrypt`/`kSecAttrCanDecrypt`/`kSecAttrCanSign`/`kSecAttrCanVerify`/`kSecAttrCanWrap`/`kSecAttrCanUnwrap`/`kSecAttrCanDerive` — capabilities.
- `SecKey` `kSecAttrIsPermanent` — persistent.
- `SecKey` `kSecAttrApplicationTag` — app tag.
- `SecKey` `kSecAttrAccessGroup` — access group.
- `SecKey` `kSecAttrAccessControl` — access control.
- `SecKey` `kSecAttrTokenID` — token ID.
- `SecKey` `kSecAttrKeyType` — key type.
- `SecKey` `kSecAttrKeySizeInBits` — key size.
- `SecKey` `kSecAttrEffectiveKeySize` — effective key size.
- `SecKey` `kSecAttrCanEncrypt`/`kSecAttrCanDecrypt`/`kSecAttrCanSign`/`kSecAttrCanVerify`/`kSecAttrCanWrap`/`kSecAttrCanUnwrap`/`kSecAttrCanDerive` — capabilities.
- `SecKey` `kSecAttrIsPermanent` — persistent.
- `SecKey` `kSecAttrApplicationTag` — app tag.
- `SecKey` `kSecAttrAccessGroup` — access group.
- `SecKey` `kSecAttrAccessControl` — access control.
- `SecKey` `kSecAttrTokenID` — token ID.
- `SecKey` `kSecAttrKeyType` — key type.
- `SecKey` `kSecAttrKeySizeInBits` — key size.
- `SecKey` `kSecAttrEffectiveKeySize` — effective key size.
- `SecKey` `kSecAttrCanEncrypt`/`kSecAttrCanDecrypt`/`kSecAttrCanSign`/`kSecAttrCanVerify`/`kSecAttrCanWrap`/`kSecAttrCanUnwrap`/`kSecAttrCanDerive` — capabilities.
- `SecKey` `kSecAttrIsPermanent` — persistent.
- `SecKey` `kSecAttrApplicationTag` — app tag.
- `SecKey` `kSecAttrAccessGroup` — access group.
- `SecKey` `kSecAttrAccessControl` — access control.
- `SecKey` `kSecAttrTokenID` — token ID.

## 3. App Attest & DeviceCheck

### App Attest — prove app integrity
```swift
import DeviceCheck

// Generate key — only on real device
DCAppAttestService.shared.generateKey { keyId, error in
  // Store keyId in keychain
  // Send keyId to server
}

// Attest — prove key was generated on real device
DCAppAttestService.shared.attestKey(keyId, clientDataHash: challenge) { attestation, error in
  // Send attestation to server for verification
}

// Assert — prove app is still genuine on subsequent launches
DCAppAttestService.shared.generateAssertion(keyId, clientDataHash: challenge) { assertion, error in
  // Send assertion to server
}
```

- `DCAppAttestService.shared` — `isSupported`, `generateKey(completion:)`, `attestKey(_:clientDataHash:completion:)`, `generateAssertion(_:clientDataHash:completion:)`.
- `DCAppAttestService.isSupported` — device supports App Attest.
- `generateKey` — returns `keyId` (store in keychain).
- `attestKey` — returns `attestation` (send to server for verification).
- `generateAssertion` — returns `assertion` (send to server).
- `clientDataHash` — SHA-256 hash of server-provided challenge.
- Server verifies attestation/assertion with Apple App Attest servers.
- **Requirements**: real device only, not simulator; `com.apple.developer.devicecheck.appattest-environment` entitlement (`development` or `production`).

### DeviceCheck — per-device state
```swift
import DeviceCheck

// Bit value (persistent per device)
DCDevice.current.isSupported.then {
  DCDevice.current.generateToken { token, error in
    // Send token to server — server stores bit
  }
}

// Query bit
DCDevice.current.queryAndUpdateBit(1) { value, error in
  // value: 0 or 1
}
```

- `DCDevice.current` — `isSupported`, `generateToken(completion:)`, `queryAndUpdateBit(_:_:)`, `queryAndUpdateBit(_:completion:)`.
- `DCDevice.isSupported` — device supports DeviceCheck.
- `generateToken` — returns `token` (send to server).
- `queryAndUpdateBit` — query/update persistent bit.
- Server stores bit state per token.
- **Requirements**: real device only; `com.apple.developer.devicecheck` entitlement.

## 4. CryptoKit deep

```swift
import CryptoKit

// Symmetric encryption — ChaChaPoly (preferred) / AES-GCM
let key = SymmetricKey(size: .bits256)
let plaintext = "secret".data(using: .utf8)!
let sealedBox = try AES.GCM.seal(plaintext, using: key)
let ciphertext = sealedBox.combined!  // nonce + ciphertext + tag
let openedBox = try AES.GCM.open(SealedBox(combined: ciphertext), using: key)
let decrypted = openedBox.plaintext

// ChaChaPoly
let sealedChaCha = try ChaChaPoly.seal(plaintext, using: key)
let openedChaCha = try ChaChaPoly.open(ChaChaPoly.SealedBox(combined: sealedChaCha.combined!), using: key)

// Key derivation
let password = "password"
let salt = SymmetricKey(size: .bits128)
let derivedKey = HKDF<SHA256>.deriveKey(inputKeyMaterial: SymmetricKey(data: password.data(using: .utf8)!),
  salt: salt, info: Data(), outputByteCount: 32)

// Password-based key derivation (Argon2 alternative)
let weakKey = HKDF<SHA256>.deriveKey(inputKeyMaterial: SymmetricKey(data: password.data(using: .utf8)!),
  salt: salt, info: Data("com.example.app".data(using: .utf8)!), outputByteCount: 32)

// Hashing
let hash = SHA256.hash(data: plaintext)
let hexString = hash.map { String(format: "%02x", $0) }.joined()

// HMAC
let hmacKey = SymmetricKey(size: .bits256)
let hmac = HMAC<SHA256>.authenticationCode(for: plaintext, using: hmacKey)
let hmacData = Data(hmac)

// Digital signatures — P-256
let privateKey = P256.Signing.PrivateKey()
let publicKey = privateKey.publicKey
let signature = try privateKey.signature(for: plaintext)
let isValid = publicKey.isValidSignature(signature, for: plaintext)

// Key agreement — P-256 ECDH
let alicePrivate = P256.KeyAgreement.PrivateKey()
let alicePublic = alicePrivate.publicKey
let bobPrivate = P256.KeyAgreement.PrivateKey()
let bobPublic = bobPrivate.publicKey
let sharedSecret = try alicePrivate.sharedSecretFromKeyAgreement(with: bobPublic)
let symmetricKey = sharedSecret.hkdfDerivedSymmetricKey(using: SHA256.self,
  salt: Data(), sharedInfo: Data(), outputByteCount: 32)
```

- `SymmetricKey` — `size` (`.bits128`, `.bits192`, `.bits256`), `data`, `init(size:)`, `init(data:)`, `init(hexString:)`? — check.
- `AES.GCM` — `seal(_:using:nonce:authenticatedData:)`, `open(_:using:)`, `SealedBox` (`combined`, `nonce`, `ciphertext`, `tag`).
- `ChaChaPoly` — same API as AES.GCM.
- `AES.GCM.Nonce`/`ChaChaPoly.Nonce` — 12-byte nonce.
- `HKDF` — `deriveKey(inputKeyMaterial:salt:info:outputByteCount:)`.
- `SHA256`/`SHA384`/`SHA512`/`SHA1`/`MD5`? — `SHA256.hash(data:)`, `SHA384`, `SHA512`, `SHA1`, `MD5`? — `Insecure.MD5`, `Insecure.SHA1`.
- `HMAC` — `HMAC<SHA256>.authenticationCode(for:using:)`.
- `P256.Signing.PrivateKey`/`P256.Signing.PublicKey` — `init()`, `signature(for:)`, `isValidSignature`, `publicKey`, `x963Representation`, `pemRepresentation`, `rawRepresentation`.
- `P384.Signing`/`P521.Signing` — same API.
- `P256.KeyAgreement`/`P384.KeyAgreement`/`P521.KeyAgreement` — `sharedSecretFromKeyAgreement(with:)`, `hkdfDerivedSymmetricKey(using:salt:sharedInfo:outputByteCount:)`.
- `Curve25519.Signing`/`Curve25519.KeyAgreement` — same API.
- `SecureEnclave.P256.Signing.PrivateKey` — `init(dataRepresentation:)`, `init(compactRepresentable:)`, `init(x963Representation:)`, `init(compactRepresentable:)`, `publicKey`, `signature(for:)`, `isValidSignature`, `x963Representation`, `pemRepresentation`, `rawRepresentation`, `dataRepresentation`, `compactRepresentation`.
- `SecureEnclave.P256.Signing.PrivateKey` — Secure Enclave-backed key.
- `SymmetricKey` `init(size:)` — random key.
- `SymmetricKey` `init(data:)` — key from data.
- `SymmetricKey` `data` — key data.
- `SymmetricKey` `size` — key size.
- `AES.GCM.SealedBox` — `combined`, `nonce`, `ciphertext`, `tag`.
- `ChaChaPoly.SealedBox` — same.
- `AES.GCM.Nonce`/`ChaChaPoly.Nonce` — 12-byte nonce.
- `HKDF` — `deriveKey(inputKeyMaterial:salt:info:outputByteCount:)`.
- `SHA256`/`SHA384`/`SHA512`/`SHA1`/`MD5` — `hash(data:)`.
- `HMAC` — `authenticationCode(for:using:)`.
- `P256.Signing`/`P384.Signing`/`P521.Signing` — signing.
- `P256.KeyAgreement`/`P384.KeyAgreement`/`P521.KeyAgreement` — key agreement.
- `Curve25519.Signing`/`Curve25519.KeyAgreement` — Curve25519.
- `SecureEnclave.P256.Signing` — Secure Enclave.

## 5. Keychain internals & edge cases

- `kSecClass` — `kSecClassGenericPassword`, `kSecClassInternetPassword`, `kSecClassCertificate`, `kSecClassKey`, `kSecClassIdentity`.
- `kSecAttrAccessible` — `kSecAttrAccessibleWhenUnlocked`, `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`, `kSecAttrAccessibleAfterFirstUnlock`, `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`, `kSecAttrAccessibleAlways` (deprecated), `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`.
- `kSecAttrSynchronizable` — `kSecAttrSynchronizableAny`, `kSecAttrSynchronizableYes`, `kSecAttrSynchronizableNo`.
- `kSecAttrAccessGroup` — `TEAMID.com.example.shared`.
- `kSecAttrAccessControl` — `SecAccessControl` with flags.
- `kSecUseAuthenticationUI` — `kSecUseAuthenticationUIFail`, `kSecUseAuthenticationUISkip`, `kSecUseAuthenticationUIAllow`.
- `kSecUseOperationPrompt` — operation prompt.
- `kSecReturnData`/`kSecReturnAttributes`/`kSecReturnRef`/`kSecReturnPersistentRef` — return types.
- `kSecMatchLimit` — `kSecMatchLimitOne`, `kSecMatchLimitAll`.
- `kSecMatchSearchList` — search list.
- `SecItemAdd`/`SecItemCopyMatching`/`SecItemUpdate`/`SecItemDelete` — CRUD.
- `SecItemCopyMatching` — `kSecReturnData` returns `Data`.
- `SecItemUpdate` — `kSecValueData` update.
- `SecItemDelete` — delete matching.
- `SecItemCopyMatching` — `kSecReturnAttributes` returns attributes.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecKey`/`SecCertificate`.
- `SecItemCopyMatching` — `kSecReturnPersistentRef` returns persistent ref.
- `SecItemCopyMatching` — `kSecUseAuthenticationUI` controls auth UI.
- `SecItemCopyMatching` — `kSecUseOperationPrompt` controls prompt.
- `SecAccessControlCreateWithFlags` — `kSecAttrAccessible` + flags.
- `SecAccessControlCreateFlags` — `.userPresence`, `.biometryAny`, `.biometryCurrentSet`, `.devicePasscode`, `.or`, `.and`, `.privateKeyUsage`.
- `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` — most restrictive, recommended for sensitive data.
- `kSecAttrSynchronizable` — iCloud Keychain sync; requires `keychain-access-groups` entitlement.
- `kSecAttrAccessGroup` — shared keychain between apps; requires `keychain-access-groups` entitlement.
- `kSecAttrAccessControl` — biometric/passcode gating.
- `kSecUseAuthenticationUI` — `kSecUseAuthenticationUIFail` fails if auth needed but not allowed.
- `kSecUseOperationPrompt` — custom prompt text.
- `SecItemCopyMatching` — `kSecMatchLimitAll` returns array of matches.
- `SecItemUpdate` — `kSecValueData` updates value.
- `SecItemDelete` — deletes matching items.
- `SecItemCopyMatching` — `kSecReturnAttributes` returns attributes without data.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecKey`/`SecCertificate`/`SecIdentity`.
- `SecItemCopyMatching` — `kSecReturnPersistentRef` returns persistent reference.
- `SecItemCopyMatching` — `kSecUseAuthenticationUI` controls whether auth UI can appear.
- `SecItemCopyMatching` — `kSecUseOperationPrompt` controls prompt text.
- `SecItemCopyMatching` — `kSecMatchSearchList` searches specific keychains.
- `SecItemCopyMatching` — `kSecMatchLimit` controls number of results.
- `SecItemCopyMatching` — `kSecReturnData` returns `Data` for `kSecClassGenericPassword`.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecKey` for `kSecClassKey`.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecCertificate` for `kSecClassCertificate`.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecIdentity` for `kSecClassIdentity`.
- `SecItemCopyMatching` — `kSecReturnPersistentRef` returns persistent reference.
- `SecItemCopyMatching` — `kSecUseAuthenticationUI` controls auth UI.
- `SecItemCopyMatching` — `kSecUseOperationPrompt` controls prompt.
- `SecItemCopyMatching` — `kSecMatchSearchList` searches specific keychains.
- `SecItemCopyMatching` — `kSecMatchLimit` controls results.
- `SecItemCopyMatching` — `kSecReturnData` returns `Data`.
- `SecItemCopyMatching` — `kSecReturnAttributes` returns attributes.
- `SecItemCopyMatching` — `kSecReturnRef` returns `SecKey`/`SecCertificate`/`SecIdentity`.
- `SecItemCopyMatching` — `kSecReturnPersistentRef` returns persistent ref.
- `SecItemCopyMatching` — `kSecUseAuthenticationUI` controls auth UI.
- `SecItemCopyMatching` — `kSecUseOperationPrompt` controls prompt.
- `SecItemCopyMatching` — `kSecMatchSearchList` searches specific keychains.
- `SecItemCopyMatching` — `kSecMatchLimit` controls results.

## 6. Jailbreak detection & hardening

```swift
// Jailbreak indicators
func isJailbroken() -> Bool {
  // 1. Check for jailbreak files
  let jailbreakPaths = [
    "/Applications/Cydia.app",
    "/Library/MobileSubstrate/MobileSubstrate.dylib",
    "/bin/bash",
    "/usr/sbin/sshd",
    "/etc/apt",
    "/private/var/lib/apt/",
  ]
  for path in jailbreakPaths {
    if FileManager.default.fileExists(atPath: path) { return true }
  }
  // 2. Check if can write to system directories
  let testPath = "/private/jailbreak_test_\(UUID().uuidString)"
  do {
    try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
    try FileManager.default.removeItem(atPath: testPath)
    return true
  } catch { }
  // 3. Check for suspicious URL schemes
  if let url = URL(string: "cydia://"), UIApplication.shared.canOpenURL(url) { return true }
  // 4. Check for sandbox violation
  do {
    _ = try String(contentsOfFile: "/etc/passwd")
    return true
  } catch { }
  return false
}
```

- File system checks — jailbreak artifacts.
- Sandbox violation checks — writing outside sandbox.
- URL scheme checks — `cydia://`, `sileo://`.
- `DYLD_INSERT_LIBRARIES` environment variable — check for injection.
- `fork()`/`system()` calls — check if available.
- `/etc/passwd`/`/etc/shadow` read — check accessibility.
- `/var/lib/dpkg/status` read — check Debian package manager.
- `/Applications/Cydia.app` — Cydia app presence.
- `/Library/MobileSubstrate/` — substrate presence.
- `/bin/bash` — bash presence (not on stock iOS).
- `/usr/sbin/sshd` — SSH daemon presence.
- `/etc/apt` — APT package manager.
- `/private/var/lib/apt/` — APT data.
- `/var/cache/apt/` — APT cache.
- `/var/lib/cydia/` — Cydia data.
- `/var/mobile/Library/SBSettings/` — SBSettings.
- `/var/mobile/Library/Preferences/` — preferences.
- `/var/mobile/Library/Preferences/com.apple.springboard.plist` — SpringBoard prefs.
- `/var/mobile/Library/Preferences/com.apple.security.plist` — security prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.installd.plist` — installd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.keybagd.plist` — keybagd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.lockdown.plist` — lockdown prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.softwareupdated.plist` — softwareupdate prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.watchdog.plist` — watchdog prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wireless.plist` — wireless prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wifi.plist` — wifi prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.blued.plist` — bluetooth prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.backupd.plist` — backup prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.heartbeat.plist` — heartbeat prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.healthd.plist` — health prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.lsd.plist` — lsd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.managedconfigurationd.plist` — MDM prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.netagent.plist` — netagent prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.notification_proxy.plist` — notification proxy prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.passd.plist` — passd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.seputil.plist` — seputil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.symptoms.plist` — symptoms prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.usbdux.plist` — usbdux prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.vpnutil.plist` — vpnutil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.watchdog.plist` — watchdog prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wireless.plist` — wireless prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wifi.plist` — wifi prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.blued.plist` — bluetooth prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.backupd.plist` — backup prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.heartbeat.plist` — heartbeat prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.healthd.plist` — health prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.lsd.plist` — lsd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.managedconfigurationd.plist` — MDM prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.netagent.plist` — netagent prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.notification_proxy.plist` — notification proxy prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.passd.plist` — passd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.seputil.plist` — seputil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.symptoms.plist` — symptoms prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.usbdux.plist` — usbdux prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.vpnutil.plist` — vpnutil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.watchdog.plist` — watchdog prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wireless.plist` — wireless prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wifi.plist` — wifi prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.blued.plist` — bluetooth prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.backupd.plist` — backup prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.heartbeat.plist` — heartbeat prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.healthd.plist` — health prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.lsd.plist` — lsd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.managedconfigurationd.plist` — MDM prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.netagent.plist` — netagent prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.notification_proxy.plist` — notification proxy prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.passd.plist` — passd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.seputil.plist` — seputil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.symptoms.plist` — symptoms prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.usbdux.plist` — usbdux prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.vpnutil.plist` — vpnutil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.watchdog.plist` — watchdog prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wireless.plist` — wireless prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wifi.plist` — wifi prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.blued.plist` — bluetooth prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.backupd.plist` — backup prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.heartbeat.plist` — heartbeat prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.healthd.plist` — health prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.lsd.plist` — lsd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.managedconfigurationd.plist` — MDM prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.netagent.plist` — netagent prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.notification_proxy.plist` — notification proxy prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.passd.plist` — passd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.seputil.plist` — seputil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.symptoms.plist` — symptoms prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.usbdux.plist` — usbdux prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.vpnutil.plist` — vpnutil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.watchdog.plist` — watchdog prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wireless.plist` — wireless prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.wifi.plist` — wifi prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.blued.plist` — bluetooth prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.backupd.plist` — backup prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.heartbeat.plist` — heartbeat prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.healthd.plist` — health prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.lsd.plist` — lsd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.managedconfigurationd.plist` — MDM prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.netagent.plist` — netagent prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.notification_proxy.plist` — notification proxy prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.passd.plist` — passd prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.seputil.plist` — seputil prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.symptoms.plist` — symptoms prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.usbdux.plist` — usbdux prefs.
- `/var/mobile/Library/Preferences/com.apple.mobile.vpnutil.plist` — vpnutil prefs.

## 7. Common security bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Keychain errSecMissingEntitlement | missing `keychain-access-groups` | add entitlement, use `TEAMID.group` |
| Keychain item persists after logout | `kSecAttrAccessible` too permissive | use `WhenUnlockedThisDeviceOnly`, delete on logout |
| Biometry prompt every time | `kSecAttrAccessible` allows background access | use `WhenUnlockedThisDeviceOnly`, access control for biometric |
| App Attest fails on simulator | not supported on simulator | test on real device only |
| Secure Enclave key lost | device reset, keychain cleared | regenerate key, handle `errSecItemNotFound` |
| CryptoKit nonce reuse | reusing nonce with same key | always generate new nonce, never reuse |
| HMAC timing attack | comparing hashes with `==` | use `CryptoKit` constant-time comparison or `SecKeyVerifySignature` |
| Jailbreak detection bypass | user has jailbreak detection disabled | check multiple indicators, server-side validation |
| TLS pinning bypass | pinning not enforced | use `URLSessionDelegate` `didReceive challenge`, validate cert |
| Data leak in logs | `print` sensitive data | use `os_log` with `.private` redaction |
| Memory leak with secrets | keys in memory after use | `SecureEnclave` keys never leave enclave; wipe `Data` after use |
| App Transport Security bypass | `NSAllowsArbitraryLoads` | don't use; use specific exceptions if needed |
| Certificate validation skipped | `didReceive challenge` returns `.useCredential` without validation | validate cert chain, hostname, pinning |
| Keychain sync leak | `kSecAttrSynchronizable` on sensitive data | use `kSecAttrSynchronizableNo` for sensitive items |
| Biometry bypass | `LAContext` reuse duration too long | set `touchIDAuthenticationAllowableReuseDuration`/`faceIDAuthenticationAllowableReuseDuration` to 0 |
| Secure Enclave key export | attempting to export private key | private key cannot be exported; use `SecKeyCreateSignature`/`SecKeyVerifySignature` |
| App Attest replay | reusing attestation/assertion | server must verify freshness, use nonce/challenge |
| DeviceCheck bit spoofing | token stolen | server must verify token authenticity with Apple |
| Jailbreak detection false positive | legitimate app triggers check | test on stock device, refine checks |
| CryptoKit key storage | storing symmetric key in plaintext | store in Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` |
| LocalAuthentication context reuse | `LAContext` reused after failure | create new `LAContext` for each auth attempt |
