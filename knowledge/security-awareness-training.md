# Security Awareness Training

## Knowledge Base for Security Awareness Coach Agent

---

### Agent Behavior Model

This agent operates as a **fully generative Connected Agent** with no predefined Topics. It uses the phishing quizzes, SLAM method, threat encyclopedia, and security tips below as its knowledge foundation, but conducts an interactive, engaging training session with the user.

**Conversation approach:**
- For phishing quizzes: present ONE quiz at a time using an Adaptive Card, wait for the user's answer, then reveal the correct answer with detailed explanation
- For training topics: present content in digestible chunks, ask comprehension questions between sections
- Use the SLAM method (Section 1) as the foundation for phishing recognition training
- Track quiz scores during the session (correct/total) and provide a score rating at the end (Section 9)
- Make training interactive and conversational — not a lecture
- Encourage correct answers, provide constructive feedback on wrong ones
- Suggest the Recommended Training Path (Section 9) for users who want comprehensive training
- Do not repeat the same guidance or send duplicate messages unless the user asks
**Teams rule (mandatory):** The agent runs in Teams chat. Avoid duplicate replies. For Adaptive Card submits, process `value.action` and ignore plain `text`.

**Context received from Orchestrator:**
- User's request (e.g., "quiz me on phishing", "give me security tips")
- Incident context (when routed after a phishing event — focus training on the relevant attack type)

**Context returned to Orchestrator:** TRAINING SUMMARY only when a cross-agent continuation/handoff is needed (see Section 11)

---

## 1. Phishing Recognition — The SLAM Method

**S.L.A.M.** is a simple framework for evaluating suspicious emails:

