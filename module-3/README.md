# 🛠️ Module 3 – Hands-on: Setting up a TUF Repository

## Overview
In this module we apply the concepts learned in the previous modules by setting up a working repository based on The Update Framework using **[TUF-on-CI](https://github.com/theupdateframework/tuf-on-ci/)**.

We will build a secure update repository where different roles are implemented using practical tools and real infrastructure:

- The **timestamp** and **snapshot** roles are implemented automatically through 
a GitHub Action. The GitHub action can sign the corresponding metadata using one 
of the following methods:
  - Using **[SigStore](https://www.sigstore.dev/)**. This approach requires no 
  additional configuration. However it is experimental and it is has limited support by TUF clients
  - Using an online Key Management System (KMS). TUF-on-CI  supports **Google Cloud KMS, Azure Key Vault, and AWS KMS**. In this module we will use  Azure Key Vault.
- The remaining roles (such as **root** and **targets**) are implemented locally 
by a user. Similarly, the user can sign the corresponding metadata using either 
SigStore or a **YubiKey**.  In this module we will use Yubikey.

This module demonstrates how a secure update repository can be created, initialized, 
and used to publish signed targets.

---



## 📚 Learning Objectives

After completing this module, you will be able to:

- Set up a **The Update Framework repository using TUF-on-CI**
- Configure **GitHub Actions** to manage repository metadata
- Use **Azure Key Vault** and **YubiKey hardware keys** to securely sign metadata
- Initialize repository roles and trust configuration
- Add and sign **target files** for distribution

---

# Hands-on Setup Guide

## 🔧 Preparation

### Azure Key Vault 

We will use Azure Key Vault for storing the keys for timestamp and snapshot roles. 
These roles are implemented by a GitHub action, therefore we need enable remote access
to a signing key. It is assumed that you have already setup an **Elliptic Curve**
signing key (RSA signing keys are not supported by TUF-on-CI). Remote access to this
key can be enabled following these steps:

* Open the **Azure portal** and go to **Microsoft Entra ID**
* In the **Manage** menu, select **App registrations** and click **New registration**.
* Give it a **Name** such as *github-tuf-on-ci* and click **Register**
* On the app’s Overview page, copy *Application (client) ID*  and *Directory (tenant) ID* 
* Separately, go to **Subscriptions** (you can type subscriptions in the search box), 
open the subscription that contains your key vault, and copy the *Subscription ID* 

Next, is to tell Azure to trust OIDC tokens coming from your GitHub repository. In
 Azure Portal, this is done by adding a federated credential to the app registration 
under Certificates & secrets. In more detail:

* Open the **Azure portal** and go to **Microsoft Entra ID**.
* In the **Manage** menu, select **App registrations** and then **View all applications in the directory**
* Select the application you created with the previous step and in the Manage menu
on the left select **certificates & secrets**.
* Click on **Add credential** and in the list of scenarios select **Github Actions deploying Azure resources**
* Fill in your **Organization** and **Repository**. For example, the corresponding values for our
demo repository (https://github.com/excid-io/cra-tuf-training-example) are *excid-io*
and *cra-tuf-training-example*. 
* In the **Entity type** select *Branch* and in the **Based on selection** textbox
type main. The subject identifier field should look like this: `repo:excid-io/cra-tuf-training-example:ref:refs/heads/main`
* Fill a **Name** and an **Organization** and click **Add**

As a final step we have to assign the *Key Vault Crypto User* role to the application
we created. First verify that you key vault used RBAC:
* Open your key vault in Azure Portal.
* In the **Settings** menu select **Access configuration**
* Make sure the permission model is Azure role-based access control

Then,  in the key vault:
* Open **Access control (IAM)**
* Click **Add** → **Add role assignment**
* Search for and select *Key Vault Crypto User*.
* Click **Next**
* For member selection, choose **User, group, or service principal**
* Click **Select members**
* Search for the app registration you created in Step 1 and select it.
* Click Review + assign.  ￼


### GitHub repository

Create a new repository using the **tuf-on-ci template**:

👉 https://github.com/new?template_name=tuf-on-ci-template&template_owner=theupdateframework

Then, perform the following configuration changes in GitHub:

- Set **Settings → Pages → Source** to **GitHub Actions**
- Go to **Settings → Environments → github-pages**
  - In **Deployment branches and tags**, change `main` to `publish`
- Go to **Settings → Actions → General**
  - Ensure **Allow GitHub Actions to create and approve pull requests** is enabled

**If you are using Azure Key Vault perform the following additional steps:**
- Go **Settings** → **Secretes** and **Variables** → **Actions** and in the **Repository secrets** 
select **New repository secret** 
- You need to create the following three secrets:
  - `AZURE_CLIENT_ID` Put here your *Application (client) ID*  
  - `AZURE_TENANT_ID` Put here your *Directory (tenant) ID* 
  - `AZURE_SUBSCRIPTION_ID` Put here your *Subscription ID* 

Then clone your repository and edit `.github/workflows/online-sign.py` and modify `online-sign` job as follows
(you can also view our demo file [here](https://github.com/excid-io/cra-tuf-training-example/blob/main/.github/workflows/online-sign.yml)):

```yml
...
jobs:
  online-sign:
    runs-on: ubuntu-latest
    permissions:
      id-token: write # for OIDC identity access
      contents: write # for committing snapshot/timestamp changes
      actions: write # for dispatching publish workflow
    steps:
      - name: Login to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
...
```

 Commit and push the modified workflow. 


### Local environment - YubiKey Preparation

We will now configure the **YubiKey** to store signing keys.

- Install **YubiKey Manager**  (https://www.yubico.com/support/download/yubikey-manager/)
- Open **YubiKey Manager**
- Navigate to **Applications** → **PIV**
- In **Certificates**, select **Configure Certificates**
- Select **Digital Signature → Generate**

Then choose:

- **Self-signed certificate**
- An **algorithm**
- A **subject** (e.g., your GitHub username)
- An **expiration date**

If not already installed, install:

👉 https://developers.yubico.com/yubico-piv-tool/

---

### Local environment - signing tools

Install the **TUF-on-CI signing tool** (Python package):

```bash
pip install tuf-on-ci-sign
```

**If you are using Azure Key Vault perform the following additional steps:**
* Install  [azure-cli](https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)
* Execute 

```bash
az login
```

`azure-cli` is used by local tools to retrieve the public key from the Azure Key Vault

---

# 📂 Local Repository Preparation

Clone your repository and inside the cloned repository create a configuration file named `.tuf-on-ci-sign.ini`

Then add the following content:

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

- For **Configuring role root**, press *Enter*
- For **Configuring role targets**, press *Enter*
- Configuring online roles, if you use Azure Key vault press *1*
  - Press *3* to select *Azure Key Vault*
  - Provide your *Azure vault name*
  - Provide your *key name*
- Press *Enter* to continue
- For **Configuring signing key** press *2* to select Yubikey and press *Enter*. Follow
the instructions in order for the initial metadata files to be singed


After initialization:

- Go to **GitHub**
- Merge the **generated pull request**

After a few minutes the repository becomes available online.

Example metadata repository:

```
https://excid-io.github.io/cra-tuf-training-example/metadata/
```

---

# 📦 Adding and Signing a Target

Assume a developer wants to publish a new file: `manifest.json`

### Step 1 – Create a signing branch

Create a branch whose name starts with: `sign/`, for example, `sign/manifest`


### Step 2 – Add the file

The developer:

- Clones the repository
- Checks out the new branch
- Adds the target file in the **targets directory**
- Commits and pushes the change

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
https://excid-io.github.io/cra-tuf-training-example/metadata/2.targets.json
```

---

## 🔗 End of Training

You have now:

- Learned the **theory of secure updates**
- Explored **The Update Framework**
- Built a **real secure update repository**

You are now ready to integrate **secure software updates** into your own products and infrastructure.
