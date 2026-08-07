# macOS app bundles, code signing, entitlements, notarization, Gatekeeper, and TCC — a practitioner's manual

> Written as a learning companion for Uri working on screenpipe. Every concept is anchored to the screenpipe app (`/Applications/screenpipe.app`, bundle id `screenpi.pe.dev`) where possible. The goal is to take you from "why does my app get a fresh 'Allow Screen Recording' prompt after a rebuild?" to "I can diagnose and fix any signing / permission issue on this machine."
>
> Read top-to-bottom the first time. After that, use §§ as a reference.
>
> Jeremy-Howard-style whole-game framing first: when you launch a Mac app, six subsystems inspect it in order — **Quarantine → LaunchServices → Gatekeeper → the kernel's code-signing enforcement → the app's entitlements → TCC**. Each one can say "no." Each one reads a specific thing from the app. This manual walks you through what each one reads, why, and how to troubleshoot when it says "no."

---

## Table of contents

1. [The `.app` bundle — what it actually is](#1-the-app-bundle--what-it-actually-is)
2. [`Info.plist` — the bundle's identity card](#2-infoplist--the-bundles-identity-card)
3. [Code signing — signatures, identifiers, and seals](#3-code-signing--signatures-identifiers-and-seals)
4. [The Designated Requirement (DR)](#4-the-designated-requirement-dr)
5. [Entitlements — what the app is *allowed to try*](#5-entitlements--what-the-app-is-allowed-to-try)
6. [The hardened runtime](#6-the-hardened-runtime)
7. [Notarization and stapling](#7-notarization-and-stapling)
8. [Gatekeeper and the quarantine bit](#8-gatekeeper-and-the-quarantine-bit)
9. [LaunchServices — the path-to-bundle-ID cache](#9-launchservices--the-path-to-bundle-id-cache)
10. [TCC — the "Allow *app* to..." system](#10-tcc--the-allow-app-to-system)
11. [How it all composes at launch time](#11-how-it-all-composes-at-launch-time)
12. [Tauri / Electron specifics](#12-tauri--electron-specifics)
13. [Diagnostic cookbook — commands that earn their keep](#13-diagnostic-cookbook--commands-that-earn-their-keep)
14. [Applied to screenpipe's current state](#14-applied-to-screenpipes-current-state)
15. [Further reading](#15-further-reading)

---

## 1. The `.app` bundle — what it actually is

A macOS `.app` looks like a single icon in Finder, but in the filesystem it's just a **directory** with a specific layout. Right-click → "Show Package Contents" and you can browse it.

Minimum anatomy:

```
screenpipe.app/
└── Contents/
    ├── Info.plist                 ← identity & capabilities declarations
    ├── MacOS/
    │   ├── screenpipe-app         ← the Mach-O executable (what actually runs)
    │   ├── bun                    ← additional helper binaries
    │   └── ffmpeg
    ├── Resources/                 ← images, icns, Assets.car, nibs, localized strings
    │   ├── icon.icns
    │   └── en.lproj/
    ├── Frameworks/                ← bundled dynamic libraries (optional)
    ├── PlugIns/                   ← app extensions (optional)
    ├── _CodeSignature/            ← signature of everything above
    │   └── CodeResources
    └── embedded.provisionprofile  ← for iOS; rare on macOS
```

Rules the kernel cares about:

- **Everything inside `Contents/` is part of the "signed code."** A signature seals the whole bundle contents, not just the executable. Change one pixel in `Resources/icon.icns` and the signature breaks.
- The Mach-O in `Contents/MacOS/<name>` must match `CFBundleExecutable` from `Info.plist` (and usually also `CFBundleName`, though that one is display-only).
- `_CodeSignature/CodeResources` is a hash of every non-executable file in the bundle. That hash is what `codesign --verify` re-computes.

**Python analogy (loose):** a `.app` is to a macOS binary what a Python *package directory with `__init__.py`* is to a bare `.py` file — it groups the executable with its metadata and resources, and tools treat the whole thing as one atomic unit.

**Concrete for screenpipe:**

- `productName` in `tauri.conf.json` decides the bundle's directory name: `"screenpipe - Development"`.
- `identifier` in `tauri.conf.json` becomes `CFBundleIdentifier` in the generated `Info.plist`: `"screenpi.pe.dev"`.
- Extra binaries like `bun` and `ffmpeg` are declared in `bundle.externalBin` (or just dropped into `Contents/MacOS/` by Tauri's resource rules).

---

## 2. `Info.plist` — the bundle's identity card

`Contents/Info.plist` is a small XML (or binary plist) file that answers "who are you?" to macOS. Every key matters because different subsystems read different keys.

Keys you'll meet in practice:

| Key | Purpose | Who reads it |
|---|---|---|
| `CFBundleIdentifier` | The globally-unique bundle ID, reverse-DNS: `com.apple.Finder`, `screenpi.pe.dev`. **TCC's primary key.** | TCC, LaunchServices, URL schemes, defaults |
| `CFBundleExecutable` | Filename in `MacOS/` to run | Launch services, kernel |
| `CFBundleVersion` | Monotonic build number (`"2.4.12"`, `"2025.1.1"`) | Updaters, version compare |
| `CFBundleShortVersionString` | Marketing version shown to user | Finder "Get Info", App Store |
| `CFBundleName` / `CFBundleDisplayName` | Human-readable name | Finder, Dock, menu bar |
| `LSMinimumSystemVersion` | Won't launch below this macOS | launch-time guard |
| `LSUIElement` / `LSBackgroundOnly` | Hide from Dock / no UI at all | Dock, menubar apps |
| `NSScreenCaptureUsageDescription` | **Required** string shown in TCC's "Allow?" prompt for screen recording | TCC |
| `NSMicrophoneUsageDescription` | Same, for mic | TCC |
| `NSCameraUsageDescription`, `NSAccessibilityUsageDescription`, `NSAppleEventsUsageDescription`, ... | Same pattern | TCC |
| `NSAppTransportSecurity` | HTTP → HTTPS policy | Foundation networking |
| `NSHighResolutionCapable` | Retina awareness | AppKit |
| `CFBundleURLTypes` | Custom URL scheme handlers (e.g. `screenpipe://`) | LaunchServices |
| `CFBundleDocumentTypes` | File types the app handles | Finder |

**Critical TCC rule:** if you want to request a TCC-protected capability (screen recording, microphone, camera, contacts, etc.) and there is **no matching `NS...UsageDescription` string** in `Info.plist`, macOS kills your app with a crash log like `This app has crashed because it attempted to access privacy-sensitive data without a usage description.`

Look at screenpipe's `Info.plist` after building:

```bash
plutil -p "/Applications/screenpipe.app/Contents/Info.plist" | head -40
```

or read a single key:

```bash
defaults read "/Applications/screenpipe.app/Contents/Info" CFBundleIdentifier
# → screenpi.pe.dev
```

**Python analogy:** `Info.plist` is `setup.py` + `pyproject.toml` merged: package name, version, entry point, required capabilities — but enforced by the OS, not just by pip.

---

## 3. Code signing — signatures, identifiers, and seals

**What signing actually does:**

1. The `codesign` tool computes a cryptographic hash of every file in the bundle.
2. Those hashes are signed with a certificate's private key.
3. The resulting signature + the cert's public chain are embedded into the Mach-O binary (`LC_CODE_SIGNATURE` load command) and into `_CodeSignature/CodeResources`.

**What the kernel does at launch:** it re-computes the hashes and verifies them against the embedded signature using the embedded cert chain. If anything changed (one byte edited, a sub-file added, the binary patched), the signature breaks and the kernel refuses to load the code — or loads it with reduced privileges, depending on policy.

### Three signing modes you'll see

| Mode | Command | What it does | When to use |
|---|---|---|---|
| **Adhoc** | `codesign --sign -` | Signs with no cert — embeds hashes but no identity. Kernel loads it, but no Gatekeeper-accepted identity, no TCC-persisted grants. | Local quick rebuilds. **What our Tauri build produced by default.** |
| **Developer ID** | `codesign --sign "Developer ID Application: Your Name (TEAMID)"` | Signed with a specific cert from your keychain. TCC and Gatekeeper recognize the cert. | Distribution outside the App Store. |
| **Apple Distribution** | `codesign --sign "Apple Distribution: ..."` | For App Store submission. | Mac App Store only. |

You can also use a **local / self-issued** codesigning certificate (what "Uri Gil Code Sign" appears to be in our case). It's not Apple-trusted for Gatekeeper, but it's trusted by TCC on this machine because the cert is in your local keychain. Enough for dev builds where you want persistent permissions.

### Keys and fields a signature carries

```bash
codesign -dvvv "/Applications/screenpipe.app" 2>&1
```

Key lines:

- `Identifier=screenpi.pe.dev` — the **code signing identifier** (NOT the same as `CFBundleIdentifier`, though by convention they match).
- `Authority=Uri Gil Code Sign` — the cert that signed it (bottom of the chain; top is `Authority=Apple Root CA` for Apple-issued certs).
- `TeamIdentifier=XXXXXXXXXX` — the 10-char Apple developer team ID (only for certs from Apple; blank for local certs).
- `Signature size=...` — the CMS blob size.
- `CDHash=...` — content digest hash, the unique fingerprint of this signed code.
- `Sealed Resources version=... rules=...` — how strictly the bundle's files are sealed.

### Useful commands

```bash
# Verify the signature is intact
codesign --verify --deep --strict --verbose=2 /Applications/screenpipe.app

# Show designated requirement
codesign -d --requirements - /Applications/screenpipe.app

# Re-sign an app with a specific cert (the fix for adhoc builds)
codesign --force --deep --sign "Uri Gil Code Sign" /path/to/app

# Strip signature entirely (sometimes needed when something is fighting you)
codesign --remove-signature /path/to/app
```

**`--deep` caveat:** Apple has been quietly deprecating `--deep` because it's often the wrong thing — it re-signs nested frameworks with the *same* identity, which isn't always what you want for complex bundles. For leaf apps like screenpipe with only a handful of bundled binaries, it's fine. For apps with many third-party frameworks, sign each framework explicitly first, then the app.

**Python analogy:** code signing is to a Mac app what **sigstore** or **PyPI's 2FA-signed releases** are to a Python package — cryptographic proof the thing you're running is the thing the author built, unmodified. Except macOS enforces it at kernel load time, not at install time.

---

## 4. The Designated Requirement (DR)

The **designated requirement** is a logical predicate embedded in a signed binary that says "*this* binary is considered the same identity as *some other* binary iff both satisfy this predicate."

Example DR (human-readable form you get from `codesign -d --requirements -`):

```
designated => anchor apple generic
  and identifier "screenpi.pe.dev"
  and (certificate leaf[field.1.2.840.113635.100.6.1.9] /* = 1 */
       or certificate 1[field.1.2.840.113635.100.6.2.6] /* Apple Dev ID Root */
          and certificate leaf[field.1.2.840.113635.100.6.1.13] /* Dev ID App */
          and certificate leaf[subject.OU] = "YOUR_TEAM_ID")
```

Parsed: *this binary is the same app as another binary iff the other has CFBundleIdentifier `screenpi.pe.dev` AND is signed by a specific cert chain rooted at Apple with the same team ID.*

**Why this matters:** TCC, Gatekeeper, keychain access groups, XPC connection validation, and several other subsystems key on **bundle ID + DR**, not on file path. That's why:

- Renaming `/Applications/Foo.app` → `/Applications/FooBar.app` doesn't break TCC grants.
- Rebuilding with the same signing cert doesn't break TCC grants.
- Rebuilding *with a different cert* (or with adhoc) **does** break TCC grants — the DR no longer matches.

**Default DR:** if you don't specify one, `codesign` generates a DR that matches the cert you signed with. That's almost always what you want.

**Custom DR:** for advanced scenarios (XPC between your helper and your main app, for instance), you can define a DR that accepts multiple certs or team IDs.

Look at the DR for any app:

```bash
codesign -d --requirements - /Applications/screenpipe.app 2>&1
```

Adhoc-signed apps have a minimal / empty-ish DR that essentially says "signed adhoc" — which matches nothing persistent, which is why TCC won't remember you.

---

## 5. Entitlements — what the app is *allowed to try*

Entitlements are a second plist, embedded *inside* the signature, that grants the binary specific capabilities. Think of them as compile-time-baked permissions: they're declared by the developer, preserved by the signature, and enforced by the kernel.

An entitlements file (`entitlements.plist`) looks like:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <key>com.apple.security.device.audio-input</key>
    <true/>
    <key>com.apple.security.device.camera</key>
    <true/>
    <key>com.apple.security.automation.apple-events</key>
    <true/>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
</dict>
</plist>
```

The file is passed at signing time with `--entitlements path/to/entitlements.plist`. Once signed, the entitlements are part of the signature — modifying them breaks the seal.

### Common entitlement buckets

**Hardened-runtime exceptions** (`com.apple.security.cs.*`) — for when your app does something macOS treats as suspicious:

- `allow-jit` — needed for JS engines, WebKit in renderer processes.
- `allow-unsigned-executable-memory` — for interpreters that generate code at runtime.
- `disable-library-validation` — allow loading dylibs not signed with the same team ID (required if you bundle third-party frameworks that aren't your own).
- `disable-executable-page-protection` — rarely needed.

**Device access** (`com.apple.security.device.*`):

- `audio-input`, `camera`, `usb`, `bluetooth` — declares that the app *wants* to access these.

**Sandbox** (`com.apple.security.app-sandbox`) — opts the app into the App Sandbox. This is **required for Mac App Store** and forbidden by most serious Apple-external tools (including screenpipe — global screen capture can't work sandboxed).

**Apple Events / automation** (`com.apple.security.automation.apple-events`) — to control other apps via AppleScript / osascript.

**Personal information** (`com.apple.security.personal-information.*`) — calendar, location, contacts. These are *declarations of intent*; TCC still gatekeeps the actual data access.

### Entitlements vs. Info.plist usage-description strings

This confuses everyone the first time.

- **`NSScreenCaptureUsageDescription`** in `Info.plist` = "the user-facing *string* shown in the 'Allow?' prompt."
- **Entitlements** (`com.apple.security.device.*`) = "my signed binary is allowed to *try* this capability at the OS level."
- **TCC grant** = "the user has said yes to letting *this specific app (bundle ID + DR)* actually use the capability."

You need *all three aligned* for a TCC-protected capability to work:
1. Correct usage-description string in `Info.plist` (so the prompt can show).
2. Correct entitlement in the signed entitlements plist (or hardened-runtime exception if your runtime requires it).
3. User-granted TCC permission.

Miss (1) → crash at first access. Miss (2) → hardened-runtime denies the syscall before TCC even sees it (for low-level resources). Miss (3) → prompt appears (or does nothing, if already denied).

Read the embedded entitlements of any signed binary:

```bash
codesign -d --entitlements :- /Applications/screenpipe.app 2>/dev/null
```

**Python analogy:** entitlements are like PEP-508 dependency markers + extras: declared metadata that tooling consults before doing the actual work. Except here "tooling" is the kernel and "the work" is a syscall.

---

## 6. The hardened runtime

A set of stricter runtime behaviors that Apple requires for notarized distribution (§ 7). Signed without hardened runtime? → notarization rejects you. Signed with it but you need exceptions? → add the right `com.apple.security.cs.*` entitlements above.

What it changes:

- Strict `DYLD_*` enforcement (environment variables are stripped).
- No unsigned executable memory (unless you add `allow-jit` or `allow-unsigned-executable-memory`).
- Library validation: all dylibs must be signed by the same team ID (unless `disable-library-validation`).
- Debugger attachment restricted.

Enable by signing with `--options=runtime` and/or `--options=hard`.

**Tauri default:** `tauri.conf.json → bundle.macOS.hardenedRuntime: true` enables it automatically. Screenpipe's config has this set.

---

## 7. Notarization and stapling

**Notarization** = you upload the signed app to Apple, Apple runs automated malware scans and static analysis, and if it passes, Apple signs a ticket that says "we looked at this build on date X and it's fine."

This is *not* App Store review — it's automated, usually 5–15 minutes.

**Stapling** = you download the notarization ticket and `xcrun stapler staple your.app`, which embeds the ticket inside the bundle so Gatekeeper can verify offline (no network round-trip to Apple's servers at user launch).

Workflow:

```bash
# 1. Build and sign
codesign --sign "Developer ID Application: ..." --options=runtime --entitlements e.plist Foo.app

# 2. Zip (notary takes zip, not .app)
ditto -c -k --keepParent Foo.app Foo.zip

# 3. Submit
xcrun notarytool submit Foo.zip \
  --apple-id you@example.com --password APP_SPECIFIC_PW --team-id TEAMID \
  --wait

# 4. Staple the ticket
xcrun stapler staple Foo.app

# 5. Distribute (dmg, pkg, raw app — stapled ticket travels with it)
```

**Why you care as a user:** a notarized + stapled app launches on a fresh Mac with one click. An unnotarized app shows "cannot be opened because Apple cannot check it for malicious software" and requires right-click → Open, or `xattr -rd com.apple.quarantine`.

**Our screenpipe dev build is NOT notarized** — that's fine on your own machine (you manually accept it once), but it's why public releases need a real `Developer ID Application` cert + notary submission.

---

## 8. Gatekeeper and the quarantine bit

### The quarantine bit

Files downloaded by a browser, mail client, AirDrop, etc., get tagged with an **extended attribute** called `com.apple.quarantine`. That bit is what triggers the "are you sure you want to open this?" dialog the first time you run something from the internet.

See it:

```bash
xattr -l ~/Downloads/some-app.zip
# com.apple.quarantine: 0083;64f21234;Safari;...
```

Remove it:

```bash
xattr -rd com.apple.quarantine /Applications/SomeApp.app
```

### Gatekeeper

The subsystem that checks quarantined apps against policy before allowing launch. Three assessments it can return:

- **Pass** — signed by Apple, App Store, or notarized Developer ID.
- **Fail, user-allowable** — signed but not notarized. User can right-click → Open once to whitelist.
- **Fail, blocked** — adhoc-signed or unsigned app from the internet. Blocked outright.

Apps *you built locally* never get a quarantine bit (your shell tools don't set it), so Gatekeeper ignores them. That's why `cargo build && open target/release/Foo.app` Just Works on your machine while handing the same `.app` to a friend makes macOS complain.

### `spctl` — Gatekeeper's CLI

```bash
# Assess an app
spctl --assess --verbose /Applications/screenpipe.app
# → /Applications/screenpipe.app: accepted
#   source=Notarized Developer ID
# (or "rejected" with a reason)

# Approve an adhoc-built local app manually
spctl --add --label "screenpipe-dev" /path/to/bundle
```

---

## 9. LaunchServices — the path-to-bundle-ID cache

`launchservicesd` maintains a cache (`~/Library/Caches/com.apple.LaunchServices.*`, system-wide copies elsewhere) that maps:

- File extensions → which app handles them
- URL schemes → which app handles them (`screenpipe://...` → screenpipe.app)
- Bundle IDs → paths on disk

When it gets wrong (usually after moving/renaming apps or having multiple copies with the same bundle ID), weird things happen: double-clicking a file opens the wrong app, `open -b com.your.app` launches an outdated version, URL schemes go to ghosts of moved apps.

**Nuke the cache:**

```bash
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister \
  -kill -r -domain local -domain system -domain user
```

(It regenerates itself on next login or first open.)

**Why this matters for us:** we discussed Option B for installing the rebuilt screenpipe (side-by-side). Two copies with the same `CFBundleIdentifier = screenpi.pe.dev` would confuse LaunchServices — `open -b screenpi.pe.dev` becomes nondeterministic. That's the concrete reason Option A (rename-and-replace) is cleaner.

---

## 10. TCC — the "Allow *app* to..." system

**TCC** = *Transparency, Consent, and Control*. It's the macOS subsystem behind every "*App X* would like to access *Resource Y*" prompt: Screen Recording, Microphone, Camera, Accessibility, Full Disk Access, Contacts, Calendars, Reminders, Photos, AppleEvents, Automation, Input Monitoring, Developer Tools, and a growing list of others.

### Where TCC lives

Two SQLite databases:

| Database | Path | Scope |
|---|---|---|
| **User TCC** | `~/Library/Application Support/com.apple.TCC/TCC.db` | Your user's grants. Most permissions live here. |
| **System TCC** | `/Library/Application Support/com.apple.TCC/TCC.db` | Machine-wide grants. Screen Recording, Accessibility, Full Disk Access live here on modern macOS. |

They are **owned by TCC's daemon (`tccd`)** and protected by SIP. Even `sudo sqlite3` can't modify the System DB on a modern SIP-enabled machine — you'd have to boot into Recovery and disable SIP, which is rarely what you actually want.

But you can *read* them (user DB, anyway):

```bash
sqlite3 "$HOME/Library/Application Support/com.apple.TCC/TCC.db" \
  "SELECT service, client, auth_value, auth_reason, last_modified
     FROM access
    WHERE client LIKE '%screenpi%';"
```

- `service` — the protected resource: `kTCCServiceScreenCapture`, `kTCCServiceMicrophone`, `kTCCServiceAccessibility`, etc.
- `client` — the bundle ID of the granted app: `screenpi.pe.dev`.
- `auth_value` — `0=denied, 1=unknown, 2=allowed, 3=limited`.
- `auth_reason` — how the grant was established (user-prompted, MDM, system).

### What TCC keys on

The grant row's effective key is a tuple:

```
(service, client, client_type, csreq)
```

- `service` — kind of resource.
- `client` — usually the bundle ID.
- `client_type` — bundle (0), script path (1), etc.
- `csreq` — a **serialized code-signing requirement blob** derived from the Designated Requirement (§4) at grant time.

When your app requests the resource, `tccd` looks up `(service, client)`, then validates the running binary's code signature against the stored `csreq`. **If the DR changed, `csreq` no longer matches, and TCC treats it as a new app — fresh prompt.** This is the single most important sentence in this manual.

### Why our adhoc-signed rebuild would re-prompt

The old binary's DR was effectively "bundle id `screenpi.pe.dev` AND signed by `Uri Gil Code Sign`". That DR was serialized into `csreq` when you first granted screen recording permission. The adhoc-signed rebuild has a DR that doesn't satisfy that predicate (no cert chain at all). So `tccd` sees a stranger.

Re-signing with the same cert (`codesign --sign "Uri Gil Code Sign"`) produces a DR that matches `csreq` — grant preserved.

### Reset TCC for one app / one service

```bash
# Reset ALL grants for a specific bundle ID
tccutil reset All screenpi.pe.dev

# Reset one service for all apps (use sparingly — bugs a lot of things)
tccutil reset ScreenCapture

# Reset one service for one app
tccutil reset ScreenCapture screenpi.pe.dev
```

**Useful when:** you suspect TCC is caching something stale; you re-signed with a different cert and want a clean slate.

### What TCC *doesn't* do

- It doesn't gatekeep resources that don't require consent (network, most filesystem reads in your home dir, etc.).
- It doesn't apply to processes running as root from `launchd` system domains.
- It's bypassable via MDM profiles (`PPPC` payloads) on managed machines — which is how enterprises pre-approve apps.

### The `NSScreenCaptureUsageDescription` trap

If you (or Tauri on your behalf) forget the `NSScreenCaptureUsageDescription` string in `Info.plist`, the *first* attempt to call `CGRequestScreenCaptureAccess()` or `SCShareableContent.current` doesn't prompt — it crashes the app with a TCC privacy violation. Always check:

```bash
defaults read "/Applications/screenpipe.app/Contents/Info" NSScreenCaptureUsageDescription
```

---

## 11. How it all composes at launch time

Full sequence the first time the user double-clicks a downloaded app:

```
1. Finder sees com.apple.quarantine xattr → asks Gatekeeper
2. Gatekeeper (spctl) evaluates:
     a. Is it signed?                   → codesign verification
     b. By a trusted authority?          → cert chain check
     c. Notarized?                      → notary ticket (embedded via staple, or online lookup)
   If pass: set approval bit, strip quarantine, move on.
   If fail: show dialog or block.
3. LaunchServices registers the bundle ID → path mapping
4. Kernel loads the Mach-O:
     a. Verifies code signature (hash of every page vs. embedded signature)
     b. Reads entitlements from the signature blob
     c. Enforces hardened-runtime rules if entitlement set says so
5. The app starts running. It calls into a TCC-protected API:
     a. macOS framework code calls XPC to `tccd`
     b. tccd looks up (service, bundle_id) in TCC.db
     c. If no row: show consent prompt (using NSXXXUsageDescription text)
        User says yes → insert row with csreq = current DR → proceed.
     d. If row exists: validate running binary against stored csreq
        Matches → allow.
        Doesn't match → deny (silently on older paths, with a prompt on newer).
6. Finally the syscall executes (screen capture, mic, etc.)
```

Each step is a separate subsystem with separate failure modes. When something "just doesn't work," the diagnostic question is always: *at which step did the denial happen?* — not *why doesn't permission work?*

**Teaching moment — debugging is like `traceroute`.**

You don't ask "is my network broken?" You ask "at which hop does the packet die?" Same here: when screen recording doesn't work, don't ask "is TCC broken?" — ask "does the signature verify? does the entitlement exist? does the DR match the stored csreq? is there a TCC grant row?" Four distinct questions, four distinct commands below.

---

## 12. Tauri / Electron specifics

### Tauri's build flow (for screenpipe)

1. `bun run build` → produces `../out/` (Next.js static export).
2. `cargo build --release` inside `src-tauri/` → produces the native Mach-O.
3. Tauri bundles both + icons + `Info.plist` (generated from `tauri.conf.json`) into a `.app`.
4. Tauri signs with `bundle.macOS.signingIdentity` if set, else **falls back to adhoc** (what bit us).
5. If `bundle.macOS.providerShortName` and a notarization key are configured, Tauri can also submit for notarization. Usually done in CI, not locally.

### Tauri's `Info.plist` generation

Tauri generates `Info.plist` from `tauri.conf.json` fields:

| `tauri.conf.json` | `Info.plist` |
|---|---|
| `identifier` | `CFBundleIdentifier` |
| `productName` | `CFBundleName` (+ bundle dir name) |
| `version` | `CFBundleShortVersionString` + `CFBundleVersion` |
| `bundle.macOS.minimumSystemVersion` | `LSMinimumSystemVersion` |
| Custom: `bundle.macOS.infoPlist` (object) | merged in verbatim |

For TCC usage strings, drop them into `tauri.conf.json`:

```json
"bundle": {
  "macOS": {
    "entitlements": "entitlements.plist",
    "signingIdentity": "Uri Gil Code Sign",
    "hardenedRuntime": true,
    "infoPlist": {
      "NSScreenCaptureUsageDescription": "screenpipe records your screen to give you 24/7 memory.",
      "NSMicrophoneUsageDescription": "screenpipe transcribes ambient audio.",
      "NSCameraUsageDescription": "optional face presence detection.",
      "NSAccessibilityUsageDescription": "UI event recording for timeline."
    }
  }
}
```

### Tauri's entitlements file

Pointed to by `bundle.macOS.entitlements` in `tauri.conf.json`. Screenpipe's is at `src-tauri/entitlements.plist`. Edit as a regular plist.

### Electron's equivalent

Similar but through `electron-builder` or `@electron/osx-sign`. The ideas transfer: Info.plist, entitlements, signing identity, hardened runtime, notarization.

---

## 13. Diagnostic cookbook — commands that earn their keep

One command per question. Keep this section nearby.

### "Is my app signed, and by whom?"

```bash
codesign -dvv /Applications/screenpipe.app 2>&1 | head -15
```

Look at: `Identifier`, `Authority`, `TeamIdentifier`, `Signature size`.

### "What does the signature's designated requirement say?"

```bash
codesign -d --requirements - /Applications/screenpipe.app 2>&1
```

### "What entitlements does the app claim?"

```bash
codesign -d --entitlements :- /Applications/screenpipe.app 2>/dev/null
```

### "Does the signature actually verify?"

```bash
codesign --verify --deep --strict --verbose=2 /Applications/screenpipe.app
```

Output `valid on disk / satisfies its Designated Requirement` = good.

### "Does Gatekeeper accept this app?"

```bash
spctl --assess --verbose /Applications/screenpipe.app
```

### "What's inside `Info.plist`?"

```bash
plutil -p /Applications/screenpipe.app/Contents/Info.plist
# or extract one key:
defaults read /Applications/screenpipe.app/Contents/Info CFBundleIdentifier
```

### "Do I have any signing certs?"

```bash
security find-identity -v -p codesigning
```

### "What TCC grants does my bundle ID have?"

```bash
sqlite3 "$HOME/Library/Application Support/com.apple.TCC/TCC.db" \
  "SELECT service, auth_value, auth_reason, datetime(last_modified,'unixepoch')
     FROM access
    WHERE client = 'screenpi.pe.dev';"
```

(Screen Recording, Accessibility, and Full Disk Access live in the **system** DB at `/Library/Application Support/com.apple.TCC/TCC.db` — readable by root only.)

### "What's in the Gatekeeper quarantine bit?"

```bash
xattr -l /Applications/screenpipe.app
```

### "What apps does LaunchServices know about with a given bundle ID?"

```bash
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -dump \
  | grep -A3 "screenpi.pe.dev"
```

### "Is a process running unsigned memory or failing hardened runtime?"

```bash
log stream --predicate 'subsystem == "com.apple.runningboard"' --style compact
# or:
log stream --predicate 'process == "tccd"'
# or:
log show --predicate 'eventMessage contains "screenpipe"' --last 10m
```

The `log` command is the modern replacement for `Console.app` on the CLI. Invaluable for TCC debugging.

### "Reset TCC for my app"

```bash
tccutil reset All screenpi.pe.dev
# Followed by: launch the app, re-grant permissions.
```

### "Strip quarantine from a locally-downloaded app"

```bash
xattr -rd com.apple.quarantine /path/to/SomeApp.app
```

### "Re-sign an adhoc-built app with my cert"

```bash
codesign --force --deep --sign "Uri Gil Code Sign" \
  --options=runtime \
  --entitlements /path/to/entitlements.plist \
  /path/to/YourApp.app
```

### "Is there a stapled notarization ticket?"

```bash
xcrun stapler validate /Applications/screenpipe.app
```

---

## 14. Applied to screenpipe's current state

Putting every concept above into one concrete scenario — our situation right now:

- **Old installed app** `/Applications/screenpipe.app`:
  - `CFBundleIdentifier = screenpi.pe.dev`
  - Signed by `Uri Gil Code Sign`
  - DR: `anchor trusted and identifier "screenpi.pe.dev" and certificate leaf[...] matches "Uri Gil Code Sign"`
  - TCC `csreq` on file: serialized form of the above DR
  - Screen recording permission: **granted** (this is what you clicked "allow" on originally)

- **New built app** `apps/.../bundle/macos/screenpipe - Development.app`:
  - `CFBundleIdentifier = screenpi.pe.dev` (same Info.plist, same identifier)
  - Signed adhoc (no cert) → DR is a minimal form that does NOT match the stored csreq
  - TCC would treat it as a different app → fresh prompt

- **Fix:** run `codesign --force --deep --sign "Uri Gil Code Sign" "screenpipe - Development.app"` before installing. The new binary now has a matching DR. TCC grant applies.

- **Long-term fix:** add `"signingIdentity": "Uri Gil Code Sign"` to `tauri.conf.json → bundle.macOS`, so every future build is signed correctly without a post-build step.

Why we rename-and-replace `/Applications/screenpipe.app` instead of installing side-by-side: **LaunchServices** (§9) caches bundle-ID → path, and two apps sharing bundle ID `screenpi.pe.dev` makes `open -b screenpi.pe.dev`, URL schemes (`screenpipe://...`), and TCC's self-identification nondeterministic.

---

## 15. Further reading

Primary sources (authoritative):

- [Code Signing Guide](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/Introduction/Introduction.html) — Apple's canonical doc. Old but still correct on the fundamentals.
- [TN2206 — macOS Code Signing in Depth](https://developer.apple.com/library/archive/technotes/tn2206/_index.html) — the bible. Updated over the years.
- [Hardened Runtime docs](https://developer.apple.com/documentation/security/hardened_runtime) — entitlement reference.
- [Notarizing macOS Software Before Distribution](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution) — notary API.
- `man codesign` — surprisingly readable.
- `man tccutil` — short but complete.
- `man spctl` — the Gatekeeper CLI.

Third-party but excellent:

- **Objective Development's "A Code Signing Survival Guide"** — practical tips.
- **Howard Oakley's blog (Eclectic Light Company)** — deep dives on Gatekeeper and TCC mechanics. Search "howard oakley tcc" or "howard oakley gatekeeper."
- **Thomas Reed / Malwarebytes** — TCC internals from a security researcher's angle.
- **Quinn "The Eskimo!" on Apple DevForums** — authoritative answers on edge cases. Search any of your confusing error messages there.

Tauri-specific:

- [Tauri macOS bundling docs](https://v2.tauri.app/distribute/sign/macos/) — signing and notarization config reference.
- [`tauri.conf.json` schema](https://schema.tauri.app/config/2) — every option explained.

---

## Appendix: vocabulary cheat sheet

| Term | One-sentence definition |
|---|---|
| **App bundle** | A `.app` directory with `Contents/MacOS/<exe>`, `Info.plist`, and resources. |
| **Mach-O** | The macOS executable format (equivalent of Linux's ELF). |
| **`Info.plist`** | The bundle's metadata file. Identity, capabilities, versions, usage strings. |
| **Code signing** | Embedding a cryptographic signature covering all of the bundle's contents, so tampering is detectable. |
| **Codesign identifier** | Short string embedded in the signature (usually same as `CFBundleIdentifier`). |
| **Designated Requirement (DR)** | A logical predicate in a signature that defines "what counts as the same identity." |
| **Entitlements** | A plist embedded in a signature declaring what capabilities the binary claims. |
| **Hardened runtime** | A stricter set of runtime checks. Required for notarization. |
| **Notarization** | Apple's automated malware scan that issues a ticket attesting a build is fine. |
| **Stapling** | Attaching the notarization ticket to the app so Gatekeeper can verify offline. |
| **Gatekeeper** | The launch-time policy enforcer for downloaded/quarantined apps. |
| **Quarantine bit** | The `com.apple.quarantine` xattr that tags downloaded files for Gatekeeper scrutiny. |
| **LaunchServices** | The OS-level router for "open a file / url / bundle id." |
| **TCC** | Transparency, Consent, Control — the "Allow app X to use Y?" subsystem. |
| **TCC service** | A category of protected resource, e.g. `kTCCServiceScreenCapture`. |
| **csreq** | A serialized Designated Requirement stored in the TCC DB to bind grants to a specific signing identity. |
| **`tccutil`** | CLI to reset TCC grants. |
| **`codesign`** | CLI to sign, verify, inspect code signatures. |
| **`spctl`** | CLI to query/configure Gatekeeper. |
| **`lsregister`** | Internal tool to rebuild LaunchServices cache. |
| **`plutil`** | Convert/validate plist files. |

If a term appears in this manual and isn't in the table, it's probably a bug report I should fix. Ping me.
