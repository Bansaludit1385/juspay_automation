# EaseMyTrip Payment Workflow Automation

Python test automation for EaseMyTrip flight checkout on desktop web (Playwright) and Android (Appium). The project uses Pytest and page objects to keep browser/device actions separate from test cases.

---

## 1. Project Overview & Tech Stack

The web flows exercise the live EaseMyTrip booking and payment journey:
`Search Flights -> Select Flight & Fare -> Guest Contact & Traveller Details -> Skip Upsells/Seat -> SafePay Payment Page -> Select Payment Method (UPI / Card / Netbanking) -> Reach Second-Factor Authentication (2FA / Bank Login / UPI QR / OTP Page)`.

These are live-site tests, not a local payment sandbox. The default card flow does not submit card details or charge a card. UPI can display a payment QR and Netbanking can open a bank portal; do not approve a payment or enter bank credentials unless you intend to make the transaction and are authorized. `MANUAL_OTP=true` pauses for manual interaction and may allow a payment to complete.

### Tech Stack
- **Language:** Python 3.10+
- **Web Automation:** Playwright (`playwright==1.65.2`)
- **Mobile Automation:** Appium Python Client (`Appium-Python-Client==6.1.8`) + `UiAutomator2`
- **Test Runner:** Pytest (`pytest==9.2.0`)
- **Reporting:** `pytest-html` (self-contained HTML report)
- **Video Recording:** Playwright native `.webm` recording (Web) + `adb shell screenrecord` `.mp4` (Android)
- **Configuration:** `python-dotenv` (`.env` / `.env.example`)
- **Design Pattern:** Page Object Model (POM) with explicit waits only (no `time.sleep`), retry decorators, and randomized human-like pacing

---

## 2. Folder Structure

```text
payment-automation/
├── web/
│   ├── pages/
│   │   ├── base_page.py           # Playwright explicit waits, human pacing, screenshots
│   │   ├── home_search_page.py    # EaseMyTrip flight search
│   │   ├── results_page.py        # Flight & fare drawer selection
│   │   ├── checkout_page.py       # Guest contact, traveller details, upsell/seat skips
│   │   ├── payment_page.py        # SafePay UPI, Credit/Debit Card, Netbanking
│   │   └── otp_page.py            # 2FA detection & optional MANUAL_OTP pause
│   └── tests/
│       ├── test_upi_payment.py
│       ├── test_card_payment.py
│       └── test_netbanking_payment.py
├── mobile/
│   ├── pages/
│   │   ├── base_mobile_page.py    # Appium WebDriverWait, UiScrollable, WebView switching
│   │   ├── home_search_page.py
│   │   ├── checkout_page.py
│   │   ├── payment_page.py
│   │   └── otp_page.py
│   └── tests/
│       ├── test_mobile_upi.py
│       ├── test_mobile_card.py
│       └── test_mobile_netbanking.py
├── utils/
│   ├── logger.py                  # Console + /reports/logs/execution.log
│   ├── helpers.py                 # Retries, delays, screenshots, video saver, masking
│   └── adb_recorder.py            # ADB screenrecord start/stop/pull manager
├── config/
│   └── settings.py                # Typed dotenv settings loader
├── reports/
│   ├── screenshots/               # Stage & failure PNG screenshots
│   ├── videos/                    # Web (.webm) & Mobile (.mp4) recordings
│   └── test_report.html           # Generated Pytest HTML report
├── conftest.py                    # Web, Mobile, video, screenshot & cleanup fixtures
├── pytest.ini
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

---

## 3. Prerequisites

### Common
- **Python:** 3.10 or newer (`python3 --version`)
- **Git:** (`git --version`)
- **Internet access:** Required to install dependencies and use the live EaseMyTrip website.

### For Web Automation
- A desktop browser is not required; Playwright installs its own Chromium browser.

### For Android Mobile Automation
- **Node.js & npm:** v18+ (`node -v && npm -v`)
- **Java JDK:** 11 or 17 (`java -version`, `JAVA_HOME` configured)
- **Android Studio / Android SDK Platform Tools (`adb`, `emulator`):**
  - `ANDROID_HOME` (or `ANDROID_SDK_ROOT`) set in shell profile
  - `$ANDROID_HOME/platform-tools` and `$ANDROID_HOME/emulator` added to `PATH`
- **Appium 2.x + UiAutomator2 Driver:**
  ```bash
  npm install -g appium
  appium driver install uiautomator2
  ```
- Either a **physical Android device** with USB Debugging enabled (`adb devices`) OR an **Android Emulator AVD** created in Android Studio Device Manager.

---

## 4. Step-by-Step Install & Setup on a Fresh Machine

Clone the repository and run the commands from the cloned project directory (the one containing `requirements.txt`):

```bash
git clone https://github.com/Bansaludit1385/juspay_automation.git
cd juspay_automation
```

### macOS / Linux
```bash
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m playwright install chromium
cp .env.example .env

