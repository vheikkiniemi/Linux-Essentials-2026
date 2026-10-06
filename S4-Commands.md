# S4 — Alpine Server: Installation, Hardening and SSH MFA

This guide follows the S4 workflow. The final login requires an SSH key and a TOTP code. After login, sudo works without another password prompt. Static IP configuration is not required; DHCP is suitable for this assignment.

## Before you begin

- Replace `linuxadmin` with your username, `alpine-s4` with your hostname, and `SERVER_IP` with the current server address.
- Commands labelled **root** run in Alpine's VM console as root. After sudo works, you can run administrative commands with `sudo` instead.
- Commands labelled **client** run on your computer, not on Alpine.
- Keep the VM console and an existing SSH session available. Test every authentication change with a fresh connection.
- Configuration blocks are file contents, not terminal commands. Use `nano` to edit them.
- Do not put private keys, TOTP secrets, QR codes or recovery codes in screenshots.

## 1. Install Alpine and test networking

**Alpine live system, root:**

```sh
setup-alpine
```

Select your keyboard layout, hostname, network interface and DHCP. Choose package mirrors and OpenSSH. For a persistent installation, select the intended virtual disk and choose **sys** installation mode. Disk installation erases the selected disk: verify that it is your empty Alpine VM disk. Complete installation, remove the installation ISO and reboot.

```sh
reboot
```

**Installed Alpine, root:**

```sh
cat /etc/alpine-release
hostname
ip addr
ip route
cat /etc/resolv.conf
ping -c 4 1.1.1.1
ping -c 4 alpinelinux.org
apk update
apk upgrade
```

The first ping tests IP connectivity; the second also requires working DNS. ICMP may be blocked on some networks, so a failed ping alone does not establish that all network access is broken. `apk update` also checks access to the package repositories.

If you need to change the hostname:

```sh
setup-hostname alpine-s4
```

Check `/etc/hosts` and adjust any entry using the old hostname if necessary.

```sh
cat /etc/hosts
```

Enable the **community** repository for your installed Alpine release if required by sudo, Fail2ban or Google Authenticator. Uncomment its matching line; do not mix stable releases or add edge repositories.

```sh
vi /etc/apk/repositories
apk update
apk add nano sudo
```

## 2. Create an administrative user

**Alpine, root:** Skip `adduser` if the account already exists.

```sh
adduser linuxadmin
addgroup linuxadmin wheel
id linuxadmin
EDITOR=nano visudo
```

In sudoers, enable this rule once:

```sudoers
%wheel ALL=(ALL:ALL) ALL
```

Save, then validate:

```sh
visudo -c
```

Log out and log in as `linuxadmin`, or start a fresh login shell from the console:

```sh
su - linuxadmin
```

**Alpine, regular user:**

```sh
whoami
id
sudo whoami
```

Enter the regular user's password. Expected output of the last command: `root`.

## 3. Configure SSH keys and initial hardening

**Alpine, root:**

```sh
apk add openssh
ssh-keygen -A
rc-update add sshd default
rc-service sshd start
rc-service sshd status
ip addr
```

### Create and transfer a client key

Reuse a suitable existing key, or create a new one. Never overwrite an existing key unintentionally. A key passphrase protects your private key and is different from the server account password.

**Linux/macOS client:**

```sh
ssh-keygen -t ed25519 -f ~/.ssh/alpine_s4
ssh-copy-id -i ~/.ssh/alpine_s4.pub linuxadmin@SERVER_IP
ssh -i ~/.ssh/alpine_s4 linuxadmin@SERVER_IP
```

**Windows PowerShell client:**

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\alpine_s4"
Get-Content "$env:USERPROFILE\.ssh\alpine_s4.pub" | ssh linuxadmin@SERVER_IP 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
ssh -i "$env:USERPROFILE\.ssh\alpine_s4" linuxadmin@SERVER_IP
```

Key transfer initially uses the account password. Verify the server fingerprint through the VM console before accepting the first connection:

```sh
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

**Alpine, regular user:**

```sh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
hostname
whoami
```

### Disable root and password login

**Alpine, root:**

