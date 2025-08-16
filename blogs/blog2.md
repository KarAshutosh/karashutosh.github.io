# Linux Hardening Guide for Ubuntu 22.04 LTS

## Table of Contents

1. Introduction to Linux Hardening
2. Why System Hardening Is Critical
3. Common Attack Vectors on Linux
4. Key Security Principles
5. System Updates & Patch Management
6. User & Group Management
7. SSH Hardening
8. File System & Permission Security
9. Firewall & Network Security
10. Service & Process Hardening
11. Audit & Monitoring
12. Kernel & System Parameter Hardening
13. Application Security
14. Backup & Recovery
15. Compliance & Best Practices
16. Quick Checklist

---

## Introduction to Linux Hardening

**Linux hardening** encompasses configuring the OS and its services to minimize vulnerabilities and reduce attack surfaces—preventing breaches and unauthorized access.

---

## Why System Hardening Is Critical

Servers face constant threats, from brute-force attacks and privilege escalation exploits to malware, misconfigurations, and insider threats. Implementing server hardening helps protect against these risks by:

- **Reducing the Attack Surface:** Disabling unnecessary services and features so only essential components are exposed.
- **Preventing Exploitation:** Securing daemons, patching misconfigurations, and keeping software up to date to block common breach points.
- **Blocking Unauthorized Access and Privilege Escalation:** Tightening user permissions and access controls to stop attackers in their tracks.
- **Mitigating Risk:** Minimizing the chances of data breaches, reputational damage, and costly downtime.
- **Ensuring Compliance:** Many regulations, such as PCI DSS and HIPAA, require secure default configurations and proactive hardening.

By systematically hardening servers, organizations build a strong security foundation that protects both their data and their reputation.

---

## Common Attack Vectors on Linux

- **Brute-force attacks & weak credentials:** Automated password-guessing attempts or leaked passwords.
- **Privilege escalation:** Exploiting misconfigured permissions, SUID binaries, or kernel vulnerabilities to gain root access.
- **Malware and rootkits:** Malicious code targeting critical system resources.
- **Vulnerable or misconfigured services:** Outdated packages, exposed network services, or misconfigured daemons such as SSH, web, or database servers.
- **Open and unmonitored network ports:** Increasing the attack surface for remote attackers.
- **Unrestricted processes and exposed data:** Running unnecessary processes or leaving sensitive data accessible.
- **Web and application exploits:** SQL injection and other attacks targeting server-side applications.
- **Local exploits:** Abuses of file or system permissions due to misconfigurations or kernel bugs.

---

## Key Security Principles

- **Least Privilege:** Users and processes have only the permissions required.
- **Defense in Depth:** Multiple layers of security across system components.
- **Separation of Duties:** Different roles/tasks for users to avoid privilege abuse.
- **Auditing and Accountability:** Log all critical changes and monitor for anomalies.
- **Security by configuration:** Disable unnecessary features/services.
- **Fail-Safe Defaults:** Systems should deny access unless explicitly allowed.
---

## System Updates & Patch Management

### Steps

1. **Manual Updates:**
  ```sh
  sudo apt update && sudo apt upgrade -y
  sudo apt full-upgrade -y
  ```
  `sudo apt full-upgrade -y` → Upgrades packages, allowing removal or installation of new dependencies if needed. This ensures that all packages are fully up-to-date and resolves dependency changes that apt upgrade cannot.

2. **Install and Configure Automatic Security Updates:**
  ```sh
  sudo apt install unattended-upgrades apt-listchanges -y
  ```
  - `sudo apt install unattended-upgrades apt-listchanges -y` → Installs:
    - `unattended-upgrades`: Automatically installs security updates.
    - `apt-listchanges`: Shows a summary of what changed in a package before upgrading. Optional but useful for sysadmins.
  ```sh
  sudo dpkg-reconfigure --priority=low unattended-upgrades
  ```

3. Review Configuration file

  Review the file `/etc/apt/apt.conf.d/50unattended-upgrades` for configuration.
  
  - This is the main configuration file where you can:
    - Choose which updates are installed automatically.
    - Configure email notifications.
    - Lock updates to only security packages or also include regular package updates.

