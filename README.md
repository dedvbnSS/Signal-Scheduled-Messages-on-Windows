# 📨 Signal Scheduled Messages on Windows

Automatically send a Signal message to a group (or a contact) every day at fixed times — e.g. **04:50** and **17:50** — on Windows, even when the laptop is asleep.

Signal Desktop has no built-in message scheduling, so this project uses [**signal-cli**](https://github.com/AsamK/signal-cli) (an unofficial command-line Signal client) linked to your account as a secondary device, plus **Windows Task Scheduler** to run it on a timer.

---

## ✨ Features

- Daily scheduled messages at any times you choose
- Works with **groups** and **individual contacts**
- Full **Cyrillic / Unicode** support (message text is read from a UTF-8 file)
- Wakes the laptop from sleep to send the message
- Waits for the network to come up after wake before sending
- Catches up on missed runs (e.g. if the PC was off)
- Logs every run to `log.txt`

---

## 📋 Requirements

| Component | Version / Notes |
|---|---|
| Windows | 10 / 11 |
| Java | **JDK/JRE 25+** (signal-cli 0.14.x is compiled for class file version 69) |
| signal-cli | 0.14.8 (or newer) |
| Signal | A phone with Signal installed — the primary device |

---

## 🚀 Setup

### 1. Install Java 25

Download **Eclipse Temurin 25** from [adoptium.net](https://adoptium.net/) and install it.

> ⚠️ If an older Java (e.g. Oracle Java 17) is also installed, it may come first in `PATH`. Check with:
> ```bat
> where java
> java -version
> ```
> The scripts below set `JAVA_HOME` explicitly, so signal-cli always uses Java 25 regardless of `PATH`.

### 2. Download signal-cli

Grab the latest `signal-cli-x.x.x.tar.gz` from the [releases page](https://github.com/AsamK/signal-cli/releases) and extract it, e.g. to:

```
C:\signal-cli-0.14.8\
```

### 3. Link signal-cli to your Signal account

```bat
set "JAVA_HOME=C:\Path\To\jdk-25"
cd /d C:\signal-cli-0.14.8\bin
signal-cli.bat link -n "PC-scheduler"
```

A QR code appears in the console. On your phone open **Signal → Settings → Linked devices → Link new device** and scan it.

Then do the first sync:

```bat
signal-cli.bat -a +380XXXXXXXXX receive
```

### 4. Find the group ID

```bat
signal-cli.bat -a +380XXXXXXXXX listGroups
```

Output example:

```
Id: AbCdEf123...xyz= Name: My Group  Active: true  Blocked: false
```

Copy the full `Id` value, **including the trailing `=`**.

> 💡 A newly created group doesn't show up? Send any message to it from your phone, then run `receive` and `listGroups` again. You can also try `signal-cli.bat -a +380XXXXXXXXX sendSyncRequest`.

### 5. Create the message file

Create `C:\signal-cli-0.14.8\message.txt` with your message text and save it as **UTF-8** (Notepad → *Save As* → *Encoding: UTF-8*).

> Passing Cyrillic text directly via `-m "..."` turns it into `????` because of Windows console encoding. Reading from a UTF-8 file via `--message-from-stdin` avoids this completely.

### 6. Create the send script

`C:\signal-cli-0.14.8\send.bat`:

```bat
@echo off
set "JAVA_HOME=C:\Path\To\jdk-25"
set "ACCOUNT=+380XXXXXXXXX"
set "GROUP_ID=AbCdEf123...xyz="
set "BASE=C:\signal-cli-0.14.8"

cd /d %BASE%\bin

rem --- Wait for the network (up to 2 minutes) after waking from sleep ---
set tries=0
:waitnet
ping -n 1 chat.signal.org >nul 2>&1 && goto online
set /a tries+=1
if %tries% geq 24 goto online
timeout /t 5 /nobreak >nul
goto waitnet

:online
echo ===== %date% %time% ===== >> %BASE%\log.txt
call signal-cli.bat -a %ACCOUNT% receive -t 5 >> %BASE%\log.txt 2>&1
call signal-cli.bat -a %ACCOUNT% send --message-from-stdin -g "%GROUP_ID%" < "%BASE%\message.txt" >> %BASE%\log.txt 2>&1
```

To send to a contact instead of a group, replace `-g "%GROUP_ID%"` with the recipient's phone number, e.g. `+380YYYYYYYYY`.

> The `receive` call keeps the linked device in sync. Signal may unlink devices that stay inactive for a long time.

Double-click `send.bat` to test it — the message should arrive in the group.

### 7. Schedule it with Task Scheduler

Run in a **regular** (non-admin) command prompt, so the tasks belong to your user — signal-cli stores its account data in your user profile:

```bat
schtasks /create /tn "Signal 04-50" /tr "C:\signal-cli-0.14.8\send.bat" /sc daily /st 04:50 /f
schtasks /create /tn "Signal 17-50" /tr "C:\signal-cli-0.14.8\send.bat" /sc daily /st 17:50 /f
```

Then open `taskschd.msc`, double-click each task (the bottom preview pane is read-only) and set:

**General**
- ✅ Run whether user is logged on or not
- Configure for: **Windows 10**

**Conditions**
- ✅ Wake the computer to run this task
- ⬜ Start the task only if the computer is on AC power
- ⬜ Start the task only if the computer is idle

**Settings**
- ✅ Allow task to be run on demand
- ✅ Run task as soon as possible after a scheduled start is missed
- ✅ If the task fails, restart every **1 minute**, up to **3** times
- ✅ Stop the task if it runs longer than **1 hour**

Click **OK** and enter your Windows password.

### 8. Enable wake timers

Without this the laptop won't wake from sleep. Press `Win+R` → `control powercfg.cpl,,3` → **Sleep → Allow wake timers → Enable** (both *On battery* and *Plugged in*).

Or from an **admin** prompt:

```bat
powercfg /setacvalueindex SCHEME_CURRENT SUB_SLEEP RTCWAKE 1
powercfg /setdcvalueindex SCHEME_CURRENT SUB_SLEEP RTCWAKE 1
powercfg /setactive SCHEME_CURRENT
```

### 9. Test

```bat
schtasks /run /tn "Signal 17-50"
```

Refresh Task Scheduler — *Last Run Result* should be `(0x0)`. To test waking from sleep, temporarily set a trigger 5 minutes ahead, put the laptop to sleep and wait.

---

## 📁 Project structure

```
C:\signal-cli-0.14.8\
├── bin\
│   └── signal-cli.bat
├── lib\
├── send.bat        # the scheduled script
├── message.txt     # message text (UTF-8)
└── log.txt         # run log (created automatically)
```

---

## 🛠 Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `UnsupportedClassVersionError ... class file version 69.0 ... up to 61.0` | Java is too old (17) | Install Java 25 and set `JAVA_HOME` |
| `where java` shows Oracle `javapath` first | Old Java takes priority in `PATH` | Set `JAVA_HOME` in the script, or remove the old Java |
| Message arrives as `????` | Console encoding breaks Cyrillic arguments | Use `--message-from-stdin` with a UTF-8 file |
| Group names show as `????` in `listGroups` | Console output encoding (cosmetic) | `chcp 65001` and `set "JAVA_TOOL_OPTIONS=-Dstdout.encoding=UTF-8"` |
| New group missing from `listGroups` | Not synced yet | Message the group from your phone, then `receive` |
| Can't edit task settings | The bottom pane is a read-only preview | Double-click the task or use *Properties* |
| Task result `0x41303` | Task hasn't run yet | Normal — wait for the first run or use *Run* |
| No message after waking from sleep | Wake timers disabled / Modern Standby | Enable wake timers; check `powercfg /a` and `powercfg /waketimers` |

---

## ⚠️ Limitations

- The laptop must be **asleep**, not **shut down** or (on many laptops) **hibernated**. If it was off, the missed run fires on next boot — late.
- An internet connection is required at send time.
- Don't unlink the `PC-scheduler` device from your phone, or sending stops working.
- Editing `send.bat` or `message.txt` is safe at any time; changes apply on the next run. Keep the file names and paths unchanged, or update the task's *Actions* tab.

---

## 🔒 Security notes

- The linked signal-cli device has **full access** to your Signal account. Keep the PC secure.
- Never share the `sgnl://linkdevice?...` link or QR code.
- Don't commit your real phone number, group ID, `log.txt` or the signal-cli data folder (`%USERPROFILE%\.local\share\signal-cli`) to Git. Suggested `.gitignore`:

```gitignore
log.txt
message.txt
*.local.bat
```

---

## 📄 Disclaimer

signal-cli is an unofficial, community-maintained client and is not affiliated with Signal Messenger LLC. Use responsibly and in accordance with Signal's terms of service.
