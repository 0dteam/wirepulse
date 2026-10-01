# Wire Pulse releases

This repository delivers **Wire Pulse updates**. The Windows app downloads its installer from the
[Releases](https://github.com/0dteam/wirepulse/releases) of this repository and installs it by itself; the
Android app reads the same information to tell people that a new version is in Google Play or on the download page.

**Everything here is public on purpose.** The apps do not trust this repository or GitHub: they trust only a digital
signature made with a key that never leaves the release PC. Anyone can read the files; nobody without that key can make
Wire Pulse install anything.

> You never edit the files in `manifest/` by hand. The release tool (`release.ps1` in the Wire Pulse source folder) writes
> them. The detailed technical rules are in `docs/UPDATES.md` of the Wire Pulse source, the developer steps in
> `docs/RELEASING.md`.

## What is in here

| Path | What it is |
|---|---|
| **Releases** (tags `v1.0.3`, `v1.0.4-beta.1`, …) | The files people download: `WirePulseSetup-<version>.exe` (installer), `WirePulse-<version>-portable-win-x64.zip`, `SHA256SUMS.txt`. Optional Android releases are tagged `android-v<version>`. |
| `manifest/stable.jws` | The **signed** description of the current stable version: which installer, its SHA-256 fingerprint, size, release notes, how many people get it now. The apps read only this `.jws` file. |
| `manifest/stable.json` | The same content, readable. It is byte-for-byte what is inside the signed file. |
| `manifest/beta.jws`, `manifest/beta.json` | The same for the Beta channel (people who chose "Beta" in Settings). |
| `manifest/android.jws`, `manifest/android.json` | The **signed** announcement of the current Android version (created by the first `android` command). Only the Android app reads it; the Windows files above never contain Android information. |
| `keys/` | The **public** halves of the signing keys (`<id>.pub.pem`, `keys.json`). Published so anyone can check the signatures. The private halves are only on the release PC. |

## How an update reaches people (Windows)

1. A few times a day, each Wire Pulse asks GitHub for `manifest/stable.jws` (or `beta.jws`). It sends nothing about the
   person or the PC — only "WirePulse/1.0.2" as the program name.
2. It checks the signature. If it is not ours, or older than one it has already seen, it ignores the file.
3. If the version is newer and this PC is in the current rollout group, it downloads the installer in the background and
   checks its SHA-256 fingerprint against the signed manifest. A single wrong byte and the file is deleted.
4. It installs the update **the next time Wire Pulse starts with Windows** (at sign-in, when "Start with Windows" is on),
   when the person clicks **Restart to update**, while they are away from a PC that has had it ready for half a day, or —
   for an **urgent** release — right away. Opening Wire Pulse from the Start menu does not install it (the person is
   waiting for the window); the banner says "Restart to update". No administrator prompt when Wire Pulse runs as
   administrator (the default). A **per-machine install that does not run as administrator** (the person turned "Run as
   administrator" off) needs a click on **Install…** and an administrator's approval for every update.
5. After installing, the new version tests itself. If the test fails, the previous version is put back automatically and
   the person sees a message. If the new version installs fine but then keeps crashing when it starts, the third start puts
   the previous version back by itself. An update that could not be undone (no previous version to go back to) installs
   only when the person clicks **Restart to update**.

People can turn automatic checks or automatic installation off, choose Stable or Beta, and press **Check for updates**
in Settings → General → Updates.

**Android:** Google Play users update through Google Play; Wire Pulse shows a notification and a banner and starts Google
Play's own update screen. People who installed the APK from the download page get a notification with a link to it.

---

## One-time setup

Do these steps once, in this order, **before building Wire Pulse 1.0.2** (the first version with automatic updates).
All commands run in PowerShell in the Wire Pulse source folder.

### 1. Create this repository

1. On GitHub, open **New repository** (`https://github.com/new`).
2. Owner: your account. Name: any (this one is **`0dteam/wirepulse`**). Visibility: **Public** (the apps download without logging in).
3. Tick **Add a README file** (this creates the `main` branch; the tool replaces the README with this one).
4. Create the repository.
5. **Required:** in the repository's **Settings → General → Releases**, turn on **Enable release immutability**. Published
   installers can then never be swapped — not even by someone who stole your token — and the release tool refuses to
   publish while it is off. Do **not** add rules that require pull requests on `main` — the release tool commits to `main`
   directly.

### 2. Create the access token for the release tool

The tool needs permission to create releases and update `manifest/` in this one repository — nothing else.

1. GitHub → your picture → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** →
   **Generate new token**.
2. Token name: `Wire Pulse releases`. Expiration: up to one year (put a reminder in your calendar a week before).
3. **Repository access**: *Only select repositories* → this repository (`wirepulse`).
4. **Permissions** → *Repository permissions* → **Contents: Read and write**. (*Metadata: Read-only* is added
   automatically.) Leave everything else at *No access*.
5. **Generate token** and copy it (GitHub shows it only once).

### 3. Tell Wire Pulse where this repository is

Open `release\update-repository.txt` in the Wire Pulse source and replace `OWNER` in the last line with your GitHub
account and repository name, for example `0dteam/wirepulse`. Commit the change. Every build made after this looks for updates
here; builds made before it never update.

### 4. Protect the signing keys with your release passphrase

```powershell
.\release.ps1 protect-keys
```

Choose a **release passphrase** (at least 12 characters; a long sentence is best), type it twice, and keep it in your
password manager. The signing keys stay on this PC, encrypted for your Windows account *and* with this passphrase: a
program that runs as you cannot use them without it. From now on the tool asks for it whenever it signs or uses the GitHub
token (once per command). It is never stored and never read from a script.

### 5. Give the token to the release tool

```powershell
.\release.ps1 set-token
```

Paste the token and press Enter (nothing is shown while you paste). The tool checks with GitHub that the token works and
may write to this repository, asks for your release passphrase, and stores the token encrypted with it (and for your Windows
account) in `C:\Users\<you>\WirePulse-Secrets\github-release-token.bin`. A token GitHub rejects is not stored, and the tool
refuses while step 3 is not done (it cannot check a token without the repository name). You can delete the copy you pasted
from anywhere else.

### 6. Upload this README and the public keys

```powershell
.\release.ps1 init
```

It shows what it will upload and asks before it does. Run it again after a key rotation (it uploads only what changed).

### 7. Back up the signing keys (important)

The signing keys were created on the release PC and exist only there. If that PC dies without a backup, no existing
installation can ever be updated automatically again. Make a backup protected with a passphrase **you** choose:

```powershell
.\release.ps1 backup-key -Out E:\WirePulse-update-keys.backup.json
.\release.ps1 verify-backup -In E:\WirePulse-update-keys.backup.json
```

(Same thing without the release tool: `dotnet run --file tools/release/update-keys.cs -- backup-key --out <file>`.)

First your release passphrase (to read the keys), then a **backup** passphrase, twice (at least 12 characters; a long
sentence is best; it may differ from the release passphrase). Keep the backup file (USB stick, second disk) and the backup
passphrase (password manager) in **different** places. Make a new backup after every key rotation.

### 8. Check everything

```powershell
.\release.ps1 status
```

It shows the repository, whether a token is stored (never the token itself), the keys (and proves they still decrypt),
the current manifests, what the apps receive, and the latest releases. The last line says whether everything is OK.

---

## Publishing a release

The build side (version number, building, tests) is in `docs/RELEASING.md` of the source. Here is what you do with the
finished build.

### Normal release: beta first, then stable in steps

```powershell
# 1. Write the release notes (plain text, English required) and commit them:
notepad release\notes\1.0.3.json
git add release\notes; git commit -m "Release notes 1.0.3"

# 2. Look before you publish: every check, the plan and the manifest change, and nothing is changed:
.\release.ps1 publish -Channel beta -Version 1.0.3 -DryRun

# 3. Publish to the Beta channel (builds 1.0.3 first if artifacts\ holds another version; asks for your release passphrase):
.\release.ps1 publish -Channel beta -Version 1.0.3

# 4. Try it: on a test PC (or your own), choose Settings > General > Updates > Update channel: Beta,
#    click Check for updates, let it install, use it for a day.

# 5. Give it to 10 % of stable users:
.\release.ps1 promote -Version 1.0.3 -Rollout 10

# 6. No problems reported after a day or two? Widen it:
.\release.ps1 rollout -Channel stable -Percent 50
.\release.ps1 rollout -Channel stable -Percent 100
```

(A build needs committed source: changes to tracked files stop it; untracked files, like notes you have not committed yet,
do not — but the release is built from what is committed, so commit the notes first.)

* `publish` checks the build (a normal build, not a test build; made from committed source; pointing at this repository;
  trusting exactly our keys), runs the installer smoke test (**close Wire Pulse on this PC first** — tray icon → Exit — the
  test refuses while it runs; it also checks that the program *inside* the installer is exactly this release), uploads the
  installer, the portable zip and the checksums as a GitHub release, checks that the release is immutable, downloads the
  installer again to prove it arrived intact, and only then signs and uploads the manifest.
* Every command that changes something shows what it will do and asks **Publish? [y/N]** first. `-Yes` answers for you.
* If it stops halfway (network), run the same command again; it continues. When everything is done it says
  "Nothing to do".
* `promote` does not rebuild anything: stable users get exactly the file beta users tested.
* The rollout group is random per installation and per version, so different people go first each time. A person who
  clicks **Check for updates** gets the new version even when their installation is not in the group yet.
* Pure beta builds (`-Version 1.0.4-beta.1`) can only go to the Beta channel.

### Urgent (security) release

Add `-Urgent` when publishing and promote to everyone at once:

```powershell
.\release.ps1 publish -Channel beta -Version 1.0.4 -Urgent
.\release.ps1 promote -Version 1.0.4 -Rollout 100
```

Or, when it cannot wait for a beta round, straight to stable (beta users get it too):

```powershell
.\release.ps1 publish -Channel stable -Version 1.0.4 -SkipBeta -Urgent
```

Urgent updates install as soon as they are downloaded (with a one-minute notice when the window is open) instead of
waiting for the next start. People who turned automatic installation off still only get a message.

### The very first release (1.0.2)

1.0.2 is the first version that updates itself, so everyone installs it **by hand once** (from this repository's
Releases page, or your download page linking to it). Publish it the normal way (`publish -Channel beta`, then
`promote -Version 1.0.2 -Rollout 100`). From 1.0.3 on, everything is automatic.

---

## When something goes wrong

**Stop a release immediately (kill switch)** — nobody else downloads or installs it; people who already installed it
keep it:

```powershell
.\release.ps1 halt -Channel stable
```

Continue later with `.\release.ps1 rollout -Channel stable -Percent 20`.

**Pull a bad release** — everybody who has it is moved to another version, even an older one:

```powershell
.\release.ps1 pull -Version 1.0.3            # back to the previous good version (1.0.2)
.\release.ps1 pull -Version 1.0.3 -To 1.0.4  # or forward to a fixed version you already published
```

Then publish a fixed version as usual. Only pull back to an older version if that version can read the data the pulled
one wrote (the tool reminds you and asks; Wire Pulse versions keep their data compatible unless the release notes say
otherwise). The pulled release is renamed "… (withdrawn)" on GitHub.

**A new version fails on some PCs** — nothing to do on your side for those PCs: when the new version's self-test fails,
the previous version is reinstalled automatically and that PC does not retry the same version. Halt or pull the release
if it affects many people.

**Someone changed this repository** — if `manifest/` was edited or reverted without the signing key, the apps ignore the
change (it has no valid signature, or an older one than they already saw), and the release tool refuses to continue
from it. Change your GitHub password, revoke the token and create a new one (`set-token`), then run your command again
with `-Force`: it continues from the last manifest your PC signed and publishes a fresh, higher-numbered one.

---

## Android

Android updates come from Google Play (or the APK download page). After the new Android version is live in Google Play,
announce it so the apps show their notification: the tool signs `manifest/android.jws`. Either host the APK here (the
tool creates the release `android-v1.0.3` and links to it):

```powershell
.\release.ps1 android -VersionName 1.0.3 -VersionCode 4 -Apk mobile\dist\1.0.3\WirePulse-1.0.3.apk
```

or point to a download page you already have (a page on your website, or an existing Releases page of this repository —
the tool checks that it opens):

```powershell
.\release.ps1 android -VersionName 1.0.3 -VersionCode 4 -DirectUrl https://lab.0d.ae/wireplus/android
```

The APK must be the **release** build: built with your release key (`mobile\android\key.properties`) after step 3 of the
setup, signed with the certificate written in `release\android-signing-cert.txt`. The tool refuses a debug-signed APK (a
phone with a debug-signed install can never install the real one over it), any other certificate, and an APK built while
the repository name was still `OWNER` (it would never look for updates). See `docs/RELEASING.md` section 6.

Add `-Urgent` to make Google Play users go through Google Play's full-screen update right away. The Windows manifests
are not touched (Windows Wire Pulse 1.0.2 would stop updating if they contained Android information), so no Windows
release is needed first. After a key rotation, run the last `android` command again (same values) to re-sign it.

---

## All commands

| Command | What it does |
|---|---|
| `.\release.ps1 status` | Everything at a glance (add `-Quick` to skip downloading the installers) |
| `.\release.ps1 verify` | Checks anonymously, exactly as the apps do, that manifests verify and every installer downloads with the right fingerprint |
| `.\release.ps1 publish -Channel beta -Version V` | New version to Beta (`-Notes <file>`, `-Urgent`, `-Rollout N`) |
| `.\release.ps1 publish -Channel stable -Version V -SkipBeta` | New version straight to Stable (and Beta) |
| `.\release.ps1 promote -Version V -Rollout N` | Beta version to Stable for N % |
| `.\release.ps1 rollout -Channel C -Percent N` | Change the share (0-100) |
| `.\release.ps1 halt -Channel C` | Kill switch (share 0) |
| `.\release.ps1 pull -Version V [-To W]` | Withdraw a version; everybody on it moves to W |
| `.\release.ps1 android -VersionName N -VersionCode C -Apk <file>` | Announce a new Android version in `manifest/android.jws` (or `-DirectUrl <url>`) |
| `.\release.ps1 protect-keys` | Once: protect the signing keys with your release passphrase |
| `.\release.ps1 set-token` | Store a new GitHub token (under your release passphrase) |
| `.\release.ps1 init` | Upload this README and the public keys |
| `.\release.ps1 keys` | Show the signing keys and prove they decrypt (never shows a private key) |
| `.\release.ps1 backup-key -Out <file>` / `verify-backup -In <file>` / `restore-key -In <file>` | Key backup, check, restore on a new PC |
| `.\release.ps1 help` | The list with all options |

Every command that changes something takes **`-DryRun`** (show everything, change nothing) and **`-Yes`** (do not ask).

---

## Keys: backup, new PC, rotation

* **Where they are:** `C:\Users\<you>\WirePulse-Secrets\update-signing\` on the release PC, encrypted for your Windows
  account and with your release passphrase. Two keys: the *current* one signs; the *next* one is already built into the
  apps and waits for a rotation. `.\release.ps1 keys` shows them and proves they still work.
* **New PC or reinstalled Windows:** copy nothing by hand; use your backup:
  `.\release.ps1 restore-key -In E:\WirePulse-update-keys.backup.json`, then `.\release.ps1 set-token` again.
* **Rotation** (only if the signing key may have been stolen, or planned at most once a year): run
  `dotnet run --file tools/release/update-keys.cs -- rotate --confirm` in the source, commit the changed public-key files,
  build and publish a new version, run `.\release.ps1 init` to update `keys/` here, and make a new backup. From then on
  every manifest also tells the apps to stop trusting the old key; re-sign the other channel once with
  `.\release.ps1 rollout -Channel <channel> -Percent <its current percentage>`. (Details: `docs/UPDATES.md` section 4.4.)
* **Never** copy a key file to GitHub, a server, a chat or an e-mail. The tool never shows private keys.

---

## Troubleshooting

| What you see | What to do |
|---|---|
| `401 Bad credentials` | The token expired or was deleted. Create a new one (setup step 2) and run `.\release.ps1 set-token`. |
| `403 … The token needs 'Contents: Read and write'` | The token lacks *Contents: Read and write* or is for another repository. Create it again (setup step 2). |
| `no GitHub token stored` | Run `.\release.ps1 set-token`. |
| `refusing to ...: the update repository is still the placeholder OWNER/wirepulse-releases` | Do setup step 3 and rebuild. |
| `refusing to sign: the signing key ... is protected by DPAPI only` | Run `.\release.ps1 protect-keys` once (setup step 4). |
| `wrong release passphrase` / `does not open with this release passphrase` | Type it again. Forgotten: restore the keys from your backup on this PC (`restore-key`, new passphrase) and run `set-token` again. |
| `... needs the release passphrase: run this command yourself in a console window` | Run it in a PowerShell window, not from a script. |
| `release immutability is off` / `release ... is not immutable` | Setup step 1, point 5: turn on release immutability, then run the same command again (a release that was not immutable went back to draft). |
| `the source folder has uncommitted changes to tracked files` | Commit (or undo) the changes the message lists, then run the command again. Untracked files do not matter. |
| `refusing to publish a DEBUG-signed APK` / `not built for <account>/<repository>` | Build the APK with your release key after setup step 3 (`docs/RELEASING.md` section 6). |
| `the release signing certificate is not pinned yet` | Check the printed SHA-256 against `keytool -list -v -keystore <your .jks>`, put it into `release\android-signing-cert.txt`, commit. |
| `-DirectUrl ... does not open` | The page does not exist (yet): use an existing page, or `-Apk <file>` to publish the APK here. |
| `refusing to publish: QA build` | You picked a test build (`artifacts\qa\`). Build the normal version (`.\build.ps1 -Version …`). |
| `refusing to publish this build: … dirty=true` | The build was made with uncommitted changes. Commit, then run the same `publish` command with `-Rebuild`. |
| `The Wire Pulse desktop app is running` (smoke test) | Exit Wire Pulse on this PC (tray icon → Exit) and run the command again. |
| `the manifest in the repository does not verify` or `… was reverted` | Someone changed `manifest/` without the key. Change your GitHub password, revoke the token, create a new one (`set-token`), then run the same command again with `-Force` (see "Someone changed this repository"). |
| People do not get the update | Wait 5 minutes (GitHub caches the manifest). Check the rollout percentage (`.\release.ps1 status`), then `.\release.ps1 verify`. In the app, Settings shows "Last checked". |
| `cannot decrypt` for a key | Windows was reinstalled or you use another account: restore from your backup (`restore-key`). |
| Upload stopped halfway | Run the same `publish` command again; it continues with the draft release. |
| `status` lists a `DRAFT` release | A publish was interrupted: run the same `publish` command again. |

---

## Security in plain words

* **Two separate locks.** GitHub stores and delivers the files; the **signature** proves they come from you. The apps
  check the signature of the manifest and the fingerprint of every installer before using them.
* **The signing key never leaves your PC.** It is encrypted for your Windows account and with your release passphrase, so
  even a program that runs as you cannot sign anything; your backup is encrypted with its own passphrase.
* **The GitHub token cannot sign.** It only lets the tool upload files to this one repository, it is stored under the same
  passphrase, and releases are immutable: nobody can swap a published installer.
* **If someone takes over your GitHub account or token**, they can delete files or stop updates for a while, but they
  **cannot** make any Wire Pulse install their files: without the signing key, the apps ignore everything they upload.
  Change your password, revoke the token and publish again.
* **If the signing key itself may have leaked**, rotate it (section above). The apps already know the next key.
* **Privacy:** the apps send no personal data when they check for updates. GitHub sees the IP address of each download,
  as with any website.
