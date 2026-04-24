---
name: wsl2-optimization
description: "Optimize WSL2 networking, create Windows desktop shortcuts from WSL, and set up WSL services for auto-start. Covers TCP tuning, MTU, mirrored networking, and Windows interop pitfalls."
tags: [wsl, wsl2, windows, networking, optimization, autostart, shortcut]
---

# WSL2 Optimization & Windows Integration

## When to Use
- User reports slow network speeds inside WSL2
- User wants desktop shortcuts or auto-start for WSL services
- User needs to run Windows executables (cmd.exe, powershell.exe) from WSL
- Setting up WSL2 mirrored networking mode

## 1. Network Speed Optimization

### TCP Buffer Tuning (immediate effect, no restart needed)

Default WSL2 TCP buffers are ~208KB — way too small for modern broadband (100Mbps+). Increase to 16MB:

    sysctl -w net.core.rmem_max=16777216
    sysctl -w net.core.wmem_max=16777216
    sysctl -w net.ipv4.tcp_rmem="4096 131072 16777216"
    sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
    sysctl -w net.core.netdev_max_backlog=5000

To persist across WSL restarts, create a drop-in file:

    sudo tee /etc/sysctl.d/99-network-tuning.conf << 'EOF'
    net.core.rmem_max = 16777216
    net.core.wmem_max = 16777216
    net.ipv4.tcp_rmem = 4096 131072 16777216
    net.ipv4.tcp_wmem = 4096 65536 16777216
    net.core.netdev_max_backlog = 5000
    EOF

Observed improvement (TCP tuning alone on NAT): ~48% download, ~132% upload, ping -69% on 100Mbps connection.

NOTE: Mirrored mode showed mixed results — ping improved further (13ms vs 20ms) but throughput was inconsistent and sometimes worse than NAT. See mirrored networking section for details.

### MTU: 1420 for NAT mode, 1500 for mirrored mode

In NAT mode (default), WSL2's Hyper-V virtual network adds encapsulation overhead. MTU 1420 outperforms 1500 — confirmed by A/B testing. Larger MTU causes packet fragmentation in the virtual layer.

    ip link set eth0 mtu 1420

In mirrored mode, the active interface (typically eth1) uses the host's real network stack with no encapsulation, so the default MTU 1500 is correct — do NOT reduce it.

### Mirrored Networking (Windows 11 22H2+ only)

Eliminates the NAT/virtual-switch layer entirely. WSL shares the host's network stack directly.

Create C:\Users\<username>\.wslconfig:

    [wsl2]
    networkingMode=mirrored
    dnsTunneling=true
    autoProxy=true

    [experimental]
    bestEffortDnsParsing=true

Requires wsl --shutdown and restart to take effect. After enabling, WSL will show the same IP as Windows host (no more 172.x.x.x NAT address).

Not available on Windows 10.

**WARNING: Mirrored mode can REDUCE throughput.** In testing (Win11 Build 26200, RTX 5090 system), mirrored mode improved ping dramatically (65ms → 13ms) but throughput actually dropped compared to NAT+TCP-tuning (18 Mbps → 3.8 Mbps). Cloudflare and other large-file downloads timed out or crawled. The user's phone on the same WiFi was fast, confirming the issue was WSL-specific.

**Always A/B test after enabling mirrored mode.** Run a download test before and after. If throughput drops, revert to NAT mode (remove the networkingMode=mirrored line and restart WSL). NAT + TCP buffer tuning is often the better practical choice.

To revert: edit .wslconfig, remove `networkingMode=mirrored`, then `wsl --shutdown` and reopen.

### Diagnosing Current State

    # Check which interface is active (mirrored mode uses eth1, not eth0!)
    ip addr show | grep -E "^[0-9]|inet " | head -20

    # Check MTU on active interface
    ip link show eth1   # or eth0 — find the UP one

    # Check TCP buffers
    sysctl net.core.rmem_max net.core.wmem_max net.ipv4.tcp_rmem net.ipv4.tcp_wmem

    # Check if mirrored mode is active (IP should be on LAN, e.g. 192.168.x.x, not 172.x.x.x)
    ip addr show | grep "inet " | grep -v 127.0.0

    # Detect Windows version (never guess — always check)
    /mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -command "(Get-CimInstance Win32_OperatingSystem).Caption"
    # NOTE: must run as non-root user (see Section 2)

### Speed Testing

