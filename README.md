# Shadow Agent — Windows Installer

## Prerequisites

- Windows 10 / 11 or Windows Server 2016+
- Administrator account
- An enrollment key from the Shadow Manager dashboard (`ENR-XXXX-XXXX-XXXX`)

---

## Installing Shadow Agent

### Step 1 — Run the installer

Right-click **`ShadowAgent-Setup-x.x.x.exe`** and choose **"Run as administrator"**.

### Step 2 — Enter enrollment details

The wizard will ask for:

| Field | Value |
|---|---|
| **Enrollment Key** | Your one-time key from the Shadow Manager dashboard (format: `ENR-XXXX-XXXX-XXXX`). Leave blank to skip enrollment and run it manually later. |
| **Manager URL** | URL of your Shadow Manager server (e.g. `https://shadow.example.com`) |

### Step 3 — Complete the wizard

Click **Install** on the summary page. The installer will:

1. Copy files to `C:\shadow`
2. Install Python packages (may take a few minutes)
3. Register two scheduled tasks:
   - **ShadowAgent** — starts at system boot, runs as SYSTEM
   - **ShadowCapture** — starts at user logon, captures screen and input
4. Enroll the device with the Shadow Manager and store credentials in `C:\shadow\creds`
5. Write Group Policy registry keys for the browser extension (Chrome and Edge)

### Step 4 — Verify

Open **Task Scheduler** and confirm both `ShadowAgent` and `ShadowCapture` are present and running. The agent log is at `C:\shadow\logs\agent.log`.

### Enrolling manually (if you skipped enrollment)

Open PowerShell as Administrator and run:

```powershell
cd C:\shadow\src
C:\shadow\python\python.exe install.py --key ENR-XXXX-XXXX-XXXX
```

Then start the tasks:

```powershell
Start-ScheduledTask ShadowAgent
Start-ScheduledTask ShadowCapture
```

---

## Installing the Browser Extension

The Shadow Agent browser extension captures browser activity (URLs, clicks, form interactions) and forwards events to the local agent. It must be installed in Chrome or Edge on the specialist's machine.

### Important: managed vs unmanaged machines

Force-install via Group Policy (written automatically by the installer) only works on **managed machines** — i.e. devices that are Azure AD joined, Active Directory domain joined, Intune enrolled, or (for Chrome) enrolled in Chrome Enterprise Core. On unmanaged machines the browser blocks self-hosted extensions from Group Policy, and manual installation is required.

---

### Option A — Automatic (managed machines only)

The installer already wrote the Group Policy registry keys for both browsers. No further action needed — Chrome and Edge will silently install the extension on the next browser restart.

To verify the policy was applied:
- **Chrome:** go to `chrome://policy` and look for `ExtensionInstallForcelist`
- **Edge:** go to `edge://policy` and look for `ExtensionInstallForcelist`

---

### Option B — Manual install (unmanaged machines)

#### Microsoft Edge

1. Open Edge and navigate to `edge://extensions`
2. Enable **Developer mode** (toggle in the bottom-left corner)
3. In a new tab, go to:
   ```
   [https://shadow.68.220.202.177.nip.io/extension/shadow-agent.zip[
   ```
   The file will download automatically.
4. Click Load Unpacked
5. Select the 'chrome-extension' folder from the unzipped directory

> Edge will show a persistent *"Extensions installed in developer mode"* banner. Click **Keep** to dismiss it. This banner reappears on each browser start on unmanaged machines.

#### Google Chrome

1. Open Chrome and navigate to `chrome://extensions`
2. Enable **Developer mode** (toggle in the top-right corner)
3. In a new tab, go to:
   ```
   [https://shadow.68.220.202.177.nip.io/extension/shadow-agent.zip[
   ```
   The file will download automatically.
4. Click Load Unpacked
5. Select the 'chrome-extension' folder from the unzipped directory

> Same developer mode banner applies. Click **Keep** to dismiss.

---

### Option C — Chrome Enterprise Core (recommended for unmanaged Chrome machines)

[Chrome Enterprise Core](https://chromeenterprise.google/browser/enrollment/) is a free Google service that lets you manage Chrome policies from the Google Admin Console without requiring domain join or MDM. Once Chrome is enrolled:

1. In the Google Admin Console, go to **Devices > Chrome > Apps & Extensions**
2. Add a new extension with the update URL:
   ```
   https://<manager-url>/extension/update.xml
   ```
3. Set the install policy to **Force install**

Chrome will then silently install the extension on all enrolled browsers, with no developer mode warnings.

---

## Uninstalling

Use **Add or Remove Programs** and uninstall **Shadow Agent**. This removes the scheduled tasks and installed files. The `C:\shadow\creds` and `C:\shadow\captures` directories are preserved so that data is not accidentally lost — delete them manually if a full removal is needed.

To also remove the browser extension registry keys:

```powershell
Remove-Item -Path "HKLM:\SOFTWARE\Policies\Google\Chrome\ExtensionInstallForcelist" -Force -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge\ExtensionInstallForcelist" -Force -ErrorAction SilentlyContinue
```
