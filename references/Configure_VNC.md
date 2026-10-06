# Configure VNC on Intel Wildcat Lake / Ubuntu 26.04.1 LTS

This document configures an independent XFCE desktop using TigerVNC and provides browser access through noVNC.

Default configuration used in this document:

- User: `renity-admin`
- VNC display: `:10`
- TigerVNC TCP port: `5910`
- noVNC TCP port: `6080`
- Resolution: `1920x1080`
- Color depth: `24`
- TigerVNC listens on localhost only; LAN clients access it through noVNC.

> Run the user-level commands as `renity-admin`. Do not run `vncpasswd` with `sudo`.

# Check Ubuntu Version and Architecture

```bash
cat /etc/os-release
uname -m
```

The expected architecture on Intel Wildcat Lake is `x86_64` (Ubuntu package architecture `amd64`).

# Install VNC Dependency

```bash
sudo apt update
sudo apt install -y \
  xfce4 xfce4-session xfce4-goodies \
  dbus-x11 xterm \
  tigervnc-standalone-server tigervnc-common \
  novnc websockify

# Package configuration > Configuring lightdm > Default display manager, please choose: gdm3  
```

Check the installed commands and package files.

```bash
command -v tigervncserver
command -v Xtigervnc
command -v websockify
dpkg -L tigervnc-standalone-server | grep -E 'tigervncserver@|vncserver.users|Xtigervnc-session' || true
```

# Change VNC Server Default Password

Switch to the VNC user if necessary.

```bash
sudo -iu renity-admin
```

Create the TigerVNC configuration directory and password.

```bash
mkdir -p /home/renity-admin/.config/tigervnc
vncpasswd /home/renity-admin/.config/tigervnc/passwd
```

The output is similar to the following.

```text
Password:
Verify:
Would you like to enter a view-only password (y/n)?
```

For normal remote control, answer `n` to the view-only password question unless a separate view-only password is required.

# Configure TigerVNC

Create the per-user TigerVNC configuration.

```bash
nano /home/renity-admin/.config/tigervnc/config
```

Add the following content.

```text
geometry=1920x1080
depth=24
localhost
alwaysshared
```

`localhost` intentionally restricts TCP port 5910 to the local machine. noVNC/websockify will connect to it through `127.0.0.1`.

# Use Your Own xstartup

Do not create `xstartup` as a directory. Create only its parent directory first.

```bash
mkdir -p /home/renity-admin/.vnc
nano /home/renity-admin/.vnc/xstartup
```

Add the following content.

```sh
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

export XDG_SESSION_TYPE=x11
export XDG_CURRENT_DESKTOP=XFCE
export DESKTOP_SESSION=xfce

exec dbus-run-session -- startxfce4
```

Configure permissions.

```bash
chmod 755 /home/renity-admin/.vnc/xstartup
chmod 700 /home/renity-admin/.vnc
chmod 700 /home/renity-admin/.config/tigervnc
chmod 600 /home/renity-admin/.config/tigervnc/passwd
```

Exit the `renity-admin` login shell if you entered it with `sudo -iu`.

```bash
exit
```

Ensure ownership is correct.

```bash
sudo chown -R renity-admin:renity-admin /home/renity-admin/.vnc
sudo chown -R renity-admin:renity-admin /home/renity-admin/.config/tigervnc
```

# Test TigerVNC Manually Before Enabling systemd

Start display `:10` manually as `renity-admin`.

```bash
sudo -iu renity-admin /usr/bin/tigervncserver :10 \
  -xstartup /home/renity-admin/.vnc/xstartup \
  -geometry 1920x1080 \
  -depth 24 \
  -localhost no
```

Check that TCP port 5910 is listening only on localhost.

```bash
sudo ss -ltnp | grep ':5910'
```

A normal result should contain `127.0.0.1:5910` and/or `[::1]:5910`, not `0.0.0.0:5910`.

Stop the manual test before configuring systemd.

```bash
sudo -iu renity-admin /usr/bin/tigervncserver -kill :10 || true
```

# Configure VNC to Run at Startup Automatically

This configuration intentionally uses a dedicated service so the VNC desktop remains independent from the local display manager and explicitly uses the custom `xstartup` file.

