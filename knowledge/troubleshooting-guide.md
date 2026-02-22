# IT Troubleshooting Guide

## Knowledge Base for IT Troubleshooter Agent

---

### Agent Behavior Model

This agent operates as a **fully generative Connected Agent** with no predefined Topics. It uses the decision trees and diagnostic procedures below as its knowledge foundation, but conducts a natural, conversational troubleshooting session with the user.

**Conversation approach:**
- Walk the user through diagnostic steps ONE AT A TIME (do not dump all steps at once)
- After each step, ask: "Did that help?" or "What happened when you tried that?"
- Use Adaptive Cards (Troubleshooting Step card) when presenting steps with images or multiple actions
- In Microsoft Teams, Adaptive Card actions must use `msteams.messageBack` with empty `text`
  to avoid duplicate replies (process `value.action` and ignore plain text)
**Teams rule (mandatory):** Avoid duplicate replies. Never send both an agent response and a separate summary for the same step.
- Collect environment details naturally during conversation (device type, OS, error messages)
- If the issue is resolved, confirm with the user and continue naturally (no system summary in chat)
- If a handoff is needed (for example Troubleshooter → Ticket Manager), compile a DIAGNOSTIC SUMMARY for transfer context

**Context received from Orchestrator:** User's initial description of the problem
**Context returned to Orchestrator:** DIAGNOSTIC SUMMARY only when a handoff/escalation is required (see Section 9)

---

## 1. Computer / Laptop Issues

### 1.1 Computer Won't Turn On
**Decision Tree:**
1. **Is the power cable plugged in and the outlet working?**
   - No → Plug in the power cable. Try a different outlet. Try a different cable if available.
   - Yes → Continue
2. **Is it a laptop?**
   - Yes → Remove the laptop from a docking station (if any). Hold the power button for 15 seconds. Plug in the charger directly. Try powering on.
   - No (Desktop) → Check that the power supply switch on the back is ON (I). Try a different power cable.
3. **Do any lights come on when pressing the power button?**
   - No lights at all → Likely a hardware issue (power supply, motherboard). **Create a ticket — requires on-site visit.**
   - Lights come on but no display → See Section 1.2 (No Display).
   - Beeps heard → Note the beep pattern. **Create a ticket** with the beep pattern.

### 1.2 No Display / Black Screen
**Decision Tree:**
1. **External monitor?**
   - Check cable connections (HDMI, DisplayPort, USB-C)
   - Try a different cable
   - Try a different port on the laptop/PC
   - Press `Win + P` and select "Duplicate" or "Extend"
2. **Laptop built-in screen?**
   - Press `Fn + F7/F8` (display toggle key — varies by manufacturer)
   - Connect an external monitor to check if the GPU works
   - Brightness keys → make sure brightness isn't at zero
3. **Still no display?**
   - Force shutdown (hold power 10 sec), wait 30 sec, power on
   - If still nothing → **Create ticket — hardware issue**

### 1.3 Blue Screen of Death (BSOD)
**Decision Tree:**
1. **Note the error code** shown on the blue screen (e.g., IRQL_NOT_LESS_OR_EQUAL, CRITICAL_PROCESS_DIED)
2. **Was anything recently installed or updated?**
   - Yes → Boot into Safe Mode (hold Shift while clicking Restart → Troubleshoot → Advanced → Startup Settings → Safe Mode). Uninstall the recent change.
   - No → Continue
3. **Is it repeating?**
   - First time → Restart normally. Monitor for recurrence. No action needed if stable.
   - Recurring → Run Windows Memory Diagnostic: `Win + R` → type `mdsched.exe` → Restart Now
   - Check disk: Open CMD as admin → `chkdsk C: /f /r` → Restart
4. **If BSOD persists after above steps** → **Create ticket with error code and frequency**

### 1.4 Slow Performance
**Decision Tree:**
1. **Check Task Manager** (Ctrl + Shift + Esc):
   - CPU > 90%? → Identify the process. If it's a known app, restart it. If unknown, note the name.
   - Memory > 90%? → Close unnecessary applications. Check for memory leaks.
   - Disk at 100%? → Check for Windows Update running, antivirus scanning, or disk indexing. Wait or reschedule these tasks.