4. **Verify Package Integrity:**
  - Only packages installed via dpkg/apt:  
    ```sh
    dpkg --verify
    ```
  - Full checksum verification:
    ```sh
    sudo apt install debsums -y
    sudo debsums -s
    ```
  - Always use official repositories and enable GPG key checking.

`dpkg --verify` checks installed packages for file changes (permissions, checksums).

`dpkg --verify` only works on packages installed via `dpkg`/`apt` and requires `debsums` to be installed for full checksum verification

---

## User & Group Management

Absolutely! Let’s go **command by command**, explaining what it does, why it’s used, and the security reasoning behind it. I’ll go section by section from the merged document.


### Listing Users

**Command:**

```bash
cut -d: -f1 /etc/passwd
```

- **What it does:**

  - `cut` extracts specific fields from a text file.
  - `-d:` sets the delimiter to `:` (colon), which separates fields in `/etc/passwd`.
  - `-f1` selects the first field, which is the username.
- **Why it’s used:**

  - To get a list of all user accounts on the system. Useful for auditing accounts and identifying unused or unexpected users.


**Command:**

```bash
awk -F: '($3 == "0") {print}' /etc/passwd
```

- **What it does:**

  - `awk` processes text line by line.
  - `-F:` sets the field separator to `:`.
  - `($3 == "0") {print}` prints lines where the 3rd field (UID) equals 0.
- **Why it’s used:**

  - UID 0 means root-level privileges. This command checks if there are any users other than `root` with full administrative access.
  - Helps detect privilege escalation risks.


### Creating Secure User Accounts

**Command:**

```bash
sudo adduser newuser
```

- **What it does:**

  - Creates a new user account interactively, asking for password and optional user details.
- **Why it’s used:**

  - Adds users securely while automatically creating home directories and necessary default settings.


**Command:**

```bash
sudo passwd -l root
```

- **What it does:**

  - Locks the password for the `root` account, preventing direct password login.
- **Why it’s used:**

  - Forces administrators to use `sudo` instead of logging in as root directly, reducing the risk of full system compromise.


**Security Note:**

- Only `root` should have UID 0. This prevents accidental creation of other accounts with unrestricted access.


### User Management

**Removing or Disabling Unused Accounts**

**Command:**

```bash
sudo awk -F: '{print $1}' /etc/passwd
```

- **What it does:**

  - Prints the first field (username) from every line in `/etc/passwd`.
- **Why it’s used:**

  - Allows administrators to audit all accounts and identify users that are no longer needed.

**Command:**

```bash
sudo deluser username
```

- **What it does:**

  - Removes a user from the system, optionally deleting their home directory.
- **Why it’s used:**

  - Reduces attack surface by removing unused accounts that could be exploited.


**Command:**

```bash
sudo usermod -L username
```

- **What it does:**

  - Locks the specified user account.
  - Prevents password login without deleting the account.
- **Why it’s used:**

  - Useful for temporarily disabling accounts without losing user data.

**Enforcing Password Policies**

