# Build & Distribution Guide for React Native ReState

This document walks you through building Android and iOS binaries from Scratch using Expo EAS and sharing installable links without publishing to app stores.

---

## 1. Prerequisites

- Node.js (>=16.x)
- Expo CLI (optional)
- [EAS CLI](https://docs.expo.dev/build/introduction/) (`npm install -g eas-cli`)
- Java Development Kit (JDK) with `keytool` (for Android keystore)
- An Expo account (`eas login`)

---

## 2. Install Dependencies

```bash
# Install EAS CLI globally
npm install -g eas-cli

# (Optional) Install Expo CLI if you use classic builds
npm install -g expo-cli
```

## 3. Java (keytool) Setup for Android

Android requires a signing keystore. You need Java’s `keytool` utility:

### 3.1 Windows via winget

Open PowerShell **as Admin**:

```powershell
winget install --id Microsoft.OpenJDK.17
```

### 3.2 Verify installation

```powershell
java -version
keytool -help
```

---

## 4. Create Android Keystore

Run from your project root:

```powershell
keytool -genkeypair `
  -v `
  -storetype PKCS12 `
  -alias app_alias `
  -keyalg RSA `
  -keysize 2048 `
  -validity 10000 `
  -keystore .\my-release-key.jks `
  -storepass YOUR_STORE_PASSWORD `
  -keypass YOUR_KEY_PASSWORD `
  -dname "CN=Sahil Hasnain, OU=Dev Team, O=Sahil Web Solutions, L=Delhi, S=Delhi, C=IN"
```

> Replace `YOUR_STORE_PASSWORD` and `YOUR_KEY_PASSWORD` with secure values you choose. Store them safely!

---

## 5. Upload Keystore to EAS

```bash
eas credentials -p android \
  --upload-keystore \
  --keystore-path my-release-key.jks \
  --keystore-password YOUR_STORE_PASSWORD \
  --key-alias app_alias \
  --key-password YOUR_KEY_PASSWORD
```

This links your keystore to your Expo project.

---

## 6. eas.json Configuration

In your `eas.json` (project root) add a profile for internal distribution. Example:

```json
{
  "cli": { "version": ">= 16.17.4" },
  "build": {
    "preview": {
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "ios": { "simulator": false }
    }
  }
}
```

- **preview**: Internal build you can share with users.
- **production**: (Optional) for Play/App Store builds.

---

## 7. Build Commands

### Android APK

```bash
eas build -p android --profile preview
```

### iOS (TestFlight / Ad Hoc)

```bash
eas build -p ios --profile preview
```

After each build finishes, Expo will print a URL. Share the Android APK link directly with Android users. For iOS:

- **TestFlight**: Expo can upload to App Store Connect for internal testing. Invite testers via TestFlight.
- **Ad Hoc**: Configure an Ad Hoc provisioning profile in `eas.json` and install the `.ipa` link on registered devices.

---

## 8. Environment Variables for Keystore Passwords

You can store passwords in environment variables in a `.env.local` file (not committed):

```bash
ANDROID_KEYSTORE_PASSWORD=YOUR_STORE_PASSWORD
ANDROID_KEY_PASSWORD=YOUR_KEY_PASSWORD
```

Then reference them in scripts or in CI workflows.

---

## 9. Common Issues & Solutions

### Hooks Order Error

**Error**: "Rendered more hooks than during previous render."
**Cause**: Conditionally calling hooks after an early return.
**Fix**: Always call hooks (e.g. `useSegments`) before any return statements to keep the call order consistent.

### Splash Screen Confusion

- **Native Splash**: Shown automatically before JS loads. Configure in `app.json`:
  ```json
  "expo": {
    "splash": {"image":"./assets/splash.png","resizeMode":"contain","backgroundColor":"#fff"}
  }
  ```
- **Custom Splash**: In-app splash (e.g. while loading fonts) in your code using state (e.g. `fontsLoaded`).

### Conditional Developer Credits

Use Expo Router’s `useSegments()` in `_layout.tsx`:

```ts
const segments = useSegments();
const showCredit = segments[0] === "sign-in";
// render credit only when showCredit is true
```

### Environment Variables in EAS

- **EXPO*PUBLIC*** variables are inlined automatically on client.
- Define secrets in **each** profile of `eas.json` under `build.[profile].env` or via EAS Dashboard.
- To read at runtime, use Expo Config: in `app.config.js`:
  ```js
  export default ({ config }) => ({
    ...config,
    extra: {
      apiUrl: process.env.EXPO_PUBLIC_API_URL,
    },
  });
  ```
  Then `import Constants from 'expo-constants';` and `Constants.expoConfig.extra.apiUrl`.

### Keystore & keytool Errors

- **keytool not found**: Install Java JDK (with keytool) via `winget install --id Microsoft.OpenJDK.17` or MSI installer.
- **EAS cloud keystore 500**: Generate keystore locally:
  ```powershell
  keytool -genkeypair -storetype PKCS12 -alias app_alias -keystore my-release-key.jks \
    -storepass <STORE_PASS> -keypass <KEY_PASS> -dname "CN=Org,OU=Dev,O=Org,L=City,S=State,C=IN"
  ```
  Then upload with:
  ```bash
  eas credentials -p android --upload-keystore --keystore-path my-release-key.jks \
    --keystore-password $STORE_PASS --key-alias app_alias --key-password $KEY_PASS
  ```

---

## 10. Doubts & Answers

**Q: How do EAS builds pick up my `.env` values?**  
A: Expo only inlines `EXPO_PUBLIC_` vars from `eas.json` (per profile) or the EAS Dashboard. Plain `.env` files are _not_ uploaded. To access at runtime, inject into `app.config.js` and read via `Constants.expoConfig.extra`.

**Q: What is `-dname` in the `keytool` command?**  
A: `-dname` specifies the certificate's _Distinguished Name_, a comma-separated string (CN=Common Name, OU=Org Unit, O=Org, L=Locality, S=State, C=Country) embedded in the self-signed cert.

**Q: Where do I get `storepass` and `keypass`?**  
A: You choose them when running `keytool -genkeypair`: the values after `-storepass` and `-keypass` are your passwords. Store them in a secure `.env` (ignored by Git) or CI secret.

**Q: What is the keystore alias?**  
A: Alias is the _name_ of the key inside your `.jks` or `.p12`. You set it via `-alias app_alias`. During EAS upload, supply the same alias so Expo knows which key to use.

**Q: Why did I see a "Rendered more hooks than during previous render" error?**  
A: You called hooks (e.g. `useSegments`) _after_ an early return (`if (!fontsLoaded) return null`). Always call hooks in the same order at the top of your component.

**Q: Native vs. custom splash screen—what’s the difference?**  
A: _Native splash_ is configured in `app.json` and shows before JS loads. _Custom splash_ (e.g. while `fontsLoaded`) is rendered in React code (in `_layout.tsx`). Use both for smooth UX.

**Q: I got "No environment variables with visibility ... found"—is that an error?**  
A: No. That message means you haven’t set any _EAS Dashboard_ secrets for that profile. If your `eas.json` profile has `env`, those vars still load correctly.

**Q: Why didn’t I get a download link after running `eas build`?**  
A: The link appears only once the remote build finishes. Use `--wait` with your command or check `eas build:list` / the Expo dashboard to see and grab the URL.

**Q: Why is my APK so large?**  
A: Common causes are bundling all of `assets/` (`

---

# Enjoy your builds!

Keep this file handy so you never have to hunt instructions again.