2. **Restart the computer** if it hasn't been restarted in 48+ hours
3. **Check available disk space:**
   - Less than 10 GB free? → Run Disk Cleanup: `Win + R` → `cleanmgr`
   - Empty Recycle Bin and Downloads folder
   - Move large files to OneDrive or network drive
4. **Check startup programs:**
   - Task Manager → Startup tab → Disable unnecessary items
5. **Run Windows Update** → Settings → Update & Security → Check for updates
6. **Still slow?** → **Create ticket — may need hardware upgrade or reimaging**

### 1.5 Overheating / Fan Noise
1. **Check ventilation** — ensure vents are not blocked
2. **Place laptop on hard, flat surface** (not blankets, pillows, or carpet)
3. **Check Task Manager** for high CPU processes
4. **Restart the computer**
5. **If persistent** → Likely needs internal cleaning or thermal paste replacement. **Create ticket for on-site service.**

### 1.6 Frozen / Unresponsive Computer
1. **Wait 2-3 minutes** — it may be processing a heavy task
2. **Try Ctrl + Alt + Delete** → Task Manager → End unresponsive application
3. **If mouse and keyboard don't respond** → Hold power button for 10 seconds to force shutdown
4. **After restart, if it freezes again immediately** → Boot into Safe Mode (hold Shift + Restart)
5. **Recurring freezes** → **Create ticket with details** (when it freezes, what you were doing)

---

## 2. Network / Connectivity Issues

### 2.1 WiFi Not Connecting
**Decision Tree:**
1. **Is WiFi enabled?**
   - Check the WiFi toggle: click the WiFi icon in the taskbar
   - Check for airplane mode: `Win + A` → verify Airplane Mode is OFF
   - Try the hardware WiFi switch (some laptops have a physical switch)
2. **Can you see the corporate WiFi network?**
   - No → Restart the WiFi adapter: Settings → Network → WiFi → turn OFF, wait 10 sec, turn ON
   - Yes but can't connect → Forget the network and reconnect. Your credentials may need refreshing.
3. **Connected but no internet?**
   - Open CMD → run `ipconfig /release` then `ipconfig /renew`
   - Run `nslookup google.com` — if it fails, DNS issue
   - Try `ipconfig /flushdns`
4. **Run Network Troubleshooter:** Settings → Network & Internet → Status → Network troubleshooter
5. **Still not working?** → Try connecting to a different WiFi network (e.g., Guest WiFi) to isolate the issue
6. **If no WiFi networks visible at all** → Device manager → Network adapters → Right-click WiFi adapter → Update/Enable. If "!" icon → **Create ticket — driver issue**

### 2.2 VPN Issues
**Decision Tree:**
1. **Is internet working?** (test by opening a website)
   - No → Fix internet first (see 2.1 or 2.3)
   - Yes → Continue
2. **VPN client opens but won't connect?**
   - Close and reopen the VPN client
   - Check your credentials — password recently changed?
   - Check if MFA prompt appears — complete it
   - Try a different VPN server/gateway if available
3. **VPN connects but can't access internal resources?**
   - Try `ping` to internal server by IP address
   - If ping works but name doesn't → DNS issue: `ipconfig /flushdns`
   - Check split tunnel settings
4. **VPN connection drops frequently?**
   - Switch to a wired connection if possible
   - Change WiFi band (2.4 GHz vs 5 GHz)
   - Check for power-saving WiFi: Device Manager → WiFi adapter → Properties → Power Management → uncheck "Allow to turn off to save power"
5. **Error messages?** → Note the exact error code/message. **Create ticket with error details.**

### 2.3 Wired Connection Not Working
1. **Check cable** — is it firmly plugged in at both ends?
2. **Try a different Ethernet cable**
3. **Try a different port** on the wall/switch
4. **Check network icon** in taskbar for "Unidentified Network" or "No Internet"
5. **Run:** `ipconfig /release` → `ipconfig /renew`
6. **Device Manager** → Network adapters → check for errors on Ethernet adapter
7. **If using a USB-to-Ethernet dongle** → try a different USB port, try reinstalling the driver
8. **Still not working?** → **Create ticket — may need switch port activation**

### 2.4 Slow Internet
1. **Speed test:** Navigate to speedtest.net or fast.com
   - Expected: >50 Mbps download on wired, >20 Mbps on WiFi
   - Below expected → Continue troubleshooting
