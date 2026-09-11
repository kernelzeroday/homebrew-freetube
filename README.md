# homebrew-freetube

> A Homebrew tap for [FreeTube](https://freetubeapp.io/) that installs the upstream build Homebrew disabled for failing the macOS Gatekeeper check.

```bash
brew tap kernelzeroday/freetube
brew install --cask kernelzeroday/freetube/freetube
```

---

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

A disabled cask cannot be installed at all — `--force` does not override it, and the unqualified name does not fall back to this tap either:

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

Use the fully qualified token. The unqualified name `freetube` still exists in `homebrew-cask` — as a disabled cask — and that copy wins name resolution, so the qualified form is the only way to ask for this one. Details in [Problem 7](#problem-7-the-unqualified-token-resolves-to-the-dead-cask).

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

The upgrade is a normal cask upgrade: Homebrew downloads the new DMG, verifies the new SHA256, replaces `/Applications/FreeTube.app`, and this tap's install step clears the quarantine bit again. Your library, subscriptions, and settings live in `~/Library/Application Support/FreeTube` and are untouched.

## Uninstall

```bash
brew uninstall --cask kernelzeroday/freetube/freetube          # keeps your subscriptions and settings
brew uninstall --cask --zap kernelzeroday/freetube/freetube    # removes them too
```

## Requirements

- macOS 12 Monterey or newer. Upstream ships 0.25.x for Monterey and later (its own cask puts those versions under `on_monterey :or_newer`), so this tap does too.
- Apple Silicon or Intel. Both architectures are pinned in the cask; the arm64 build is a thin arm64 binary, so there is no Rosetta involved.

---

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
- Add a helper, agent, or launch daemon

### Verify the artifact yourself

Download the DMG and compare hashes:

```bash
shasum -a 256 freetube-0.25.3-beta-mac-arm64.dmg
# 2b445d64f5e56a873debea50c785cd41400bdabc387f0d67fc7be745b2b9146e   (arm64)
# 40fb6c671ec75905e035968ec0c14bfe717643730af80536e625d191455f49bf   (x64)
```

Both values match the disabled `homebrew-cask` definition of FreeTube 0.25.3 — Homebrew's own maintainers checked these bytes when the cask was still installable.

---

## How this works

### Problem 1: A disabled cask is uninstallable, not just un-installable

`disable!` is not a warning. The cask cannot be installed at all, so no install-time option can rescue it. Worse, the failure is quiet: `brew install --cask freetube` prints a warning and exits **0**, so scripts and users see success and no app.

The blast radius extends past installing. The installed FreeTube 0.25.2 on the machine this tap was built on had an `INSTALL_RECEIPT.json` pointing at `homebrew/cask`, which meant `brew uninstall --cask kernelzeroday/freetube/freetube` failed with:

```
Error: Cask 'kernelzeroday/freetube/freetube' is unavailable.
```

and `brew untap` refused to remove a half-used tap because it "contains installed casks". Disabling a cask leaves the people who already installed it with no clean path in either direction. See [Problem 8](#problem-8-upgrading-out-of-the-official-cask) for what does work.

### Problem 2: FreeTube's macOS builds are ad-hoc signed

From the 0.25.3 arm64 build:

```
$ codesign -dv --verbose=4 FreeTube.app
Identifier=io.freetubeapp.freetube
Format=app bundle with Mach-O thin (arm64)
Signature=adhoc
TeamIdentifier=not set
```

"Ad-hoc" means the app carries a code signature that macOS can verify against itself, but that was not issued by an Apple Developer ID and has never been through Apple's notarization service. The signature is structurally valid:

```
$ codesign --verify --deep --strict --verbose=2 FreeTube.app
FreeTube.app: valid on disk
FreeTube.app: satisfies its Designated Requirement
```

That is not enough for the security assessment Gatekeeper performs:

```
$ spctl --assess --type execute -vvv /Applications/FreeTube.app
/Applications/FreeTube.app: rejected
```

`spctl` rejects the app whether or not it is quarantined. What quarantining changes is *enforcement* — see the next problem.

The x64 build is the same story: `Mach-O thin (x86_64)`, `Signature=adhoc`, `TeamIdentifier=not set`. FreeTube is not doing anything wrong here; it simply has not paid for an Apple Developer Program membership ($99/year) and run the notarization step that membership unlocks.

The request to fix this is [FreeTubeApp/FreeTube#9161](https://github.com/FreeTubeApp/FreeTube/issues/9161), filed 2026-05-20, closed as not planned 27 minutes later as a duplicate. The project has not signed the app as of 0.25.3.

### Problem 3: Quarantine, not the signature, is what blocks launch

Gatekeeper only evaluates a bundle that carries `com.apple.quarantine`. Downloads get it attached; that is the entire mechanism. A quarantined, unnotarized app is what produces *"FreeTube is damaged and can't be opened. You should move it to the Trash."* — and since macOS 15 Sequoia, the Control-click → Open bypass is gone, so the only recovery is System Settings → Privacy & Security → Allow Anyway.

On the affected machine, the blocking state was visible directly:

```
$ xattr -l /Applications/FreeTube.app
com.apple.provenance:
com.apple.quarantine: 0381;6aa34c65;Waterfox;2246B794-…
```

After this tap installs the same app, the only extended attribute left is `com.apple.provenance`, and the app launches:

```
$ xattr -l /Applications/FreeTube.app
com.apple.provenance:

$ open -a FreeTube && pgrep -fl "FreeTube.app/Contents/MacOS/FreeTube"
27948 /Applications/FreeTube.app/Contents/MacOS/FreeTube
```

So the whole fix is one hash-verified `xattr -dr com.apple.quarantine`. The rest of this tap exists to make that automatic, repeatable, and tied to a pinned artifact instead of to a user's memory.

### Problem 4: Homebrew 6 has no way at all to opt out of quarantine

Homebrew does not merely inherit the quarantine bit from the download — it applies and propagates it deliberately (`cask/download.rb` and `cask/quarantine.rb` in the Homebrew source):

- `Quarantine.cask!` stamps a LaunchServices *web download* quarantine onto the downloaded artifact, with its own agent name and the cask's URL and homepage as provenance
- `Quarantine.propagate` carries that attribute from the mounted container onto the app that gets installed
- `Quarantine.inherit_user_approval!` runs on upgrade and re-quarantines the new bundle while carrying forward a *user-approved* flag for the paths the previous version had

That last one is the current model, and it is the reason there is no escape hatch to point users at: Homebrew's idea is that Gatekeeper prompts once, the user approves the app in System Settings, and the approval survives upgrades. It works — but it requires the app to be launched and approved by hand, which is the step a package manager is supposed to remove.

Homebrew has historically offered a `--no-quarantine` install option. It no longer does:

```
$ brew install --cask --dry-run --no-quarantine kernelzeroday/freetube/freetube
Usage: brew install [options] formula|cask [...]
```

It is not in `brew install --help`, it is not mentioned anywhere in the [Cask Cookbook](https://docs.brew.sh/Cask-Cookbook) or the [Manpage](https://docs.brew.sh/Manpage), and `Cask::Quarantine.release!` — the library method that removes the attribute, whose own body carries the comment *"Fully remove quarantine only when explicitly requested"* — has no call sites anywhere in Homebrew's Ruby source as of 6.0.22.

So there is no flag to document, no stanza to write, and no way for a cask to ask for this. The attribute has to be removed after the fact, which is what this cask does — after Homebrew has verified the hash, and with the app bundle otherwise untouched.

### Problem 5: `postflight` is the wrong stanza for a new cask

The obvious mechanism is the legacy `postflight do … end` Ruby block. `brew audit --cask --strict` accepts it, and Homebrew still executes it. But `brew style` inside the tap rejects it:

```
Casks/freetube.rb:29:3: C: Cask/InstallSteps: Casks must use postflight_steps instead of postflight.
```

Homebrew 6 has a declarative install-step API — `preflight_steps`, `postflight_steps`, `uninstall_preflight_steps`, `uninstall_postflight_steps` — described in the [Cask Cookbook](https://docs.brew.sh/Cask-Cookbook) as "declarative file preparation steps run after artifact installation", and stored in Homebrew's JSON API so that installing a cask does not require evaluating arbitrary Ruby. The cops enforce it: as of 2026-09-10, GitHub code search found **0** casks in `Homebrew/homebrew-cask` using legacy `postflight do` and **83** using `postflight_steps do`.

The declarative form is not limited to file shuffling. `CASK_ALLOWED_STEP_METHODS` includes `COMMAND_STEP_METHODS = [:run, :terminate_process]`, and `run` takes a command plus argv:

```ruby
postflight_steps do
  run "/usr/bin/xattr", args: ["-dr", "com.apple.quarantine", "{{appdir}}/FreeTube.app"]
end
```

`{{appdir}}` is one of Homebrew's absolute template tokens (alongside `{{staged_path}}`, `{{HOMEBREW_PREFIX}}`, and friends), expanded at run time. `run` steps execute through `SystemCommand` with no sandbox profile, which is what makes touching the app bundle's xattrs legal; Homebrew's own cask for Parallels uses exactly this shape for `xattr` (`run "xattr", args: ["-d", "com.apple.FinderInfo", "{{staged_path}}/…"]`).

`run` defaults to `must_succeed: true`, so if the step ever fails the install fails loudly instead of leaving a quarantined app behind.

### Problem 6: Cask cops that only run inside a tap

`brew style` run against a loose `.rb` file outside a tap applies the wrong cop set and reports a clean bill of health it has not earned. This actually happened while building this tap: `brew style homebrew-freetube/Casks/freetube.rb` from the parent directory returned "no offenses detected" for a file that, checked from inside the tap, had two. Always lint from inside the tap:

```bash
cd "$(brew --repository kernelzeroday/freetube)"
brew style Casks/freetube.rb
```

Two cask cops bit here, and both are undocumented outside the source:

- **`Cask/StanzaGrouping`.** `[:version, :sha256]` is one stanza group and `[:url, :appcast, :name, :desc, :homepage]` is another, so a blank line must separate them — but nothing else in the header may be separated. A comment inside the header counts as a break in the group and needs its own blank line before it.
- **`Homebrew/OSDependsOn`.** `depends_on macos: ">= :monterey"` is flagged; the cop wants the symbol form `depends_on macos: :monterey`, which Homebrew reads as "Monterey or newer".

### Problem 7: The unqualified token resolves to the dead cask

Tapping this repo does **not** make `brew install --cask freetube` work. With both taps present:

```
$ brew info --cask freetube
==> freetube (FreeTube): 0.25.3
Disabled because it does not pass the macOS Gatekeeper check! It was disabled on 2026-09-01.
Installed (on request)
/opt/homebrew/Caskroom/freetube/0.25.3 (289.2MB)
```

Note what that output is: the disabled `homebrew-cask` definition, with *your* install listed underneath it. `homebrew-cask` wins resolution of the bare token, and the disabled cask refuses to do anything with it. The qualified token is not a stylistic preference, it is the only working form:

```bash
brew install  --cask kernelzeroday/freetube/freetube
brew upgrade  --cask kernelzeroday/freetube/freetube
brew uninstall --cask kernelzeroday/freetube/freetube
```

One asymmetry worth knowing: `brew list --cask --versions` does not accept the qualified token and wants the bare name.

### Problem 8: Upgrading out of the official cask

For anyone already running the official cask, the obvious sequence — uninstall, then install from this tap — is the one that does not work, because the installed receipt points at the now-disabled `homebrew/cask` definition:

```
$ brew uninstall --cask kernelzeroday/freetube/freetube
Error: Cask 'kernelzeroday/freetube/freetube' is unavailable.
```

What does work is upgrading straight through, which replaces 0.25.2 with 0.25.3 in place using this tap's definition:

```
$ brew upgrade --cask kernelzeroday/freetube/freetube
==> Upgrading freetube
  0.25.2 -> 0.25.3
==> Quitting application 'io.freetubeapp.freetube'...
==> Backing up App 'FreeTube.app' to '/opt/homebrew/Caskroom/freetube/0.25.2/FreeTube.app'
==> Removing App '/Applications/FreeTube.app'
==> Moving App 'FreeTube.app' to '/Applications/FreeTube.app'
==> Purging files for version 0.25.2 of Cask freetube
🍺  freetube was successfully upgraded!
```

If you would rather start from a clean slate, remove the bundle and let Homebrew forget the old install — verified working, and it deletes nothing but the app:

```bash
rm -rf /Applications/FreeTube.app "$(brew --prefix)/Caskroom/freetube"
brew install --cask kernelzeroday/freetube/freetube
```

---

## Decisions, and the alternatives that were rejected

**Building FreeTube from source.** The [sibling Metasploit tap](https://github.com/kernelzeroday/homebrew-metasploit) builds from git HEAD because Rapid7 publishes no ARM64 installer at all. FreeTube is the opposite case: upstream already ships a thin arm64 DMG and a thin x86_64 DMG. An Electron source build would drag in Node, Yarn, and `electron-builder` for a ~300 MB app that is *still* ad-hoc signed when it finishes, solving nothing the DMG does not already solve. Building locally does dodge quarantine — locally produced files are not quarantined — but it buys that at the cost of a long, fragile build for a variant of the app nobody else runs. Not worth it.

**Re-signing the app locally.** [INFERENCE] Re-signing with a self-signed certificate would not produce a signature Gatekeeper trusts either; it would replace upstream's ad-hoc signature with a different ad-hoc signature, and it would break the guarantee this tap makes that the installed bundle is byte-for-byte what upstream published. Rejected.

**Telling users to run `xattr` themselves.** That is the workaround, and it is documented below for anyone who prefers it, but as the only option it fails twice: it has to be repeated after every update, and its failure mode is an app that silently will not open.

**Leaning on Homebrew's quarantine handling instead of doing this in the cask.** There is nothing to lean on. Homebrew 6 always quarantines cask downloads, has no install-time flag to skip it, and the built-in approval carry-forward only helps once the user has opened the app and allowed it by hand in System Settings. [Problem 4](#problem-4-homebrew-6-has-no-way-at-all-to-opt-out-of-quarantine) has the details.

**Using an existing multi-app tap.** [`SoftwareRat/unsigned-tap`](https://github.com/SoftwareRat/homebrew-unsigned-tap) is a good idea and tracks the whole upstream set of casks disabled for `fails_gatekeeper_check` — roughly 430 of them. Its FreeTube entry, checked 2026-09-10, is pinned to **0.23.15** (last touched 2026-04-27), three releases behind upstream's 0.25.3. A tap that exists but is three versions stale does not solve the problem for anyone. This tap covers one app and stays current.

**Shipping a legacy branch for older macOS.** The disabled upstream cask carried an `on_big_sur :or_older` branch pinned to 0.23.15. This tap ships one current version instead: that legacy branch cannot be tested here, upstream's current stanza is Monterey-and-newer, and 0.23.15 predates the build system that produces today's DMGs. Anyone on macOS 11 who wants that specific build should take it from the upstream releases page directly.

---

## Repository layout

```
homebrew-freetube/
├── Casks/
│   └── freetube.rb    # the cask: version, both arch hashes, url, livecheck, install step
└── README.md
```

The `homebrew-` prefix in the repository name is what lets Homebrew infer the tap name `kernelzeroday/freetube`.

## Working on this tap

`brew tap <user>/<tap> <path>` **clones** the repository, it does not link to it. Edits in your working copy are invisible to Homebrew until you commit and refresh the clone:

```bash
git -C /path/to/homebrew-freetube commit -am "…"
git -C "$(brew --repository kernelzeroday/freetube)" pull --ff-only
```

Three more things that will cost you time if you do not know them:

- `brew untap kernelzeroday/freetube` refuses while the cask is installed ("contains the following installed casks"). Uninstall first, or use `brew untap --force`.
- `brew list --cask --versions` wants the bare token `freetube`, not the qualified one.
- Testing install/uninstall locally can trip Homebrew's automatic `autoremove` sweep, which will remove formulae you have lying around unrequested and unrequired. Expect unrelated packages to disappear from `brew list` the first time you uninstall a cask.

## Keeping this tap current

FreeTube tags **every** GitHub release as a pre-release, including stable ones, so the tag and file name contain `-beta` even when the build is the release everyone should use. Two consequences:

- Do not go looking for the release through the GitHub "latest release" API — `https://api.github.com/repos/FreeTubeApp/FreeTube/releases/latest` returns **HTTP 404** for this repo, because there is no non-prerelease release. List releases and take the newest.
- The `-beta` suffix is baked into the URL, so it is easy to update `version` and forget the URL.

```bash
V=0.25.4
for a in arm64 x64; do
  curl -sSLO "https://github.com/FreeTubeApp/FreeTube/releases/download/v${V}-beta/freetube-${V}-beta-mac-${a}.dmg"
done
shasum -a 256 freetube-${V}-beta-mac-*.dmg
```

Then update `version` and both `sha256` values in `Casks/freetube.rb` and check the result:

```bash
brew livecheck --cask kernelzeroday/freetube/freetube     # should report the new version as current
brew audit --cask --strict kernelzeroday/freetube/freetube
(cd "$(brew --repository kernelzeroday/freetube)" && brew style Casks/freetube.rb)
```

`livecheck` is set up to match the DMG URL's version, so it will flag an outdated pin for you:

```
$ brew livecheck --cask kernelzeroday/freetube/freetube
freetube: 0.25.3 ==> 0.25.3
```

---

## Verification log

Everything below was run on 2026-09-10 on an Apple M2 (macOS 26, Darwin 25.6) with Homebrew 6.0.22, against FreeTube 0.25.3 installed from the published tap.

| Check | Command | Result |
|---|---|---|
| arm64 artifact hash | `shasum -a 256 freetube-0.25.3-beta-mac-arm64.dmg` | `2b445d64…14e6`, matches the disabled `homebrew-cask` value |
| x64 artifact hash | `shasum -a 256 freetube-0.25.3-beta-mac-x64.dmg` | `40fb6c67…49bf`, matches the disabled `homebrew-cask` value |
| Signature | `codesign -dv --verbose=4 FreeTube.app` | `Signature=adhoc`, `TeamIdentifier=not set` |
| Integrity | `codesign --verify --deep --strict` | `valid on disk`, `satisfies its Designated Requirement` |
| Gatekeeper | `spctl --assess --type execute -vvv` | `rejected` (before and after quarantine removal — enforcement is what changes) |
| Homebrew quarantines downloads | `xattr -l` on the cached DMG | `com.apple.quarantine: 0281;6aa34e51;;…` |
| Quarantine opt-out | `brew install --cask --no-quarantine …` | invalid option; 6.0.22 has no such flag and `Quarantine.release!` has no call sites |
| Installed state | `xattr -l /Applications/FreeTube.app` | `com.apple.provenance` only, no quarantine |
| Version | `defaults read …/Info.plist CFBundleShortVersionString` | `0.25.3` |
| Arch | `file /Applications/FreeTube.app/Contents/MacOS/FreeTube` | `Mach-O 64-bit executable arm64` |
| Launches | `open -a FreeTube`, then `pgrep -fl` | main process + 4 Electron helpers, renderer using `~/Library/Application Support/FreeTube` |
| Install step runs | plant `com.apple.quarantine`, `brew reinstall` | attribute gone afterwards |
| Lint | `brew audit --cask --strict` | clean |
| Lint | `brew style` (inside the tap) | `1 file inspected, no offenses detected` |
| Livecheck | `brew livecheck --cask` | `0.25.3 ==> 0.25.3` |
| Fresh install from GitHub tap | `brew tap kernelzeroday/freetube && brew install --cask …` | `freetube was successfully installed!` |
| Upgrade from official cask | `brew upgrade --cask kernelzeroday/freetube/freetube` | `0.25.2 -> 0.25.3`, `was successfully upgraded!` |
| Uninstall | `brew uninstall --cask kernelzeroday/freetube/freetube` | app removed, `~/Library/Application Support/FreeTube` preserved |
| Recovery | `rm -rf` app + Caskroom, reinstall | clean install, app launches |

## Troubleshooting

**FreeTube won't open after installing.** Check for a quarantine bit and clear it:

```bash
xattr -l /Applications/FreeTube.app
xattr -dr com.apple.quarantine /Applications/FreeTube.app
```

**`Warning: Not upgrading freetube, it is disabled...`.** The unqualified name resolved to `homebrew-cask`'s disabled copy, not this tap's. Use the fully qualified token for install, upgrade, and uninstall.

**`Error: Cask 'kernelzeroday/freetube/freetube' is unavailable.`** You are coming from the official cask; uninstalling through the old receipt does not work. Upgrade instead, or use the `rm -rf` recovery above.

**Coming from the official cask, in one line.** `brew upgrade --cask kernelzeroday/freetube/freetube` replaces 0.25.2 with 0.25.3 and leaves your library and settings in place.

**You would rather do this without a tap.** Install the upstream DMG and run one command:

```bash
xattr -dr com.apple.quarantine /Applications/FreeTube.app
```

That is the entire workaround; you just have to remember it after every update.

---

## A note to the FreeTube maintainers

FreeTube is a well-maintained project with over 21,000 stars and first-class macOS builds for both architectures. The only thing standing between it and a normal install experience is an Apple Developer Program membership ($99/year) and the notarization step it unlocks. The request was filed as [#9161](https://github.com/FreeTubeApp/FreeTube/issues/9161) and closed as not planned within the hour as a duplicate.

Notarization is a paid gate, not a quality bar, and it is reasonable for a volunteer project to decline to pay it. But the cost is borne by every macOS user, and by Homebrew users twice over: the moment the app fails the Gatekeeper check, Homebrew drops the cask, and a dropped cask is not a warning — it is an outage that also strands everyone who already installed it. If FreeTube ever does sign and notarize, this tap becomes unnecessary and will be deleted. That is the best possible outcome.

## A note to Homebrew maintainers

Disabling a cask for `fails_gatekeeper_check` removes an app from the one channel most macOS users have, and there is no longer an opt-out to point them at: `--no-quarantine` is gone as of 6.0.22, so quarantine now depends on the user approving each app by hand in System Settings. On top of that, `brew install` reports the refusal as a warning with exit status 0, so the failure is easy to miss, and `brew uninstall` of an existing install fails outright, because the receipt points at a definition that can no longer be loaded.

Three small things would go a long way for the ~3,000 people a year who install FreeTube through Homebrew: a "how to install this anyway" section on disabled cask pages, a non-zero exit status when a disabled cask defuses an install, and letting `brew uninstall` clean up an install whose cask has been disabled since.

---

## License

This tap is MIT licensed. FreeTube itself is [AGPL-3.0](https://github.com/FreeTubeApp/FreeTube/blob/master/LICENSE.md); no FreeTube code is vendored here — the cask only points at upstream release artifacts.

If FreeTube or the Homebrew maintainers want to adopt, adapt, or supersede this cask — please do.
