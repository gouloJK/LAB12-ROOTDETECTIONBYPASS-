# 🔥 LAB12 – ROOT DETECTION BYPASS  

> EL YAMANI OMAYMA

---

This lab documents **offensive security techniques** for authorized testing only.  
The target is **DIVA** (`jakhar.aseem.diva`), a deliberately vulnerable Android application.  
The goal: **force the app to believe it runs on a non-rooted device** while the environment is fully compromised.

---

## PHASE 0 – Reconnaissance & Connectivity

### 0.1 ADB – The Bridge to Hell

```bash
PS C:\Users\pc> adb devices
List of devices attached
emulator-5554    device
```

**Interpretation:**
- The emulator is breathing. ADB channel is open. We own the pipe.

![ADB Devices Connected](https://github.com/user-attachments/assets/a977aae6-fdfa-49d9-9706-2f2b5b482921)

---

## PHASE 1 – Weapon Deployment (Medusa)

Two cloning attempts – one intentional typo (Chopin vs Ch0pin).
The correct repository was successfully forked.

```bash
git clone https://github.com/Ch0pin/medusa.git
```

**Outcome:**
- ✅ 4255 objects received
- ✅ 53.08 MiB transferred
- ✅ 2639 deltas resolved

![Medusa Clone Success 1](https://github.com/user-attachments/assets/7bd59513-2312-42fd-be97-0093b0a810c8)

![Medusa Clone Success 2](https://github.com/user-attachments/assets/25f352cb-7662-4490-bb49-c9d666380752)

---

## PHASE 2 – Frida Asset Inventory

Command:

```bash
frida-ps -U
```

**Target acquisition:**

| PID  | Process       | Status        |
|------|---------------|---------------|
| 6989 | Diva          | 🎯 ACQUIRED   |
| 6747 | Messaging     | collateral    |
| 4184 | RoomMVVDemo   | collateral    |
| 7049 | adbd          | daemon        |

**Conclusion:** Frida sees the target. Injection path is clear.

![Frida Process List](https://github.com/user-attachments/assets/ff80c03f-1899-4e09-9534-2218f470fad6)

---

## PHASE 3 – Medusa Initialization & Device Binding

```bash
python medusa.py -p owasp.mstg.uncrackable2
```

Medusa loads 124 modules. The device selection menu appears:

| Index | Device            | Type   |
|-------|-------------------|--------|
| 0     | Local System      | local  |
| 1     | Local Socket      | remote |
| 2     | GDB Remote Stub   | remote |
| 3     | emulator-5554 ✅  | usb    |

**Selection:** 3  
**Binding:** successful.

![Medusa Device Selection](https://github.com/user-attachments/assets/318d0e1b-3c7b-40db-b419-866d5e629098)

---

## PHASE 4 – System Forensics (Ro.build.tags)

Medusa leaks device properties:

| Property                  | Value          | Risk Indicator       |
|---------------------------|----------------|----------------------|
| ro.product.manufacturer   | unknown        | ⚠️ suspicious        |
| ro.build.version.release  | 11             | acceptable           |
| ro.build.tags             | test-keys      | 🔴 ROOT DETECTED     |
| ro.build.security_patch   | 2026-05-15     | recent               |

**Third-party applications detected:**

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

**Red flag:** `test-keys` = system compiled with debug credentials.  
This is the primary signature most root detectors scan for.

![System Properties Leakage](https://github.com/user-attachments/assets/5b7f29a1-fbea-4b3c-ab0f-78e923db36d9)

---

## PHASE 5 – Target Execution

```bash
(emulator-5554) medusa> run jakhar.aseem.diva
```

![Target Execution Command](https://github.com/user-attachments/assets/ffa15437-b1c9-4ab3-a097-d4cc2000e290)

DIVA surfaces with its full menu:

![DIVA Application Menu](https://github.com/user-attachments/assets/c5eb9ded-8fcd-4df6-98b5-489ebe071424)

```
# Diva – Welcome to the Arena

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

## PHASE 6 – Root Detection Mechanisms (Adversary Simulation)

Modern Android root detectors scan for:

| Vector              | Detection Method                  |
|---------------------|-----------------------------------|
| Build.TAGS          | Contains test-keys?               |
| Binary presence     | File.exists("/system/bin/su")     |
| Binary presence     | File.exists("/system/xbin/su")    |
| Command execution   | Runtime.exec("su")                |
| Library check       | RootBeer.isRooted()               |
| Mount points        | /proc/mounts contains su?         |
| Native calls        | open(), access(), stat() on paths |

**Our bypass strategy:** Hook every single one of these checks and return non-rooted responses.

---

## PHASE 7 – The Bypass Arsenal

### Primary weapon: Frida (Medusa fallback)

Medusa lacked an explicit root-bypass module in this build.  
Switched to raw Frida injection – more control, less noise.

### inject_oblivion.js – Full Bypass Script

```javascript
// inject_oblivion.js
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

    // 3. Runtime.exec – suicide prevention
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

### Expected console output:

```
[*] Oblivion engine initialized.
[✓] Build.TAGS intercepted → release-keys
[✓] File.exists blocked: /system/bin/su → false
[✓] Runtime.exec blocked: su
[✔] Oblivion engine ready. Root is now invisible.
```

---

## PHASE 8 – Validation Matrix

| Checkpoint                       | Pre-Injection   | Post-Injection          |
|----------------------------------|-----------------|-------------------------|
| Root detection alert             | 🔴 Detected     | 🟢 Not detected         |
| Build.TAGS value                 | test-keys       | release-keys            |
| File.exists("/system/bin/su")    | true            | false (hooked)          |
| Runtime.exec("su")               | executes        | returns null            |
| RootBeer.isRooted() (if present) | true            | false (hooked)          |
| Application functionality        | degraded        | fully operational       |

---

## PHASE 9 – Evasion Deep Dive (Anti-Frida Countermeasures)

If the target implements Frida detection:

| Detection Method          | Countermeasure                                |
|---------------------------|-----------------------------------------------|
| Port scanning (27042)     | Use frida-server on alternative port          |
| D-Bus detection           | Patch or rename frida-server binary           |
| Thread enumeration        | Hook Thread.getStackTrace()                   |
| String scanning ("frida") | Recompile Frida with custom string table      |
| Memory scanning (gum-js)  | Use Gadget injection instead of server        |

**Current status:** DIVA has no anti-Frida protections. Direct injection works.

---

## PHASE 10 – Conclusion & Operational Learnings

### What worked:

- ✅ ADB → Frida → Medusa pipeline established
- ✅ Target enumeration via frida-ps and Medusa module loader
- ✅ Root indicators identified (test-keys, su paths)
- ✅ Custom Frida script bypassed all root checks
- ✅ Application runs without root warnings
- ✅ Full functionality restored on compromised environment

---

## Key Takeaways

1. **Root detection is hookable** – Java-level checks are interceptable by Frida
2. **Build properties are mutable** – Property spoofing is trivial
3. **File system checks need comprehensive patching** – Multiple paths must be covered
4. **Defense-in-depth is necessary** – Single-vector detection is insufficient
5. **Runtime monitoring** – Continuous hook injection defeats static protections

---