# Run the three live web payment flows
.venv/bin/python -m pytest web/tests
```

### Windows (PowerShell)
```powershell
py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m playwright install chromium
copy .env.example .env

# Run the three live web payment flows
.\.venv\Scripts\python.exe -m pytest web/tests
```

The setup above installs both web and mobile Python dependencies. Android tests additionally require the Android SDK, an emulator or connected device, and a running Appium server as described below.

If you already activated the virtual environment, you can use `python -m pytest ...` instead of the full `.venv` Python path in the commands below. On Linux, if Chromium reports missing system libraries, install them with `.venv/bin/python -m playwright install --with-deps chromium` (this may require administrator privileges).

---

## 5. Exact Commands to Start Android Emulator & Appium Server

### 1. List and Start Android Emulator (or Connect Physical Phone via USB)
```bash
# List available AVDs
emulator -list-avds

# Start an AVD (replace <AVD_NAME> with your emulator name, e.g. Pixel_7_API_34)
emulator -avd <AVD_NAME> -no-snapshot-load &

# Verify device/emulator is online
adb devices
```

### 2. Start Appium Server
In a dedicated terminal window:
```bash
appium --address 127.0.0.1 --port 4723 --allow-insecure=uiautomator2:chromedriver_autodownload
```
*(The `--allow-insecure=uiautomator2:chromedriver_autodownload` flag allows Appium v3/v4 + UiAutomator2 to handle WebView payment gateways automatically.)*

### 3. Configure `.env` for Android
Update `.env` with your device UDID from `adb devices` and either `ANDROID_APP_PATH` (path to `.apk`) or `ANDROID_APP_PACKAGE` + `ANDROID_APP_ACTIVITY` for the installed EaseMyTrip app.

The repository does not include an Android APK. Install the authorized EaseMyTrip app on your device/emulator first, or set `ANDROID_APP_PATH` to an APK you are permitted to use. Check `adb devices` before starting the mobile tests.

---

## 6. How to Run Tests

### Run All Web Tests (Playwright)
```bash
# macOS / Linux: headed mode is enabled by default in .env.example
.venv/bin/python -m pytest web/tests

# Windows PowerShell
.\.venv\Scripts\python.exe -m pytest web/tests
```

The web tests use the live EaseMyTrip site, so flight availability, network conditions, CAPTCHA/anti-bot checks, and site changes can affect results. By default, card submission and manual OTP are disabled. The UPI flow can display a payment QR and the Netbanking flow reaches the bank portal; do not approve a payment or enter payment credentials unless you intend to do so and are authorized. Use the site and automation only in accordance with its terms.

### Run All Mobile Tests (Appium + Android)
*(Requires running Appium server on `127.0.0.1:4723` and connected Android device/emulator.)*
```bash
# macOS / Linux
.venv/bin/python -m pytest mobile/tests

# Windows PowerShell
.\.venv\Scripts\python.exe -m pytest mobile/tests
```

### Run a Single Payment Method Test
```bash
# Web
.venv/bin/python -m pytest web/tests/test_upi_payment.py
.venv/bin/python -m pytest web/tests/test_card_payment.py
.venv/bin/python -m pytest web/tests/test_netbanking_payment.py

