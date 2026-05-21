# LAB 9 Android Attack Surface Analysis 
## Drozer Deep Dive | Defensive Security Audit

**Author** : Omayma El Yamani   
**Audit Type** : Authorized defensive assessment  
**Target Environment** : Android Emulator (API 34)  
**Toolstack** : Drozer v3.1.0 + ADB + DIVA v1.0

---

## Lab Index

| Phase | Objective | Status |
|-------|-----------|--------|
| 0 | Pre-audit checks & tool validation | ✅ |
| 1 | Drozer session establishment | ✅ |
| 2 | Target deployment (DIVA.apk) | ✅ |
| 3 | Attack surface enumeration | ✅ |
| 4 | Component-level introspection | ✅ |
| 5 | Data exposure verification | ✅ |
| 6 | Static credential discovery | ✅ |
| 7 | Risk matrix & OWASP mapping | ✅ |

---

## 🔧 Phase 0 – Pre‑audit Environment Verification

### ADB connectivity check

```powershell
PS C:\Users\pc> adb devices
List of devices attached
64a65e1f2db6c5f9    device
```
✅ Emulator detected and authorized for debugging.

### Drozer installation validation
```powershell
PS C:\Users\pc> drozer version
drozer Console (v3.1.0)
```
✅ Drozer console ready.

## Phase 1 – Drozer Communication Channel
### Establishing the bridge
```powershell
PS C:\Users\pc> drozer console connect
Selecting 64a65e1f2db6c5f9 (Google sdk_gphone64_x86_64 14)
Console output – ASCII art banner confirmed successful handshake.
```

```
drozer
drozer Console (v3.1.0)
dz>
```

🔐 The encrypted channel between host and emulator is operational.
All subsequent commands will traverse this secure tunnel.

### Drozer console session established
<img width="845" height="440" alt="image" src="https://github.com/user-attachments/assets/99118a7c-b4ae-409a-a6a8-bdbe5eabc64a" />

## 📦 Phase 2 – Target Deployment (DIVA)

### Installing the vulnerable test application
```powershell
PS C:\Users\pc> adb install C:\Users\pc\Downloads\DIVA.apk
```
Performing Streamed Install
Success

🎯 DIVA (Damn Insecure and Vulnerable App) – deliberately flawed Android application designed for security training.

#### Successful APK installation
<img width="656" height="55" alt="image" src="https://github.com/user-attachments/assets/36ba33ab-e9de-4d95-bd59-a0003d0d210a" />

## Phase 3 – Attack Surface Enumeration

### Package discovery
```
drozer
dz> run app.package.list -f diva
jakhar.aseem.diva (Diva)
```

🔍 Package identified: jakhar.aseem.diva

### Global attack surface summary
```
drozer
dz> run app.package.attacksurface jakhar.aseem.diva
Attack Surface:
  3 activities exported
  0 broadcast receivers exported
  1 content providers exported
  0 services exported
  is debuggable
```

| Component Type | Exported Count | Risk Level |
|---|---|---|
| Activities | 3 | 🟡 Medium-High |
| Content Providers | 1 | 🔴 High |
| Debuggable flag | YES | 🟡 Medium |

⚠️ Debuggable=true → allows runtime instrumentation and memory inspection.

#### Attack surface summary
<img width="584" height="206" alt="image" src="https://github.com/user-attachments/assets/e3783c51-a9fb-4c10-87ce-8c542f984cce" />

## 🔬 Phase 4 – Component‑level Introspection

### 4.1 Activity inventory
```
drozer
dz> run app.activity.info -a jakhar.aseem.diva
Package: jakhar.aseem.diva
  jakhar.aseem.diva.MainActivity      Permission: null
  jakhar.aseem.diva.APICredsActivity  Permission: null
  jakhar.aseem.diva.APICreds2Activity Permission: null
```

🔴 **Finding** : All three activities are exported="true" with no permission protection.
Any third-party application can launch them arbitrarily.

### 4.2 Service & Receiver scan
```
drozer
dz> run app.service.info -a jakhar.aseem.diva
No exported services.

dz> run app.broadcast.info -a jakhar.aseem.diva
No matching receivers.
```

