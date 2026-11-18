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
  server's address and port, the database name, the user name and the password.
- **From Vega:** the connector key (it starts with `vcc_`), and whether this clinic uses
  Vega's *Production* or *Beta* (test clinics). Vega shows the key once when it creates
  the connector.
- Outbound HTTPS (port 443) from the PC to the internet.

## 1. Download and check

1. From the release, download `vc-connector.exe`.
2. Check that the exe is genuine. Open a Command Prompt in the download folder (in File
   Explorer, type `cmd` in the address bar and press Enter) and run:
   ```bat
   certutil -hashfile vc-connector.exe SHA256
   ```
   The result must equal the SHA-256 of `vc-connector.exe` in the release notes (and in
   the release's `SHA256SUMS.txt`). If it doesn't, don't run the file; tell Vega.
3. The program is not code-signed, so Windows may say "Windows protected your PC". Only
   after the checksum matches, choose **More info → Run anyway**.

## 2. Run the setup

Double-click `vc-connector.exe` and answer its questions:

1. Windows asks whether the program may make changes: choose **Yes**.
2. **Language:** 1 for English, 2 for Thai.
3. **Which Vega:** Production or Beta, as Vega told you.
4. **The connector key:** paste it and press Enter. Paste with a **right-click** in the
   window (or Ctrl+V). Nothing appears while you paste; that is normal. The setup checks
   the key with Vega and says plainly if it is wrong.
5. **The database:** the setup suggests `127.0.0.1` (this PC), port `3306`, database
   `winclinic` and login `vega_readonly`: press Enter to accept each, or type the clinic's
   value. Then type the password (hidden) and press Enter. The setup connects at once and
   says plainly if the address, the login or the read access is wrong.

The setup then installs everything: the program in `C:\Program Files\VegaConnected`, the
settings in `C:\ProgramData\VegaConnected\connector` (only administrators can open it; the
key and the password are encrypted for this PC), a check of everything, and the Windows
service **Vega Connected Connector**. It ends with "The service is running": press Enter
to close the window and tell Vega.

The service starts by itself when the PC starts, even before anyone logs in, and restarts
itself after a failure. The first sync loads the clinic's records, which can take a few
minutes; then it syncs every few minutes.

The log file is `C:\ProgramData\VegaConnected\connector\state\vc-connector.log`. It holds
counts, times and errors, never patient data, the key or the password.

## Every day

- When the PC is off, nothing syncs. When it starts again, the connector catches up by
  itself.
- When Vega can't be reached for a moment, the connector tries again within a minute or
  two by itself.
- Clinic admins can press **Sync now** in Vega's clinic app to sync at once, for example
  after closing a visit.
- The connector doesn't depend on Windows' language or date settings: keep them as
  WinClinic needs them. The PC's clock and the WinClinic database's clock should agree;
  the check warns when they don't.

## Updating to a new version

Download and check the new exe as in step 1, double-click it and choose **Update to this
version**. It stops the service, replaces the program and starts it again. The settings,
the key and the password stay as they are.

## Changing the key or the database settings

Double-click `vc-connector.exe` (the downloaded one, or the one in
`C:\Program Files\VegaConnected`) and choose **Change the key or the database settings**.
Press Enter to keep a current value, including the key and the password. The service
restarts with the new settings.

## Moving to a new PC

1. Ask Vega for a **new key for the same connector** (not a new connector: a clinic's
   database is synced by one connector only).
2. Install on the new PC with steps 1–2, using the new key.
3. When Vega confirms that the new PC has synced, uninstall on the old PC (below). Vega
   then revokes the old key.

## Uninstall

Double-click `vc-connector.exe` and choose **Uninstall**. It stops and removes the
service, then asks whether to delete the settings folder (the encrypted key and password,
and the sync progress): answer **n** to keep it if you will install again on this PC. If
you ran the setup from `C:\Program Files\VegaConnected`, delete that folder after closing
the window.

## When something is wrong

- **Run the check:** double-click `vc-connector.exe` and choose **Run the check**. Each
  line should start with `[ok]`; a `[FAIL]` line says what is wrong, a `[warn] clock` line
  says the PC's and the database's clocks disagree.
- **The service isn't running:** run the setup again and choose **Update to this
  version**, which starts it; or in an administrator Command Prompt run
  `sc query VegaConnectedConnector`.
- **"another vc-connector process is using …":** the connector is already running (for
  example the service, while you try it in a Command Prompt). Only one can run at a time.
- **The database can't be reached:** check that the WinClinic server is on and that this PC
  can reach its address and port. The connector keeps trying by itself.
- **The key was refused:** ask Vega for a new key and enter it with **Change the key or
  the database settings**.
- **What to send Vega:** a photo of the setup window, or the check's output, and the log
  file. Never send the key or the database password.

## Appendix: install by hand

The setup does all of this; use these steps only if Vega asks you to. Use an
**administrator** Command Prompt (Start → type `cmd` → right-click → *Run as
administrator*), in the download folder.

### A. Install the program

```bat
mkdir "C:\Program Files\VegaConnected"
copy vc-connector.exe "C:\Program Files\VegaConnected\"
```

### B. Create the settings folder

Only administrators and the system may open it (the codes name those two groups in every
Windows language). Download `config.example.toml` from the release too.

```bat
mkdir C:\ProgramData\VegaConnected\connector
icacls C:\ProgramData\VegaConnected\connector /inheritance:r /grant:r *S-1-5-32-544:(OI)(CI)F *S-1-5-18:(OI)(CI)F
copy config.example.toml C:\ProgramData\VegaConnected\connector\config.toml
notepad C:\ProgramData\VegaConnected\connector\config.toml
```

### C. Fill in the settings file

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

### D. Store the key and the database password

```bat
cd "C:\Program Files\VegaConnected"
set VC_CONNECTOR_CONFIG=C:\ProgramData\VegaConnected\connector\config.toml
vc-connector set-key
vc-connector set-key --source-password
```

Each asks for its value: paste with a right-click and press Enter (nothing appears; that is
normal).

### E. Check everything

```bat
vc-connector check
```

Each line should start with `[ok]` and the last line should say `check passed`.

### F. Install and start the service

```bat
vc-connector service install --config C:\ProgramData\VegaConnected\connector\config.toml
vc-connector service start
sc query VegaConnectedConnector
```

`sc query` should show the state `4  RUNNING`. Uninstall by hand with
`vc-connector service stop`, `vc-connector service remove`, then delete
`C:\Program Files\VegaConnected` and `C:\ProgramData\VegaConnected`. The log is
`vc-connector.log` in the settings folder's `state` folder.