```sh
cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.before-s4
nano /etc/ssh/sshd_config
```

Set these options in the global section, before any `Match` blocks. Edit existing entries; do not simply append conflicting duplicates. Check included configuration files too.

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
AuthenticationMethods publickey
LogLevel VERBOSE
```

```sh
sshd -t
sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|authenticationmethods|loglevel) '
rc-service sshd reload
```

`sshd -t` normally prints nothing on success. Apply changes only if it succeeds. If reload is unsupported, use `rc-service sshd restart` with console access available.

**Client:** Open a new terminal and test your key. Then test password-only login:

```sh
ssh -i PATH_TO_PRIVATE_KEY linuxadmin@SERVER_IP
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password linuxadmin@SERVER_IP
```

The first succeeds; the second is refused. Replace `PATH_TO_PRIVATE_KEY` with the Linux or Windows path used above.

## 4. Configure logging and Fail2ban

This example uses Alpine's BusyBox syslog and `/var/log/messages`. If you already use another logger, retain it and set Fail2ban's `logpath` to the file containing SSH events. Do not run competing syslog daemons.

**Alpine, root:**

```sh
rc-update add syslog boot
rc-service syslog start
rc-service syslog status
logger -t s4 'S4 logging test'
tail -n 40 /var/log/messages
apk add fail2ban iptables
mkdir -p /etc/fail2ban/jail.d
nano /etc/fail2ban/jail.d/s4-sshd.local
```

File contents:

```ini
[sshd]
enabled = true
filter = sshd
port = ssh
backend = polling
logpath = /var/log/messages
maxretry = 5
findtime = 10m
bantime = 10m
banaction = iptables-multiport
```

This example explicitly uses an iptables ban action and installs its command-line tools. Fail2ban adds rules when banning; do not flush existing firewall rules. If using a different firewall, choose a compatible action instead.

Inspect enabled jails and check for conflicting existing SSH jail settings:

```sh
grep -R -n -E '^\[|^enabled|^logpath|^backend|^banaction' /etc/fail2ban/jail.local /etc/fail2ban/jail.d 2>/dev/null
fail2ban-client -t
rc-update add fail2ban default
rc-service fail2ban start
fail2ban-client status
fail2ban-client status sshd
fail2ban-client get sshd logpath
```

If Fail2ban is already running, apply changes with:

```sh
rc-service fail2ban restart
```

Make a fresh SSH connection and confirm actual SSH events are recorded:

```sh
grep -E 'sshd|sshd-auth' /var/log/messages | tail -n 40
fail2ban-regex /var/log/messages /etc/fail2ban/filter.d/sshd.conf
```

An active jail is not proof that its filter matches your failed logins. After generating failures, inspect the regex results. If zero relevant failures match, check log format, log path and the installed filter before expecting a ban.

## 5. Enable TOTP for SSH

### Install the PAM-enabled server and synchronise time

**Alpine, root:**

```sh
apk add openssh-server-pam google-authenticator
apk info -L openssh-server-pam
setup-ntp
date -u
rc-status
```

Follow `setup-ntp` to enable a time synchronisation service. Check the selected service with `rc-service SERVICE_NAME status`; OpenNTPD, chrony or BusyBox NTP may be used depending on your installation. The authenticator device must also keep accurate time.

Inspect how your installed SSH service selects the PAM-enabled implementation:

```sh
cat /etc/conf.d/sshd
grep -n -E 'command|pam|sshd' /etc/init.d/sshd
```

Package layouts vary across Alpine releases. Use the package-provided PAM implementation; do not assume that merely installing PAM changes an already running daemon. If your release requires a daemon path selection in `/etc/conf.d/sshd`, select the PAM path shown by the installed package and service script. Do not invent a path from another release.

### Enrol the regular account

**Alpine, regular user — not root:**

```sh
google-authenticator
chmod 600 ~/.google_authenticator
ls -l ~/.google_authenticator
```

Choose time-based tokens, enrol the secret in your authenticator, and save the configuration. Enable rate limiting and prevent token reuse when prompted. Store emergency codes securely. Do not use `cat` on the secret file for submission evidence. Each account that needs SSH MFA must enrol separately.

### Configure the SSH PAM authentication stack

**Alpine, root:**

```sh
cp -p /etc/pam.d/sshd /etc/pam.d/sshd.before-s4
cat /etc/pam.d/sshd
nano /etc/pam.d/sshd
```

In the SSH-specific PAM file, replace the password authentication part (typically an `auth include base-auth` line) with:

```text
auth required pam_google_authenticator.so
```

Preserve existing account, session and password-management lines, and required access restrictions such as `pam_nologin`. Remove or disable other `auth` rules that request the account password or allow authentication to skip TOTP. Do not edit shared `base-auth` files for this SSH-only change. Do not add `nullok` or use a `sufficient` TOTP rule.

Inspect any included auth stacks too. A line that appears earlier with a bypassing control flag can defeat a later TOTP requirement. `sshd -t` does not validate PAM logic: fresh-login tests are essential.

### Configure the final SSH authentication requirements

```sh
cp -p /etc/ssh/sshd_config /etc/ssh/sshd_config.key-only
nano /etc/ssh/sshd_config
```

Replace the earlier global authentication settings with:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication yes
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive:pam
LogLevel VERBOSE
```

