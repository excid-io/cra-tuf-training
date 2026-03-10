# 🛠️ Module 3 – Hands-on: Setting up a TUF Repository

## Overview
In this module we apply the concepts learned in the previous modules by setting up a **working repository based on The Update Framework** using **[TUF-on-CI](https://github.com/theupdateframework/tuf-on-ci/)**.

We will build a secure update repository where different roles are implemented using practical tools and real infrastructure:

- The **timestamp role** is implemented automatically through a **GitHub Action** using one-time signing keys provided by **[SigStore](https://www.sigstore.dev/)**.  
  This approach requires no manual configuration, although it may not yet be supported by all TUF clients. As alternatives, TUF-on-CI also supports **Google Cloud KMS, Azure Key Vault, and AWS KMS**.

- The remaining roles (such as **root** and **targets**) are implemented locally by a user who signs metadata using **keys stored in a YubiKey**.

This module demonstrates how a secure update repository can be created, initialized, and used to publish signed targets.

---



## 📚 Learning Objectives

After completing this module, you will be able to:

- Set up a **The Update Framework repository using TUF-on-CI**
- Configure **GitHub Actions** to manage repository metadata
- Use **YubiKey hardware keys** to securely sign metadata
- Initialize repository roles and trust configuration
- Add and sign **target files** for distribution

---

# 🧰 Hands-on Setup Guide

## Preparation

Create a new repository using the **tuf-on-ci template**:

👉 https://github.com/new?template_name=tuf-on-ci-template&template_owner=theupdateframework

Then perform the following configuration changes in GitHub:

- Set **Settings → Pages → Source** to **GitHub Actions**
- Go to **Settings → Environments → github-pages**
  - In **Deployment branches and tags**, change `main` to `publish`
- Go to **Settings → Actions → General**
  - Ensure **Allow GitHub Actions to create and approve pull requests** is enabled

---

# 🔐 YubiKey Preparation

We will now configure the **YubiKey** to store signing keys.

1. Install **YubiKey Manager**  
   https://www.yubico.com/support/download/yubikey-manager/

2. Open **YubiKey Manager**

3. Navigate to:

```
Applications → PIV
```

4. In **Certificates**, select **Configure Certificates**

5. Select **Digital Signature → Generate**

Then choose:

- **Self-signed certificate**
- An **algorithm**
- A **subject** (e.g., your GitHub username)
- An **expiration date**

If not already installed, install:

👉 https://developers.yubico.com/yubico-piv-tool/

---

# 🔧 Install the TUF Signing Tool

Install the **TUF-on-CI signing tool** (Python package):

```bash
pip install tuf-on-ci-sign
```

---

# 📂 Local Repository Preparation

Clone your repository and create the configuration file:

```
.tuf-on-ci-sign.ini
```

Example configuration:

```ini
[settings]

# Path to PKCS#11 module (optional)
# If not provided, tuf-on-ci-sign will probe known locations
# pykcs11lib = /usr/lib/x86_64-linux-gnu/libykcs11.so

# GitHub username
user-name = @my-github-username

# Git remote used for pulling repository updates
pull-remote = origin

# Remote used for pushing changes
push-remote = origin
```

---

# 🚀 Repository Initialization

Inside the cloned repository run:

```bash
tuf-on-ci-delegate sign/init
```

During the configuration:

- For **Configuring role root**, select **1**
- Press **Enter** to set your GitHub handle as:
  - **root role**
  - **targets role**

For **online roles**, keep the default **SigStore** configuration.

For **signing key configuration**, select:

```
2 – YubiKey
```

Then follow the instructions.

After initialization:

1. Go to **GitHub**
2. Merge the **generated pull request**

After a few minutes the repository becomes available online.

Example metadata repository:

```
https://excid-io.github.io/tuf-on-ci-example/metadata/
```

---

# 📦 Adding and Signing a Target

Assume a developer wants to publish a new file:

```
manifest.json
```

### Step 1 – Create a signing branch

Create a branch whose name starts with:

```
sign/
```

Example:

```
sign/manifest
```

### Step 2 – Add the file

The developer:

1. Clones the repository
2. Checks out the new branch
3. Adds the target file in the **targets directory**
4. Commits and pushes the change

At this stage the file **is not yet part of the repository** because it must still be signed.

### Step 3 – Sign the target

The signer runs:

```bash
tuf-on-ci-sign sign/manifest
```

The signature is processed by the corresponding **GitHub Action**, which creates a **pull request**.

Once the pull request is merged, the repository is updated and a new **targets metadata file** is generated.

Example:

```
https://excid-io.github.io/tuf-on-ci-example/metadata/2.targets.json
```

---

## 🔗 End of Training

You have now:

- Learned the **theory of secure updates**
- Explored **The Update Framework**
- Built a **real secure update repository**

You are now ready to integrate **secure software updates** into your own products and infrastructure.
