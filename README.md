# From 47% CPU to Cryptojacking: How I Found and Removed 1,997 Malware Files From My Own Laptop

**Author:** Gagandeep Singh  
**Program:** Cybersecurity & AI, Humber Polytechnic  
**Date:** April 2026  
**Tags:** `#cybersecurity` `#cryptojacking` `#incidentresponse` `#malwareanalysis` `#mitreattack`

---

## The Story Begins

It was almost midnight on a Saturday when I noticed my laptop's fan spinning louder than usual. I wasn't running anything heavy — no games, no rendering, no virtual machines. Just a browser with a few tabs open.

I opened Task Manager out of habit.

**CPU: 48%.**

I sat up a little straighter.

As a Cybersecurity & AI student at Humber Polytechnic, an unexplained 48% CPU spike isn't something you ignore. It's something you investigate. What followed was a three-hour journey down a rabbit hole that ended with **1,997 malware files removed**, a confirmed Monero cryptominer, a password stealer, and an honest reckoning with my own past.

Here's how it went.

---

## Step 1: Initial Detection

The first thing I noticed in Task Manager was two processes named `AddInProcess.exe` — one using 28.8% CPU, the other 18.7%. Together, almost half my CPU.

`AddInProcess.exe` is a legitimate Microsoft .NET host process used to run add-ins for Office and Visual Studio. It's signed by Microsoft. It lives in a system folder. Most people would see it and move on.

But two of them, eating CPU, while I wasn't using any Office apps?

Suspicious.

---

## Step 2: Verifying the File

I right-clicked the process and chose **"Open file location."** It took me to:C:\Windows\Microsoft.NET\Framework64\v4.0.30319\AddInProcess.exe 
That's the legitimate Microsoft path. So the file itself wasn't fake.

For about a minute, I thought I'd been wrong. Maybe iCloud's Outlook integration was misbehaving. That's a known issue.

Then I decided to go one step further.

---

## Step 3: The Smoking Gun

In Task Manager → Details tab, I added the **"Command line"** column. This shows what arguments a process was launched with.

What I saw stopped me cold: AddInProcess.exe --algo rx/0 -o xmr-us-east1.nanopool.org:10300 ...
Let me decode this:

- `--algo rx/0` → **RandomX algorithm**, used for mining **Monero (XMR)**
- `-o xmr-us-east1.nanopool.org` → **Nanopool**, a well-known Monero mining pool
- `xmr` → Monero, a privacy-focused cryptocurrency favored by attackers

This wasn't iCloud. This wasn't a buggy add-in.

**Someone was using my laptop to mine cryptocurrency for them.**

This is called **cryptojacking** — and the technique used here is a classic example of **LOLBin abuse** (Living Off The Land Binaries). Instead of dropping their own miner that antivirus would catch, the attacker abused a legitimate, signed Microsoft binary to do the mining. To Task Manager, it just looks like a normal Windows process.

Clever. And confirmed malicious.

---

## Step 4: Confirming Persistence

I killed both `AddInProcess.exe` processes. They came back within seconds — with new PIDs.

Something was launching them automatically. The malware had **persistence**.

I tried to investigate via Task Scheduler. When I opened it, I got an error:

> *"The selected task 'Microsoft' no longer exists."*