# Or by marker across Web/Mobile (macOS / Linux)
.venv/bin/python -m pytest -m upi
.venv/bin/python -m pytest -m card
.venv/bin/python -m pytest -m netbanking

# Windows PowerShell
.\.venv\Scripts\python.exe -m pytest -m upi
.\.venv\Scripts\python.exe -m pytest -m card
.\.venv\Scripts\python.exe -m pytest -m netbanking
```

---

## 7. How to Enable Manual OTP Mode & Live Card Testing

Never hardcode real card numbers or OTPs in code. Edit `.env`:

```ini
# Pause at the second-factor / OTP / Bank Login page and wait for you to enter OTP manually
MANUAL_OTP=true
MANUAL_OTP_TIMEOUT_SEC=120

# Optional: Only enable when testing with your own authorized card details
SUBMIT_CARD_PAYMENT=false
CARD_NUMBER=
CARD_EXPIRY=
CARD_CVV=
CARD_HOLDER_NAME=Aarav Sharma
```

When `MANUAL_OTP=true`, run in headed mode (`HEADLESS=false`) so you can type the OTP directly into the browser/device:
```bash
# macOS / Linux
HEADLESS=false MANUAL_OTP=true .venv/bin/python -m pytest web/tests/test_netbanking_payment.py

# Windows PowerShell
$env:HEADLESS="false"
$env:MANUAL_OTP="true"
.\.venv\Scripts\python.exe -m pytest web/tests/test_netbanking_payment.py
```

---

## 8. Where to Find Reports, Screenshots & Videos

- **HTML Test Report:** `reports/test_report.html` (`open reports/test_report.html` on macOS)
- **Screenshots (every stage + failures):** `reports/screenshots/*.png`
- **Web Video Recordings (`.webm`):** `reports/videos/web_*.webm`
- **Mobile Video Recordings (`.mp4`):** `reports/videos/mobile_*.mp4`
- **Step-by-Step Execution Logs:** `reports/logs/execution.log`

Pytest creates the reports and evidence directories during a run. Screenshots, videos, and logs are local test output and are ignored by Git; the HTML report is refreshed on each test run.

---

## 9. Test Scope

The web test files cover card form readiness, UPI payment initiation, and Netbanking redirection. The Android test files cover the equivalent payment-method flows on a configured device. Because these flows depend on live services and local device setup, results can vary; run the tests in your own environment to verify them.

---

## 10. Troubleshooting

- **`ERR_CERT_AUTHORITY_INVALID` on HTTPS sites:** Check system date/time. `conftest.py` sets `ignore_https_errors=True` for Playwright test contexts.
- **EaseMyTrip shows `"Fare Not Available!"` on same-day search:** `HomeSearchPage.search_default_flights()` automatically advances the flight listing query to `+5 days` to avoid sold-out same-day airline reconfirmation failures.
- **Playwright `BrowserType.launch` missing binary:** Run `.venv/bin/python -m playwright install chromium` (macOS/Linux) or `.\.venv\Scripts\python.exe -m playwright install chromium` (Windows PowerShell) from the project directory.
- **Tests time out or cannot reach checkout:** Confirm internet access and try again later. Flight availability and live-site changes are outside the test suite's control.
- **Unexpected test failure:** Review the terminal output, `reports/test_report.html`, `reports/logs/execution.log`, and the latest screenshot in `reports/screenshots/`.
- **Appium `Could not find a connected Android device`:** Check `adb devices` returns `device` (not `unauthorized` or empty), and ensure `ANDROID_UDID` in `.env` matches `adb devices`.
- **Mobile locators differ on newer EaseMyTrip APK versions:** Inspect the screen using Appium Inspector and update resource IDs in `mobile/pages/` (text-based `UiSelector().textContains(...)` fallbacks are already included).
- **`adb screenrecord` fails on emulator:** Some x86/arm emulators restrict hardware H.264 encoding; lower `ADB_SCREENRECORD_BITRATE=2000000` in `.env` or set `RECORD_MOBILE_VIDEO=false`.
# juspay_automation
# juspay_automation