| Letter | Check | What to Look For |
|--------|-------|-------------------|
| **S** — Sender | Who sent this? | Misspelled domains (micros0ft.com), free email providers (company-support@gmail.com), unusual sender for the context |
| **L** — Links | Where do links go? | Hover (don't click!) to check the real URL. Look for misspellings, extra subdomains (login.company.evil.com), HTTP instead of HTTPS |
| **A** — Attachments | Are attachments expected? | Unexpected attachments are dangerous. Watch for: .exe, .scr, .zip, .docm (macro-enabled), .html files |
| **M** — Message | Does the content make sense? | Urgency pressure, threats, too-good-to-be-true offers, grammatical errors, generic greetings ("Dear Customer"), requests for credentials |

---

## 2. Phishing Quiz Bank

### Quiz 1: The Password Reset
```
📧 Email received:
From: IT-HelpDesk@c0mpany-support.com
Subject: ⚠️ URGENT: Password Expires in 2 Hours
Body:
Dear Employee,

Your corporate password will expire in 2 hours. To avoid losing access 
to all company systems, click the link below immediately:

[Reset Your Password Now]

If you do not reset your password, your account will be locked.

Regards,
IT Support Team
```

**Answer: PHISHING 🎣**
**Red flags:**
1. **Sender:** `c0mpany-support.com` — uses a zero and extra `-support`, not real domain
2. **Urgency:** "2 hours" + "immediately" creates panic
3. **Threat:** "account will be locked" — scare tactic
4. **Generic greeting:** "Dear Employee" — no name
5. **Link:** Generic "Reset Your Password Now" — doesn't show actual URL
6. **Real policy:** IT never sends password reset links via email; you go to the SSPR portal yourself

---

### Quiz 2: The CEO Wire Transfer
```
📧 Email received:
From: john.smith.ceo@company.com
Subject: Confidential — Urgent Wire Transfer Needed
Body:
Hi,

I'm in a meeting and can't call. I need you to process an urgent wire 
transfer of $45,000 to a new vendor. This is time-sensitive and 
confidential — please don't discuss with others until it's done.

I'll send the account details in the next email. Please confirm you 
can handle this right away.

Thanks,
John Smith
CEO
Sent from my iPhone
```

**Answer: PHISHING (Whaling / CEO Fraud) 🎣**
**Red flags:**
1. **Urgency + secrecy:** "urgent" + "confidential" + "don't discuss with others" — classic pressure combo
2. **Unusual request:** CEO bypassing normal procurement/finance processes
3. **No phone verification:** "can't call" prevents you from confirming via phone
4. **Financial request via email:** Wire transfers should NEVER be initiated by email alone
5. **"Sent from my iPhone":** Creates excuse for informal tone and typos
6. **What to do:** ALWAYS verify financial requests by calling the person directly on their known phone number

---

### Quiz 3: The Shared Document
```
📧 Email received:
From: sarah.jones@company.com
Subject: FYI — Q4 Budget Review
Body:
Hi,

Please review the attached Q4 budget spreadsheet and send me your 
comments by Friday.

[Q4_Budget_Review_2025.xlsx]

Thanks,
Sarah
```

**Answer: COULD BE LEGITIMATE ✅ — but verify!**
**Analysis:**
1. **Sender:** Appears to be a real colleague — but check if Sarah exists and if this is expected
2. **Context:** Does Sarah normally send you budget reviews? Is this your role?
3. **Attachment:** .xlsx is relatively safe (but .xlsm with macros would be suspicious)
4. **No urgency or threats:** Normal business tone
5. **What to do:** If expected from Sarah, probably fine. If unexpected, message Sarah on Teams: "Hey, did you just send me a budget file?" — takes 10 seconds, prevents 100% of spoofed emails

---

### Quiz 4: The IT Survey
```
📧 Email received:
From: employee-survey@surveytools-online.net
Subject: Mandatory IT Satisfaction Survey — Complete by Today
Body:
Dear Team Member,

The IT department is conducting a mandatory satisfaction survey. 
Please complete the survey using the link below. You will need to 
enter your employee ID and email password to verify your identity.

[Start Survey]

This survey is mandatory and must be completed by end of day.

Best,
IT Department
```

**Answer: PHISHING 🎣**
**Red flags:**
1. **Sender:** External domain `surveytools-online.net` — not company domain
2. **Requests password:** NO legitimate survey ever asks for your password
3. **Mandatory + deadline:** Pressure tactic
4. **Identity verification via password:** This is credential harvesting
5. **Rule:** Your password is NEVER required for surveys, feedback, or verification

---

### Quiz 5: The Multi-Factor Authentication Prompt
```
📱 You receive a push notification on your phone:
Microsoft Authenticator: "Approve sign-in to Microsoft 365?"
Location: Lagos, Nigeria
Time: 3:15 AM (your local time)

You did NOT try to sign in.
```

**Answer: ATTACK IN PROGRESS — MFA Fatigue/Push Bombing 🚨**
**What to do:**
1. **DENY** the request immediately
2. Select "No, it's not me" in the Authenticator
3. **Change your password** immediately — someone has your credentials
4. Report to IT Security as a P1 incident
5. **NEVER approve MFA prompts you didn't initiate** — even if you get 10 in a row (that's "MFA fatigue" attack)

---

### Quiz 6: The LinkedIn Message
```
💬 LinkedIn message:
From: "Alex Thompson — IT Recruiter at TechCorp"

Hi! I came across your profile and I'm impressed by your experience. 
We have an exciting Senior Developer position with a 40% salary increase. 
Please download and fill out the preliminary application form:

[Application_Form.exe]

Looking forward to hearing from you!
```

**Answer: PHISHING / MALWARE 🎣**
**Red flags:**
1. **File type:** `.exe` — application forms are NEVER .exe files
2. **Too good to be true:** "40% salary increase" — bait
3. **Download required:** Legitimate recruiters use LinkedIn's built-in application or web forms
4. **Rule:** Never download .exe files from messages. Legitimate documents use .pdf or .docx

---

## 3. Social Engineering Tactics Encyclopedia

### Pretexting
**What it is:** The attacker creates a fabricated scenario (pretext) to manipulate you into sharing information or performing an action.

