# TextLens AI — AI-Powered Handwritten Text Recognition & Summarizer

TextLens AI is a cross-platform mobile application built with React Native for Android and iOS that digitizes English handwritten notes using on-device optical character recognition (OCR). It enables users to transcribe, listen to (TTS), copy, and optionally summarize handwritten notes with Google Gemini AI, persisting all scans and results to a personal recent history dashboard.

## 🚀 Key Features

* **Authentication & Profiles:** User sign-up, sign-in, and persistent sessions via Firebase Authentication.
* **On-Device Handwriting Recognition (OCR):** Low-latency, privacy-preserving Latin script English text extraction using Google ML Kit.
* **Clipboard & Speech Actions:** One-tap clipboard copy and native text-to-speech (TTS) playback for recognized text.
* **Intelligent Summarization:** Optional popup asking users if they want an AI-generated bullet-point summary powered by Google Gemini (`gemini-1.5-flash`).
* **Scan History Dashboard:** Automatic synchronization of handwritten images, detected text, and generated summaries stored in Cloud Firestore and Cloud Storage.

## 🛠️ Tech Stack

### Mobile Client (Frontend)

* **Framework:** [React Native](https://reactnative.dev/) (TypeScript)
* **Navigation:** React Navigation (Native Stack)
* **Camera / Gallery:** `react-native-image-picker`
* **OCR Engine:** `@react-native-ml-kit/text-recognition` (Google ML Kit on-device)
* **Text-to-Speech:** `react-native-tts`
* **Clipboard:** `@react-native-clipboard/clipboard`
* **State & Data Fetching:** TanStack Query (React Query) + Zustand

### Backend & Cloud Services

* **Authentication:** Firebase Authentication (Email/Password, Google OAuth)
* **Serverless Backend:** Firebase Cloud Functions (v2, Node.js/TypeScript)
* **AI Model:** Google Gemini API (`@google/generative-ai` running `gemini-1.5-flash`)
* **Database:** Cloud Firestore
* **Asset Storage:** Cloud Storage for Firebase

## 🏛️ System Architecture

```
┌────────────────────────────────────────────────────────┐
│                   React Native Client                  │
│       (Camera Capture, History Dashboard, Audio)       │
└───────────────┬────────────────────────┬───────────────┘
                │                        │
       On-Device OCR & TTS          HTTPS / Callable
        (Offline Capable)                │
     ┌──────────────────────┐            ▼
     │  • Google ML Kit     │   ┌────────────────────────┐
     │    Text Recognition  │   │ Firebase Auth / Rules  │
     │  • React Native TTS  │   └───────────┬────────────┘
     └──────────────────────┘               │
                                ┌───────────┴────────────┐
                                ▼                        ▼
                     ┌─────────────────────┐  ┌─────────────────────┐
                     │ Cloud Storage / DB  │  │   Cloud Functions   │
                     │ • Image Blobs       │  │ • summarizeText     │
                     │ • User Scan History │  │ • Gemini 1.5 Flash  │
                     └─────────────────────┘  └─────────────────────┘
```

## 📡 API & Cloud Function Endpoints

The architecture utilizes authenticated, serverless callable functions to keep secret keys out of the mobile client binary.

### 1. `summarizeText` (Firebase Callable Function)

Generates an executive summary of recognized English handwritten text using Gemini.

* **Type:** Firebase HTTPS Callable (`onCall`)
* **Security:** Requires an authenticated Firebase user session token (`request.auth.uid`).
* **Request Payload:**

  ```json
  {
    "text": "String (Required) — Extracted OCR text to summarize"
  }
  ```

* **Success Response (`200 OK`):**

  ```json
  {
    "summary": "• First key point\n• Second key point\n• Action item"
  }
  ```

* **Error Codes:**
  * `unauthenticated`: Invoked without valid Firebase Auth credentials.
  * `invalid-argument`: Missing, empty, or non-string `text` field.
  * `internal`: Model processing or network failure.

## 🗄️ Firestore Database Schema

### Collection Path: `users/{userId}/scans/{scanId}`

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `id` | `string` | Unique document ID |
| `imageUrl` | `string` | Public/signed download URL from Cloud Storage |
| `imageStoragePath` | `string` | Firebase storage reference path (`scans/{uid}/{timestamp}.jpg`) |
| `detectedText` | `string` | Raw English text produced by Google ML Kit OCR |
| `summary` | `string \| null` | Generated summary from Gemini (or `null` if skipped) |
| `createdAt` | `timestamp` | Firestore server timestamp |

## 🔒 Security Rules

### Firestore Security Rules (`firestore.rules`)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/scans/{scanId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

### Cloud Storage Security Rules (`storage.rules`)

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /scans/{userId}/{fileName} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

## 📂 Project Directory Structure

```text
├── .github/
│   └── workflows/
│       ├── deploy-functions.yml    # Continuous backend deployment
│       └── build-mobile.yml        # Android & iOS build automation
├── android/                        # Android native configuration & manifests
├── ios/                            # iOS workspace, Podfile, & Info.plist
├── functions/                      # Firebase Cloud Functions (Backend)
│   ├── src/
│   │   └── index.ts                # Gemini API integration & callable endpoints
│   ├── package.json
│   └── tsconfig.json
├── src/                            # React Native Application Source
│   ├── components/                 # Reusable UI (Buttons, Modals, AudioPlayer)
│   ├── navigation/                 # AppNavigator (AuthStack vs MainStack)
│   ├── screens/
│   │   ├── Auth/
│   │   │   ├── LoginScreen.tsx
│   │   │   └── RegisterScreen.tsx
│   │   ├── DashboardScreen.tsx     # Scan feed & history items
│   │   └── ScanScreen.tsx          # Camera, ML Kit OCR, and Summary popup
│   ├── services/                   # Firebase, OCR, and TTS abstraction layer
│   └── types/                      # TypeScript declarations & schemas
├── App.tsx                         # Root provider setup
├── package.json
└── README.md
```

## 🚀 CI/CD Pipelines (GitHub Actions)

### 1. Backend Deployment (`.github/workflows/deploy-functions.yml`)

Deploys Cloud Functions and updates Firestore / Storage security rules on pushes to `main`.

```yaml
name: Deploy Backend & Rules

on:
  push:
    branches:
      - main
    paths:
      - 'functions/**'
      - 'firestore.rules'
      - 'storage.rules'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: 'npm'
          cache-dependency-path: functions/package-lock.json

      - name: Install Functions Dependencies
        run: |
          cd functions
          npm ci

      - name: Build TypeScript
        run: |
          cd functions
          npm run build

      - name: Deploy to Firebase
        uses: w9jds/firebase-action@master
        with:
          args: deploy --only functions,firestore:rules,storage
        env:
          GCP_SA_KEY: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_KEY }}
```

### 2. Mobile CI & Release Build (`.github/workflows/build-mobile.yml`)

Runs TypeScript validation, tests, and compiles Android release binaries.

```yaml
name: Mobile CI / Android Build

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci
      - run: npx tsc --noEmit

  build-android:
    needs: lint-and-typecheck
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: 'npm'

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: '17'

      - name: Install Root Dependencies
        run: npm ci

      - name: Decode Keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > android/app/release.keystore
        env:
          ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}

      - name: Build Android Release AAB
        working-directory: android
        run: ./gradlew bundleRelease --no-daemon
        env:
          MYAPP_UPLOAD_STORE_FILE: release.keystore
          MYAPP_UPLOAD_KEY_ALIAS: ${{ secrets.ANDROID_KEY_ALIAS }}
          MYAPP_UPLOAD_STORE_PASSWORD: ${{ secrets.ANDROID_STORE_PASSWORD }}
          MYAPP_UPLOAD_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release.aab
          path: android/app/build/outputs/bundle/release/app-release.aab
```

## ⚙️ Setup & Installation

### Prerequisites

* Node.js >= 18.x
* Yarn or NPM
* Android Studio (with Android SDK platform tools) and/or Xcode for iOS development
* A configured [Firebase Project](https://console.firebase.google.com/) with Authentication, Firestore, Storage, and Cloud Functions enabled

### Required GitHub Secrets for CI/CD

Configure the following secrets under **Repository Settings > Secrets and variables > Actions**:

| Secret Name | Description |
| :--- | :--- |
| `FIREBASE_SERVICE_ACCOUNT_KEY` | Service account JSON key generated from GCP IAM for Firebase deployment |
| `ANDROID_KEYSTORE_BASE64` | Base64-encoded release `.keystore` or `.jks` file |
| `ANDROID_KEY_ALIAS` | Key alias used when generating the signing key |
| `ANDROID_STORE_PASSWORD` | Password for the keystore store |
| `ANDROID_KEY_PASSWORD` | Password for the specific key alias |

### 1. Clone & Install Client Dependencies

```bash
git clone https://github.com/your-username/textlens-ai.git
cd textlens-ai
npm install
```

### 2. Native Dependencies Setup

* **iOS:**

  ```bash
  cd ios && pod install && cd ..
  ```

  Ensure `NSCameraUsageDescription` and `NSPhotoLibraryUsageDescription` exist in `ios/TextLensAI/Info.plist`.

* **Android:**
  Add `android.permission.CAMERA` and storage permissions to `android/app/src/main/AndroidManifest.xml`.

### 3. Backend (Functions) Setup

```bash
cd functions
npm install
firebase functions:config:set gemini.key="YOUR_GEMINI_API_KEY"
npm run deploy
```

### 4. Running the Application

```bash
# Start the Metro bundler
npm start

# In a separate terminal run Android or iOS
npm run android
# or
npm run ios
```