The comma requires the key and then PAM authentication. Keyboard-interactive now carries the TOTP prompt; disabling it would prevent this MFA flow.

```sh
sshd -t
sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|usepam|authenticationmethods) '
rc-service sshd restart
rc-service sshd status
```

Restart here so that the PAM-enabled implementation selected by the service replaces the previous running server. Keep the console available. If `UsePAM` is unsupported, stop and correct the daemon selection before continuing.

**Client, fresh terminal:**

```sh
ssh -o ControlMaster=no -o ControlPath=none -i PATH_TO_PRIVATE_KEY linuxadmin@SERVER_IP
```

Enter the current authenticator code at the verification prompt. Expected: successful login without the server account password. A private-key passphrase prompt is separate and can still appear.

## 6. Configure passwordless sudo

**Alpine, root:**

```sh
EDITOR=nano visudo -f /etc/sudoers.d/s4-linuxadmin
```

File contents (substitute your username):

```sudoers
linuxadmin ALL=(ALL:ALL) NOPASSWD: ALL
```

```sh
chmod 440 /etc/sudoers.d/s4-linuxadmin
visudo -c
```

Confirm `/etc/sudoers` includes `/etc/sudoers.d` using `@includedir` or `#includedir`. If it does not, add the rule using the main `visudo` editor instead. Inspect `sudo -l` for conflicting later rules.

**Alpine, regular user:**

```sh
sudo -l
sudo -k
sudo -n whoami
```

Expected: `root` with no password prompt. `-k` clears cached credentials; `-n` refuses to prompt. The rule grants this account full passwordless administrative access for the assignment. SSH TOTP is checked at login, not again for sudo.

## 7. Test the completed configuration

Run client tests one at a time. Replace the private-key path. These options prevent reuse of an authenticated multiplexed connection.

### Valid key and correct TOTP — success

```sh
ssh -o ControlMaster=no -o ControlPath=none -i PATH_TO_PRIVATE_KEY -o IdentitiesOnly=yes linuxadmin@SERVER_IP
```

### Key only — must not complete login

```sh
ssh -o ControlMaster=no -o ControlPath=none -i PATH_TO_PRIVATE_KEY -o IdentitiesOnly=yes -o PreferredAuthentications=publickey linuxadmin@SERVER_IP
```

### Valid key and incorrect TOTP — refused

Use the normal fresh-login command and enter one incorrect code. Stop after the rejection. Avoid accidentally consuming the ban threshold on your management client.

### Password only — refused

```sh
ssh -o ControlMaster=no -o ControlPath=none -o PubkeyAuthentication=no -o PreferredAuthentications=password linuxadmin@SERVER_IP
```

### Inspect and test Fail2ban

**Alpine, root:**

```sh
tail -n 80 /var/log/messages
grep -E 'sshd|sshd-auth|fail2ban' /var/log/messages | tail -n 80
fail2ban-client status sshd
fail2ban-client get sshd maxretry
fail2ban-client get sshd findtime
fail2ban-client get sshd bantime
iptables -S
```

