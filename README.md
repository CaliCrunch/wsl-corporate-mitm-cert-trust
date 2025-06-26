# SSL Certificate Bypass for MITM Proxies in WSL & Linux

Best-practice steps for importing and trusting your organization's MITM (Man-in-the-middle) SSL certificate on any Linux distribution inside WSL (Windows Subsystem for Linux) or on a standalone machine. Especially useful for avoiding `self-signed-certificate`-related errors in tools like `curl`, `npm`, `git`, package managers, etc.  

> These instructions assume you control or trust the proxy certificate you're installing.

---

## Table of Contents

1. [Prerequisites](#-prerequisites)  
2. [Step 1: Export Your MITM Certificate](#-step-1-export-your-mitm-certificate)  
3. [Step 2: Trust on Windows (Optional for WSL)](#-step-2-trust-on-windows-optional-for-wsl)  
4. [Step 3: Import into Linux / WSL](#-step-3-import-into-linux--wsl)  
   - [Debian & Ubuntu](#debian--ubuntu)  
   - [RHEL, CentOS & Fedora](#rhel-centos--fedora)  
   - [openSUSE & SLE](#opensuse--sle)  
   - [Arch Linux & derivatives](#arch-linux--derivatives)  
   - [Alpine Linux](#alpine-linux)  
5. [Step 4: Configure Common Tools](#-step-4-configure-common-tools)  
   - [Git](#git)  
   - [npm / Node.js](#npm--nodejs)  
   - [Python / pip](#python--pip)  
   - [Java (keytool)](#java-keytool)  
   - [curl / wget](#curl--wget)  
6. [Verification & Troubleshooting](#-verification--troubleshooting)  
7. [Cleanup / Uninstall](#-cleanup--uninstall)  
8. [License](#-license)  

---

## Prerequisites

- **Root / sudo** access on your Linux / WSL distro  
- The MITM proxy's public certificate in PEM format (`.cer`, `.crt`, or you’ll convert to `.crt`)  

---

## 1. Export Your MITM Certificate

1. **From Windows (corporate environments):**  
   2. Using browser 
      - Open **Chrome** (or really any modern browser), navigate to any HTTPS site you know is MITM'd.  
      - Click the padlock --> **Certificate** --> **Details** --> **Copy to File…** --> export as **Base-64 encoded X.509 (.CER)** --> save `corp-proxy.cer`.  
   3. Using certificate managers
      - Open up `certlm` or `certmgr` depending on where the MITM is.
      - Right-click on the certificate --> **Export...** --> Export as a `.cer` file. 

2. **From Linux / WSL directly (if you know host:port):**  
   ```bash
   openssl s_client -showcerts -connect proxy.company.com:443 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > corp-proxy.pem
   ```
   Rename `corp-proxy.pem` --> `corp-proxy.crt` for consistency.

---

## 2. Trust on Windows (Optional for WSL)

> WSL doesn’t auto-inherit the Windows store, but you may still want Windows tools to trust it too.

```powershell
# Run PowerShell as Administrator:
certutil -addstore -user ROOT "C:\path\to\corp-proxy.cer"
```

---

## 3. Import into Linux / WSL

Copy your `corp-proxy.crt` into the Linux filesystem. E.g.:

```bash
# from Windows path C:\Users\You\Downloads
cp /mnt/c/Users/You/Downloads/corp-proxy.crt ~/
```

Then follow your distro's instructions:

### Debian & Ubuntu

```bash
sudo cp ~/corp-proxy.crt /usr/local/share/ca-certificates/
sudo chmod 644 /usr/local/share/ca-certificates/corp-proxy.crt
sudo update-ca-certificates
```

### RHEL, CentOS & Fedora

```bash
sudo cp ~/corp-proxy.crt /etc/pki/ca-trust/source/anchors/
sudo chmod 644 /etc/pki/ca-trust/source/anchors/corp-proxy.crt
sudo update-ca-trust extract
```

### openSUSE & SLE

```bash
sudo cp ~/corp-proxy.crt /etc/pki/trust/anchors/
sudo chmod 644 /etc/pki/trust/anchors/corp-proxy.crt
sudo update-ca-certificates
```

### Arch Linux & derivatives

```bash
sudo mkdir -p /etc/ca-certificates/trust-source/anchors/
sudo cp ~/corp-proxy.crt /etc/ca-certificates/trust-source/anchors/
sudo trust extract-compat
```

### Alpine Linux

```bash
sudo cp ~/corp-proxy.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

---

## 4. Configure Common Tools

Even after updating the OS trust store, some tools need explicit config:

### Git

```bash
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
```

### npm / Node.js

```bash
npm config set cafile "~/corp-proxy.crt"
# Or via env var
export NODE_EXTRA_CA_CERTS=~/corp-proxy.crt
```

### Python / pip

```ini
# ~/.pip/pip.conf
[global]
trusted-host =
    pypi.org
    files.pythonhosted.org
cert = /home/you/corp-proxy.crt
```

### Java (keytool)

```bash
sudo keytool -import -trustcacerts \
  -alias corp-proxy -file ~/corp-proxy.crt \
  -keystore /etc/ssl/certs/java/cacerts \
  -storepass changeit
```

### curl / wget

```bash
curl --cacert ~/corp-proxy.crt https://internal.site/
wget --ca-certificate=~/corp-proxy.crt https://internal.site/
```

---

## Verification & Troubleshooting

- **Verify OS store:**  
  ```bash
  openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt ~/corp-proxy.crt
  ```
- **Test with curl:**  
  ```bash
  curl -v https://internal.site/ 2>&1 | grep "verify"
  ```
- Double-check file permissions (should be world-readable) and that you ran the correct update command.

---

## Cleanup / Uninstall

1. Remove the cert file from the anchors directory.  
2. Rerun the update command (`update-ca-certificates` or `update-ca-trust extract`, etc.).  
3. Remove any tool-specific configs (Git, npm, pip).

---