```bash
# Remove TightVNC Default Configuration
sudo rm ~/.config/tigervnc/config
sudo nano /etc/systemd/system/vncserver@.service
```

Add the following content.

```ini
[Unit]
Description=TigerVNC server on display :%i
After=network.target
Wants=network.target

[Service]
Type=forking
User=renity-admin
Group=renity-admin
WorkingDirectory=/home/renity-admin
Environment=HOME=/home/renity-admin

# Remove a stale TigerVNC session for this display if one exists.
ExecStartPre=-/usr/bin/tigervncserver -kill :%i

# Start an independent XFCE/X11 desktop.
# VNC is restricted to localhost; noVNC connects through 127.0.0.1.
ExecStart=/usr/bin/tigervncserver :%i -xstartup /home/renity-admin/.vnc/xstartup -geometry 1920x1080 -depth 24 -localhost no
ExecStop=/usr/bin/tigervncserver -kill :%i

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> Do not manually remove `/tmp/.X11-unix/X10` or `/tmp/.X10-lock` during normal operation. If another X server legitimately owns display `:10`, deleting its lock/socket can damage that running session. Diagnose the process first.

# Configure noVNC

Create the noVNC service.

```bash
sudo nano /etc/systemd/system/novnc.service
```

Add the following content.

```ini
[Unit]
Description=noVNC WebSocket Proxy for TigerVNC display :10
After=network.target vncserver@10.service
Requires=vncserver@10.service

[Service]
Type=simple
ExecStart=/usr/bin/websockify --web=/usr/share/novnc/ 6080 127.0.0.1:5910
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

# Apply Changes and Start Services

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now vncserver@10.service
sudo systemctl enable --now novnc.service
```

# Check Services

```bash
sudo systemctl status vncserver@10.service --no-pager -l
sudo systemctl status novnc.service --no-pager -l
sudo ss -ltnp | egrep '(:5910|:6080)'
```

Expected network behavior:

```text
5910 -> localhost only (127.0.0.1 / ::1)
6080 -> noVNC/websockify; reachable from permitted LAN clients
```

Check the VNC processes.

```bash
ps -ef | grep -E '[X]tigervnc|[t]igervncserver|[w]ebsockify'
```

# Configure UFW Firewall (If UFW Is Enabled)

Check firewall status.

```bash
sudo ufw status verbose
```

If UFW is active and the local network is `192.168.0.0/24`, allow noVNC only from that network.

```bash
sudo ufw allow from 192.168.0.0/24 to any port 6080 proto tcp
```

Do **not** open TCP 5910 when using the localhost-only design in this document.

If your LAN uses a different subnet, replace `192.168.0.0/24` with the correct subnet.

# Find the Ubuntu Host IP Address

```bash
hostname -I
```

For example, if the server address is `192.168.0.100`, use the URLs below.

# Browse noVNC Page and Enter Password Manually

```text
http://192.168.0.100:6080/vnc.html
```

Enter the VNC password configured by `vncpasswd` when prompted.

# Optional noVNC Auto-Connect URL

An auto-connect URL can be used without embedding the password.

```text
http://192.168.0.100:6080/vnc_lite.html?host=192.168.0.100&port=6080&autoconnect=true
```

Do **not** place the VNC password in the URL. URLs can be retained in browser history, proxy logs, screenshots, and other records.

# Restart VNC / noVNC

```bash
sudo systemctl restart vncserver@10.service
sudo systemctl restart novnc.service
```

# Stop VNC / noVNC

```bash
sudo systemctl stop novnc.service
sudo systemctl stop vncserver@10.service
```

# Disable Automatic Startup

```bash
sudo systemctl disable novnc.service
sudo systemctl disable vncserver@10.service
```

# Troubleshooting

## Check systemd Logs

```bash
sudo journalctl -u vncserver@10.service -b --no-pager -n 200
sudo journalctl -u novnc.service -b --no-pager -n 200
```

Check TigerVNC user logs.

```bash
sudo -iu renity-admin bash -lc 'find ~/.local/state/tigervnc ~/.vnc -maxdepth 1 -type f -name "*.log" -print 2>/dev/null'
```

If a log file is shown, inspect it with, for example:

```bash
sudo -iu renity-admin bash -lc 'tail -n 200 ~/.local/state/tigervnc/*.log 2>/dev/null || tail -n 200 ~/.vnc/*.log 2>/dev/null'
```

## VNC Service Fails to Start

First stop the service and check whether display `:10` is already in use.

```bash
sudo systemctl stop vncserver@10.service
ps -ef | grep -E '[X]tigervnc.*:10|[X]org.*:10|[X]wayland.*:10'
sudo ss -ltnp | grep ':5910' || true
ls -l /tmp/.X10-lock /tmp/.X11-unix/X10 2>/dev/null || true
```

If no process owns display `:10`, try a manual foreground-style diagnostic start from the VNC user account:

```bash
sudo -iu renity-admin /usr/bin/tigervncserver :10 \
  -xstartup /home/renity-admin/.vnc/xstartup \
  -geometry 1920x1080 \
  -depth 24 \
  -localhost no
