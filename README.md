# homebrew-freetube

## A Homebrew tap that should not need to exist.

As of 2026-09-10, the [official FreeTube cask](https://formulae.brew.sh/cask/freetube) is **disabled**:

```ruby
disable! date: "2026-09-01", because: :fails_gatekeeper_check
```

`brew info --cask freetube` reports:

```
==> freetube (FreeTube): 0.25.3
Disabled because it does not pass the macOS Gatekeeper check! It was disabled on 2026-09-01.
```

A disabled cask cannot be installed at all — not with `--no-quarantine`, not with `--force` — and the unqualified name does not fall back to this tap either:

```
$ brew install --cask freetube
Warning: Not upgrading freetube, it is disabled because it does not pass the macOS Gatekeeper check! It was disabled on 2026-09-01.
```

That command exits 0 and changes nothing on disk. Users are left with a manual `.dmg` download and a trip through System Settings, or nothing at all.

**This tap provides the missing cask.** It installs the same upstream artifact the official cask pointed at, pinned by SHA256, with the quarantine attribute cleared after installation so the app launches.

---

## Install

```bash
brew tap kernelzeroday/freetube
brew install --cask kernelzeroday/freetube/freetube
```

Use the fully qualified token. The unqualified name `freetube` still exists in `homebrew-cask` — as a disabled cask — so the qualified form is the only unambiguous way to ask for this one.

If you have `HOMEBREW_REQUIRE_TAP_TRUST` set (and you should):

```bash
brew tap kernelzeroday/freetube
brew trust --tap kernelzeroday/freetube
brew install --cask kernelzeroday/freetube/freetube
```

## Update

```bash
brew update
brew upgrade --cask kernelzeroday/freetube/freetube
```

## Uninstall

```bash
brew uninstall --cask kernelzeroday/freetube/freetube   # keeps your subscriptions and settings
brew uninstall --cask --zap kernelzeroday/freetube/freetube   # removes them too
```

---

## Why Gatekeeper rejects it

FreeTube's macOS builds are **ad-hoc signed**: the binary carries a code signature, but not one issued by an Apple Developer ID, and it has never been through Apple's notarization service. From the 0.25.3 arm64 build:

```
$ codesign -dv --verbose=4 FreeTube.app
Identifier=io.freetubeapp.freetube
Format=app bundle with Mach-O thin (arm64)
Signature=adhoc
TeamIdentifier=not set
```

An ad-hoc signature is enough for macOS to verify that the app is internally consistent — `codesign --verify --deep --strict` passes and the app satisfies its designated requirement — but it is not enough for Gatekeeper. With the download quarantine bit set, which is what Homebrew (and every browser) applies to downloaded files, launch is blocked:

```
$ spctl --assess --type execute -vvv /Applications/FreeTube.app
/Applications/FreeTube.app: rejected
```

Gatekeeper only evaluates apps that carry the quarantine attribute. Clear the attribute on a hash-verified artifact and the app runs normally; leave it and macOS 15+ shows "FreeTube is damaged and can't be opened" with no right-click-to-open bypass. This tap clears it with a declarative `postflight_steps` install step, immediately after Homebrew has verified the SHA256 of the downloaded disk image.

This is not a new idea, and it is not specific to FreeTube: it is the same mechanism every unofficial macOS distribution channel uses. The difference here is that the bytes are pinned to an exact upstream release, so `xattr -dr com.apple.quarantine` is being applied to an artifact whose hash you can check yourself.

## What this tap does and does not do

Does:

- Downloads `freetube-#{version}-beta-mac-#{arch}.dmg` from `github.com/FreeTubeApp/FreeTube` releases
- Verifies the SHA256 against the value in `Casks/freetube.rb` before installing anything
- Installs `FreeTube.app` to `/Applications`
- Removes `com.apple.quarantine` from the installed bundle

Does not:

- Modify the app, its signature, or its entitlements
- Re-sign anything
- Patch or redirect the app's network traffic
- Do anything at runtime

If you would rather verify by hand, download the DMG yourself and compare:

```bash
shasum -a 256 freetube-0.25.3-beta-mac-arm64.dmg
# 2b445d64f5e56a873debea50c785cd41400bdabc387f0d67fc7be745b2b9146e   (arm64)
# 40fb6c671ec75905e035968ec0c14bfe717643730af80536e625d191455f49bf   (x64)
```

Both hashes match the values in the disabled `homebrew-cask` definition of FreeTube 0.25.3.

## If you would rather strip the attribute yourself

The cask's only non-standard step is one `xattr` call. You can install the official DMG manually and run it yourself:

```bash
xattr -dr com.apple.quarantine /Applications/FreeTube.app
```

That is the entire workaround. This tap exists because doing it by hand — and remembering to redo it after every update — is not a package manager.

## Other taps that cover this

[`SoftwareRat/unsigned-tap`](https://github.com/SoftwareRat/homebrew-unsigned-tap) hosts a broad set of casks that Homebrew disabled for `fails_gatekeeper_check`, and it is a reasonable choice if you want many of them. Its FreeTube entry, checked 2026-09-10, is pinned to **0.23.15** (last modified 2026-04-27), while upstream is at 0.25.3 — it has missed several releases. This tap tracks one app and stays current.

## Requirements

- macOS 12 Monterey or newer, matching the upstream 0.25.x build requirement
- Apple Silicon or Intel — both architectures are pinned in the cask

## Keeping this tap current

Upstream ships releases under tags like `v0.25.3-beta` (every release is marked pre-release on GitHub, including stable builds), so the version and both hashes have to be updated together:

```bash
V=0.25.4
for a in arm64 x64; do
  curl -sSLO "https://github.com/FreeTubeApp/FreeTube/releases/download/v${V}-beta/freetube-${V}-beta-mac-${a}.dmg"
done
shasum -a 256 freetube-${V}-beta-mac-*.dmg
```

Then update `version` and the two `sha256` values in `Casks/freetube.rb`, and check the result:

```bash
brew livecheck --cask kernelzeroday/freetube/freetube
brew audit --cask --strict kernelzeroday/freetube/freetube
(cd "$(brew --repository kernelzeroday/freetube)" && brew style Casks/freetube.rb)
```

Run `brew style` from inside the tap, as above. Pointed at a loose file outside a tap, it applies the wrong cop set and reports a clean bill of health it has not actually earned.

## Troubleshooting

**FreeTube still won't open after installing.** Check for a re-applied quarantine bit and clear it:

```bash
xattr -l /Applications/FreeTube.app
xattr -dr com.apple.quarantine /Applications/FreeTube.app
```

**Coming from the official cask.** `brew uninstall --cask freetube` cannot clean up the old install: the unqualified name resolves to the cask Homebrew disabled, and that definition can no longer be loaded. Upgrade straight through instead — this replaces the old install in one step and leaves `~/Library/Application Support/FreeTube` (subscriptions, settings, playlists) untouched:

```bash
brew upgrade --cask kernelzeroday/freetube/freetube
```

To start clean instead, delete the bundle and let Homebrew forget the old install:

```bash
rm -rf /Applications/FreeTube.app "$(brew --prefix)/Caskroom/freetube"
brew install --cask kernelzeroday/freetube/freetube
```

**`Warning: Not upgrading freetube, it is disabled...`.** The unqualified name resolves to `homebrew-cask`'s disabled copy, not this tap's. Use the fully qualified token `kernelzeroday/freetube/freetube` for install, upgrade, and uninstall.

---

## A note to the FreeTube maintainers

FreeTube is a well-maintained, 21k-star project with native arm64 and x64 macOS builds; the only thing standing between it and a normal install experience is an Apple Developer Program membership ($99/year) and the notarization step it unlocks. The request was filed as [#9161](https://github.com/FreeTubeApp/FreeTube/issues/9161) and closed as not planned within the hour as a duplicate.

Notarization is a paid gate, not a quality bar, and it is reasonable for a volunteer project to decline to pay it. But the cost is borne by every macOS user, and by Homebrew users twice over: the moment the app fails the Gatekeeper check, Homebrew drops the cask. If the project ever does sign and notarize, this tap becomes unnecessary and will be deleted — that is the best possible outcome.

## A note to Homebrew maintainers

Disabling a cask for `fails_gatekeeper_check` removes it from the one channel most macOS users have. The documented escape hatch, `--no-quarantine`, does not apply to a disabled cask, so the disable is not a warning — it is an outage. A short "how to install this anyway" section on the disabled cask's page, or a quarantine-free option that survives, would go a long way for the ~3,000 people a year installing FreeTube through Homebrew.

---

## License

This tap is MIT licensed. FreeTube itself is [AGPL-3.0](https://github.com/FreeTubeApp/FreeTube/blob/master/LICENSE.md); no FreeTube code is vendored here — the cask only points at upstream release artifacts.

If FreeTube or the Homebrew maintainers want to adopt, adapt, or supersede this cask — please do.
