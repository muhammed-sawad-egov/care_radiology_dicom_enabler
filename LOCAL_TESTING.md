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
**Settings → Secrets and variables → Actions**.

Secrets and variables are per-repository, so these must be set on **your
fork**, not upstream. Enable Actions first (§3) or the pages are inert.

### Secrets tab

<https://github.com/muhammed-sawad-egov/care_radiology_dicom_enabler/settings/secrets/actions>
→ **New repository secret**. Values here are masked in workflow logs.

| Secret | Example | Purpose |
|---|---|---|
| `CARE_BASE_URL` | `https://care.example.org` | Your CARE instance, no trailing slash |
| `CARE_API_TOKEN` | *see below* | Authenticates worklist and webhook calls |

**`CARE_API_TOKEN` must be the complete `Authorization` header value.** Both
call sites add it to the header verbatim, with no scheme prepended:

```csharp
// WorklistItemsProvider.cs:332 and MppsHandler.cs:76
client.DefaultRequestHeaders.Add("Authorization", token);
```

So if your CARE instance expects `Authorization: Bearer abc123`, the secret
must contain `Bearer abc123`, not `abc123`. Storing the bare token is the
most common cause of a `403 Forbidden` in `logs\WorklistItems*.txt`.

**`CARE_BASE_URL` must not end in a slash.** It is string-concatenated at
`WorklistItemsProvider.cs:322`, so a trailing slash produces a double slash
in the request path.

### Variables tab

<https://github.com/muhammed-sawad-egov/care_radiology_dicom_enabler/settings/variables/actions>
→ **New repository variable**.

These appear in the artifact filename, so they must **not** be secrets —
GitHub would mask them to `***` and corrupt the name.

| Variable | Example | Purpose |
|---|---|---|
| `CARE_BACKEND` | `2` | `0` = static list, `1` = database, `2` = CARE server |
| `CARE_MODALITY` | `CT` | Modality filter; leave unset for all modalities |
| `CARE_FROM_DATE` | `2025-01-01 08:00:00` | Earliest worklist entry to fetch |

Set `CARE_BACKEND` to `2`. It selects the mode that queries your CARE server,
and `MppsHandler.cs:42-46` returns early for any other value — so with the
wrong backend the MPPS status webhooks are skipped silently, with no error.

### Confirming they were picked up

The **Patch CARE_MWL_Service App.config** step in the run log prints each
value it applied, with the token reduced to a length:

```
  - Backend: 2
  - Server URL: https://care.example.org
  - API Token: (set, 47 chars)
  - Modality: CT
```

`No CARE settings supplied - keeping App.config defaults.` instead means
nothing resolved — the names are misspelled, or they were added to the wrong
tab, or to the upstream repository rather than your fork.

### Which files hold the server URL

**Two services call CARE, through differently named keys in separate config
files.** This matters because setting only the worklist URL leaves image
upload pointing at the shared staging server, and the upload still succeeds —
so nothing looks wrong until you notice studies arriving on the wrong host.

| Service | Key | Drives |
|---|---|---|
| `CARE_MWL_Service` | `careBaseUrl` / `careToken` | Worklist C-FIND, MPPS status webhook |
| `CARE_SCU_Service` | `careBackendURL` / `staticAPIKey` | DICOM image upload, study webhook |

Both send the credential as a bare `Authorization` header with no scheme
prepended (`WorklistItemsProvider.cs:332`, `MppsHandler.cs:76`,
`Plexus_SCU_Service.cs:139` and `:213`), so a single `CARE_API_TOKEN` secret
covers `careToken` and `staticAPIKey` alike.

The root `App.config`, which becomes `CARE_DICOM_Enabler.exe.config`, carries
a third copy of `careBackendURL` and `staticAPIKey` that no code reads. The
build patches it anyway so the archive does not ship the staging URL and
token in a file someone might later wire up.

`careBackendURL` is `TrimEnd('/')`-ed at `Plexus_SCU_Service.cs:61`, so a
trailing slash is harmless there. `careBaseUrl` is not — see §2 above.

### Repointing an install you have already extracted

These are plain XML files sitting beside the executables, so there is no need
to rebuild. In the extracted folder, edit **both**:

| File | Key to change |
|---|---|
| `CARE_MWL_Service.exe.config` | `careBaseUrl`, `careToken` |
| `CARE_SCU_Service.exe.config` | `careBackendURL`, `staticAPIKey` |

Then restart the services so .NET re-reads them — `ConfigurationManager`
loads the file once at process start:

```powershell
Restart-Service CAREMWL, CARESCU
```

Confirm from the logs that the upload target changed:

```powershell
Select-String -Path logs\*.txt -Pattern "Uploading to" | Select-Object -Last 3
```

Any line still naming `staging.carehmis.dpdns.org` means the SCU service is
running with the old config, or was not restarted.

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
services, `cfg\common.cfg`, an empty `logs\` directory, and `Emulator\` with
whatever that folder holds — currently just `Modality-Emulator-3.1.5.0.zip`.

> The build copies every file present in `Emulator\`, rather than naming
> `Initializer.bat`, `Initializer.ps1` and `schema.sql` individually. Those
> three are not in the repository yet, and GitHub Actions runs `pwsh` with
> `$ErrorActionPreference = 'stop'`, so a `Copy-Item` naming a missing file
> fails the step and no artifact is produced at all. The step logs a warning
> for each absent initializer instead. Once the scripts are committed to
> `Emulator\`, they are picked up automatically with no workflow change.

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
