# The commands for the S3

## 1. Add your user to the `sudo` group

On the Debian Server:

```bash
su -
apt update
apt install sudo
usermod -aG sudo YOUR_USER
```

Log out and back in as `YOUR_USER`, then verify:

```bash
groups
sudo whoami
```

`sudo whoami` should print `root`.

## 2. Confirm that SSH key login works

From Debian Desktop or Windows PowerShell, open a **new terminal**:

```bash
ssh YOUR_USER@SERVER_IP
```

Confirm that you can log in without entering the **server account’s password**. Keep your existing server session open.

## 3. Disable SSH password authentication

On the Debian Server, create a configuration file:

```bash
sudo nano /etc/ssh/sshd_config.d/01-keys.conf
```

Add:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Check the configuration, then reload SSH:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Verify the effective settings:

```bash
sudo sshd -T | grep -Ei '^(pubkey|password|kbdinteractive)authentication'
```

From a **new client terminal**, test key login again:

```bash
ssh YOUR_USER@SERVER_IP
```

OR create a **new user** in debian server and test login:

```bash
adduser testuser
```

To check that password-only authentication is refused, run this on the client:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password YOUR_USER@SERVER_IP
```

The password-only attempt should fail. Keep the previously working SSH session open until these checks are complete.

## 4. Install and configure Fail2ban

On the Debian Server:

```bash
sudo apt update
sudo apt install fail2ban python3-systemd
sudo nano /etc/fail2ban/jail.d/sshd.local
```

Add:

```ini
[sshd]
enabled = true
backend = systemd
```

Start Fail2ban and check the SSH jail:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

If you edited the jail file after starting Fail2ban, apply the change:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

## 5. Inspect SSH and Fail2ban activity

Show recent events from the current boot:

```bash
sudo journalctl -u ssh -u fail2ban -b -n 80
```

> [!TIP]
> In a command, each space-separated part has a role. For example:

```bash
sudo journalctl -u ssh -u fail2ban -b -n 80
```

| Part | Name | Meaning |
|---|---|---|
| `sudo` | Command | Runs the following command with administrative privileges. |
| `journalctl` | Command | Reads the systemd journal. |
| `-u` | Option (or flag) | Filters entries by a systemd unit. |
| `ssh` | Option argument | The unit name supplied to the first `-u`. |
| `-u fail2ban` | Option and its argument | Also includes events from the Fail2ban unit. |
| `-b` | Option (flag) | Shows entries from the current boot. |
| `-n 80` | Option and its argument | Shows the most recent 80 entries. |

**Useful distinction:** `-b` works on its own, so it is a *flag*. The options `-u` and `-n` need a value after them; those values are called *option arguments*. You may also hear people call options **command-line switches**.

---

Follow new SSH events live:

```bash
sudo journalctl -u ssh -f
```

After controlled failed login attempts from a separate client, check the jail again:

```bash
sudo fail2ban-client status sshd
```

Look at the failed-attempt and banned-address counts. **A logged failure alone does not prove an address was banned**; use the jail status to verify the result.