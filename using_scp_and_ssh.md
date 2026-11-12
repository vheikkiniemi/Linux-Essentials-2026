# SCP – Secure File Transfer Using SSH

SCP (Secure Copy Protocol) is a command-line tool used to securely transfer files and folders between computers over an SSH connection.

SCP is available on Linux and Windows PowerShell (with OpenSSH installed). It uses SSH authentication, so you can connect using a password or an SSH key.

In these examples, we use:

- Local computer: Windows PowerShell or Debian Desktop
- Remote computer: Debian or Alpine Linux Server
- Remote username: `linuxadmin`
- Remote IP address: `192.168.1.100`

## Part 1 – Download files and folders

Download means copying data from the remote Linux server to your local computer.

### 1.1 Download a single file

Run this command on your local computer:

```
scp linuxadmin@192.168.1.100:/home/linuxadmin/report.txt .
```

This downloads `report.txt` from the server to your current local directory.

The `.` means the current directory.

### 1.2 Download a file to a specific folder

Windows PowerShell:

```
scp linuxadmin@192.168.1.100:/home/linuxadmin/report.txt C:\Users\Student\Downloads\
```

Linux terminal:

```
scp linuxadmin@192.168.1.100:/home/linuxadmin/report.txt ~/Downloads/
```

### 1.3 Download an entire folder

Use the `-r` option to copy folders recursively, including their contents.

```
scp -r linuxadmin@192.168.1.100:/home/linuxadmin/project .
```

This downloads the entire `project` folder into your current directory.

## Part 2 – Upload files and folders

Upload means copying data from your local computer to the remote Linux server.

### 2.1 Upload a single file

```
scp report.txt linuxadmin@192.168.1.100:/home/linuxadmin/
```

This uploads `report.txt` from your current local directory to the remote user's home directory.

### 2.2 Upload a file from a specific folder

Windows PowerShell:

```
scp C:\Users\Student\Documents\report.txt linuxadmin@192.168.1.100:/home/linuxadmin/
```

Linux terminal:

```
scp ~/Documents/report.txt linuxadmin@192.168.1.100:/home/linuxadmin/
```

### 2.3 Upload an entire folder

```
scp -r project linuxadmin@192.168.1.100:/home/linuxadmin/
```

This uploads the `project` folder and all its contents to the server.

## SCP command structure

The general syntax is:

```
scp [options] source destination
```

| Component              | Meaning                            |
| ---------------------- | ---------------------------------- |
| `scp`                  | Secure copy command                |
| `-r`                   | Copy folders recursively           |
| `-P 2222`              | Use SSH port 2222 instead of 22    |
| `-i ~/.ssh/id_ed25519` | Specify an SSH private key         |
| `user@host:`           | Remote username and server address |
| `.`                    | Current local directory            |
| `~/`                   | Current user's home directory      |

Important: Run SCP commands on the computer where you want to initiate the transfer. SCP uses SSH, so the remote server must be reachable and allow SSH access. You also need appropriate file permissions.

## Quick exercise

Practice secure file transfers between your Debian Desktop and Debian Server.

1. Create a text file named `test.txt` on Debian Desktop.
2. Upload it to Debian Server using SCP.
3. Verify that the file exists on the server using `ls`.
4. Download the same file back to Debian Desktop under a different filename.
5. Create a folder containing two files and upload the entire folder using `scp -r`.

Expected result: You can securely upload and download both individual files and entire folders using SCP.

# SSH – Execute Commands on a Remote Linux Server

SSH (Secure Shell) allows you to securely connect to a remote Linux server and execute commands without physically accessing the server.

You can use SSH from Windows PowerShell or a Linux terminal.

In these examples:

- Local computer: Windows PowerShell or Debian Desktop
- Remote server: Debian or Alpine Linux
- Username: `linuxadmin`
- Server IP address: `192.168.1.100`

## Part 1 – Connect to a remote server

Open PowerShell or a Linux terminal and run:

```
ssh linuxadmin@192.168.1.100
```

After authentication, you can execute Linux commands directly on the remote server.

For example:

```
hostname
whoami
ip addr
```

To close the remote session:

```
exit
```

## Part 2 – Execute a single remote command

You can execute a command without opening an interactive SSH session.

General syntax:

```
ssh user@server "command"
```

For example, display the remote server's hostname:

```
ssh linuxadmin@192.168.1.100 "hostname"
```

Check the server's uptime:

```
ssh linuxadmin@192.168.1.100 "uptime"
```

Display the server's IP addresses:

```
ssh linuxadmin@192.168.1.100 "ip addr"
```

The command executes on the remote server, and the result is displayed in your local terminal. The SSH connection then closes automatically.

## Part 3 – Execute multiple remote commands

You can combine several commands using `&&`.

```
ssh linuxadmin@192.168.1.100 "hostname && uptime && free -h"
```

The commands execute sequentially. With `&&`, the next command runs only if the previous command succeeds.

You can also use `;` to execute commands sequentially regardless of whether an earlier command fails.

```
ssh linuxadmin@192.168.1.100 "hostname; uptime; free -h"
```

## Part 4 – Execute administrative commands

Some commands require administrator privileges.

For example, restart the SSH service on Debian:

```
ssh -t linuxadmin@192.168.1.100 "sudo systemctl restart ssh"
```

The `-t` option allocates a terminal on the remote server. This is useful when `sudo` needs to ask for a password.

Other examples:

Check SSH service status:

```
ssh -t linuxadmin@192.168.1.100 "sudo systemctl status ssh"
```

Update the Debian package list:

```
ssh -t linuxadmin@192.168.1.100 "sudo apt update"
```

Note: Alpine Linux normally uses OpenRC rather than systemd. For example, check SSH status with `sudo rc-service sshd status`.

## Part 5 – Execute a local script on a remote server

You can execute a local Bash script remotely without first copying it to the server.

On Debian Desktop:

```
ssh linuxadmin@192.168.1.100 "bash -s" < script.sh
```

This sends the contents of your local `script.sh` file to Bash running on the remote server.

Alternatively, upload the script first:

```
scp script.sh linuxadmin@192.168.1.100:/home/linuxadmin/
```

Then execute it remotely:

```
ssh linuxadmin@192.168.1.100 "bash /home/linuxadmin/script.sh"
```

The second approach also works from Windows PowerShell when the local script file is accessible.

## Quick exercise

Practice remote administration using SSH.

1. Connect to your Debian Server using SSH.
2. Display the server's hostname and IP addresses.
3. Close the SSH session.
4. Execute `hostname` remotely without opening an interactive session.
5. Execute `hostname`, `uptime`, and `free -h` using a single SSH command.
6. Check the SSH service status remotely using `sudo`.

Expected result: You can execute commands and perform basic administrative tasks on a remote Linux server directly from your local computer.

Remember the difference:

- `ssh` – Execute commands on a remote server.
- `scp` – Transfer files and folders between computers.

Both use SSH for secure communication.