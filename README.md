# Android-SSL-Pinning-Bypass-Guide
A comprehensive guide on how to bypass SSL pinning on a rooted Android device using Frida, Objection, and Caido. This procedure involves moving CA certificates to the system store and injecting scripts into the target application.

<img width="1176" height="630" alt="imagen" src="https://github.com/user-attachments/assets/46251b0a-2bac-4ea8-9688-aa6834654718" />


## Prerequisites
- Rooted Android Device.

- Linux Workstation.

- ADB (Android Debug Bridge) installed.

- Caido or Burp Suite for traffic interception.

- Frida-tools & Objection installed on your PC.

___

## Phase 1: Installing CA Certificate as a System Root

Android 7.0+ ignores user-added certificates for most apps. We need to "trick" the system into believing our proxy certificate is a trusted Authority.

### 1.1 Generate the "Magic" Hash

If your certificate is in PEM format (e.g., ca.crt), run:
``` bash
openssl x509 -inform PEM -subject_hash_old -in "ca.crt" | head -1
```
Note the 8-character output (e.g., 9a12b3c4).

### 1.2 Prepare and Push the Certificate
``` bash
# Rename the cert using the hash obtained above
cp "ca.crt" HASH.0

# Push it to the device storage
adb push HASH.0 /sdcard/
```
### 1.3 Move to System Store (The "Attack")

Access the device shell to move the certificate to the trusted directory:
``` bash
adb shell
su

# Remount the system partition as Read-Write (RW)
mount -o rw,remount /

# Move the cert to the system cacerts folder
cp /sdcard/HASH.0 /system/etc/security/cacerts/

# Set correct permissions (Read for all, Write for root)
chmod 644 /system/etc/security/cacerts/HASH.0

# Reboot to apply changes
reboot

```
___

## Phase 2: Proxy Configuration (Caido)

1. Mobile Settings: * Long press your Wi-Fi > Modify Network.

    - Advanced options > Proxy > Manual.

    - Hostname: Your PC IP (e.g., 192.168.3.18).

    - Port: 8080.

2. Caido Setup: * Go to Instances > Edit/Create new instance.

    - Change the listening address from 127.0.0.1 to 0.0.0.0.
  
    - Activate Enable invisible proxying

    - Ensure the interceptor is active (Green/Intercepting).

___

## Phase 3: Frida-Server Setup

### 3.1 Download and Prepare
Download the latest frida-server-*-android-arm64.xz from the [Frida Releases](https://github.com/frida/frida/releases).
``` bash
# Decompress the file
unxz frida-server-*-android-arm64.xz

# Rename for simplicity and set execution permissions
mv frida-server-*-android-arm64 frida-server
chmod +x frida-server

# Upload to the device
adb push frida-server /data/local/tmp/
```
### 3.2 Execution
``` bash
adb shell
su
chmod 755 /data/local/tmp/frida-server

# Run it in the background
/data/local/tmp/frida-server &
```
To verify it is working, run ```frida-ps -U``` on your PC. You should see a list of running processes.

___

At this point, your mitm is running, but the certificate pinning of the app will block you, thats why we do the next part, the theory of the ssl pinning will be in at the end of this repo.    

## Phase 4: Bypassing SSL Pinning
With everything in place, we use Objection to hook into the app and disable the pinning logic.
1. Launch the Objection console:
   ``` bash
   objection -g "app process" explore
   ```
   Rememeber, at this point you should have Frida installed so you can run ```bash frida-ps -U``` to see the process running.
2. Execute the bypass command:
   Once the "app.process" on (android) prompt appears, type:
   ``` bash
   android sslpinning disable
   ```
___

## Troubleshooting
- Read-only file system: Some newer Android versions require adb disable-verity before remounting /system.
- Architecture mismatch: Ensure you are using arm64 for modern devices. x86 is only for emulators.
- Frida Connection: Ensure the version of frida on your PC matches the version of frida-server on the mobile.

___
# Theory of the mitm

<img width="903" height="576" alt="imagen" src="https://github.com/user-attachments/assets/e5ca6d59-a6c3-4380-bfa2-917c5409647f" />

To understand why we need this complex setup, we must understand how Android handles trust and how applications defend themselves, so lets take a deep breath in the theory.

### 1. Standard TLS Trust Model
Normally, when your phone connects to a server (e.g., `google.com`), a **Hierarchy of Trust** occurs:
1.  **The Handshake:** The server sends its public certificate.
2.  **The Verification:** Android checks its **System Trust Store** (`/system/etc/security/cacerts/`) to see if a trusted Certificate Authority (CA), like DigiCert or Let's Encrypt, signed that certificate.
<img width="767" height="289" alt="imagen" src="https://github.com/user-attachments/assets/e079bd1a-c1f8-4508-b5a8-9094188cfb56" />

3.  **The Green Light:** If the signature is valid and the CA is trusted, the encrypted connection is established.



### 2. The Android 7.0+ Barrier
Before Android 7, you could simply install a proxy certificate as a "User Certificate". Now, Android ignores user-installed CAs for almost all apps. This is why **Phase 1** of this guide focuses on moving the certificate to the **System Store** using Root access. This forces the OS to treat our proxy (Caido) as a globally trusted authority.

### 3. What is SSL Pinning?
Even if the OS trusts our certificate, high-security apps implement **SSL Pinning**. This is a "Zero Trust" approach:
- **Standard Behavior:** "I trust anyone the System Trust Store trusts."
- **SSL Pinning:** "I don't care who the System trusts. I only trust **THIS** specific certificate (or public key) hardcoded inside my binary."

When the app detects the proxy certificate, it sees that it doesn't match the "pinned" one and kills the connection immediately to prevent a Man-In-The-Middle (MITM) attack.
(Note: If you launch the app before completing Phase 4, you will likely encounter a connection error. Testing this is recommended to confirm the app implements SSL Pinning; if it doesn't, this lack of protection could be considered a security vulnerability)


### 4. How Frida Breaks the Logic
**Frida** is a dynamic instrumentation toolkit. Instead of modifying the app's source code (which is hard and easily detectable), we inject ourselves into the app's process in **RAM** while it is running.

When we run `android sslpinning disable` via **Objection**, we are performing **Function Hooking**:
1.  We locate the internal functions responsible for certificate validation (e.g., `checkServerTrusted` in Java or `OkHttp`'s `CertificatePinner`).
2.  We overwrite these functions in real-time. 
3.  When the app asks: *"Is this certificate valid?"*, our injected script intercepts the question and always returns **TRUE**, effectively blinding the app's security sensors.

### 4.1 Why Frida-server?
There are two ways to bypass SSL Pinning: Static (modifying the APK file) or Dynamic (using Frida). We choose Frida for several key reasons:

  - Bypassing Integrity Checks: Many modern apps have "Self-Checksum" or "Signature Verification" mechanisms. If you modify the APK file statically (repackaging), the app will detect it and refuse to run. Frida operates in RAM, so the original APK file remains untouched and "legit" in the eyes of the OS.
  - No Repackaging Hassle: Decyling, modifying Smali code, and resigning an APK is time-consuming and prone to errors. Frida allows us to achieve the same result in seconds with a single command.
  - Versatility: One single Frida script can target multiple libraries at once. Whether the app uses OkHttp, TrustManager, or Network Security Configuration, Frida hooks into all of them simultaneously.
  - Volatile and Stealthy: Since the bypass only exists in memory while Frida is running, it doesn't leave permanent "scars" on the application. Stopping the Frida-server restores the app to its original state.