2. **Close bandwidth-heavy applications** (streaming, large downloads)
3. **Large file transfers?** → Consider scheduling them for off-hours
4. **WiFi?** → Move closer to the access point. Avoid obstacles/interference.
5. **Multiple devices affected?** → Likely a network infrastructure issue. **Create ticket for network team.**

---

## 3. Email / Outlook Issues

### 3.1 Outlook Won't Open / Crashes
**Decision Tree:**
1. **Start Outlook in Safe Mode:** `Win + R` → type `outlook.exe /safe` → Enter
   - Opens in Safe Mode? → Likely an add-in conflict. Go to File → Options → Add-ins → Manage COM Add-ins → Disable all → Restart Outlook normally → Re-enable one at a time to find the culprit
   - Doesn't open in Safe Mode → Continue
2. **Repair Office:** Control Panel → Programs → Microsoft 365 → Change → Quick Repair
   - If Quick Repair fails → try Online Repair
3. **Create new Outlook profile:**
   - Control Panel → Mail → Show Profiles → Add → set up new profile
   - If new profile works → data migration may be needed
4. **Check for updates:** File → Office Account → Update Options → Update Now
5. **Still crashing?** → **Create ticket with error details and crash timing**

### 3.2 Can't Send or Receive Email
1. **Check internet connection** (load a website)
2. **Check Outlook status bar** at the bottom — does it say "Connected" or "Disconnected"?
   - Disconnected → Click "Connected to: Microsoft Exchange" → try reconnecting
3. **Check Outbox** — is the email stuck there?
   - Yes → Open the stuck email, close it, try Send/Receive (F9)
   - If stuck persistently → Move it to Drafts, delete from Outbox, resend
4. **Check email size** — attachment too large? (Limit: 25 MB for external, 150 MB for internal)
   - Too large → Use OneDrive sharing link instead
5. **Check mailbox size:** File → Account Settings → check mailbox quota
   - Near limit → Archive old emails, empty Deleted Items and Junk
6. **Run Send/Receive manually:** Send/Receive tab → Send/Receive All Folders (F9)
7. **Still failing?** → Try Outlook Web App (outlook.office.com) to confirm if the issue is local or server-side

### 3.3 Mailbox Full / Storage Issues
1. **Check mailbox size:** File → Info → Mailbox Settings → Storage
   - Standard limit: 50 GB (E3) / 100 GB (E5)
2. **Quick cleanup actions:**
   - Empty Deleted Items folder
   - Empty Junk Email folder
   - Sort by size: search `size:>10MB` to find large emails
   - Use Mailbox Cleanup: File → Info → Cleanup Tools → Mailbox Cleanup
3. **Enable Online Archive** (if available): Settings → Forwarding and IMAP → enable archive
4. **Setup auto-archive rules:** File → Options → Advanced → AutoArchive Settings

### 3.4 Calendar / Meeting Issues
1. **Can't see others' calendars?**
   - They need to share their calendar with you: Calendar → Properties → Permissions
   - For room calendars → check with admin
2. **Meeting room booking issues?**
   - Double-check room availability
   - Some rooms require approval → wait for confirmation
   - If room always shows busy → **Create ticket** for room mailbox check
3. **Time zone issues?**
   - File → Options → Calendar → Time zones → verify correct zone

---

## 4. Printer Issues

### 4.1 Printer Offline / Not Printing
**Decision Tree:**
1. **Is the printer powered on?** Check display panel / lights
2. **Is it connected?** 
   - Network printer: check network cable / WiFi
   - USB printer: check USB cable
3. **On your computer:**
   - Settings → Printers & Scanners → find the printer
   - Right-click → "See what's printing" → Cancel all stuck jobs
   - Right-click printer → "Set as default"
4. **Restart print spooler:**
   - `Win + R` → `services.msc` → find "Print Spooler" → Restart
5. **Remove and re-add the printer:**
   - Settings → Printers & Scanners → Remove the printer → Add again
6. **Network printer still offline?** → Try printing from another computer to isolate
7. **Hardware issues (paper jam, toner, etc.)?** → Check printer display for error messages

### 4.2 Print Quality Issues
1. **Streaky/faded prints?** → May need new toner/ink cartridge
2. **Run cleaning cycle** from the printer's built-in menu
3. **Print a test page** from the printer itself (not from computer)
4. **Check paper type** — use correct paper for the printer
5. **For persistent quality issues** → **Create ticket for printer maintenance**

