# Fork, Build and Test Locally

How to host this repository under your own GitHub account, produce a build
archive configured against your own CARE instance, and run it on a local
Windows machine for testing.

The work lives on the `testing` branch of the fork
<https://github.com/muhammed-sawad-egov/care_radiology_dicom_enabler>.

---

## 1. Push to the fork

### 1.1 Set your commit identity

This clone had no `user.name` / `user.email` configured, so commits failed
until they were set (repository-local, not global):

```bash
git config user.name  "Muhammed Sawad"
git config user.email "muhammed.sawad@egovernments.org"
```

### 1.2 Add the fork as a remote and push the testing branch

The existing `origin` points at `care-ecosystem/care_radiology_dicom_enabler`,
which you cannot push to. Add the fork alongside it:

```bash
git remote add mine https://github.com/muhammed-sawad-egov/care_radiology_dicom_enabler.git
git checkout -b testing
git push -u mine testing
```

Git Credential Manager is already configured on this machine, so the first
push opens a browser window to authorise GitHub. If you would rather use a
Personal Access Token, generate one with the **repo** scope at
<https://github.com/settings/tokens> and paste it as the password.

To make the fork the default push target instead of `care-ecosystem`:

```bash
git remote rename origin upstream
git remote rename mine   origin
```

> **The fork is public, and the build archive carries your API token in
> cleartext.** A fork of a public repository cannot be made private. If the
> token is anything other than a throwaway staging value, run the build in a
> separate private repository instead — create one at <https://github.com/new>
> and push this branch there.

### 1.4 A note on the nested `care_radiology/` directory

The working tree contains `care_radiology/`, which is a **separate Git
repository** (the Python CARE backend plugin, from
`care-ecosystem/care_radiology`). It is now listed in `.gitignore`, because
committing a nested repository produces a broken "gitlink" — an entry that
records a commit hash with no way to fetch it, so anyone cloning your
repository gets an empty directory.

If you also need the plugin, push it to its own repository:

```bash
cd care_radiology
git remote add mine https://github.com/muhammed-sawad-egov/care_radiology.git
git push -u mine main
```

Its `trigger-care-build.yml` workflow needs a `CARE_REPO_DISPATCH_TOKEN`
secret to dispatch builds to `care-ecosystem/care`. Without write access to
that repository, delete the workflow or the run will fail on every push.

---

## 2. Configure your CARE settings in GitHub

The build workflow reads the CARE connection settings and bakes them into
`CARE_MWL_Service.exe.config`. Set them once under
**Settings → Secrets and variables → Actions** in your new repository.

### Secrets tab

Values here are masked in workflow logs.

| Secret | Example | Purpose |
|---|---|---|
| `CARE_BASE_URL` | `https://care.example.org` | Your CARE instance, no trailing slash |
| `CARE_API_TOKEN` | *your radiology plugin token* | Authenticates worklist and webhook calls |

### Variables tab

These appear in the artifact filename, so they must **not** be secrets —
GitHub would mask them to `***` and corrupt the name.

| Variable | Example | Purpose |
|---|---|---|
| `CARE_BACKEND` | `2` | `0` = static list, `1` = database, `2` = CARE server |
| `CARE_MODALITY` | `CT` | Modality filter; leave unset for all modalities |
| `CARE_FROM_DATE` | `2025-01-01 08:00:00` | Earliest worklist entry to fetch |

Set `CARE_BACKEND` to `2` — that is the mode which queries your CARE server.

### Precedence

Each setting resolves to the first non-empty value of:

1. The `workflow_dispatch` input, for a one-off manual run
2. The repository secret or variable, as the standing default
3. The value committed in `CARE_MWL_Service/App.config`

So the secrets apply automatically to every push build, and you can still
override any of them for a single manual run without changing them.

---

## 3. Produce the build archive

1. Go to the **Actions** tab → **Build CARE DICOM Enabler** → **Run workflow**.
2. Leave the CARE fields blank to use your secrets and variables.
3. When the run finishes, download the artifact from the run summary page.

