# VisionClaw on iPhone from Windows — Step by Step

---

## STEP 0: Get what you need ready

You need 4 things before starting. Do all of these first:

### 0A — Get a Gemini API key (free)

1. Open https://aistudio.google.com/apikey
2. Sign in with your Google account
3. Click "Create API Key"
4. Copy the key and save it somewhere (notepad is fine)

### 0B — Install iTunes on your PC

1. Open the Microsoft Store on your PC
2. Search "iTunes"
3. Install it (this gives Windows the iPhone USB drivers)
4. You don't need to open it, just have it installed

### 0C — Install Sideloadly on your PC

1. Go to https://sideloadly.io
2. Click "Download for Windows"
3. Run the installer, install it
4. Don't open it yet

### 0D — Have a GitHub account

1. If you don't have one, go to https://github.com and sign up (free)
2. If you already have one, make sure you're signed in

---

## STEP 1: Fork the VisionClaw repo

1. Open this link: https://github.com/Intent-Lab/VisionClaw
2. In the top right, click the **Fork** button
3. On the next page, leave everything as default
4. Click **Create fork**
5. Wait a few seconds — you now have your own copy at `github.com/YOUR_USERNAME/VisionClaw`

---

## STEP 2: Add the build workflow file

You need to add one file to your fork. This tells GitHub how to build the app.

1. On your fork's page, click the **Add file** dropdown (green button area, top right of the file list)
2. Click **Create new file**
3. In the "Name your file" box at the top, type exactly:
   ```
   .github/workflows/build-ios.yml
   ```
   (GitHub will auto-create the folders as you type the slashes)

4. In the big editor box below, paste this entire content:

```yaml
name: Build VisionClaw iOS (unsigned IPA)

on:
  workflow_dispatch:
    inputs:
      gemini_api_key:
        description: "Gemini API key (or leave empty to use repo secret)"
        required: false
        type: string

jobs:
  build:
    runs-on: macos-15
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Select Xcode
        run: |
          XCODE_PATH=$(ls -d /Applications/Xcode*.app 2>/dev/null | sort -V | tail -1)
          echo "Using Xcode at: $XCODE_PATH"
          sudo xcode-select -s "$XCODE_PATH"
          xcodebuild -version

      - name: Create Secrets.swift
        run: |
          API_KEY="${{ inputs.gemini_api_key }}"
          if [ -z "$API_KEY" ]; then
            API_KEY="${{ secrets.GEMINI_API_KEY }}"
          fi
          if [ -z "$API_KEY" ]; then
            API_KEY="YOUR_GEMINI_API_KEY"
            echo "No API key provided. You can add it later in the app Settings."
          fi

          cat > samples/CameraAccess/CameraAccess/Secrets.swift << SWIFT_EOF
          import Foundation

          enum Secrets {
            static let geminiAPIKey = "$API_KEY"
            static let openClawHost = "http://placeholder.local"
            static let openClawPort = 18789
            static let openClawHookToken = "placeholder"
            static let openClawGatewayToken = "placeholder"
            static let webrtcSignalingURL = "ws://placeholder:8080"
          }
          SWIFT_EOF

          echo "Secrets.swift created."

      - name: Resolve Swift packages
        working-directory: samples/CameraAccess
        run: |
          xcodebuild -resolvePackageDependencies \
            -project CameraAccess.xcodeproj \
            -scheme CameraAccess \
            -clonedSourcePackagesDirPath ../spm_cache

      - name: Build for device (unsigned)
        working-directory: samples/CameraAccess
        run: |
          xcodebuild build \
            -project CameraAccess.xcodeproj \
            -scheme CameraAccess \
            -configuration Release \
            -sdk iphoneos \
            -derivedDataPath build \
            -clonedSourcePackagesDirPath ../spm_cache \
            CODE_SIGNING_ALLOWED=NO \
            CODE_SIGN_IDENTITY="-" \
            DEVELOPMENT_TEAM="" \
            PRODUCT_BUNDLE_IDENTIFIER="com.visionclaw.app" \
            AD_HOC_CODE_SIGNING_ALLOWED=YES \
            CODE_SIGNING_REQUIRED=NO

      - name: Package unsigned IPA
        working-directory: samples/CameraAccess
        run: |
          APP_PATH=$(find build/Build/Products/Release-iphoneos -name "*.app" -type d | head -1)
          if [ -z "$APP_PATH" ]; then
            echo "ERROR: .app not found"
            find build -name "*.app" -type d
            exit 1
          fi
          echo "Found app at: $APP_PATH"

          mkdir -p ipa/Payload
          cp -r "$APP_PATH" ipa/Payload/
          cd ipa
          zip -r ../VisionClaw-unsigned.ipa Payload
          echo "IPA created."
          ls -lh ../VisionClaw-unsigned.ipa

      - name: Upload IPA artifact
        uses: actions/upload-artifact@v4
        with:
          name: VisionClaw-unsigned-IPA
          path: samples/CameraAccess/VisionClaw-unsigned.ipa
          retention-days: 30
```

5. Scroll down and click **Commit changes**
6. In the popup, leave "Commit directly to the main branch" selected
7. Click **Commit changes**

---

## STEP 3: Add your Gemini API key as a GitHub secret

1. On your fork's page, click **Settings** (top menu bar, far right)
2. In the left sidebar, click **Secrets and variables**
3. Click **Actions**
4. Click the green **New repository secret** button
5. Fill in:
   - **Name**: `GEMINI_API_KEY`
   - **Secret**: paste your Gemini API key from Step 0A
6. Click **Add secret**

---

## STEP 4: Run the build

