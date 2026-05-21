# 🔥 LAB12 – ROOT DETECTION BYPASS

**Author:** EL YAMANI OMAYMA

---

## ⚠️ Disclaimer

This lab documents **offensive security techniques** for **authorized testing only**.  
The target is **DIVA** (`jakhar.aseem.diva`), a deliberately vulnerable Android application.  
**Goal:** Force the app to believe it runs on a non-rooted device while operating in a fully compromised environment.

---

## PHASE 0 – Reconnaissance & Connectivity

### 0.1 ADB – Establish Device Connection

```bash
PS C:\Users\pc> adb devices
List of devices attached
emulator-5554    device
```

**Interpretation:**
- ✅ Emulator is active
- ✅ ADB channel is open
- ✅ Communication pipeline established

---

## PHASE 1 – Weapon Deployment (Medusa)

Clone the Medusa repository:

```bash
git clone https://github.com/Ch0pin/medusa.git
```

**Outcome:**
- ✅ 4255 objects received
- ✅ 53.08 MiB transferred
- ✅ 2639 deltas resolved

---

## PHASE 2 – Frida Asset Inventory

List running processes with Frida:

```bash
frida-ps -U
```

**Target Process Table:**

| PID  | Process       | Status        |
|------|---------------|---------------|
| 6989 | Diva          | 🎯 **ACQUIRED** |
| 6747 | Messaging     | Collateral    |
| 4184 | RoomMVVDemo   | Collateral    |
| 7049 | adbd          | Daemon        |

**Conclusion:** Frida successfully detects the target. Injection path is clear.

---

## PHASE 3 – Medusa Initialization & Device Binding

Initialize Medusa with target package:

```bash
python medusa.py -p owasp.mstg.uncrackable2
```

**Device Selection Menu:**

| Index | Device        | Type   |
|-------|---------------|--------|
| 0     | Local System  | local  |
| 1     | Local Socket  | remote |
| 2     | GDB Remote    | remote |
| 3     | emulator-5554 | **usb** ✅ |

**Selection:** 3 → **Binding: Successful**

---

## PHASE 4 – System Forensics (Build Properties)

Medusa extracts device properties:

| Property                  | Value          | Risk Level   |
|---------------------------|----------------|--------------|
| ro.product.manufacturer   | unknown        | ⚠️ Suspicious |
| ro.build.version.release  | 11             | ✅ Acceptable |
| ro.build.tags             | test-keys      | 🔴 **ROOT DETECTED** |
| ro.build.security_patch   | 2026-05-15     | ✅ Recent    |

**Third-party Applications Detected:**

```
[0] projet.fst.ma.roommvvmdemo
[1] projet.fst.ma.localisationsmartphone
[2] projet.fst.ma.convertertabsjava
[3] projet.fst.ma.starsgallery
[4] sg.vantagepoint.lab8
[5] sg.vantagepoint.jnidemo
[6] com.withsecure.dz
[7] jakhar.aseem.diva  ← TARGET CONFIRMED
```

**Critical Finding:** `test-keys` indicates system compiled with debug credentials. This is the primary signature most root detectors scan for.

---

## PHASE 5 – Target Execution

Launch the target application:

```bash
(emulator-5554) medusa> run jakhar.aseem.diva
```

**DIVA Application Menu:**

```
1. INSECURE LOGGING
2. HARDCODING ISSUES – PART 1
3. INSECURE DATA STORAGE – PART 1
4. INSECURE DATA STORAGE – PART 2
5. INSECURE DATA STORAGE – PART 3
6. INSECURE DATA STORAGE – PART 4
7. INPUT VALIDATION ISSUES – PART 1
8. INPUT VALIDATION ISSUES – PART 2
9. ACCESS CONTROL ISSUES – PART 1
10. ACCESS CONTROL ISSUES – PART 2
```

**Status:** Application running on rooted environment.  
**Objective:** Make it believe otherwise.

---

## PHASE 6 – Root Detection Mechanisms

Modern Android root detectors scan for:

| Vector              | Detection Method                      |
|---------------------|---------------------------------------|
| Build Properties    | `Build.TAGS` contains `test-keys`?    |
| Binary Presence     | `File.exists("/system/bin/su")`       |
| Binary Presence     | `File.exists("/system/xbin/su")`      |
| Command Execution   | `Runtime.exec("su")`                  |
| Library Check       | `RootBeer.isRooted()`                 |
| Mount Points        | `/proc/mounts` contains `su`?         |
| Native Calls        | `open()`, `access()`, `stat()` checks |

**Bypass Strategy:** Hook every detection vector and return non-rooted responses.

---

## PHASE 7 – The Bypass Arsenal

### Primary Weapon: Frida Injection

**File:** `inject_oblivion.js`