**File:** `/etc/login.defs`

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   7
PASS_MIN_LEN    12
```

- **What they do:**

  - `PASS_MAX_DAYS`: Maximum number of days a password is valid before expiration.
  - `PASS_MIN_DAYS`: Minimum days before a password can be changed.
  - `PASS_WARN_AGE`: Days before expiry to warn the user.
  - `PASS_MIN_LEN`: Minimum password length.

- **Why it’s used:**

  - Ensures passwords are regularly changed and meet a baseline length, which reduces risk from weak or compromised passwords.

**Command:**

```bash
sudo apt install libpam-pwquality -y
```

- **What it does:**

  - Installs the PAM module `libpam-pwquality` for enforcing password complexity rules.
- **Why it’s used:**

  - Strengthens passwords by requiring combinations of upper/lowercase letters, digits, and special characters.


**File Edit:** `/etc/pam.d/common-password`

```
password requisite pam_pwquality.so retry=3 minlen=12 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
```

- **What it does:**

  - Configures PAM to enforce password quality:

    - `retry=3`: Allows 3 attempts to enter a valid password.
    - `minlen=12`: Minimum length 12 characters.
    - `dcredit=-1, ucredit=-1, ocredit=-1, lcredit=-1`: Requires at least one digit, uppercase, other (special), and lowercase letter.
- **Why it’s used:**
  - Prevents weak passwords and enforces complexity policies for system security.


### Implementing Multi-Factor Authentication (MFA)

**Command:**

```bash
sudo apt install libpam-google-authenticator
```

- **What it does:**

  - Installs the PAM module for Google Authenticator-based MFA.
- **Why it’s used:**

  - Adds a second authentication factor, requiring both a password and a time-based one-time code, improving login security.

**Command:**

```bash
google-authenticator
```

- **What it does:**

  - Runs the setup for MFA for a specific user.
  - Generates QR codes, secret keys, and recovery codes.
- **Why it’s used:**
  - Links the user account to a Google Authenticator app or compatible TOTP app.


**Edit File:** enable in `/etc/pam.d/sshd` file:

```
auth required pam_google_authenticator.so
```

- **What it does:**

  - Tells SSH to require MFA authentication for logins.
- **Why it’s used:**

  - Forces all SSH logins to require the one-time code, reducing risk from password theft.

Here’s a **fully merged and comprehensive SSH Hardening section**, combining all your sources without missing any details:


---

## SSH Hardening

SSH hardening reduces the risk of unauthorized access, brute-force attacks, and other remote exploits.

### SSH Configuration File

- The main configuration file is:

```
/etc/ssh/sshd_config
```

All changes below are applied here. After editing, SSH must be reloaded or restarted to apply changes.



### Change Default SSH Port

- **Purpose:** Reduces exposure to automated attacks targeting the default port `22`.

**Edit `/etc/ssh/sshd_config`:**

```
Port 2222
```

**Restart or reload SSH:**

```bash
sudo systemctl restart sshd   # or reload: sudo systemctl reload sshd
```



### Disable Root Login & Password Authentication

- **Purpose:** Prevents direct root login and enforces key-based authentication.

**In `/etc/ssh/sshd_config`:**

```
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
```

- **Restart SSH:**

```bash
sudo systemctl restart sshd
```



### Use SSH Keys with Passphrases

- **Purpose:** Key-based authentication is more secure than passwords. Adding a passphrase protects private keys.

**Generate a secure key pair:**

```bash
ssh-keygen -t ed25519 -a 100
```

- `-t ed25519`: Uses the Ed25519 algorithm, which is secure and fast.
- `-a 100`: Number of rounds for key derivation function (increases resistance to brute-force if the key is stolen).

**Deploy public key:**

- Copy the public key to the server’s user account:

```bash
~/.ssh/authorized_keys
```

- Ensure proper permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```



### Enable Intrusion Prevention with Fail2Ban

- **Purpose:** Protects against brute-force login attempts by banning IPs after multiple failed attempts.

**Install Fail2Ban:**

```bash
sudo apt install fail2ban -y
```

**Configure Fail2Ban:**

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

**Example configuration for SSH:**

```
[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 1h
```

**Start and enable service:**

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```



### Configure Idle Timeouts

- **Purpose:** Automatically disconnects inactive sessions, reducing exposure from abandoned SSH sessions.

**In `/etc/ssh/sshd_config`:**

```
ClientAliveInterval 300   # Sends a keepalive message every 5 minutes
ClientAliveCountMax 2     # Disconnect after 2 missed keepalives
```

- Alternative configuration sets `ClientAliveCountMax 0` to disconnect immediately after inactivity.



### Enforce Strong Key Exchange Algorithms and Ciphers

- **Purpose:** Ensures only modern, secure cryptography is used for SSH connections.

**Add to `/etc/ssh/sshd_config`:**

```
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com
```



### Summary of Recommended `/etc/ssh/sshd_config` Settings

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
ClientAliveInterval 300
ClientAliveCountMax 2
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com
```

- After making all changes:

```bash
sudo systemctl restart sshd
```

Here’s a **fully merged and comprehensive “File System & Permission Security” section**, combining all your sources without missing anything:


---

## File System & Permission Security

