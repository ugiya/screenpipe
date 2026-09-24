---
name: build-from-source
description: Build, sign, and install screenpipe from source on Uri's Mac, preserving macOS TCC permissions across the upgrade. USE WHEN update screenpipe, rebuild the app, install a new local build, sync fork with upstream, permissions lost after rebuild, ort-sys 403, cdn.pyke.io blocked.
tools:
  - Bash
  - Read
  - Edit
---

# Build screenpipe from source (personal macOS build)

Local-only procedure for Uri's fork. Produces a build signed with his own certificate so
macOS keeps the existing Screen Recording / Microphone / Accessibility grants instead of
re-prompting. Not an upstream workflow — do not include this in a PR to `screenpipe/screenpipe`.

Licensing: the source is available for personal, non-commercial use, so building and running
your own copy is permitted (`README.md`, `LICENSE.md`). Only the prebuilt app they distribute
is behind a subscription.

## Why signing identity is the whole game

macOS TCC binds a permission grant to a **Designated Requirement** — `(bundle id, signing
certificate)` — not to a file path or version. Sign the new build with the same cert as the
installed one and every grant survives. Sign adhoc (Tauri's default) or with a different cert
and macOS treats it as a brand-new app: fresh prompts, and screen recording silently stops.

Background reading: `docs/macos-signing-and-tcc.md` (§4 DR, §10 TCC, §14 applied case).

## Preflight

```bash
security find-identity -v -p codesigning          # expect: "Uri Gil Code Sign"
codesign -d --requirements - /Applications/screenpipe.app 2>&1 | tail -1   # record this DR
defaults read /Applications/screenpipe.app/Contents/Info CFBundleShortVersionString
df -h ~ | tail -1                                  # need ~15 GB free for target/
```

Save that DR string. The new bundle's DR must match it byte for byte before you install.

## 1. Sync the fork

`origin` = `screenpipe/screenpipe` (upstream), `fork` = `ugiya/screenpipe`.

```bash
git fetch origin
git checkout main && git merge --ff-only origin/main
git checkout -b update/vX.Y.Z-uri main
```

Re-apply personal patches on the new branch by hand — do not cherry-pick blindly. Upstream
moves files constantly (the chat 5xx-retry tweak migrated from `components/standalone-chat.tsx`
to `components/chat/standalone/hooks/use-pi-foreground-events.ts`). Find the new home with
`grep -rn "<nearby unique string>"`, not by path.

Every source file needs the header from `CLAUDE.md` — check before committing.

## 2. Seed the ONNX Runtime cache (required on this network)

`ort-sys` downloads a prebuilt ONNX Runtime from `cdn.pyke.io`, and that CDN returns **HTTP 403
to this network** (Cloudflare block — reproduced from curl, the home-lab VM, and real Chrome).
The build fails with `ort-sys failed to download prebuilt binaries ... http status: 403`.

Check whether the cache already has it before doing anything:

```bash
ORTSYS=$(ls -d ~/.cargo/registry/src/*/ort-sys-* | tail -1)
grep "^none	aarch64-apple-darwin" "$ORTSYS/build/download/dist.txt"
# → none  aarch64-apple-darwin  <url>  <sha256>
ls ~/Library/Caches/ort.pyke.io/dfbin/aarch64-apple-darwin/<sha256>/libonnxruntime.a
```

If that file exists, skip this section — `ort-sys` skips the download whenever the
content-addressed directory is present.

Otherwise fetch it through a US runner on the fork:

1. On a temp branch, add a one-job workflow that `curl`s the URL, checks it with
   `sha256sum -c`, and uploads it via `actions/upload-artifact`. Trigger on that branch only.
   Verify no other workflow fires on an arbitrary branch push (they are filtered by
   branch/path/tag — confirm with a quick scan of `.github/workflows/`).
2. `gh run list --repo ugiya/screenpipe --branch <temp>` then
   `gh run download <id> --repo ugiya/screenpipe -n <artifact> -D <dir>`
3. Verify locally: `shasum -a 256 <file>` must equal the `dist.txt` hash. Do not proceed otherwise.
4. The payload is a **raw LZMA2 stream**, not `.xz` — `tar` and `xz` will not open it:

```bash
python3 -c "
import lzma, tarfile, io, os
raw = lzma.LZMADecompressor(format=lzma.FORMAT_RAW,
        filters=[{'id': lzma.FILTER_LZMA2, 'dict_size': 1<<26}]
      ).decompress(open('<file>.tar.lzma2','rb').read())
dest = os.path.expanduser('~/Library/Caches/ort.pyke.io/dfbin/aarch64-apple-darwin/<sha256>')
os.makedirs(dest, exist_ok=True)
tarfile.open(fileobj=io.BytesIO(raw)).extractall(dest)
"
```

The archive holds a single flat `libonnxruntime.a`; `ort-sys` links it with
`rustc-link-search=native=<that dir>`, so the flat layout matters.

5. Delete the temp branch locally and on the fork.

## 3. Build

```bash
cd apps/screenpipe-app-tauri
bun install
bun tauri build --no-sign          # NOT `bun run tauri:build`
```

`scripts/build_macos.sh` (what `tauri:build` runs) hardcodes an upstream developer's signing
identity that is not in this keychain. Always build unsigned and sign in step 4.

Cold build is roughly 25-50 minutes. Run it in the background and poll. Output:
`src-tauri/target/release/bundle/macos/screenpipe - Development.app`

**Check the bundle identity before signing.** Upstream changed the base `tauri.conf.json` from
`screenpi.pe.dev` / `screenpipe - Development` to `screenpi.pe` / `screenpipe` in `acd070441`
(2026-09-16). The fork carries a two-line override back to the dev identity. After every sync:

```bash
grep -n '"identifier"\|productName' src-tauri/tauri.conf.json   # want screenpi.pe.dev
APP="src-tauri/target/release/bundle/macos/screenpipe - Development.app"
plutil -extract CFBundleIdentifier raw "$APP/Contents/Info.plist"   # after the build, same
```

If the bundle comes out as `screenpipe.app` / `screenpi.pe`, the DR gate in step 4 fails and every
TCC grant would be lost. Some behavior does key off the identifier; an audit on 2026-09-24 found
the URL scheme, keychain service (`com.screenpipe.app`), data dir, updater, single-instance and
autostart all compatible with the dev identity. The dev identity also turns off remote
announcements (`lib/announcements.ts` `isDevOrE2EAppIdentity`). Upstream's
`scripts/build_macos.sh` hardcodes `screenpipe.app` and breaks with this name — another reason
not to use it.

A `cargo:warning` about `APPLE_SIGNING_IDENTITY missing` for `mlx.metallib` is expected — step 4
signs it.

## 4. Sign with the local certificate

Order matters: nested code first, bundle last. `--timestamp=none` because a local cert has no
Apple timestamp authority.

```bash
APP="src-tauri/target/release/bundle/macos/screenpipe - Development.app"
IDENT="Uri Gil Code Sign"
xattr -cr "$APP"

# 1. Mach-O sidecars
for b in bun ffmpeg ffprobe libonnxruntime.dylib; do
  codesign --force --timestamp=none --options=runtime --sign "$IDENT" "$APP/Contents/MacOS/$b"
done

# 2. mlx.metallib — lives in Contents/MacOS/, so codesign demands a signature on it
codesign --force --timestamp=none --options=runtime --entitlements src-tauri/entitlements.plist \
  --sign "$IDENT" "$APP/Contents/MacOS/mlx.metallib"

# 3. The bundle
codesign --force --timestamp=none --options=runtime --entitlements src-tauri/entitlements.plist \
  --sign "$IDENT" "$APP"

codesign --verify --strict --verbose=2 "$APP"      # want: satisfies its Designated Requirement
```

Skipping `mlx.metallib` makes the bundle signature fail with `code object is not signed at all`.

### Gate: DR must match before installing

```bash
codesign -d --requirements - "$APP" 2>&1 | tail -1
codesign -d --requirements - /Applications/screenpipe.app 2>&1 | tail -1
```

Identical strings → permissions survive. Different → stop and fix the signing identity;
installing anyway costs every TCC grant.

The entitlements file adds calendar and Apple Events capabilities that the old build lacked.
That is safe: entitlements and the hardened-runtime flag are not part of the DR.

## 5. Back up before first launch

A large version jump runs one-way DB migrations. The data directory is ~35 GB, but the
databases clone instantly on APFS with `cp -c` (copy-on-write, no disk consumed until they
diverge):

```bash
osascript -e 'tell application "screenpipe" to quit'
pgrep -lf "/Applications/screenpipe.app"           # must be empty before copying

BK=~/.screenpipe-backup-$(date +%Y%m%d)
mkdir -p "$BK" && cd ~/.screenpipe
for f in db.sqlite db.sqlite-wal db.sqlite-shm vector_db.sqlite secrets.sqlite store.bin; do
  [ -e "$f" ] && cp -c "$f" "$BK/$f"
done
```

Do not back up `data/` (the video files, ~11 GB) — migrations do not rewrite it.

## 6. Install (rename-and-replace)

Never install side-by-side. Two bundles sharing `screenpi.pe.dev` make LaunchServices
nondeterministic about `open -b`, the `screenpipe://` URL scheme, and the app's own
self-identification (`docs/macos-signing-and-tcc.md` §9).

```bash
mv /Applications/screenpipe.app /Applications/screenpipe-<oldver>.bak.app
ditto "$APP" /Applications/screenpipe.app        # ditto preserves the signature; cp does not
defaults read /Applications/screenpipe.app/Contents/Info CFBundleShortVersionString
codesign --verify --strict /Applications/screenpipe.app
```

## 7. Verify

```bash
open -a /Applications/screenpipe.app
sleep 25
pgrep -lf "/Applications/screenpipe.app/Contents/MacOS/screenpipe-app"
tail -40 "$(ls -t ~/.screenpipe/screenpipe-app*.log | head -1)"
```

Healthy: `App version: X.Y.Z`, server listening, no TCC prompt on screen.

Known non-failure: `Skipping server auto-start: screenpipe account access required` means the
upstream account gate (`src-tauri/src/recording.rs`) is paused pending sign-in — not a build or
signing problem.

Then sleep/wake the Mac and confirm vision capture resumes (the historical regression this fork
originally patched).

## Rollback

```bash
osascript -e 'tell application "screenpipe" to quit'
rm -rf /Applications/screenpipe.app
mv /Applications/screenpipe-<oldver>.bak.app /Applications/screenpipe.app
cp -c ~/.screenpipe-backup-<date>/db.sqlite ~/.screenpipe/db.sqlite   # only if migrations broke it
```

Keep the old bundle and the DB backup through a few days of soak before deleting.

## Failure modes seen in practice

| Symptom | Cause | Fix |
|---|---|---|
| `ort-sys ... http status: 403` | `cdn.pyke.io` blocks this network | Step 2 |
| `code object is not signed at all ... mlx.metallib` | metallib in `Contents/MacOS/` unsigned | Sign it before the bundle |
| Fresh "Allow Screen Recording" prompt | DR changed (adhoc or wrong cert) | Re-sign with `Uri Gil Code Sign`, reinstall |
| `open -b screenpi.pe.dev` hits the wrong copy | two bundles, same id | Remove the extra; `lsregister -kill -r -domain local -domain user` |
| Recording never starts, log says account required | upstream entitlement gate | Sign in, or patch `recording_access_policy` |
| Bundle is `screenpipe.app`, id `screenpi.pe` | upstream base identity changed (2026-09-16) | Keep the fork's `tauri.conf.json` override to `screenpi.pe.dev` |
| Native tests panic in build.rs: `resource path bun-aarch64-apple-darwin doesn't exist` | sidecars not provisioned in a fresh worktree | `bun scripts/gen-skill-content.js && bun scripts/pre_build.js`, then `bun run test:tauri <filter>` |
| Upstream `main` tip fails to compile | mid-flight upstream change | Build the newest `app-v*` release tag instead of the `main` tip |