That itself is suspicious. Real scheduled tasks have proper names inside the `\Microsoft\Windows\` folder structure — never just `Microsoft` as a top-level item. The malware was likely creating, executing, and deleting tasks dynamically to evade detection.

At this point, I stopped trying to hunt manually and reached for the right tool.

---

## Step 5: Deploying Malwarebytes

I downloaded Malwarebytes (free version) and started a Threat Scan.

Within seconds, before the scan even finished, real-time protection caught something:

> **Malware blocked**  
> **Type:** Malware  
> **Name:** `Trojan.MCrypt.MSIL.Generic`  
> **Path:** `C:\Users\[user]\AppData\Roaming\Values\Current.exe`

There it was. The persistence mechanism. A fake executable hidden in AppData under a folder innocuously named "Values," disguised as `Current.exe`.

A second alert came almost immediately:

> **We blocked a connection to a potentially risky site**  
> **Address:** `154.12.226.43:56002`  
> **App:** `C:\Users\[user]\AppData\...\conhost.exe`

A **second piece of malware** — this one impersonating `conhost.exe` (a legitimate Windows process) but living in AppData instead of `System32`. It was trying to phone home to its command-and-control (C2) server.

Two indicators of compromise, caught in real time.

---

## Step 6: The Full Scan

Then the full scan results came in.

**1,997 threats detected.**

Not 5. Not 50. **One thousand nine hundred and ninety-seven.**

The detections fell into clear families:

| Threat Family | What It Is |
|---|---|
| `Trojan.Miner.DDS` | The cryptominer |
| `Trojan.MCrypt.MSIL.Generic` | Crypto miner module |
| `Trojan.PCrypt.MSIL` | **Password stealer** |
| `PUP.Optional.MediaArena.DDS` | Bundled adware |
| `Trojan.Agent.Gen` | Generic trojan components |

Locations included:

- `C:\Users\[user]\AppData\Roaming\` (user-level persistence)
- `C:\Windows\SysWOW64\` (system-level — concerning)
- `C:\Windows\Temp\` (staging area)
- `HKLM\SOFTWARE\...` (registry persistence)

I selected all 1,997 items and clicked **Quarantine**.

---

## Step 7: The Aftermath

After quarantine and a restart, I checked Task Manager again.

**CPU: 14%. Memory: 40%.**

The two `AddInProcess.exe` processes were gone. The fan went quiet. The laptop felt fast again — actually fast, the way it used to feel.

But the relief was mixed with a harder realization.

---

## How Did This Happen?

Two years ago, before I started studying cybersecurity, I installed **cracked software** on this laptop.

That's the honest origin. No mystery zero-day, no sophisticated phishing, no supply-chain attack. Just a younger me thinking I was getting "free" software, when in fact I was paying with my CPU cycles, my passwords, and my privacy.

For two years, attackers had a foothold on my machine. They were:
- Mining Monero with my electricity and my hardware
- Likely harvesting passwords from my browser
- Establishing persistence I never noticed

The cryptominer was just the most visible symptom. The `Trojan.PCrypt.MSIL` — the password stealer — is the part that should scare anyone reading this. Saved browser passwords. Banking sessions. School portal credentials. All probably exfiltrated long before I noticed the CPU spike.

---

## Mapping This to MITRE ATT&CK

For anyone studying cybersecurity, here's how this incident maps to the **MITRE ATT&CK framework**:

| Tactic | Technique | Evidence |
|---|---|---|
| **Initial Access** | T1189 — Drive-by Compromise (likely) | Cracked software installer |
| **Execution** | T1218 — Signed Binary Proxy Execution | Abuse of `AddInProcess.exe` |
| **Persistence** | T1547 — Boot or Logon Autostart | `Current.exe` in AppData\Roaming |
| **Defense Evasion** | T1036 — Masquerading | Fake `conhost.exe` outside System32 |
| **Credential Access** | T1555 — Credentials from Password Stores | `Trojan.PCrypt.MSIL` |
| **Command & Control** | T1071 — Application Layer Protocol | C2 to `154.12.226.43:56002` |
| **Impact** | T1496 — Resource Hijacking | Monero mining via Nanopool |

This is real-world threat intelligence — not from a textbook, from my own laptop.

---

## Indicators of Compromise (IOCs)

For other defenders:

**Network IOCs:**
- IP: `154.12.226.43:56002`
- Domain: `xmr-us-east1.nanopool.org`
- Mining algorithm: `rx/0` (RandomX)

**File IOCs:**
- `%APPDATA%\Roaming\Values\Current.exe` — Trojan.MCrypt.MSIL.Generic
- `%APPDATA%\Local\...\conhost.exe` — masqueraded process

**Behavioral IOCs:**
- Unexplained sustained 40%+ CPU usage
- `AddInProcess.exe` running with `--algo` arguments
- Auto-respawning processes after termination
- Scheduled tasks with non-standard names

---

## Lessons Learned

**1. Cracked software is never free.** It costs your security, your privacy, and eventually your time when you have to clean up the mess. Use student licenses (most colleges including Humber provide them), open-source alternatives, or save up for legitimate copies.

**2. Trust your instincts.** A 48% CPU spike with no explanation is a signal, not a coincidence. Investigate.

**3. The command line tells the truth.** Process names lie. File paths can be faked. But command-line arguments show what a process is *actually* doing.

**4. LOLBins are real.** Attackers don't always drop new malware. They often abuse what's already on your system. This means signature-based detection alone isn't enough — behavioral analysis matters.

**5. Cleanup ≠ Trust.** Even after Malwarebytes removed 1,997 files, I can never be 100% sure this Windows install is clean. Two years of compromise can leave deep traces. The right call is a full Windows reset.

---

## What's Next

This week I'm doing a clean Windows reinstall. Backing up only documents (no executables, no AppData, no browser profiles). Reimaging the drive completely. Reinstalling only legitimate software. Setting up a password manager. Enabling 2FA everywhere. Treating my personal device the way a professional treats a work laptop — because that's what it should be.

This experience didn't shake my interest in cybersecurity. It deepened it. There's something genuinely powerful about going from *"why is my fan loud"* to mapping a real-world attack to MITRE ATT&CK in one night, on your own machine, with your own evidence.

If you're a fellow student, a hiring manager, or just someone who used to download cracked Photoshop — I hope this writeup is useful.

And if you're thinking, *"could this be happening on my laptop right now?"* — open Task Manager. Check your CPU. Look at the command lines.

You might be surprised what you find.

---

## About the Author

**Gagandeep Singh** is a Cybersecurity & AI student at **Humber Polytechnic** in Toronto, Canada. He's targeting SOC Analyst and incident response roles after graduation. This case study was performed on his own personal device, with documentation captured at every stage.

---

*Have you dealt with a similar incident? Found this writeup useful? Drop a comment — I'd love to hear other defenders' stories.*