speedtest-cli can hang/timeout especially under mirrored mode. Use curl or wget instead:

    # Download test (pick a nearby server)
    wget --report-speed=bits -O /dev/null "http://speedtest.london.linode.com/100MB-london.bin"

    # Or with curl
    curl -o /dev/null -w "Speed: %{speed_download} bytes/s\nTime: %{time_total}s\n" https://speed.cloudflare.com/__down?bytes=25000000

    # Ping test
    ping -c 5 8.8.8.8

## 1b. Windows → WSL Port Access ("This site can't be reached")

### Symptoms

- Service runs fine inside WSL: `curl http://127.0.0.1:PORT` returns 200
- Windows browser at `http://localhost:PORT` shows "This site can't be reached / ERR_CONNECTION_RESET"
- Often appears after Windows sleep, WiFi switch, VPN toggle, or WSL restart
- `ss -tlnp` shows service bound to `127.0.0.1:PORT` only

### Root Cause

WSL2's built-in localhost forwarding (the magic that makes Windows `localhost:PORT` reach into WSL) is implemented by `wslrelay.exe` on the Windows side. It's fragile — silently breaks on network changes, sleep/wake, or when another process grabs the port briefly.

### Fix (permanent, survives sleep/reboot)

Two-part fix:
1. Make the service bind `0.0.0.0` instead of `127.0.0.1` so WSL IP is reachable
2. Add a Windows `netsh portproxy` rule to forward `127.0.0.1:PORT` → `<WSL-IP>:PORT`

**Step 1** — find how the service binds. Usually an env var or CLI flag. Examples:

    # Hermes Web UI: HERMES_WEBUI_HOST=0.0.0.0
    # Jupyter: --ip=0.0.0.0
    # Ollama: OLLAMA_HOST=0.0.0.0
    # Generic Python: look for app.run(host='127.0.0.1')

Verify after restart:

    ss -tlnp | grep PORT   # should show 0.0.0.0:PORT not 127.0.0.1:PORT

**Step 2** — add Windows portproxy rule (requires admin). WSL IP changes on every WSL restart, so read it dynamically:

    WSLIP=$(hostname -I | awk '{print $1}')
    su - <winuser> -c "/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -NoProfile -Command \"Start-Process powershell -Verb RunAs -ArgumentList '-NoProfile','-Command','netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=PORT connectaddress=$WSLIP connectport=PORT'\""

**UAC prompt appears on Windows desktop** — user must click Yes. There is no way to bypass UAC from WSL without pre-configured scheduled task with elevated privileges.

### PITFALL: UAC sometimes doesn't appear at all

When chaining `su - <user> -c 'cmd.exe /c "powershell ... Start-Process -Verb RunAs -ArgumentList ..."'`, UAC may silently fail to appear — no error, no prompt, nothing happens. Causes:

- Quote nesting too deep: PowerShell's `-ArgumentList` parser mangles triple-nested quotes
- Focus-stealing protection: Windows 11 sometimes suppresses UAC for background-initiated requests
- Script path unreachable: If the .ps1 file was written to a path that doesn't exist on Windows (e.g. wrong username), Start-Process silently does nothing

**Workaround 1 (preferred): skip portproxy entirely.** If the service is bound to `0.0.0.0`, the user can access it via the WSL IP directly: `http://<WSL-IP>:8787`. No admin needed, works immediately. Trade-off: WSL IP changes on reboot, and the user has to remember/look up the IP. Offer to make a Windows desktop shortcut that runs `wsl hostname -I` at click time and opens the browser — see Section 3.

**Workaround 2: write the .ps1 to a verified-writable Windows path first.** Run `ls -la /mnt/c/Users/` to discover the actual Windows username (it is NOT necessarily the same as the WSL Linux username — common mismatch). Then `write_file` to `/mnt/c/Users/<WIN-USERNAME>/setup.ps1` and verify with `ls -la` before invoking Start-Process. Hardcode the WSL IP inside the .ps1 rather than passing it via -ArgumentList to avoid quote-escaping hell.

**Workaround 3: tell the user to run the netsh command manually.** Output a one-liner they paste into an admin PowerShell themselves. Saves time when UAC is being uncooperative.

### PITFALL: Windows username ≠ WSL Linux username

Do not assume `/mnt/c/Users/<wsl-user>/` exists. Always verify:

    ls /mnt/c/Users/ | grep -v -E "^(Public|Default|All Users|Default User|WsiAccount|desktop\.ini)$"

Real case: WSL user and Windows user had different names (mismatch scenario). Writing to `/mnt/c/Users/<WSL_USER>/script.ps1` silently succeeds with `write_file` (creates a directory on Windows side? depends on WSL version) but the file isn't actually reachable for PowerShell's Start-Process.

