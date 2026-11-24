# 🚨 SECURITY RESEARCH REPORT: The CCIT S18 Pro Max — Hardware Spoofing and Pre-Installed Spyware (Midnight Mimosa)

**Target Device:** CCIT S18 Pro Max (Designed by CCIT in UAE / Made in China)
**Investigator:** Independent Security Researcher (BlackArch Linux Environment)
**Date:** October 2026
**Classification:** Consumer Fraud, Hardware Spoofing, Pre-Installed Advanced Persistent Malware

---

## 📝 Executive Summary

The CCIT S18 Pro Max is marketed as a high-end flagship smartphone, claiming 16GB of RAM, 1TB of storage, an Android 16 operating system, and a powerful processor. However, a forensic analysis using Android Debug Bridge (ADB) and the `/proc/` filesystem reveals a completely different reality. The device is a low-end counterfeit built on a 2018-era MediaTek MT6765 (Helio P35) chipset, equipped with only 3GB of actual RAM and 32GB of storage, running a spoofed Android 12 ROM.

More alarmingly, the device ships with a pre-installed, system-level spyware package identified as **`com.android.sys.bcprot`**. This malware belongs to the **"Midnight Mimosa"** campaign discovered by Bitdefender, a severe preinstalled Android firmware malware that operates with system-level privileges, silently installs other malicious applications, and can turn the device into a node within a proxy botnet.

---

## 🏢 Part 1: Manufacturer and Product Background

**Manufacturer:** CCIT
**Origin:** Designed in the UAE, Manufactured in China
**Product:** CCIT S18 Pro Max

The device is distributed through various online marketplaces, including Jumia, Emag, and Amazon UAE, often with wildly inconsistent specifications to deceive buyers. While some listings claim "16GB RAM + 1TB Storage" and "Android 16", others advertise "24GB RAM (12GB + 12GB extensible)" and "Android 14". This deliberate inconsistency is a hallmark of counterfeit operations. The device is sold as a budget alternative to flagship smartphones, but it is, in fact, a repackaged low-end device with a spoofed operating system.

---

## 🕵️‍♂️ Part 2: The Hardware Illusion (Marketing vs. Reality)

The device's `build.prop` and Settings application were heavily modified to display false information. By bypassing the user interface and reading the raw hardware data via ADB, the following discrepancies were confirmed:

| Feature | CCIT Claims (Packaging/Settings) | Actual Hardware (Verified via ADB) |
| :--- | :--- | :--- |
| **Processor** | Snapdragon 8 Gen 3 (Implied) | **MediaTek MT6765** (Helio P35 - Released 2018) |
| **RAM** | 16 GB | **2.8 GB** (`MemTotal: 2850852 kB`) |
| **Storage** | 1 TB | **~19 GB Available** (32GB eMMC) |
| **Display** | 6.9" FullHD, 120Hz | **720x1600** (HD+), 60Hz |
| **OS** | Android 16 | **Android 12** (`ro.build.version.release=12`) |
| **Battery** | 6000 mAh | Unverified (Suspicious API reporting) |

**Forensic Evidence:**
The following commands were used to extract the true hardware specifications:
```bash
adb shell cat /proc/cpuinfo | grep Hardware  # Output: MT6765
adb shell cat /proc/meminfo | head -n 1      # Output: MemTotal: 2850852 kB
adb shell wm size                            # Output: Physical size: 720x1600
adb shell getprop ro.build.version.release   # Output: 12
```
*Note: The "Android 16" Easter Egg in the settings is a static image and cannot be interacted with. The device's `ro.treble.enabled` property is also spoofed to `true`, misleading users into thinking it supports Generic System Images (GSIs).*

---

## 🦠 Part 3: The Spyware Payload — Midnight Mimosa (`com.android.sys.bcprot`)

While scanning the system packages (`adb shell pm list packages -s`), a highly suspicious package was discovered: **`com.android.sys.bcprot`**. This package is a known variant of the **Midnight Mimosa** malware family, which is documented by Bitdefender to be preinstalled in the firmware of low-cost MediaTek devices across 150+ countries.

### 🚨 Dangerous Permissions Granted (granted=true)
Using `adb shell dumpsys package com.android.sys.bcprot`, it was confirmed that the malware possessed the following permissions, which were actively granted:

*   🔴 **`BIND_NOTIFICATION_LISTENER_SERVICE`**: Allows the malware to read **every notification** on the device (including OTPs, banking alerts, and WhatsApp messages).
*   🔴 **`INSTALL_PACKAGES`**: Allows the malware to **silently download and install** other malicious APKs without user consent.
*   🔴 **`SYSTEM_ALERT_WINDOW`**: Can draw over other apps, enabling phishing attacks through fake login screens.
*   🔴 **`WRITE_SECURE_SETTINGS`**: Can modify core system settings.
*   🔴 **`REBOOT`**: Can restart the device remotely.
*   🔴 **`GRANT_RUNTIME_PERMISSIONS`**: Can grant itself any permission it needs on demand.

### 🛡️ How It Operates
According to Bitdefender's research, Midnight Mimosa operates as follows:

