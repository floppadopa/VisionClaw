# VisionClaw — iPhone setup from Windows (via GitHub Actions)

No Mac needed. This guide builds the iOS app in the cloud and installs it on your iPhone.

## Prerequisites

- **iPhone** running iOS 17+
- **Apple ID** (free — no $99 developer account needed)
- **GitHub account** (free)
- **Gemini API key** — get one free at https://aistudio.google.com/apikey
- **Sideloadly** — download at https://sideloadly.io (Windows installer)
- **iTunes** — install from Microsoft Store (needed for iPhone USB drivers)
- **USB cable** — Lightning or USB-C to connect iPhone to PC

---

## Step 1: Fork the repo on GitHub

1. Go to https://github.com/Intent-Lab/VisionClaw
2. Click **Fork** (top right)
3. This creates your own copy at `github.com/YOUR_USERNAME/VisionClaw`

## Step 2: Add the build workflow

The workflow file is already in this repo at `.github/workflows/build-ios.yml`.

If you forked the original repo (which doesn't have this file), upload it:

1. In your fork on GitHub, click **Add file > Upload files**
2. Create the folder path `.github/workflows/`
3. Upload the `build-ios.yml` file from this repo

Or use the GitHub web editor:
1. Go to your fork
2. Press `.` (period) to open github.dev editor
3. Create `.github/workflows/build-ios.yml`
4. Paste the contents of the workflow file from this repo

## Step 3: Add your Gemini API key as a secret

1. In your fork, go to **Settings > Secrets and variables > Actions**
2. Click **New repository secret**
3. Name: `GEMINI_API_KEY`
4. Value: paste your Gemini API key
5. Click **Add secret**

(Or skip this — you can enter the key directly in the app's Settings screen after install.)

## Step 4: Run the build

1. In your fork, go to **Actions** tab
2. Click **Build VisionClaw iOS (unsigned IPA)** in the left sidebar
3. Click **Run workflow** (button on the right)
4. Optionally paste your Gemini API key in the input field
5. Click **Run workflow**

Wait ~10-15 minutes. When the green checkmark appears:

6. Click on the completed workflow run
7. Scroll down to **Artifacts**
8. Download **VisionClaw-unsigned-IPA**
9. Unzip it — you'll get `VisionClaw-unsigned.ipa`

## Step 5: Install Sideloadly on Windows

1. Download from https://sideloadly.io
2. Install it
3. Make sure **iTunes** is installed (from Microsoft Store)

## Step 6: Install the IPA on your iPhone

1. Connect your iPhone to your PC via USB
2. Trust the computer on your iPhone if prompted
3. Open **Sideloadly**
4. Drag `VisionClaw-unsigned.ipa` into Sideloadly
5. Enter your **Apple ID** email
6. Click **Start**
7. Enter your Apple ID password when prompted
8. If you have 2FA, enter the verification code

Sideloadly will sign the app with your free Apple ID and install it on your iPhone.

**Important:** After install, on your iPhone go to:
**Settings > General > VPN & Device Management > [your Apple ID email] > Trust**

## Step 7: Enable Developer Mode on Ray-Ban glasses

1. Open the **Meta AI** app on your iPhone
2. Go to **Settings** (gear icon, bottom left)
3. Tap **App Info**
4. Tap the **App version number 5 times**
5. Go back to Settings — toggle **Developer Mode** on

## Step 8: Use VisionClaw

1. Open the **VisionClaw** app on your iPhone
2. If you didn't embed the API key in the build, tap **Settings** (gear icon) and enter your Gemini API key
3. **Without glasses:** Tap "Start on iPhone" to use your phone camera
4. **With glasses:** Tap "Start Streaming" to use your Ray-Ban camera
5. Tap the **AI button** and start talking!

---

## FAQ

### The app expires after 7 days?
Yes — free Apple ID signing lasts 7 days. After that, re-install with Sideloadly (same steps, takes 2 minutes). A paid Apple Developer account ($99/year) makes it last 1 year.

### Sideloadly asks for an app-specific password?
If your Apple ID has 2FA, you may need an app-specific password:
1. Go to https://appleid.apple.com
2. Sign in > Security > App-Specific Passwords
3. Generate one and use it in Sideloadly

### The build fails on GitHub Actions?
- Check the workflow logs for errors
- Make sure you forked the repo correctly
- The Meta DAT SDK package should resolve automatically (it's a public GitHub package)

### Can I update the API key without rebuilding?
Yes! The app has a **Settings screen** where you can change the Gemini API key, system prompt, and all other settings at runtime.

### Free GitHub Actions minutes?
Free tier gives 2,000 minutes/month. macOS runners bill at **10x rate**, so each 15-min build costs ~150 minutes. You get about **13 builds per month** free.
