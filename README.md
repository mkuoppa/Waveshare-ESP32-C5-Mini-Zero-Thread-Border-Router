# Waveshare ESP32-C5 (Mini/Zero) → Thread Border Router for Home Assistant

Building the Waveshare ESP32-C5 into an OpenThread RCP (Radio Co-Processor) dongle with external antenna support, compiled from source on Windows, connected to Home Assistant (HAOS on Raspberry Pi 4) via USB.

---

## Overview

The ESP32-C5 acts as a dumb 802.15.4 radio dongle (**RCP** — Radio Co-Processor) over USB. Home Assistant's **OpenThread Border Router (OTBR)** add-on does all the actual Thread/IPv6 routing — the ESP itself just relays radio packets.

Board specifics:
- **Board:** Waveshare ESP32-C5 Mini/Zero
- **External antenna switch pin:** GP26 (IO26) — set **HIGH** to select external antenna, per Waveshare's own docs
- **Firmware branch:** `v6.0-beta1` of `esp-idf` (contains a USB-RCP connection-stability patch not yet in stable releases)

---

## 1. Install prerequisites

- **Git for Windows** — [git-scm.com](https://git-scm.com/download/win). During install, keep "Git from the command line" selected.
- **Python 3.10+** — from python.org. Check "Add Python to PATH" during install.

Do all following steps in **PowerShell**.

---

## 2. Enable Windows UTF-8 support (do this before building)

ESP-IDF's build tooling requires a UTF-8 system locale. On non-English Windows installs (e.g. Swedish), this isn't on by default and env vars like `$env:LC_ALL` or `$env:PYTHONUTF8` won't fix it — it's a genuine OS-level setting.

1. `Win + R` → type `intl.cpl` → Enter
2. **Administrative** tab → **Change system locale...**
3. Check **"Beta: Use Unicode UTF-8 for worldwide language support"**
4. OK → **restart your computer**

---

## 3. Clone ESP-IDF (the correct branch)

```powershell
mkdir C:\esp
cd C:\esp
git clone -b v6.0-beta1 --recursive https://github.com/espressif/esp-idf.git
```

Keep the path short (`C:\esp`) — long paths break the Windows build.

---

## 4. Install tools for the C5 target

```powershell
cd C:\esp\esp-idf
.\install.bat esp32c5
```

---

## 5. Activate the environment (repeat every new PowerShell window)

```powershell
cd C:\esp\esp-idf
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\export.ps1
```

- `Set-ExecutionPolicy -Scope Process` only relaxes the policy for this one window — safe, temporary, no system-wide change.
- If PowerShell's execution policy is fully locked down and this fails, use `cmd.exe` instead and run `export.bat` there — but then do **all** subsequent commands in that same `cmd.exe` window, since `export.bat`'s environment variables won't propagate back into PowerShell.

---

## 6. Configure the RCP example

```powershell
cd examples\openthread\ot_rcp
idf.py set-target esp32c5
idf.py menuconfig
```

In the menuconfig UI:
`Component config → OpenThread → Thread Core Features → Thread Radio Co-Processor Feature → RCP transport type` → select **USB** → save (`S`) → exit (`Q`).

---

## 7. Add the external-antenna GPIO code

File: `C:\esp\esp-idf\examples\openthread\ot_rcp\main\esp_ot_rcp.c`

Open it directly from your current directory:
```powershell
notepad .\main\esp_ot_rcp.c
```
(or `code .\main\esp_ot_rcp.c` if you have VS Code)

Make **three** changes:

```c
#include "esp_vfs_eventfd.h"
#include "driver/gpio.h"                       // 1. ADD THIS INCLUDE

#if CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE
#include "ot_examples_common.h"
#endif
// ... (unchanged code) ...

#define TAG "ot_esp_rcp"

extern void otAppNcpInit(otInstance *instance);

// 2. ADD THIS WHOLE FUNCTION
static void select_external_antenna(void)
{
    gpio_set_direction(GPIO_NUM_26, GPIO_MODE_OUTPUT);
    gpio_set_level(GPIO_NUM_26, 1);   // HIGH = external antenna (Waveshare C5 Mini/Zero)
}

void app_main(void)
{
    select_external_antenna();          // 3. ADD THIS LINE — first statement in app_main()

    // ... rest of app_main() stays exactly as it was ...
}
```

> Note: your actual `esp_ot_rcp.c` may use `esp_openthread_platform_config_t` (v6.0-beta1) rather than the older `esp_openthread_config_t` seen in some community posts — that's just a version difference and doesn't affect this edit.

---

## 8. (Optional) Apply the USB connection-dropout patch

Some users see the RCP-over-USB link hang after several hours with `otbr-agent` logging `Wait for response timeout`. This is reportedly fixed in `v6.0-beta1`, but the patch can be applied explicitly if needed:

```powershell
curl.exe -L https://gist.githubusercontent.com/rayanamal/f0bdc614ea630e865a8a09e905ffa453/raw/5ac5bfffc67f4a961ea5c830aa9240b6d07f7b4d/connection-drop-fix.patch -o fix.patch
git apply fix.patch
```

Use `curl.exe` explicitly — plain `curl` in PowerShell is aliased to `Invoke-WebRequest`, which doesn't support curl-style flags like `-L`.

If `git apply` fails saying the patch doesn't apply cleanly, it likely means this fix is already included in `v6.0-beta1` — skip the step and continue.

---

## 9. Find your COM port

Plug the C5 into your Windows PC via USB (data cable, not charge-only). Check **Device Manager → Ports (COM & LPT)** for the assigned `COMx`. The C5 has USB built into the chip, so Windows should detect it natively without extra drivers.

---

## 10. Build and flash

```powershell
idf.py -p COM5 build flash
```

(replace `COM5` with your actual port)

---

## 11. Connect to Home Assistant

1. Unplug the C5 from Windows, plug it into a USB port on your Pi 4 (HAOS).
2. Install the **OpenThread Border Router** add-on.
3. In its config:
   - **Device**: select `/dev/serial/by-id/usb-Espressif_USB_JTAG...` — **not** `/dev/ttyAMA...`
   - Disable **Hardware Flow Control**
4. Start the add-on. The OTBR integration should auto-appear in HA.
5. If it asks for a URL instead of auto-configuring, remove the integration and let the add-on re-add it.

---

## 12. Share Thread credentials with your phone

For commissioning Matter-over-Thread devices: open the **Home Assistant Companion app** → Settings → Thread → **Send credentials to phone**.

---

## Troubleshooting quick reference

| Symptom | Fix |
|---|---|
| `Support for Unicode is required...` on `idf.py` commands | Enable Windows Beta UTF-8 setting (Step 2) + reboot — env vars alone won't fix this |
| `export.ps1 cannot be loaded... running scripts is disabled` | `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` before running it |
| `curl -L ... : parameter cannot be found` | Use `curl.exe` explicitly, or switch to `Invoke-WebRequest -Uri ... -OutFile ...` |
| OTBR log shows `Wait for response timeout` / hangs after hours | Apply the connection-drop-fix patch (Step 8), or confirm you're on `v6.0-beta1` |
| Weak range despite external antenna | Confirm GP26 GPIO patch (Step 7) was applied and board was rebuilt/reflashed after |
| OTBR integration asks for a manual URL | Remove integration, let the add-on re-add it automatically |