1.  **System-Level Disguise:** The malware disguises itself as a core system component, often labeled simply as "System," without an icon.
2.  **Bypassing Defenses:** It temporarily disables the Google Play Store to avoid Play Protect detection, silently installs payload apps, and then re-enables the store to avoid suspicion.
3.  **Ad Fraud and Botnet:** The malware generates hidden background ad clicks to steal revenue and turns infected devices into residential-proxy relay nodes, making them part of large botnets for DDoS attacks.
4.  **Data Exfiltration:** It collects device identifiers (IMEI, Android ID, MAC address) and grants itself access to Accessibility services, notifications, and SMS permissions.

---

## 🔬 Part 4: Forensic Methodology and Commands Used

The analysis was conducted on a **BlackArch Linux** environment using the following tools and commands:

### 1. Package Inspection
```bash
adb shell pm list packages -s | grep -E "system.lite|sys.prot|sys.gmsprot|sys.bcprot"
# Output: package:com.android.sys.bcprot
```

### 2. Permission Analysis
```bash
adb shell dumpsys package com.android.sys.bcprot | grep -i "android.permission"
# Revealed extensive granted permissions, including INSTALL_PACKAGES and BIND_NOTIFICATION_LISTENER_SERVICE.
```

### 3. Network Monitoring
```bash
adb shell netstat -tuln | grep ESTABLISHED
# Confirmed no active external connections when the device was offline.
```

### 4. Application Verification
```bash
adb shell pm list packages -3
# Confirmed no user-installed apps were present, indicating the malware did not silently install additional APKs before removal.
```

---

## 🛡️ Part 5: Remediation and Mitigation

### Temporary Remediation (User-Space Removal)
The malware was successfully removed from the current user space using:
```bash
adb shell pm uninstall -k --user 0 com.android.sys.bcprot
# Output: Success
```
Additional suspicious packages were also removed:
*   `com.abfota.systemUpdate` (Fake updater)
*   `com.kst.weather` (Suspicious weather app)
*   `com.example.switchbootanim` (Amateur developer artifact)
*   `com.AgeTest`, `com.wan.faceunlock`, `com.st.nfc.dta`

### ⚠️ Critical Limitation
This removal is **NOT permanent**. The malware files reside in the read-only `/system` partition. A **Factory Reset will re-activate the spyware**. As Bitdefender and ThaiCERT note, permanent removal requires either **firmware-level cleanup** or **disabling the component via ADB** as a temporary mitigation.

### 💡 Recommended Protection Measures
*   **Do not use** this device for sensitive accounts (banking, primary email, cryptocurrency).
*   **Use it only as a secondary device** for basic tasks (calls, YouTube, light browsing).
*   **Re-run the ADB cleanup script** after any factory reset.
*   **Do not attempt to flash a custom ROM** unless you have a verified device tree and are prepared for the risk of bricking the device.

---

## 📢 Call to Action and Direct Accusations

**To the Developer and Security Community:**
*   **Do not purchase** the CCIT S18 Pro Max or similar unbranded devices from AliExpress, local markets, or social media ads.
*   **Spread awareness** about the Midnight Mimosa campaign and the risks of counterfeit MediaTek devices.

**To the Manufacturer (CCIT):**
*   You are directly accused of **Counterfeiting (تزييف)**, **Consumer Fraud (غش تجاري)**, and **Cyber Espionage (تجسس سيبراني)**.
*   Distributing devices with pre-installed notification interceptors, silent installers, and botnet capabilities is a criminal offense under international cybersecurity laws.

**To Google:**
*   How did this device pass CTS (Compatibility Test Suite) if it falsely reports the OS version and uses spoofed Android APIs? (Or did it bypass GMS certification entirely?)

**To Cybersecurity Researchers:**
*   This device serves as a case study for the dangers of supply-chain attacks in the budget smartphone market. Further analysis of the `com.android.sys.bcprot` payload and its command-and-control infrastructure is encouraged.

---

## 📚 References

1.  Bitdefender Labs. (2026, October 8). *The phone was compromised before the user turned it on: the rise of Midnight Mimosa*. https://www.bitdefender.com/en-us/blog/labs/midnight-mimosa-malware
2.  PhoneArena. (2026, October 16). *Check your phone right now: Severe malware discovered hidden inside thousands of budget Androids*. https://www.phonearena.com/news/Check-your-phone-right-now-Severe-malware-discovered-hidden-inside-thousands-of-budget-Androids_id184101
3.  Security Affairs. (2026, October 11). *Bitdefender finds preinstalled Android malware you can't uninstall, seen in 150 countries*. https://securityaffairs.com/200759/malware/bitdefender-finds-preinstalled-android-malware-you-cant-uninstall-seen-in-150-countries.html
4.  ThaiCERT. (2026, October 12). *Midnight Mimosa Malware Found Preinstalled on Budget Android Smartphones, Difficult to Remove and Seen in More Than 150 Countries*. https://www.thaicert.or.th/en/2026/10/12/midnight-mimosa-malware-found-preinstalled-on-budget-android-smartphones-difficult-to-remove-and-seen-in-more-than-150-countries/

---

**Report prepared by an Independent Security Researcher using BlackArch Linux and ADB.**
**All forensic evidence is documented and verifiable.**