✅ Services and Broadcast Receivers are properly locked down.

### 4.3 Content Provider discovery
```
drozer
dz> run app.provider.info -a jakhar.aseem.diva
Authority: jakhar.aseem.diva.provider.notesprovider
```

🔴 **Critical finding** : One Content Provider is exported without any permission requirement.

#### Component enumeration results
<img width="779" height="262" alt="image" src="https://github.com/user-attachments/assets/2cf0a123-d515-46a4-84fb-3b5a65af704c" />

<img width="595" height="449" alt="image" src="https://github.com/user-attachments/assets/670e3bbc-f9ff-468b-89f9-487792f028ec" />

## 💥 Phase 5 – Data Exposure Verification (Defensive)

### 5.1 Content Provider unauthorized access test
```
drozer
dz> run app.provider.query content://jakhar.aseem.diva.provider.notesprovider/notes/
Extracted records :
```

| _id | title | note |
|---|---|---|
| 5 | Exercise | Alternate days running |
| 4 | Expense | Spent too much on home theater |
| 6 | Weekend | b33333333333r |
| 3 | holiday | Either Goa or Amsterdam |

🔴 **Severity : CRITICAL**

The Content Provider exposes full read access to all stored notes without any authentication or permission check.
Any malicious app installed on the device or any ADB-connected tool can exfiltrate this data.

#### Data extraction proof
<img width="903" height="185" alt="image" src="https://github.com/user-attachments/assets/336ea4d1-6d16-45e1-8af5-068b1a173fc7" />

## 🕵️ Phase 6 – Static Credential Discovery

### Manual inspection of APICredsActivity
Launching the activity via the launcher reveals:

```
# Vendor API Credentials

API Key: 123secretapikey123
API User name: diva
API Password: p@ssword
```

🔴 **Severity : CRITICAL**

Hardcoded credentials in plaintext.
An attacker with access to the APK (reverse engineering) or physical device access can retrieve these secrets instantly.

#### Plaintext API credentials
<img width="491" height="232" alt="6 (3)" src="https://github.com/user-attachments/assets/2c007486-a033-4de7-9dee-45511a37c333" />

## Phase 7 – Risk Assessment Matrix

| ID | Component | Vulnerability | Likelihood | Impact | Severity |
|---|---|---|---|---|---|
| CR-01 | APICredsActivity | Hardcoded API credentials | High | Critical | 🔴 CRITICAL |
| CR-02 | notesprovider | Unauthenticated data access | High | Critical | 🔴 CRITICAL |
| MD-01 | MainActivity, APICredsActivity, APICreds2Activity | Exported without permissions | Medium | Medium-High | 🟠 HIGH |
| MD-02 | Application manifest | Debuggable flag enabled | Medium | Medium | 🟡 MEDIUM |

## Remediation Roadmap (OWASP MASVS compliant)

| Vulnerability | MASVS Reference | Remediation Action |
|---|---|---|
| Hardcoded credentials | MSTG-STORAGE-6 | Migrate secrets to Android Keystore system or use secure configuration injection (build-time environment variables) |
| Exported Content Provider | MSTG-PLATFORM-2 | Set android:exported="false". If export required, enforce custom signature-level permission |
| Exported Activities | MSTG-PLATFORM-1 | Set android:exported="false" for all non-launcher activities. Keep only MAIN/LAUNCHER exported |
| Debuggable flag | MSTG-CODE-2 | Set android:debuggable="false" in production builds |

## Code correction example (AndroidManifest.xml)
```xml
<!-- Secure Content Provider -->
<provider
    android:name=".NotesProvider"
    android:authorities="jakhar.aseem.diva.provider.notesprovider"
    android:exported="false" />

<!-- Non-exported activities -->
<activity android:name=".APICredsActivity" android:exported="false" />
<activity android:name=".APICreds2Activity" android:exported="false" />

<!-- Production hardening -->
<application
    android:debuggable="false"
    ...>
```

## Final Conclusion
This defensive audit of DIVA using Drozer revealed:

- 2 critical vulnerabilities (unauthenticated Content Provider + hardcoded credentials)
- 2 high/medium findings (exported activities + debuggable flag)