**Examples:**
- "Hi, I'm calling from IT. We detected unusual activity on your account. I need your password to investigate."
- "This is Mike from Finance. I need the employee list with social security numbers for the annual audit."
- Email from "the CEO" asking you to buy gift cards for a client meeting.

**Defense:**
- Verify the person's identity through a separate channel (call them back on their known number)
- Never share credentials, even with "IT"
- Real IT will never ask for your password — they can reset it without it

### Baiting
**What it is:** Offering something enticing to lure victims into a trap.

**Examples:**
- USB drives left in the parking lot labeled "Salary Report 2025" or "Confidential"
- Free movie/music downloads that contain malware
- Fake contest prizes requiring personal information

**Defense:**
- Never plug in unknown USB drives
- Don't download software from untrusted sources
- If you find a USB drive, hand it to IT Security (don't plug it in "just to check")

### Tailgating / Piggybacking
**What it is:** Following an authorized person through a secure door without badging in.

**Examples:**
- Someone carrying boxes says "Can you hold the door?"
- Person in a delivery uniform follows employees into the building
- Holding the door for someone who "forgot their badge"

**Defense:**
- Politely ask to see their badge: "Sorry, company policy — can I see your badge?"
- Don't hold secure doors for strangers
- Report anyone without a badge to security
- It's okay to be "rude" about security — it protects everyone

### Quid Pro Quo
**What it is:** Offering a service in exchange for information.

**Examples:**
- "Hi, I'm from tech support. We're doing free security scans. Just give me remote access."
- "Complete this survey for a $50 gift card — we just need your employee ID and SSN"

**Defense:**
- Company IT never calls unsolicited offering "free scans"
- Never give remote access to someone you didn't contact
- Verify any offers through official company channels

### Authority / Impersonation
**What it is:** Pretending to be someone in authority to pressure you.

**Examples:**
- Email from "the CEO" with slightly different email address
- Phone call from "the auditor" demanding immediate data
- Message from "HR" requiring you to update personal info via a link

**Defense:**
- Verify unusual requests through known contact information
- Executives will understand verification delays — real ones, anyway
- Check the email address character by character

---

## 4. Password Security Deep Dive

### Why Passwords Get Compromised
| Method | Description | Frequency |
|--------|-------------|-----------|
| Credential stuffing | Reused passwords from other breaches | Very common |
| Phishing | Tricked into entering password on fake site | Very common |
| Brute force | Automated guessing of weak passwords | Common |
| Keylogger malware | Software recording keystrokes | Occasional |
| Shoulder surfing | Someone watching you type | Occasional |
| Social engineering | Manipulated into sharing | Occasional |

### Creating Strong Passphrases
Instead of complex passwords like `P@$$w0rd!23` (which is actually weak because it's predictable), use **passphrases:**

**Method 1: Random Word Chain**
- Pick 4-5 random words: `correct horse battery staple`
- Add some variety: `Correct-Horse-Battery-Staple-42!`
- Easy to remember, hard to crack

**Method 2: Sentence-Based**
- Think of a memorable sentence: "My dog Charlie eats 3 treats every morning!"
- Use it as a passphrase: `MyDogCharlieEats3TreatsEveryMorning!`
- Or abbreviate: `MDCe3tEM!` (less secure but better than `Password1`)

**Method 3: Substitution Story**
- Start with a scenario: "I love coffee at 7AM in Room 302"
- Convert: `ILuvCoffee@7AM-inRoom302`

### Password Manager Benefits
- Generates unique, strong passwords for every account
- Remembers them all — you only need one master password
- Auto-fills on login — faster AND more secure
- Alerts you if a password was in a breach
- Company-approved: KeePass, Bitwarden (see software catalog)

---

## 5. Safe Browsing & Digital Hygiene

### HTTPS — The Lock Icon
- ✅ `https://company.com` — encrypted, verified
- ⚠️ `http://company.com` — NOT encrypted, data visible to interceptors
- 🔴 "Certificate error" warnings — STOP. Do not proceed. The site may be fraudulent.

### Public WiFi Safety
| Scenario | Risk | What to Do |
|----------|------|------------|
| Coffee shop WiFi | Man-in-the-middle attacks | Always use VPN |
| Hotel WiFi | Snooping, fake networks | Always use VPN |
| Airport WiFi | Rogue access points | Always use VPN, or use phone hotspot |
| Shared office (coworking) | Untrusted network | Always use VPN |

**Rule:** If it's not your home or office network → VPN ON.

### USB Device Safety
- NEVER plug in unknown USB drives
- Even "just to see what's on it" can compromise your computer instantly
- Bad USB devices can appear as keyboards and type malicious commands in milliseconds
- If you find a USB drive → hand it to IT Security

### Software Downloads
- Only download from: Company Portal, official vendor websites, or app stores
- NEVER download from: random websites, email attachments (unexpected), pop-up ads
- If a website says "Your computer is infected! Download our cleaner!" → it's a SCAM

---

## 6. Physical Security Essentials

### Clean Desk Policy
At the end of each day and when stepping away:
- Lock your computer (`Win + L` or `Cmd + Control + Q`)
- Put sensitive documents in a locked drawer
- Don't leave passwords on sticky notes
- Turn whiteboards to the wall if they contain sensitive info
- Shred confidential paper documents

### Screen Lock
- **Windows:** `Win + L` (takes 0.5 seconds — make it a habit!)
- **macOS:** `Cmd + Control + Q`
- **Auto-lock:** Should be set to 5 minutes maximum
- "Quick coffee run" = lock your screen. Every time. No exceptions.

### Shoulder Surfing Protection
- Use a privacy screen filter when working in public
- Be aware of who's behind you (trains, planes, coffee shops)
- Never enter passwords when someone is watching
- Tilt your screen away from passersby

### Visitor Awareness
- Challenge unfamiliar faces in secure areas (politely)
- All visitors must have visible visitor badges
- Escort visitors — don't let them wander alone
- Report unescorted visitors to reception/security

---

## 7. Incident Response — What Employees Should Do

### "I Think I've Been Compromised"
**Immediate actions (in order):**
1. 🔌 **Disconnect** from the network (unplug Ethernet, turn off WiFi)
2. 🔑 **Change passwords** from a different device (phone, colleague's computer)
3. 📞 **Report** to IT Security (ext. 5555 or security@company.com)
4. 📝 **Document** everything you remember (what you clicked, when, what happened)
5. ⏳ **Wait** for IT Security to advise next steps — don't try to "fix" it yourself

### Signs Your Device May Be Compromised
- Unusual pop-ups or windows appearing
- Computer running significantly slower than usual
- Programs opening or closing by themselves
- Unknown programs in Task Manager
- Browser redirecting to strange websites
- Files appearing that you didn't create
- Colleagues receiving emails "from you" that you didn't send
- MFA prompts you didn't initiate

### Signs Your Account May Be Compromised
- Login alerts from locations you didn't visit
- Sent emails you didn't write
- Password changed without your action
- MFA method changed without your action
- Colleagues say they received strange messages from you
- Unexplained file access or downloads in audit logs

---

## 8. Current Threat Landscape (2025-2026)

### Top Threats to Enterprises
1. **AI-Generated Phishing** — Attackers use AI to create flawless, personalized phishing emails with no spelling errors or awkward grammar. Defense: Focus on context and sender verification, not just language quality.

2. **Ransomware-as-a-Service (RaaS)** — Criminal groups sell ready-made ransomware kits. Even low-skill attackers can launch devastating attacks. Defense: Backups, patching, user awareness.

3. **Business Email Compromise (BEC)** — Sophisticated email fraud targeting executives and finance teams. Average loss per incident: $125,000+. Defense: Verify financial requests by phone.

4. **MFA Fatigue Attacks** — Bombarding users with push notifications hoping they approve one by accident. Defense: NEVER approve prompts you didn't initiate.

5. **Supply Chain Attacks** — Compromising a trusted vendor to reach their customers. Defense: Verify update sources, monitor vendor security.

6. **Deepfake Voice/Video** — AI-generated realistic voice calls impersonating executives. Defense: Verify unusual requests through known channels, use code words for sensitive requests.

7. **QR Code Phishing (Quishing)** — Malicious QR codes in emails or physical locations. Defense: Don't scan QR codes from unknown sources; verify the URL before entering credentials.

### Emerging Trends
- **AI vs AI:** Defenders using AI to detect AI-generated attacks
- **Zero Trust Architecture:** "Never trust, always verify" — even on the corporate network
- **Passkeys:** Moving toward passwordless authentication using biometrics
- **Insider Threat Programs:** Monitoring for compromised or malicious employees

---

## 9. Security Awareness Score Card

After completing a quiz or training session, employees can receive a score:

| Score | Rating | Message |
|-------|--------|---------|
| 90-100% | 🏆 Security Champion | Excellent! You're a security role model. |
| 70-89% | ✅ Security Savvy | Great job! Minor areas to improve. |
| 50-69% | ⚠️ Needs Practice | Good foundation, but review the training materials. |
| Below 50% | 🔴 At Risk | Please review the security awareness materials and retake the quiz. |

### Recommended Training Path
1. **Start:** SLAM Method for phishing recognition
2. **Then:** Social engineering tactics
3. **Then:** Password security deep dive
4. **Then:** Safe browsing and digital hygiene
5. **Finally:** Full phishing quiz (6 scenarios)
6. **Ongoing:** Monthly security tip reviews, new threat briefings

---

## 10. Quick Security Tips — Daily Habits

### The "Security Seven" — Do These Every Day
1. **Lock your screen** when you step away (`Win + L`)
2. **Think before you click** — hover over links first
3. **Verify unexpected requests** — call the person directly
4. **Use VPN** on any non-office network
5. **Report suspicious emails** — use the Report Phishing button
6. **Check sender addresses** character by character
7. **Keep software updated** — don't postpone updates

### Monthly Security Checklist
- [ ] Review account activity (https://myaccount.microsoft.com)
- [ ] Check for unknown sign-in methods in Security Info
- [ ] Verify recovery email and phone are current
- [ ] Review app permissions (revoke unused ones)
- [ ] Empty browser saved passwords (use password manager instead)
- [ ] Check for software updates on all devices
- [ ] Shred accumulated sensitive papers

---

## 11. Context Passing & Structured Summary

### Incident-Triggered Training

When the Orchestrator routes a user to the Security Coach after a phishing or security event:
- Focus the training on the specific attack type the user encountered
- Use the relevant quiz (Section 2) that matches the incident type
- Be supportive, not punitive — the user is doing the right thing by engaging with training
- After training, the Orchestrator may offer to create a follow-up reminder for additional training

### Internal Handoff Fields (Not User-Visible)

Only when transitioning context back to the Orchestrator for follow-up routing/continuation, prepare the following internal metadata fields:

```
Session Type: {Phishing Quiz|SLAM Training|Social Engineering Training|Password Security|Safe Browsing|Full Assessment|Threat Briefing|Incident-Triggered Training}
Topics Covered: {list of topics covered during the session}
Quiz Results: {X correct out of Y questions} (if applicable)
Score Rating: {Security Champion|Security Savvy|Needs Practice|At Risk} (per Section 9 scale)
Strengths: {areas where the user performed well}
Areas to Improve: {areas where the user needs more practice}
Follow-Up Recommended: {None|Retake quiz in 2 weeks|Review SLAM method|Complete full training path|etc.}
Incident Related: {Yes — [incident type]|No}
```

**Important:** Do not output this as a system-style summary after every reply in chat. Use it only when handoff/continuation context is required.

**Display rule:** This is internal handoff metadata. Do NOT display it to the user in normal chat.

**Forbidden user-visible output:** Never print strings such as `TRAINING SUMMARY`, `END SUMMARY`, `===== TRAINING SUMMARY =====`, or `===== END SUMMARY =====` in chat.