### 4.3 Printer Driver Issues
1. **Check if driver is installed:** Settings → Printers → select printer → Print properties
2. **Update driver:** Device Manager → Print queues → right-click → Update driver
3. **Download latest driver** from manufacturer's website or company software portal
4. **Driver conflicts?** → Remove all printer drivers, restart, install fresh

---

## 5. Microsoft Teams Issues

### 5.1 Audio / Microphone Issues
1. **Check Teams audio settings:** Click your profile → Settings → Devices
   - Verify correct Speaker and Microphone are selected
   - Use "Make a test call" to verify
2. **Check Windows audio settings:** Right-click speaker icon → Sound Settings
   - Check Input device is correct
   - Check App volume settings → Teams volume
3. **Is the microphone muted?** Check both:
   - Teams mute button (microphone icon)
   - Physical mute button on headset
4. **Close other apps** that might be using the microphone (Zoom, Webex, etc.)
5. **Bluetooth headset?** → Try removing and re-pairing. Switch to USB if possible.
6. **Try in the browser version** (teams.microsoft.com) to check if it's an app issue

### 5.2 Camera / Video Issues
1. **Is the camera enabled in Teams?** Check video button during a call
2. **Privacy slider/cover** → physically check the camera cover isn't closed
3. **Windows camera permissions:** Settings → Privacy → Camera → Allow apps to access camera → Teams = ON
4. **Another app using camera?** → Close Zoom, Webex, or other video apps
5. **Update Teams:** Click profile → Check for updates
6. **External webcam?** → Try a different USB port. Check Device Manager for errors.

### 5.3 Screen Sharing Not Working
1. **Presenter role needed?** → Ask the meeting organizer to make you a presenter
2. **GPU rendering issue:** Settings → General → disable "GPU hardware acceleration" → Restart Teams
3. **Multiple monitors?** → Make sure you're sharing the correct screen
4. **VPN?** → High bandwidth required. Try lowering screen share quality.

### 5.4 Teams Not Loading / Slow
1. **Clear Teams cache:**
   - Close Teams completely (check system tray)
   - `Win + R` → `%appdata%\Microsoft\Teams` → Delete all contents
   - Restart Teams
2. **Check for updates:** Profile → Check for updates
3. **New Teams vs Classic Teams:** If on old version, upgrade to "New Teams"
4. **Restart computer**
5. **Try web version** (teams.microsoft.com) as alternative

---

## 6. Login / Authentication Issues

### 6.1 Can't Log In
**Decision Tree:**
1. **What error message do you see?**
   - "Wrong password" → Caps Lock off? Try typing password in a text editor first to verify. If truly forgotten → See password reset below.
   - "Account locked" → Too many failed attempts. Wait 30 minutes for auto-unlock, or contact IT.
   - "Account disabled" → **Create urgent ticket** — may require admin intervention.
   - "Password expired" → Follow the password change prompt. If no prompt, use https://myaccount.microsoft.com
2. **Can you log into other services?** (e.g., Outlook Web, Teams Web)
   - Yes → Local machine issue. Try restarting. Check cached credentials.
   - No → Account-level issue. Password may be expired.

### 6.2 Password Reset
1. **Self-service:** Go to https://passwordreset.microsoftonline.com/
   - Must have SSPR set up previously with MFA method
   - Follow verification steps (phone/email/authenticator)
2. **Can't use self-service?** → **Create ticket** — admin will reset and provide temporary password
3. **After reset:** You'll be prompted to create a new password
   - Must meet policy: 12+ characters, mix of upper/lower/numbers/symbols
   - Cannot reuse last 12 passwords

### 6.3 MFA Issues
1. **Not receiving MFA prompt?**
   - Check phone isn't on Do Not Disturb / Silent mode
   - Check Microsoft Authenticator app is installed and notifications enabled
   - Try "I can't use my app right now" for alternative method (SMS, phone call)
2. **New phone?**
   - Go to https://myaccount.microsoft.com → Security info → Add method → Authenticator app
   - If you can't log in at all → **Create ticket** for MFA reset
3. **MFA keeps asking every time?**
   - Check "Don't ask again for 30 days" option (if available)
   - Clear browser cookies for microsoft.com
   - Make sure device is marked as "Compliant" in Intune