```

Then inspect the TigerVNC log before stopping it.

```bash
sudo -iu renity-admin bash -lc 'tail -n 200 ~/.local/state/tigervnc/*.log 2>/dev/null || tail -n 200 ~/.vnc/*.log 2>/dev/null'
sudo -iu renity-admin /usr/bin/tigervncserver -kill :10 || true
```

## XFCE Shows a Black or Blank Screen

Verify the startup script.

```bash
sudo -iu renity-admin cat /home/renity-admin/.vnc/xstartup
sudo -iu renity-admin test -x /home/renity-admin/.vnc/xstartup && echo OK
```

Verify required commands.

```bash
command -v startxfce4
command -v dbus-run-session
```

Reinstall the required session packages if necessary.

```bash
sudo apt install --reinstall xfce4-session dbus-x11
```

Then restart both services.

```bash
sudo systemctl restart vncserver@10.service
sudo systemctl restart novnc.service
```

## noVNC Page Opens but Cannot Connect

Verify both ports and services.

```bash
sudo systemctl is-active vncserver@10.service
sudo systemctl is-active novnc.service
sudo ss -ltnp | egrep '(:5910|:6080)'
```

Test the noVNC HTTP endpoint locally.

```bash
curl -I http://127.0.0.1:6080/vnc.html
```

If port 6080 is listening locally but another LAN computer cannot open it, check UFW and the network path.

```bash
sudo ufw status numbered
ip -br addr
ip route
```

# Security Notes

- TigerVNC port `5910` is intentionally bound to localhost only.
- Only noVNC port `6080` needs LAN access in this configuration.
- Plain `http://` does not provide HTTPS encryption. For use outside a trusted LAN, place noVNC behind an HTTPS reverse proxy, VPN, or SSH tunnel rather than exposing port 6080 directly to the Internet.
- Do not embed VNC passwords in URLs.
- Use a strong VNC password and restrict firewall access to trusted networks.

# Final Verification After Reboot

Reboot the system.

```bash
sudo reboot
```

After the system returns, verify automatic startup.

```bash
sudo systemctl is-enabled vncserver@10.service
sudo systemctl is-active vncserver@10.service
sudo systemctl is-enabled novnc.service
sudo systemctl is-active novnc.service
sudo ss -ltnp | egrep '(:5910|:6080)'
```

Then browse to:

```text
http://<Ubuntu-IP>:6080/vnc.html
```

```text
http://<Ubuntu-IP>:6080/vnc_lite.html?host=192.168.0.100&port=6080&password=70604376&autoconnect=true
```

The XFCE VNC desktop should be available independently of the local graphical login session.


# Important. Please configure Ubuntu 26.04.1 LTS Default Terminal
## Configure Ubuntu 26.04.1 LTS Default Terminal
Please type command as follows to choose Default Terminal: */usr/bin/xfce4-terminal.wrapper*.
Do not choose: *ptyxis*.
```bash
# Do not choose:　ptyxis
sudo update-alternatives --config x-terminal-emulator
```

```text
Selection    Path
--------------------------------
* 0          /usr/bin/ptyxis
  ...
  N          /usr/bin/xfce4-terminal.wrapper
  ...
  
```

## Verify Ubuntu 26.04.1 LTS Default Terminal
```bash
＃ Output should be: /usr/bin/xfce4-terminal.wrapper
readlink -f /etc/alternatives/x-terminal-emulator
```