Securing the file system ensures that sensitive files, directories, and system resources are protected against unauthorized access or tampering. 

### Setting Secure Permissions

- **Tools:**

  - `chmod` → set file/directory permissions
  - `chown` → set ownership
  - `umask` → set default permissions for new files

**Commands:**

```bash
chmod 700 filename       # Full access for owner only
chmod 600 filename       # Read/write for owner only
chown root:root filename # Set owner and group to root
umask 027                # Default permission mask for new files/directories
```

- **Implementation:** Place `umask 027` in `/etc/profile` or `/etc/login.defs` to enforce system-wide defaults.



### Typical Secure Permission Settings

| File/Directory      | Owner | Permissions |
| ------------------- | ----- | ----------- |
| /boot, /etc/cron.\* | root  | 700         |
| /etc/crontab        | root  | 600         |
| /etc/passwd         | root  | 644         |
| /etc/shadow         | root  | 640         |
| /var/log            | root  | 700         |

- **Why:** Restrict access to critical system files to prevent unauthorized read/write operations.

- **Check for dangerous permissions**:

```bash
find / -perm /6000
```


### Securing `/tmp`, `/var`, and Other Sensitive Directories

- **Purpose:** Prevent execution of malicious scripts in temporary directories and sensitive mounts.

**Mount options in `/etc/fstab`:**

```
tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev 0 0
tmpfs /var/tmp tmpfs defaults,noexec,nosuid,nodev 0 0
tmpfs /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0
```

- **Remount sensitive directories:**

```bash
sudo mount -o remount,noexec,nosuid,nodev /home
```

- **Create separate partitions** for `/var`, `/var/tmp`, `/var/log` to isolate data and prevent accidental or malicious filling of the root filesystem.



### Disk Encryption (LUKS)

- **Purpose:** Protects data at rest, especially on laptops, cloud VMs, or systems storing sensitive information.

**Install necessary tool:**

```bash
sudo apt install cryptsetup
```

**Encrypt a disk/partition:**

```bash
sudo cryptsetup luksFormat /dev/sdX
```

- **Notes:**

  - During Ubuntu installation, full-disk encryption options are available and recommended.
  - For existing installations, follow advanced guides to migrate data to LUKS-encrypted volumes.


Here’s a **fully merged and comprehensive “Firewall & Network Security” section**, combining all your sources without missing anything:

---

## Firewall & Network Security

A properly configured firewall and network security setup reduces exposure to attacks, ensures only necessary services are accessible, and provides logging for monitoring.


### UFW (Uncomplicated Firewall)

- **Install UFW:**

```bash
sudo apt install ufw -y
```

- **Set default policies:**

```bash
sudo ufw default deny incoming   # Deny all incoming connections by default
sudo ufw default allow outgoing  # Allow all outgoing connections by default
```

- **Allow essential ports/services only** (allowlisting is preferred over blocklisting):

```bash
sudo ufw allow 2222/tcp          # Custom SSH port
sudo ufw allow 80/tcp            # HTTP
sudo ufw allow 443/tcp           # HTTPS
```

- **Enable UFW:**

```bash
sudo ufw enable
sudo ufw status verbose          # Check current rules
```

- **Why:**

  - Reduces attack surface by allowing only required services.
  - Denying everything by default ensures unintentional services aren’t exposed.


### Enable Logging

- **Command:**

```bash
sudo ufw logging on
```

- **Log location:** `/var/log/ufw.log`
- **Purpose:**

  - Helps monitor attempted connections and detect suspicious activity.


### Disable Unused Network Services

- **List all services:**

```bash
sudo ss -tulwn                    # Show listening TCP/UDP ports
sudo systemctl list-unit-files --type=service
```

- **Stop and disable unnecessary services:**

```bash
sudo systemctl stop service_name
sudo systemctl disable service_name
```

- **Examples of common unnecessary services to disable:**

```bash
sudo systemctl disable avahi-daemon
sudo systemctl disable cups
sudo systemctl disable rpcbind
```

- **Why:**

  - Reduces attack surface by preventing unneeded daemons from running.


### Advanced: Using iptables

- **Purpose:** Provides more granular control than UFW.

**Examples:**