---

## 7. Mobile Device Issues

### 7.1 Company Email on Phone
1. **iPhone/iOS:**
   - Settings → Accounts → Add Account → Microsoft Exchange
   - Use your company email address
   - May prompt for Intune Company Portal installation → Install and enroll
2. **Android:**
   - Install Outlook for Android from Play Store
   - Sign in with company email
   - May prompt for Intune Company Portal → Install and enroll
3. **Not receiving notifications?**
   - Check phone notification settings for Outlook
   - Check Focus/Do Not Disturb modes
   - Check Outlook in-app notification settings

### 7.2 MDM / Intune Issues
1. **Company Portal not enrolling?**
   - Ensure latest version of Company Portal app
   - Check OS version meets minimum requirements (iOS 16+, Android 12+)
   - Restart phone and try again
2. **Compliance issues?**
   - Check Company Portal → Device Status for non-compliance reasons
   - Common: OS outdated, no PIN set, jailbroken/rooted device
3. **Can't install company apps?** → Check Company Portal app for available apps. If app not listed → **Create ticket**.

---

## 8. Troubleshooting Quick Reference Card

| Symptom | First Step | Second Step | Escalate If |
|---------|-----------|------------|-------------|
| Won't turn on | Check power | Force reset (15 sec) | No lights at all |
| BSOD | Note error code | Safe Mode | Recurring 3+ times |
| Slow PC | Task Manager check | Restart | Persists after restart |
| No WiFi | Toggle WiFi off/on | Forget & reconnect | No networks visible |
| VPN down | Check internet first | Restart VPN client | Error message |
| Outlook crash | Safe Mode start | Repair Office | Can't open in Safe Mode |
| Can't print | Restart spooler | Remove/re-add printer | Hardware issue |
| Teams audio | Check device settings | Test call | Hardware fault |
| Can't login | Check caps lock | Self-service reset | Account disabled |
| Phone email | Reinstall Outlook | Re-enroll Intune | Company Portal fails |

---

## 9. Information to Collect for Ticket Escalation

When creating a ticket after troubleshooting, always include:

1. **Device info:** Laptop model, asset tag (sticker on bottom), OS version
2. **Description:** What the user was trying to do
3. **Error messages:** Exact wording, screenshots if possible
4. **Steps already tried:** Everything attempted during troubleshooting
5. **When it started:** Date/time, any changes around that time
6. **Frequency:** One-time, intermittent, or constant
7. **Impact:** How many people affected, business impact
8. **Workaround:** Is there a temporary workaround available?

### DIAGNOSTIC SUMMARY Format

Only when a cross-agent handoff is needed (especially Troubleshoot → Ticket), compile the following structured summary and return it to the Orchestrator.

Do NOT print this template in user-visible chat.
Do NOT output strings such as "===== DIAGNOSTIC SUMMARY =====" or "===== END SUMMARY =====" to the user.
Use natural language in chat; keep this format internal for handoff metadata only.

```
===== DIAGNOSTIC SUMMARY =====
Issue: {one-line description of the problem}
Category: {Computer|Network|Email|Printer|Teams|Login|Mobile}
Severity: {P1-Critical|P2-High|P3-Medium|P4-Low}
Resolved: {Yes|No|Partial}

Environment:
- Device: {laptop model / asset tag}
- OS: {Windows 11 23H2 / macOS 14.x / etc.}
- Application: {Outlook, Teams, VPN client, etc. — if applicable}

Steps Attempted & Results:
1. {Step description} → {Result: resolved / not resolved / partial}
2. {Step description} → {Result}
3. {Step description} → {Result}
...

Observations:
- {Error messages noted}
- {Patterns observed (e.g., "occurs every morning", "after Windows Update")}
- {Other relevant details}

Recommendation: {Resolved — no further action | Create ticket — requires on-site visit | Create ticket — needs admin/specialist | Route to Access Provisioner — needs password reset}
===== END SUMMARY =====
```

**Important:** The DIAGNOSTIC SUMMARY ensures that if the user is handed off to the Ticket Manager, the ticket can be auto-populated with all troubleshooting details. The user should NOT be asked again for information already captured during the troubleshooting session.

**Display rule:** This summary is internal handoff metadata. Do NOT display raw summary blocks to the user in normal chat.