The artifact is a zip named
`CARE_DICOM_Enabler-Release-AnyCPU-custom-CT-CARE-main-<run>-<sha>`
containing the whole `bin\Release\` tree: the WinForms UI, the four Windows
services, `cfg\common.cfg` and an empty `logs\` directory.

> The API token is written in cleartext into `CARE_MWL_Service.exe.config`
> inside this archive, and GitHub Actions artifacts are downloadable by anyone
> with read access to the repository. Keep the repository private, and treat
> the archive as a credential-bearing file.

### Building locally instead

A local build needs **MSBuild v15+** — this machine has only the .NET
Framework 4.0 MSBuild, which cannot build these `ToolsVersion="15.0"`
projects. Install either Visual Studio 2019+ or the standalone
[Build Tools for Visual Studio](https://visualstudio.microsoft.com/downloads/)
with the *.NET desktop build tools* workload, then:

```powershell
nuget restore CARE_DICOM_Enabler.sln
msbuild CARE_DICOM_Enabler.sln /p:Configuration=Release /p:Platform="Any CPU" /m
```

All ten projects output to the shared `bin\Release\` directory.
The .NET SDK 10 already installed cannot substitute — `dotnet build` does not
support these legacy `.csproj` files.

---

## 4. Run it on your local machine

### 4.1 Prerequisites

* **Windows**, with an **Administrator** shell — the services register with
  the Windows Service Control Manager
* **MySQL 8.0+** — the services log studies, series and patients locally
* **.NET Framework 4.7.2** runtime (present on Windows 10/11 by default)
* **DCMTK** (optional, for testing) — `choco install dcmtk`

### 4.2 Create the database

```powershell
mysql -u root -p < ci\schema.sql
```

This creates the `plexus_mi2` schema with the `study`, `series`, `patient`
and related tables.

### 4.3 Point `cfg\common.cfg` at your database

`cfg\common.cfg` ships with an **AES-encrypted connection string for the
CARE staging environment**, which will not work against your MySQL instance.
The encryption is AES-128-CBC, zero IV, PKCS7, with the key
`3DEA271411CD4AA0AC1499ACF35B0A9E` (visible in
`.github/workflows/integration-test-staging.yml`).

The supported way to regenerate it is the **GenerateConnectionString**
project included in the solution: run it, enter your server, database, user
and password, use *Test Connection* to confirm, then save — it writes the
encrypted value into `cfg\common.cfg`.

To regenerate it by hand, encrypt
`Server=localhost;Database=plexus_mi2;Uid=root;Pwd=yourpassword;`
with those parameters and replace the `<connectString>` element.

Also update in `cfg\common.cfg`:

| Element | Set to |
|---|---|
| `authURL` | `https://care.example.org/api/token/` — your instance |
| `mwlport` | `2008` — the port modalities query for the worklist |
| `sscpport` | `2007` — the port modalities send images to |
| `sscuhost` / `sscuport` | Your PACS, if forwarding images onward |
| `uname` / `pwd` | Encrypted app credentials, via the same tool |

### 4.4 Install and start the services

Unzip the archive, then from an **Administrator** PowerShell inside the
extracted `bin\Release\` folder, launch `CARE_DICOM_Enabler.exe`, log in, and
use the **Server Manager** screen to install and start the services. That is
the path `SETUP.md` documents and the one to prefer.

To do it directly instead:

```powershell
New-Service -Name CAREMWL      -BinaryPathName "$PWD\CARE_MWL_Service.exe"      -StartupType Manual
New-Service -Name CAREStoreSCP -BinaryPathName "$PWD\CARE_StoreSCP_Service.exe" -StartupType Manual
Start-Service CAREMWL, CAREStoreSCP
```

Confirm both are listening:

```powershell
Get-Service CAREMWL, CAREStoreSCP
netstat -an | Select-String ":200[78]\s.*LISTENING"
```

### 4.5 Verify end to end

```powershell
# Reachability
echoscu -v localhost 2008 -aet TESTSCU -aec MODALITYSCP   # worklist service
echoscu -v localhost 2007 -aet TESTSCU -aec STORAGESCP    # storage service

# Fetch the worklist your CARE instance is serving
findscu -v -W -k "0008,0050=" -k "0010,0010=" -k "0010,0020=" `
  -k "0040,0100[0].0008,0060=CT" `
  localhost 2008 -aet TESTSCU -aec MODALITYSCP
```

A successful `findscu` returns patient and accession values from your CARE
instance. Then check the logs:

```powershell
Get-Content logs\WorklistItems*.txt -Tail 50
```

Look for `Successfully fetched N worklist items`.

### 4.6 Common failures

| Symptom | Cause |
|---|---|
| `403 Forbidden` in `WorklistItems*.txt` | `careToken` wrong, or lacks radiology permissions |
| `Successfully fetched 0 worklist items` | Token is valid; no entries match your modality filter or `careFromDate` |
| Service starts then stops immediately | `cfg\common.cfg` connection string cannot reach MySQL |
| Port 2007/2008 not listening | Service failed to start, or another process holds the port |
| `findscu` reports association rejected | Called AE title does not match `mwlaetitle` in `cfg\common.cfg` |

Service startup failures are recorded in `logs\*.txt` next to the executable,
and in Windows Event Viewer under **Windows Logs → Application**.

---

## 5. Credentials already committed to this repository

These values are in the Git history of the upstream repository. Replace them
with your own, and treat the originals as compromised rather than as
defaults to keep.

| File | Value |
|---|---|
| `CARE_MWL_Service/App.config` | `careToken` = `RADOMSECRET`, `careBaseUrl` → staging |
| `App.config` | MySQL password `inzin@123` in `connectionstring` |
| `Sample_ModalitySCP/App.config` | Same MySQL password |
| `cfg/common.cfg` | Encrypted connection string, `uname`, `pwd` for staging |
| `.github/workflows/integration-test-staging.yml` | AES key `3DEA271411CD4AA0AC1499ACF35B0A9E`; default token `RADOMSECRET` |

Because the AES key is committed alongside the ciphertext it protects, the
encrypted values in `cfg/common.cfg` should be considered readable by anyone
with the repository. Rotate any credential you reuse elsewhere.