```bash
sudo iptables -A INPUT -s <IP_ADDRESS> -j DROP             # Block a specific IP
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT        # Allow HTTP
```

- **Save rules persistently:**

```bash
sudo iptables-save > /etc/iptables/rules.v4
```

- **Why:**

  - Useful for fine-tuned rules, rate-limiting, and temporary blocks beyond UFW’s scope.


Here’s a **fully merged and comprehensive “Service & Process Hardening” section**, combining all your sources without missing anything:

---

## Service & Process Hardening

Hardening services and processes reduces the attack surface by disabling unnecessary services, restricting permissions, and enforcing mandatory access controls.


### Disabling Unnecessary Services

- **Purpose:** Unused services can be exploited by attackers. Removing or disabling them minimizes risk.

**Remove unnecessary packages:**

```bash
sudo apt remove telnet rsh-client rsh-redone-client -y
```

**List enabled services:**

```bash
sudo systemctl list-unit-files --type=service --state=enabled
```

**Disable unneeded services:**

```bash
sudo systemctl disable service_name
sudo systemctl stop service_name
```

- **Why:** Prevents unneeded daemons from running, reducing memory usage and attack surface.


### Restricting Service Permissions

- **Purpose:** Limit the capabilities of running services so that even if compromised, damage is minimized.

**Edit a service unit file:**

```bash
sudo systemctl edit service_name
```

**Add security directives under `[Service]`:**

```
User=nobody
Group=nogroup
ProtectSystem=full      # Makes the filesystem read-only for the service except /etc, /var, /tmp
ProtectHome=yes         # Restricts access to user home directories
NoNewPrivileges=true    # Prevents the service from gaining new privileges
```

- **Why:** Enforces the principle of least privilege for services.


### AppArmor / SELinux

- **Purpose:** Enforce mandatory access controls on applications to restrict what they can do on the system.

**Check AppArmor status:**

```bash
sudo aa-status
```

**Enforce profiles for applications:**

```bash
sudo aa-enforce /etc/apparmor.d/*
sudo aa-enforce /etc/apparmor.d/usr.bin.yourapp
```

- **Notes:**
  - Ubuntu defaults to **AppArmor**, which is practical and easier to manage.
  - SELinux is more powerful but requires additional setup and is less common on Ubuntu.

---

## Audit & Monitoring

### Install and Enable Auditd

Auditd is the Linux Audit Daemon, used for monitoring system events and changes to critical files.

```bash
sudo apt update
sudo apt install auditd audispd-plugins -y
sudo systemctl enable auditd --now
sudo systemctl start auditd
```

**Example audit rules:**

```bash
# Monitor changes to user accounts
sudo auditctl -w /etc/passwd -p wa -k user-modify
sudo auditctl -w /etc/shadow -p wa -k auth-modify

# Monitor generic system changes (example)
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
```

> **Explanation:**
> `-w <file>` → watch this file
> `-p wa` → watch for write (w) and attribute (a) changes
> `-k <key>` → attach a custom identifier to the event for easy searching


### Log Management

Proper log management ensures that audit events and system activity are recorded and reviewable.

**Using rsyslog:**

```bash
sudo apt install rsyslog -y
sudo systemctl enable rsyslog --now
```

- Configuration file: `/etc/rsyslog.conf`
- Aggregates system logs and audit events.

**Using journalctl:**

```bash
journalctl --verify
journalctl -xe
```

- `--verify` checks for log integrity.
- `-xe` shows recent logs with priority and explains errors.


### Intrusion Detection & File Integrity Monitoring**

**AIDE (Advanced Intrusion Detection Environment)**

```bash
sudo apt install aide -y
sudo aideinit
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
sudo aide --check
```

- Automate periodic integrity checks with cron jobs, e.g., daily or weekly.
- Detects unauthorized changes to system files and configuration.

**OSSEC (Open Source HIDS)**

- Host-based Intrusion Detection System (HIDS).
- Steps: install via package or from source, then configure rules and integrate with central logging for multi-server environments.
- Provides alerts for suspicious activity and central management of multiple hosts.


### Notes and Best Practices

