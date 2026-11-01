# VS Code (Windows) → Ubuntu remote: develop and debug a Qt GUI app with X11 forwarding

Setup: Windows PC runs VS Code and an X server (VcXsrv). Ubuntu PC holds the source, compiler and debugger and runs the app. The app window is displayed on Windows through X11 forwarding over SSH.

```
VS Code (Windows) --SSH--> Ubuntu: build, gdb, app
        ^                        |
        +---- X11 over SSH ------+   (window drawn by VcXsrv on Windows)
```

---

## 1. Ubuntu (remote), one-time

```bash
sudo apt update
sudo apt install openssh-server build-essential cmake ninja-build gdb xauth x11-apps x11-utils
# plus Qt dev packages, e.g. qt6-base-dev, or a Qt install under /opt/Qt
```

`/etc/ssh/sshd_config` (also check `/etc/ssh/sshd_config.d/*.conf` for overrides):
```
X11Forwarding yes
X11UseLocalhost yes
```
```bash
sudo systemctl enable --now ssh
sudo systemctl restart ssh
```

## 2. Windows, one-time

1. **SSH key** (PowerShell):
   ```powershell
   ssh-keygen -t ed25519
   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh user@remote-ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
   ```
2. **VcXsrv**: install, then run **XLaunch** with:
   - Multiple windows
   - **Display number: 0** (must match the DISPLAY variable below)
   - Start no client
   - Extra settings: leave "Native opengl" unchecked for QML apps; tick "Disable access control" if you get authentication errors
   - Allow `vcxsrv.exe` through Windows Firewall (Private networks)
3. **DISPLAY variable** (PowerShell):
   ```powershell
   setx DISPLAY "localhost:0.0"
   ```
   Then fully close VS Code (check Task Manager) and reopen it.
4. **SSH config** `C:\Users\<you>\.ssh\config`:
   ```
   Host devpc
       HostName <remote-ip>
       User <remote-user>
       ForwardX11 yes
       ForwardX11Trusted yes
       ServerAliveInterval 30
   ```
5. **VS Code user settings:**
   ```json
   "remote.SSH.enableX11Forwarding": true
   ```

## 3. VS Code remote window

1. Install the **Remote - SSH** extension.
2. `F1` → **Remote-SSH: Connect to Host…** → `devpc`, then open the project folder on the remote.
3. Install on the remote (**Install in SSH: devpc**): C/C++ (`ms-vscode.cpptools`), CMake Tools, clangd (optional).

## 4. Verify GUI forwarding BEFORE debugging

In the VS Code integrated terminal (runs on Ubuntu):
```bash
echo $DISPLAY     # expect localhost:10.0 (the number can change per session)
xclock            # a clock window should appear on the Windows desktop
```
Do not go further until `xclock` works.

## 5. Project configuration (on the remote)

**CMake**: Debug build, `CMAKE_BUILD_TYPE=Debug`, `CMAKE_EXPORT_COMPILE_COMMANDS=ON` (a `CMakePresets.json` with a `debug` preset is convenient).

**`.vscode/tasks.json`**
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build-debug",
      "type": "shell",
      "command": "cmake --build build/debug",
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": "$gcc"
    }
  ]
}
```

**`.vscode/launch.json`**
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug MyApp",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/build/debug/myapp",
      "cwd": "${workspaceFolder}/build/debug",
      "MIMode": "gdb",
      "miDebuggerPath": "/usr/bin/gdb",
      "preLaunchTask": "build-debug",
      "environment": [
        { "name": "DISPLAY", "value": "localhost:10.0" },
        { "name": "QT_QUICK_BACKEND", "value": "software" },
        { "name": "QT_XCB_GL_INTEGRATION", "value": "none" }
      ],
      "setupCommands": [
        { "text": "-enable-pretty-printing", "ignoreFailures": true }
      ]
    }
  ]
}
```
- Set `DISPLAY` to the value printed by `echo $DISPLAY` in step 4.
- The two `QT_*` variables are only for QML/OpenGL apps; drop them for plain Widgets apps.

## 6. Run and debug

Set a breakpoint and press **F5**. VS Code builds, gdb launches the app on Ubuntu, and the window appears on Windows.

Tips:
- A breakpoint freezes the window (no repaint). This is normal.
- Use logpoints or conditional breakpoints in paint or mouse-move handlers.
- In the Debug Console, prefix gdb commands with `-exec`, e.g. `-exec info threads`.

---

## What went wrong in our case, and the fix

| Step | Symptom | Cause / fix |
|---|---|---|
| 1 | `echo $DISPLAY` empty in the VS Code terminal | X11 forwarding not requested/established. Enable `remote.SSH.enableX11Forwarding`, make sure `xauth` is installed on Ubuntu and `X11Forwarding yes` is set, kill the VS Code server (`F1` → **Remote-SSH: Kill VS Code Server on Host…**), reconnect, open a NEW terminal |
| 2 | `$DISPLAY` = `localhost:10.0` but `xclock` could not open the window | Windows side not matching. **Fix: run `setx DISPLAY "localhost:0.0"` again, set the XLaunch display number to 0, restart VcXsrv and VS Code** |

## Troubleshooting checklist (in order)

1. VcXsrv running (tray icon), display number **0**.
2. `echo $env:DISPLAY` in PowerShell shows `localhost:0.0`; if not, `setx` again and fully restart VS Code.
3. Windows Firewall allows `vcxsrv.exe`.
4. Authentication errors → tick "Disable access control" in XLaunch, keep `ForwardX11Trusted yes`.
5. On Ubuntu: `xauth list` shows an entry, `ss -ltn | grep 6010` shows the forwarded port.
6. "X11 forwarding request failed on channel 0" → the Windows SSH client lacks `xauth`; add `XAuthLocation` pointing to an `xauth.exe` (Git for Windows, Cygwin, MobaXterm).
7. Isolate: run `ssh -Y devpc` then `xclock` in plain PowerShell. If it works there but not in VS Code, the issue is only VS Code's terminal environment.
8. After changing any SSH or X11 setting: kill the VS Code server, reconnect, open a new terminal.

## Alternatives if X11 is too slow (common with QML/OpenGL)

- **VNC / NoMachine** on Ubuntu, then set `"DISPLAY": ":1"` (or the session's display) in `launch.json`.
- **Remote PC has a monitor**: set `"DISPLAY": ":0"` and run `xhost +local:` in the remote desktop session; the window shows on the remote screen.
- **Headless tests**: `QT_QPA_PLATFORM=offscreen` or `xvfb-run`.