1. On your fork's page, click the **Actions** tab (top menu bar)
2. You might see a message "Workflows aren't being run on this forked repository" — click **I understand my workflows, go ahead and enable them**
3. In the left sidebar, click **Build VisionClaw iOS (unsigned IPA)**
4. On the right side, click the **Run workflow** dropdown
5. Leave "Branch: main" selected
6. You can optionally paste your Gemini API key in the input (or leave empty if you set the secret in step 3)
7. Click the green **Run workflow** button
8. The page will refresh — click on the new workflow run that appeared (it has a yellow spinning icon)
9. **Wait 10-15 minutes** for it to finish. The icon will turn to a green checkmark when done.

If it fails (red X), click on it to see the error logs. Common issues:
- The Meta DAT SDK package failed to resolve — try running again
- Xcode version mismatch — the workflow auto-selects the latest

---

## STEP 5: Download the IPA

1. Once the build shows a green checkmark, click on the workflow run
2. Scroll down to the **Artifacts** section at the bottom
3. Click **VisionClaw-unsigned-IPA** to download it
4. A `.zip` file downloads to your PC
5. Open the zip and extract `VisionClaw-unsigned.ipa` to your Desktop (or anywhere you'll find it)

---

## STEP 6: Connect your iPhone to your PC

1. Plug your iPhone into your PC with a USB cable (Lightning or USB-C)
2. On your iPhone, a popup says "Trust This Computer?" — tap **Trust**
3. Enter your iPhone passcode
4. If iTunes opens, just close it — you don't need it

---

## STEP 7: Install the app with Sideloadly

1. Open **Sideloadly** on your PC
2. Your iPhone should appear in the dropdown at the top (if not, reconnect USB and restart Sideloadly)
3. Click the big IPA icon on the left, or drag `VisionClaw-unsigned.ipa` onto the Sideloadly window
4. In the **Apple ID** field, enter your Apple ID email
5. Click **Start**
6. A popup asks for your Apple ID password — enter it
7. If you have 2-factor authentication:
   - Enter the 6-digit code that appeared on your iPhone/Mac
   - If it asks for an "app-specific password", see the troubleshooting section below
8. Wait for the progress bar to complete — it says "Done" when finished

---

## STEP 8: Trust the app on your iPhone

The app is installed but you can't open it yet — Apple requires you to trust it first.

1. On your iPhone, open **Settings**
2. Go to **General**
3. Scroll down and tap **VPN & Device Management** (or "Profiles & Device Management")
4. You'll see your Apple ID email listed under "Developer App"
5. Tap on it
6. Tap **Trust "[your email]"**
7. Tap **Trust** again to confirm

---

## STEP 9: Enable Developer Mode on your Ray-Ban glasses

1. On your iPhone, open the **Meta AI** app (or "Meta View" if that's what it's called)
2. Make sure your Ray-Ban glasses are connected
3. Tap the **Settings** gear icon (bottom left)
4. Tap **App Info**
5. Tap the **App version number** rapidly **5 times**
6. Go back to Settings — a new **Developer Mode** toggle has appeared
7. Turn it **ON**

---

## STEP 10: Use VisionClaw!

1. Open the **VisionClaw** app on your iPhone
2. If you see "Gemini API key not configured":
   - Tap the **Settings gear icon** in the app
   - Paste your Gemini API key
   - Go back
3. **Test without glasses first:**
   - Tap **"Start on iPhone"**
   - Tap the **AI button** (sparkle icon)
   - Say "Hey, what do you see?" — the AI will describe what your iPhone camera sees
   - Say anything else — it's a real-time voice conversation
4. **With your Ray-Ban glasses:**
   - Make sure they're connected via Bluetooth in the Meta AI app
   - In VisionClaw, tap **"Start Streaming"**
   - Tap the **AI button**
   - Talk naturally — the AI sees through your glasses and responds via voice

---

## Troubleshooting

### "Untrusted Developer" when opening the app
You skipped Step 8. Go to Settings > General > VPN & Device Management and trust your Apple ID.

### App stops working after 7 days
Free Apple ID signing expires every 7 days. Just redo Steps 6-8 with Sideloadly (same IPA file, takes 2 minutes). A paid Apple Developer account ($99/year) makes it last 1 year.

### Sideloadly asks for "app-specific password"
1. Go to https://appleid.apple.com
2. Sign in
3. Go to **Sign-in and Security > App-Specific Passwords**
4. Click **Generate an app-specific password**
5. Name it "Sideloadly"
6. Copy the password and paste it in Sideloadly

### Sideloadly says "device not found"
- Make sure iTunes is installed (Microsoft Store version)
- Try a different USB cable
- On your iPhone, check you tapped "Trust This Computer"
- Restart Sideloadly

### Build failed on GitHub Actions
- Click on the failed run to see error logs
- If it says "package resolution failed": just click **Re-run all jobs**
- If it says "no signing certificate": that's expected and already handled by the workflow (CODE_SIGNING_ALLOWED=NO)
- If you get a persistent error, open an issue on this repo with the error log

### The AI can't hear me
- Make sure you granted microphone permission when the app asked
- Speak clearly at normal volume
- Check that your iPhone isn't on silent mode

### Glasses not connecting
- Make sure Developer Mode is ON (Step 9)
- Make sure glasses are paired in the Meta AI app first
- Close and reopen VisionClaw
- Try toggling Bluetooth off and on

---

## Summary of what's happening

```
Your Ray-Ban glasses → (Bluetooth) → iPhone → (WebSocket) → Gemini AI (cloud)
                                                                    ↓
                                                              Voice response
                                                                    ↓
                                              iPhone speaker ← (WebSocket) ←
```

The glasses stream their camera at ~1 frame/sec and audio in real-time to your iPhone.
The VisionClaw app relays everything to Google's Gemini AI via WebSocket.
Gemini responds with voice that plays through your iPhone speaker.