If Fail2ban writes its own file:

```sh
tail -n 80 /var/log/fail2ban.log
```

For live SSH log monitoring, use the following and press Ctrl+C to stop:

```sh
tail -f /var/log/messages
```

From a separate client IP, make controlled failed attempts, then inspect the jail counters and ban rules. Separate terminals on the same computer normally share an IP. Clients behind the same NAT may also share the address seen by the server. Use the console to recover if necessary.

To remove a test ban (replace `TEST_CLIENT_IP`):

```sh
fail2ban-client set sshd unbanip TEST_CLIENT_IP
```

A logged failure may not match the filter or reach five failures within ten minutes. If a ban is reported, also verify that a new connection from the banned address is blocked. Existing connections can remain open.

### Reboot and check persistence

**Alpine, regular user:**

```sh
sudo reboot
```

Check the address in the VM console after reboot; DHCP may assign a different address. Reconnect with the key and TOTP. Then:

```sh
hostname
ip addr
ip route
sudo -k
sudo -n whoami
sudo rc-service sshd status
sudo rc-service syslog status
sudo rc-service fail2ban status
sudo fail2ban-client status sshd
sudo rc-update show
sudo sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|usepam|authenticationmethods) '
sudo tail -n 40 /var/log/messages
```

Verify network connectivity, SSH MFA, passwordless sudo, logging and Fail2ban.

## Recovery and common problems

| Problem | Check or action |
| --- | --- |
| Package not found | Check `/etc/apk/repositories`, enable community for the same release, then run `apk update`. |
| Key login fails | Check username, address, key path, `authorized_keys`, ownership and permissions. Use `ssh -v` on the client and inspect server logs. |
| No TOTP prompt | Inspect effective AuthenticationMethods, keyboard-interactive and UsePAM settings; confirm PAM-enabled daemon selection and restart SSH. |
| Password requested after the key | Inspect the SSH PAM auth stack for remaining password authentication rules. |
| Correct TOTP rejected | Check server/device time, correct enrolled account, file ownership and permissions; wait for a fresh code. |
| Sudo still requests a password | Run `sudo -l`, verify the username and included directory, and inspect conflicting rule order with `visudo`. |
| Fail2ban does not ban | Check actual SSH log path, filter matches, threshold, time window, ignore list and firewall action. |
| Lost SSH access | Use the VM console; repair configuration there before reopening SSH. |

Restore the pre-MFA key-only SSH configuration from the console if necessary:

```sh
cp -p /etc/ssh/sshd_config.key-only /etc/ssh/sshd_config
sshd -t
rc-service sshd restart
```

This temporarily removes the TOTP requirement. Repair MFA and re-test before submitting. Do not restore an insecure configuration as your final solution.

## Submission evidence

Capture the hostname, fresh MFA login, effective SSH settings, `sudo -k` plus `sudo -n whoami`, active services, jail status and relevant log entries. Explain the controlled failure test and post-reboot results. Keep all authentication secrets out of screenshots.

## Official references

- [Alpine installation](https://wiki.alpinelinux.org/wiki/Installation)
- [Alpine SSH server](https://wiki.alpinelinux.org/wiki/Setting_up_a_SSH_server)
- [Alpine OpenSSH PAM and Google Authenticator](https://wiki.alpinelinux.org/wiki/HOWTO_OpenSSH_2FA_with_password_and_Google_Authenticator) — its password + TOTP example must be adapted for this assignment's key + TOTP requirement.
- [Alpine syslog](https://wiki.alpinelinux.org/wiki/Syslog)
- [Alpine Fail2ban](https://wiki.alpinelinux.org/wiki/Fail2ban)
- [OpenSSH server configuration](https://man.openbsd.org/sshd_config)
- [Google Authenticator PAM module](https://github.com/google/google-authenticator-libpam)
- [Fail2ban jail configuration](https://github.com/fail2ban/fail2ban/blob/master/man/jail.conf.5)

Prepared for S4. Commands should be checked against your installed Alpine release. This guide was reviewed against documentation; it was not executed on a live Alpine VM.