Verify from Windows (PowerShell, non-elevated):

    netsh interface portproxy show v4tov4

Or from WSL:

    su - <winuser> -c '/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -NoProfile -Command "netsh interface portproxy show v4tov4"'

### Auto-update portproxy on WSL restart (WSL IP changes!)

The WSL IP changes every reboot. Put this in a Windows startup script (run as admin via Task Scheduler or shortcut with "Run as administrator"):

    $wslIp = (wsl.exe -d Ubuntu-24.04 --exec hostname -I).Trim().Split()[0]
    netsh interface portproxy reset
    netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=8787 connectaddress=$wslIp connectport=8787

Alternative: use WSL2 mirrored mode (Section 1) which eliminates the IP translation entirely, but has throughput trade-offs.

### Quick Diagnosis

    # 1. Service running and listening where?
    ss -tlnp | grep PORT
    #    0.0.0.0:PORT  = accessible from Windows via WSL IP
    #    127.0.0.1:PORT = only accessible via WSL2 localhost forwarding (fragile)

    # 2. WSL IP
    hostname -I | awk '{print $1}'

    # 3. From WSL, verify both paths work:
    curl -sS -o /dev/null -w "loopback -> %{http_code}\n" http://127.0.0.1:PORT/
    curl -sS -o /dev/null -w "wsl-ip -> %{http_code}\n" http://$(hostname -I | awk '{print $1}'):PORT/

    # 4. From Windows (run in PowerShell):
    #    Test-NetConnection -ComputerName 127.0.0.1 -Port PORT
    #    Test-NetConnection -ComputerName <WSL-IP> -Port PORT

### When NOT to do this

- If service handles sensitive data and binding 0.0.0.0 would expose it to LAN — set a password first, or stick with portproxy pointing to WSL IP with Windows firewall blocking inbound on LAN interfaces.
- Quick one-off: `wsl --shutdown` in Windows PowerShell often revives localhost forwarding temporarily.

## 2. Windows Interop from WSL

### PITFALL: root user cannot run Windows executables

cmd.exe, powershell.exe, and other Windows binaries fail with "Invalid argument" when run as root in WSL. Must switch to a regular user:

    # This FAILS as root:
    /mnt/c/Windows/System32/cmd.exe /c "echo hello"   (returns "Invalid argument")

    # This WORKS — switch to a non-root user:
    su - <username> -c '/mnt/c/Windows/System32/cmd.exe /c "echo hello"'

For PowerShell scripts:

    su - <wsluser> -c 'cd /mnt/c/Users/<winuser> && /mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -ExecutionPolicy Bypass -File "C:\path\to\script.ps1"'

Find the WSL username from /etc/wsl.conf ([user] default=xxx) or ls /home/.
Find the Windows username: ls /mnt/c/Users/ | grep -v -E "^(Public|Default|All Users|Default User|desktop.ini)$"

## 3. Creating Windows Desktop Shortcuts from WSL

Write a PowerShell script that uses WScript.Shell COM to create .lnk files:

    $WshShell = New-Object -ComObject WScript.Shell
    $Shortcut = $WshShell.CreateShortcut("$env:USERPROFILE\Desktop\MyApp.lnk")
For URL/web shortcuts (open a URL in default browser):

    $Shortcut.TargetPath = "http://localhost:8787"
    $Shortcut.IconLocation = "C:\Windows\System32\shell32.dll,14"

For WSL command shortcuts:

    $Shortcut.TargetPath = "C:\Windows\System32\wsl.exe"
    $Shortcut.Arguments = "-u root -- bash -lic /path/to/script.sh"
    $Shortcut.IconLocation = "C:\Windows\System32\wsl.exe,0"
    $Shortcut.Save()

Save to a Windows-accessible path, then execute via non-root user (see Section 2).

### Dynamic-IP Shortcut Pattern (for services bound to WSL IP)

When the service is bound to 0.0.0.0 (see Section 1b) and the user wants a one-click launcher, the WSL IP changes every reboot — so a static URL shortcut won't work. Use a 3-file pattern:

**File 1: `C:\Users\<winuser>\launch.ps1`** — detects current WSL IP at click time:

    $ErrorActionPreference = 'Stop'
    $wslIp = (wsl.exe -d Ubuntu-24.04 --exec hostname -I).Trim().Split(' ')[0]
    if (-not $wslIp) {
        # WSL not running — kick it awake, then retry
        wsl.exe -d Ubuntu-24.04 --exec true | Out-Null
        Start-Sleep -Seconds 2
        $wslIp = (wsl.exe -d Ubuntu-24.04 --exec hostname -I).Trim().Split(' ')[0]
    }
    if (-not $wslIp) { exit 1 }
    Start-Process "http://${wslIp}:8787"

**File 2: `C:\Users\<winuser>\launch.vbs`** — silent wrapper (no black PowerShell window flashing):

    Set oShell = CreateObject("WScript.Shell")
    oShell.Run "powershell.exe -NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File ""C:\Users\<winuser>\launch.ps1""", 0, False

**File 3: the .lnk itself** — target is wscript.exe:

    $sc.TargetPath = "C:\Windows\System32\wscript.exe"
    $sc.Arguments = '"C:\Users\<winuser>\launch.vbs"'
    $sc.WorkingDirectory = "C:\Users\<winuser>"
    $sc.IconLocation = "C:\Windows\System32\shell32.dll,14"

Why three files and not a direct `wsl.exe hostname -I`-as-TargetPath? Because shortcuts can only run one executable with static arguments — they can't capture output and chain commands. VBS wrapper is what hides the console window flash when running PowerShell.

### PITFALL: OneDrive Desktop redirection — user "can't see" the shortcut

Windows 10/11 with OneDrive enabled often **redirects the Desktop folder** to `C:\Users\<winuser>\OneDrive\Desktop\`. The user sees one thing, but `C:\Users\<winuser>\Desktop\` might be empty or hidden.

When user reports "I don't see the icon":

1. Check if OneDrive has a Desktop folder:

        ls /mnt/c/Users/<winuser>/OneDrive/Desktop/ 2>&1

2. If it exists, copy the .lnk there (or create it there in the first place). If both exist, write to the OneDrive one — that's the visible one.

3. If OneDrive doesn't have Desktop but user still can't see it:
   - Tell user to press F5 on desktop to refresh Explorer
   - Check for desktop organizers (Fences, Stardock) that might hide it
   - Have user navigate to `C:\Users\<winuser>\Desktop\` directly in File Explorer to confirm file exists

**Detection logic to use at shortcut-create time:**

    $desktop = [Environment]::GetFolderPath('Desktop')
    # This returns the REDIRECTED path if OneDrive has taken over — use it, don't hardcode C:\Users\<u>\Desktop

Always use `[Environment]::GetFolderPath('Desktop')` in PowerShell to get the correct path, rather than building `$env:USERPROFILE\Desktop` manually.

## 4. Auto-Start WSL Services on Windows Login

### Use Startup Folder (no admin rights needed)

Register-ScheduledTask requires admin elevation and often fails. Use the Startup folder instead:

    $StartupFolder = [Environment]::GetFolderPath("Startup")
    $WshShell = New-Object -ComObject WScript.Shell
    $Shortcut = $WshShell.CreateShortcut("$StartupFolder\MyService.lnk")
    $Shortcut.TargetPath = "C:\Windows\System32\wsl.exe"
    $Shortcut.Arguments = "-u root -- bash -lc /path/to/service-script.sh"
    $Shortcut.WindowStyle = 7   # 7 = Minimized
    $Shortcut.Save()

The service script should start processes with nohup ... & and exit quickly (don't block with exec).

## Pitfalls
- Never guess Windows version — always check via PowerShell (Get-CimInstance Win32_OperatingSystem).Caption or cmd.exe ver
- MTU 1500 is slower than 1420 in WSL2 NAT mode — don't "fix" it back to 1500. But in mirrored mode, 1500 is correct (no virtual encapsulation)
- speedtest-cli (Python) often hangs or times out inside WSL2, especially under mirrored mode — use wget or curl for speed testing instead
- TCP buffer changes are lost on WSL restart unless written to sysctl.conf
- Mirrored mode is NOT always faster — it improves ping/latency but can tank throughput on some setups. Always benchmark before and after. NAT + TCP tuning is the safer default.
- Mirrored mode changes IP addressing — services binding to specific IPs may break
- Mirrored mode may use eth1 (not eth0) as the active interface — eth0 can be DOWN; always check ip addr to find the UP interface
- .wslconfig is per-Windows-user, lives at C:\Users\<user>\.wslconfig, only read at WSL startup
- PowerShell path: use full path /mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe — powershell.exe alone isn't on PATH inside WSL
- Always run Windows exes via non-root user (su - <user>) — root triggers "Invalid argument"
