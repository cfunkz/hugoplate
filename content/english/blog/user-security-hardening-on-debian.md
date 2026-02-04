---
title: How to Secure SSH on Debian 11 & 12 with User Creation and Fail2Ban
meta_title: Secure SSH on Debian 11 & 12 with Fail2Ban and SSH Keys
description: Learn how to secure SSH on Debian 11 and 12 by creating a non-root
  user, disabling password logins, using SSH keys, and adding Fail2Ban
  protection.
---
# How to Secure SSH on Debian 11 & 12 (Users, Keys, Fail2Ban)

When you spin up a fresh Debian (or any distro), SSH is usually the first thing exposed to the internet and also the first thing attackers start poking at.

This guide walks through a simple, proven setup:

* create a non-root user
* lock SSH down to key-based access only
* add Fail2Ban to block brute-force attempts

Everything here works on **Debian 11 (Bullseye)** and **Debian 12 (Bookworm)**.

***

## 1. Create a non-root user

Running day-to-day admin tasks as root isn’t a great idea. A separate user with sudo access is safer and easier to audit.

```bash
# Create a new user
adduser cfunkz

# Give the user sudo access
usermod -aG sudo cfunkz

# Switch to the new user
su - cfunkz
```

From here on, you should be logged in as this user, not root.

***

## 2. Set up SSH key authentication

Password logins are the weakest part of most SSH setups. Keys are simpler, faster, and far more secure.

Create the SSH directory and lock down permissions:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Add your public key:

```bash
nano ~/.ssh/authorized_keys
# Paste your public key here
```

Fix permissions so SSH will actually accept the key:

```bash
chmod 600 ~/.ssh/authorized_keys
```

If permissions are wrong, SSH will silently ignore the file.

***

## 3. Harden the SSH configuration

Now we disable root login and all password-based authentication.

Edit the SSH daemon config:

```bash
sudo nano /etc/ssh/sshd_config
```

Update or add the following options:

```text
# Network settings
Port 22
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

# Disable root login
PermitRootLogin no

# Key-based authentication only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Disable password authentication
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM no
PermitEmptyPasswords no

# General hardening
PermitTunnel no
AllowAgentForwarding yes
AllowTcpForwarding yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
ClientAliveInterval 120
ClientAliveCountMax 3

# SFTP
Subsystem sftp /usr/lib/openssh/sftp-server
```

At this point:

* root can’t log in over SSH
* passwords are completely disabled
* only users with valid SSH keys get access

Before restarting SSH, **make sure you can log in with your key**.

***

## 4. Restart SSH

Apply the changes:

```bash
sudo systemctl restart sshd
```

Keep your current session open and test a new SSH connection in another terminal. If something’s wrong, you’ll still have a way back in.

***

## 5. Install and configure Fail2Ban

Fail2Ban watches logs and automatically blocks IPs that keep failing authentication. It’s a simple layer that stops most background noise immediately.

### Install Fail2Ban

```bash
sudo apt update
sudo apt install fail2ban -y
```

### Configure the SSH jail

Copy the default config so it doesn’t get overwritten by updates:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit it:

```bash
sudo nano /etc/fail2ban/jail.local
```

Enable and configure the SSH jail:

```ini
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
findtime = 600
bantime = 3600
```

What these do:

* **maxretry**: failed attempts before a ban
* **findtime**: time window to count failures
* **bantime**: how long the IP stays banned (seconds)

### Optional: protect other services

If you’re running web servers or other exposed services, you can enable additional jails (e.g. `nginx`, `apache`, `vsftpd`) in the same file.

***

### Start and enable Fail2Ban

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo systemctl status fail2ban
```

### Check Fail2Ban status

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

This shows active jails and currently banned IPs.

***

## Final notes

* SSH now only accepts key-based logins
* root login is disabled entirely
* Fail2Ban blocks repeated login attempts automatically
* works on Debian 11 and 12

Before logging out for good, **double-check your SSH key works**. Locking yourself out is still a common mistake.