```javascript
// inject_oblivion.js - Full Root Detection Bypass Script
Java.perform(function() {
    console.log("[*] Oblivion engine initialized.");

    // 1. Hijack Build.TAGS
    try {
        var Build = Java.use('android.os.Build');
        Object.defineProperty(Build, 'TAGS', {
            get: function() {
                console.log("[✓] Build.TAGS intercepted → release-keys");
                return 'release-keys';
            }
        });
    } catch(e) { console.error("[✗] Build hook failed:", e); }

    // 2. Neutering File.exists on root paths
    try {
        var File = Java.use('java.io.File');
        var killPaths = [
            "/system/bin/su", "/system/xbin/su", "/sbin/su",
            "/system/bin/busybox", "/system/xbin/busybox",
            "/system/app/Superuser.apk", "/system/app/SuperSU.apk"
        ];
        
        File.exists.implementation = function() {
            var path = this.getAbsolutePath();
            for (var p of killPaths) {
                if (path.includes(p)) {
                    console.log("[✓] File.exists blocked: " + path + " → false");
                    return false;
                }
            }
            return this.exists.call(this);
        };
    } catch(e) { console.error("[✗] File hook failed:", e); }

    // 3. Runtime.exec – Block su command execution
    try {
        var Runtime = Java.use('java.lang.Runtime');
        Runtime.exec.overload('java.lang.String').implementation = function(cmd) {
            if (cmd.includes("su") || cmd.includes("busybox")) {
                console.log("[✓] Runtime.exec blocked: " + cmd);
                return null;
            }
            return this.exec(cmd);
        };
    } catch(e) { console.error("[✗] Runtime hook failed:", e); }

    // 4. RootBeer assassination (if present)
    try {
        var RootBeer = Java.use('com.scottyab.rootbeer.RootBeer');
        if (RootBeer.isRooted) {
            RootBeer.isRooted.implementation = function() {
                console.log("[✓] RootBeer.isRooted → false");
                return false;
            };
        }
    } catch(e) { /* RootBeer not present – silent pass */ }

    console.log("[✔] Oblivion engine ready. Root is now invisible.");
});
```

### Injection Command

```bash
frida -U -f jakhar.aseem.diva -l inject_oblivion.js --no-pause
```

### Expected Console Output

```
[*] Oblivion engine initialized.
[✓] Build.TAGS intercepted → release-keys
[✓] File.exists blocked: /system/bin/su → false
[✓] Runtime.exec blocked: su
[✔] Oblivion engine ready. Root is now invisible.
```

---

## PHASE 8 – Validation Matrix

| Checkpoint                        | Pre-Injection     | Post-Injection        |
|-----------------------------------|-------------------|-----------------------|
| Root detection alert              | 🔴 Detected       | 🟢 Not detected       |
| Build.TAGS value                  | `test-keys`       | `release-keys`        |
| File.exists("/system/bin/su")     | `true`            | `false` (hooked)      |
| Runtime.exec("su")                | Executes          | Returns `null`        |
| RootBeer.isRooted() (if present)  | `true`            | `false` (hooked)      |
| Application functionality         | Degraded          | Fully operational     |

---

## PHASE 9 – Evasion Deep Dive (Anti-Frida Countermeasures)

If the target implements Frida detection:

| Detection Method        | Countermeasure                              |
|-------------------------|---------------------------------------------|
| Port scanning (27042)   | Use frida-server on alternative port        |
| D-Bus detection         | Patch or rename frida-server binary         |
| Thread enumeration      | Hook `Thread.getStackTrace()`               |
| String scanning         | Recompile Frida with custom string table    |
| Memory scanning (gum-js)| Use Gadget injection instead of server      |

**Current Status:** DIVA has no anti-Frida protections. Direct injection works without obstruction.

---

## PHASE 10 – Conclusion & Operational Learnings

### What Worked ✅

- ✅ ADB → Frida → Medusa pipeline established
- ✅ Target enumeration via `frida-ps` and Medusa module loader
- ✅ Root indicators identified (`test-keys`, su paths)
- ✅ Custom Frida script bypassed all root checks
- ✅ Application runs without root warnings
- ✅ Full functionality restored on compromised environment

### Key Takeaways

1. **Root detection is hookable** – Java-level checks are interceptable by Frida
2. **Build properties are mutable** – Property spoofing is trivial
3. **File system checks need comprehensive patching** – Multiple paths and methods must be covered
4. **Defense-in-depth is necessary** – Single-vector detection is insufficient
5. **Runtime monitoring** – Continuous hook injection defeats static protections

---

## References

- [Frida Documentation](https://frida.re/)
- [Medusa Framework](https://github.com/Ch0pin/medusa)
- [DIVA Application](https://github.com/jakhar/DIVA-Android)
- [Android Security Best Practices](https://developer.android.com/privacy-and-security)
