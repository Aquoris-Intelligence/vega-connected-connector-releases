# Vega Connected connector for Windows

The connector links a clinic's WinClinic records to Vega Connected. It runs as a Windows
service on a PC at the clinic, reads the WinClinic database with a read-only login and
sends the records to Vega over HTTPS. It opens no inbound ports and never changes
anything in WinClinic.

Download it from the [releases page](https://github.com/Aquoris-Intelligence/vega-connected-connector-releases/releases).
Use the newest release **not** marked *Pre-release*, unless Vega gives you a specific
version: pre-releases are betas for test clinics.

## What you need

- A Windows 10, Windows 11 or Windows Server PC at the clinic that can reach the WinClinic
  database and is on during opening hours. It may be the WinClinic server itself.
- An administrator account on that PC.
- **From the clinic:** a read-only (SELECT-only) login to the WinClinic database: the
  server's address and port, the database name, the user name and the password. Vega tells
  the clinic which tables the login must be able to read.
- **From Vega:** the API address and the connector key (it starts with `vcc_`). Vega shows
  the key once when it creates the connector.
- Outbound HTTPS (port 443) from the PC to the API address.

## 1. Download and check

1. From the release, download `vc-connector.exe` and `config.example.toml`.
2. Check that the exe is genuine. Open a Command Prompt in the download folder and run:
   ```bat
   certutil -hashfile vc-connector.exe SHA256
   ```
   The result must equal the SHA-256 of `vc-connector.exe` in the release notes (and in
   the release's `SHA256SUMS.txt`). If it doesn't, don't run the file; tell Vega.
3. The program is not code-signed, so Windows may say "Windows protected your PC". Only
   after the checksum matches, choose **More info → Run anyway**.

## 2. Install the program

Open a Command Prompt **as administrator** (Start → type `cmd` → right-click → *Run as
administrator*) and run, from the download folder:

```bat
mkdir "C:\Program Files\VegaConnected"
copy vc-connector.exe "C:\Program Files\VegaConnected\"
```

## 3. Create the settings folder

The settings folder will hold the key and the database password (encrypted), so only
administrators and the system may open it. In the same administrator Command Prompt:

```bat
mkdir C:\ProgramData\VegaConnected\connector
icacls C:\ProgramData\VegaConnected\connector /inheritance:r /grant:r Administrators:(OI)(CI)F SYSTEM:(OI)(CI)F
copy config.example.toml C:\ProgramData\VegaConnected\connector\config.toml
notepad C:\ProgramData\VegaConnected\connector\config.toml
```

## 4. Fill in the settings file

In Notepad, change only these lines and save:

```toml
api_base_url = "https://…"     # the API address from Vega
log_format = "text"            # easier to read than json

[source]
host = "…"                     # the WinClinic database server, e.g. 192.168.1.10
port = 3306                    # its port
user = "…"                     # the read-only login from the clinic
database = "…"                 # the WinClinic database name
```

Don't type the key or the password into this file: the next step stores them encrypted.

## 5. Store the key and the database password

In the administrator Command Prompt:

```bat
cd "C:\Program Files\VegaConnected"
set VC_CONNECTOR_CONFIG=C:\ProgramData\VegaConnected\connector\config.toml
vc-connector set-key
vc-connector set-key --source-password
```

- `set-key` asks for the **API key**: paste the `vcc_…` key and press Enter. Nothing
  appears while you type or paste; that is normal. In a Command Prompt, paste with a
  **right-click** (Ctrl+V may not work).
- `set-key --source-password` asks for the **database password** the same way.
- If pasting keeps failing, pipe the value instead, then close the window so it isn't
  left on screen:
  ```bat
  echo PASTE-THE-KEY-HERE| vc-connector set-key
  ```
  (no space before the `|`).

Both are encrypted for this PC only. On another PC they must be stored again.

## 6. Check everything

```bat
vc-connector check
```

Each line should start with `[ok]` and the last line should say `check passed`. A
`[FAIL]` line says what is wrong, for example a wrong address or port, a refused login
or password, a table the login cannot read, or a refused key. Fix it and run `check`
again, or send the output to Vega.

## 7. Install and start the service

```bat
vc-connector service install --config C:\ProgramData\VegaConnected\connector\config.toml
vc-connector service start
sc query VegaConnectedConnector
```

`sc query` should show the state `4  RUNNING`. The service, **Vega Connected Connector**,
starts by itself when the PC starts, even before anyone logs in, and restarts itself after
a failure. The first sync loads the clinic's records, which can take a few minutes; then
it syncs every few minutes. Tell Vega that it is running.

The log file is `C:\ProgramData\VegaConnected\connector\state\vc-connector.log`. It holds
counts, times and errors, never patient data, the key or the password.

## Every day

- When the PC is off, nothing syncs. When it starts again, the connector catches up by
  itself.
- Clinic admins can press **Sync now** in Vega's clinic app to sync at once, for example
  after closing a visit.
- The connector doesn't depend on Windows' language or date settings: keep them as
  WinClinic needs them.

## Updating to a new version

Download and check the new exe as in step 1, then in an administrator Command Prompt:

```bat
cd "C:\Program Files\VegaConnected"
vc-connector service stop
copy /Y %USERPROFILE%\Downloads\vc-connector.exe "C:\Program Files\VegaConnected\"
vc-connector service start
vc-connector --version
```

The settings, the key and the password stay as they are.

## Moving to a new PC

1. Ask Vega for a **new key for the same connector** (not a new connector: a clinic's
   database is synced by one connector only).
2. Install on the new PC with steps 1–7, using the new key.
3. When Vega confirms that the new PC has synced, uninstall on the old PC (below). Vega
   then revokes the old key.

## Uninstall

In an administrator Command Prompt:

```bat
cd "C:\Program Files\VegaConnected"
vc-connector service stop
vc-connector service remove
```

Then delete `C:\Program Files\VegaConnected` and `C:\ProgramData\VegaConnected`. The
second folder holds the encrypted key and password.

## When something is wrong

- **The service isn't running:** `sc query VegaConnectedConnector`; start it with
  `vc-connector service start` from `C:\Program Files\VegaConnected`.
- **"another vc-connector process is using …":** the connector is already running (for
  example the service, while you try it in a Command Prompt). Only one can run at a time.
- **The database can't be reached:** check that the WinClinic server is on and that this PC
  can reach its address and port. The connector keeps trying by itself.
- **The key was refused:** ask Vega for a new key and store it with `vc-connector set-key`;
  the service picks it up within a few minutes.
- **What to send Vega:** the output of `vc-connector check` and the log file. Never send
  the key or the database password.