- Regularly update audit rules for critical system files (`/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, etc.).
- Ensure logs are centralized, backed up, and rotated to prevent disk space exhaustion.
- Combine AIDE with auditd logs for comprehensive monitoring.
- Consider integrating OSSEC with SIEM solutions for enterprise environments.

---

## Kernel & System Parameter Hardening

Kernel and system parameter hardening helps secure the OS against network attacks, information leaks, and unauthorized access.


### Network Security Parameters (`/etc/sysctl.conf`) 

For permanent configuration, edit `/etc/sysctl.conf`.

| Parameter                                   | Recommended Value      | Purpose                                                |
| ------------------------------------------- | ---------------------- | ------------------------------------------------------ |
| net.ipv4.ip\_forward                        | 0                      | Disable IPv4 forwarding                                |
| net.ipv4.conf.all.rp\_filter                | 1                      | Enable source IP address validation (anti-IP spoofing) |
| net.ipv4.conf.default.rp\_filter            | 1                      | Same as above                                          |
| net.ipv4.tcp\_syncookies                    | 1                      | Enable SYN flood protection                            |
| net.ipv4.conf.all.accept\_source\_route     | 0                      | Disable source-routed packets                          |
| net.ipv4.conf.default.accept\_source\_route | 0                      | Same as above                                          |
| net.ipv4.icmp\_echo\_ignore\_broadcasts     | 1                      | Ignore ICMP broadcast requests                         |
| net.ipv4.conf.all.send\_redirects           | 0                      | Disable sending ICMP redirects                         |
| net.ipv6.conf.all.disable\_ipv6             | 1 (if IPv6 not needed) | Disable IPv6                                           |
| net.ipv6.conf.default.disable\_ipv6         | 1 (if IPv6 not needed) | Disable IPv6                                           |
| kernel.randomize\_va\_space                 | 2                      | Enable Address Space Layout Randomization (ASLR)       |
| fs.suid\_dumpable                           | 0                      | Disable core dumps for SUID binaries                   |

Disabling prevents attackers from finding sensitive info in dumps.

**Apply the changes immediately:**

```bash
sudo sysctl -p
```

> **Explanation of important parameters:**
>
> - `net.ipv4.ip_forward = 0` → Prevents IP forwarding to reduce risk of acting as a router.
> - `rp_filter = 1` → Reject packets with spoofed source addresses.
> - `tcp_syncookies = 1` → Mitigates SYN flood DoS attacks.
> - `accept_source_route = 0` → Prevent attackers from specifying packet routes.
> - `icmp_echo_ignore_broadcasts = 1` → Protects against Smurf attacks.
> - `disable_ipv6 = 1` → Turn off IPv6 if not required.
> - `kernel.randomize_va_space = 2` → Randomizes memory layout to hinder exploits.
> - `fs.suid_dumpable = 0` → Prevents sensitive information leaks via core dumps.


### Restrict Core Dumps (`/etc/security/limits.conf`) 

To prevent sensitive information leakage through core dumps:

```
- hard core 0
- soft core 0
```

- This disables core dumps globally for all users.
- `fs.suid_dumpable = 0` further ensures SUID binaries cannot produce core dumps.


### Notes and Best Practices 

- Always backup `/etc/sysctl.conf` before making changes:

```bash
sudo cp /etc/sysctl.conf /etc/sysctl.conf.bak
```

- After editing `/etc/sysctl.conf`, apply changes:

```bash
sudo sysctl -p
```

- Regularly review kernel and sysctl settings to ensure alignment with security policies.
- Use combination of `sysctl`, `limits.conf`, and system hardening guides for comprehensive security.

---

## Application Security

Application security involves hardening applications, containers, and services to minimize the attack surface and enforce least privilege.


### Least Privilege & Sandboxing 

- Run all applications under dedicated low-privilege users.
  Example for a web application:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin mywebapp
```

- Use **AppArmor** or **SELinux** profiles to confine major apps such as web servers, databases, and custom applications.
- Avoid running services as root wherever possible.


### Container Security (Docker / Podman) 

- Use rootless containers whenever possible.
- Regularly update container images and scan them for vulnerabilities.
- Limit container capabilities and use security profiles (e.g., AppArmor).
- Example Docker installation and user setup:

```bash
sudo apt install docker.io -y
sudo usermod -aG docker youruser
```

- Run Docker Bench Security to check for common misconfigurations:

```bash
docker run -it --net host --pid host --userns host --cap-add audit_control \
-v /etc:/etc -v /usr/bin/docker-containerd:/usr/bin/docker-containerd \
--label docker_bench_security \
docker/docker-bench-security
```


### Web Application Hardening 

**Apache**

- Disable directory listing.
- Remove test/example files.
- Configure SSL/TLS properly (`ssl.conf`).
- Restrict server tokens.
- Configuration file: `/etc/apache2/apache2.conf`

**Nginx**

- Disable server tokens:

```nginx
server_tokens off;
```

- Use strong TLS protocols:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

- Configuration file: `/etc/nginx/nginx.conf`

**PHP**

- Disable risky functions in `php.ini`:

```ini
disable_functions = exec,passthru,shell_exec,system
```

- Limit accessible directories:

```ini
open_basedir = /var/www/html:/tmp
```

- Set `expose_php = Off`
- Limit file uploads and execution permissions.

**MySQL / MariaDB**

- Bind database service to localhost unless remote access is required:

```ini
bind-address = 127.0.0.1
```

- Use strong user passwords.
- Disallow remote root login.
- Remove anonymous users.


### Best Practices 

- Always run applications and services with **minimal privileges**.
- Maintain **updated containers and images** to avoid vulnerabilities.
- Regularly scan for misconfigurations in containers and web apps.
- Combine **AppArmor/SELinux** with application hardening for layered security.
- Maintain proper TLS configurations for web servers and enforce secure coding practices in applications.


---

## Backup & Recovery

Reliable backups are essential for recovering from system failures, ransomware, or accidental data loss. Backups should be **automated, encrypted, and regularly tested**.


### Backup Strategies

- Automate regular backups using tools like:

  - **rsync**: Lightweight file synchronization.
  - **rsnapshot**: Incremental backup system.
  - **Borg**: Deduplicating and encrypted backups.
  - **Restic**: Encrypted and deduplicated backups.

- Store backups **offsite** or **offline** to protect against local disasters and ransomware.

- Maintain **backup logs** securely for auditing and verification.


### Backup Encryption 

- Use built-in encryption options of backup tools (Borg, Restic).
- Or use **GPG** for manual encryption:

```bash
gpg -c backup.tar.gz
```

- Ensure the encryption key or passphrase is stored securely and separately from the backup.


### Testing Recovery Procedures 

- Regularly **test restores** to ensure backup integrity.
- Verify both **partial file recovery** and **full system restoration**.
- Document recovery steps and review periodically to ensure they work under emergency conditions.


### Example: Restic Encrypted Backup 

```bash
sudo apt install restic
restic init --repo /path/to/backup
restic backup /path/to/data
```

- Test recovery regularly:

```bash
restic restore latest --target /restore/path
```


### Best Practices 

- Automate backups with cron or systemd timers.
- Keep multiple backup versions for redundancy.
- Encrypt backups to protect sensitive data.
- Store backup logs securely and monitor for failures.
- Periodically verify that both encryption and restore procedures work correctly.

---

## Compliance & Best Practices


Ensuring compliance with recognized security benchmarks and continuously assessing your system is essential for maintaining a secure Ubuntu 22.04 environment.


### CIS Benchmarks 

- Download and review the [CIS Ubuntu 22.04 Benchmark](https://www.cisecurity.org/benchmark/ubuntu_linux/).
- Perform checks using tools or manually:

**Lynis Audit:**

```bash
sudo apt update
sudo apt install lynis -y
sudo lynis audit system
```

- Review the output for actionable recommendations and follow CIS guidelines.

**OpenSCAP Audit:**

```bash
sudo apt install openscap-scanner -y
# Evaluate using OVAL definitions
sudo oscap oval eval --results results.xml --report report.html oval:com.ubuntu.jammy
# Or XCCDF profiles
sudo oscap xccdf eval --profile <profile_id> --report report.html /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

- Use **Ubuntu Security Guide (USG)** for additional CIS alignment if desired.


### Security Scanning Tools 

- **Lynis:** Performs comprehensive system audits and provides remediation suggestions.
- **OpenSCAP:** Scans using standardized security content for compliance (CIS, NIST, or custom profiles).
- Schedule regular scans to maintain compliance and detect drift from best practices.


### Continuous Security Assessment

- Integrate regular security scanning into **CI/CD pipelines** for applications and system configurations.
- Schedule recurring audits using Lynis or OpenSCAP.
- Perform internal penetration testing periodically.
- Keep up to date with security advisories:

  - Ubuntu security mailing lists
  - Official CIS and OpenSCAP content updates
  - Vulnerability feeds and patch notices


### Best Practices

- Review CIS benchmark results and implement recommended changes.
- Maintain detailed reports of scans and audits for tracking compliance.
- Automate compliance checks where possible using scripts or security management tools.
- Ensure continuous monitoring to detect misconfigurations or deviations from security policies.

---

## Quick Checklist

| Area                            | Key Hardening Steps / Actions                                                                                     | Command Example / File Path / Notes                                                              |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **System Updates**              | Enable unattended-upgrades; regular `apt update`                                                                  | `/etc/apt/apt.conf.d/50unattended-upgrades`                                                      |
| **User Accounts**               | Remove unused accounts; enforce strong password policy; MFA                                                       | `/etc/passwd`, `/etc/shadow`, `/etc/login.defs`                                                  |
| **SSH**                         | Change default port; disable root login; enforce key-based auth; enable fail2ban                                  | `/etc/ssh/sshd_config`, `/etc/fail2ban/`                                                         |
| **Firewall / Network**          | Deny all incoming by default; allowlist essential ports; configure UFW                                            | `ufw`, `iptables`                                                                                |
| **Filesystem**                  | Secure `/tmp`, restrict permissions, encrypt disks                                                                | `chmod`, `umask`, LUKS/cryptsetup                                                                |
| **Service Security**            | Remove/disable unneeded services; apply AppArmor or SELinux profiles                                              | `systemctl`, `/etc/apparmor.d/`                                                                  |
| **Audit & Monitoring**          | Enable `auditd`; configure logging; install AIDE; centralized logging                                             | `/etc/audit/auditd.conf`, `/etc/rsyslog.conf`                                                    |
| **Kernel & Sysctl**             | Harden via `/etc/sysctl.conf`; disable IPv6 if not needed; restrict core dumps                                    | `/etc/sysctl.conf`, `/etc/security/limits.conf`                                                  |
| **Application Security**        | Run apps as non-root; enforce least privilege; isolate containers; harden web servers, PHP, MySQL                 | AppArmor, SELinux, container configs, `/etc/apache2/`, `/etc/nginx/`, `/etc/php/`, MySQL configs |
| **Containers (Docker/Podman)**  | Use rootless containers; update and scan images; limit capabilities; run Docker Bench Security                    | `docker`, `docker/docker-bench-security`                                                         |
| **Backup & Recovery**           | Automate regular backups; encrypt; store offsite/offline; test restores                                           | rsnapshot, Borg, Restic, GPG                                                                     |
| **Compliance & Best Practices** | Follow CIS benchmarks; scan with Lynis/OpenSCAP; integrate into CI/CD; schedule audits; perform penetration tests | `/usr/local/lynis`, `oscap`, Ubuntu Security Guide                                               |


### Additional Recommendations 

- Always test all changes in a **non-production environment**.
- Document all configurations and maintain **change logs**.
- Perform periodic reviews, audits, and vulnerability scans.
- Keep up to date with security advisories and patches.
- Use **layered security**: combine OS hardening, audit, kernel tweaks, app security, backup, and compliance.

By implementing the measures above, Ubuntu 22.04 LTS servers can be robustly defended against contemporary attacks and misconfigurations—supporting secure production workloads and regulatory compliance.

---

Follow me for more security write-ups:
[X/Twitter](https://x.com/KAshSecurity) | [YouTube](https://www.youtube.com/@SecurityWithKAsh) | [GitHub](https://github.com/KAshSecurity)