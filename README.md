# How releases work

```
 your website (Render)  ──────────►  everybody, instantly (the app is a window onto it)
 your source repository ──(tag)───►  GitHub Actions builds the installer
                                           │
                                           ▼
                        public "releases" repository  ◄── people download here
                                           │
                                           ▼
                        installed apps check it and update themselves (Windows)
```

**Two kinds of updates.** Changes to the *website* (lyrics, scoring, rooms, buttons, fixes) go live for everyone the moment you push to Render.
You only publish a new *app version* when the wrapper itself changes (new Electron version, new window behaviour, new Android permission, new icon).

## One-time setup

1. **Create the public releases repository**, for example `karaoke-night-app`. Put the file `releases-repo/README.md` in it as its README. Nothing else is needed there.
2. **Create an access token.** GitHub > Settings > Developer settings > Personal access tokens > *Fine-grained tokens* > Generate.
   Repository access: **Only select repositories** > the releases repository. Permissions: **Contents: Read and write**. Copy the token.
3. **In your SOURCE repository** (the one that contains the `apps/` folder): Settings > Secrets and variables > Actions > **New repository secret**:
   name `RELEASES_TOKEN`, value = the token.
4. **Tell the project where the releases live**:
   - `apps/desktop-electron/package.json` > `build.publish` > set `owner` and `repo`.
   - `public/js/download-page.js` > `RELEASES` = `owner/repo` (the website’s Download page then always links to the newest release).
5. On your PC, in `apps/desktop-electron`, run `npm install` once (it adds `electron-updater` to `package-lock.json`), and commit the changed `package-lock.json`.
6. Copy the folder `.github/workflows/` from this kit to the root of the source repository and commit.

## Every release (Windows)

1. Raise `"version"` in `apps/desktop-electron/package.json` (for example `1.1.0` > `1.1.1`) and commit.
2. `git tag app-v1.1.1` then `git push origin app-v1.1.1`.
3. Actions > *Release desktop app* builds it and publishes **Release v1.1.1** in the releases repository, with `KaraokeNight-Setup.exe`.
   Edit the release notes there if you like.
4. People who already have the app get it automatically (within about 6 hours, installed when they close the app).
   New people use the download link from the Releases page or the website’s Download page.

The very first install is always manual; every later version updates by itself.

## Android APK (optional)

Android has no silent updates outside the Play Store, so people install the new `.apk` over the old one. That only works if **every APK is signed with the
same key**, so make the key once and keep it safe (if you lose it, everybody must uninstall before updating).

1. Create the key (needs Java; one line, answer the questions, remember the passwords):
   `keytool -genkeypair -v -keystore release.keystore -alias karaoke -keyalg RSA -keysize 2048 -validity 10000`
2. Turn it into text: macOS/Linux `base64 -w0 release.keystore`, Windows PowerShell
   `[Convert]::ToBase64String([IO.File]::ReadAllBytes("release.keystore"))`.
3. In the source repository add the secrets `ANDROID_KEYSTORE_BASE64` (that text), `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS` (`karaoke`),
   `ANDROID_KEY_PASSWORD`, and a **variable** `RELEASES_REPO` = `owner/repo`.
4. Release the desktop version first (the Release must exist), then Actions > *Release Android APK* > **Run workflow** > type the same version.
5. In `public/js/download-page.js` set `APK_PUBLISHED = true`.

## Worth knowing

- **The app loads your website address.** It is stored in `apps/desktop-electron/config.json` (and in `capacitor.config.json` for Android). If that address ever changes,
  installed apps stop working until they are updated. A **custom domain** that you control, instead of the `onrender.com` address, protects you from that.
- **Windows SmartScreen** warns about unsigned installers until enough people have installed it (or you buy a code-signing certificate). The warning is normal for hobby projects.
- **Turn auto-update off** for a build with `"autoUpdate": false` in `apps/desktop-electron/config.json`.
- Treat `RELEASES_TOKEN` and the keystore like passwords. Never commit them.
