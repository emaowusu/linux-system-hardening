# Linux System Hardening Guide for DevOps & DevSecOps Beginners (Ubuntu 22.04)

This guide walks you through practical Linux system hardening steps, specifically for **Ubuntu 22.04**, using a **VirtualBox VM** for safe testing. These steps are beginner-friendly for DevOps and DevSecOps learners.

> ⚠️ **Disclaimer:** These steps are intended for **learning and lab practice** only. They improve baseline security but do not guarantee full protection. Incorrect changes may lock you out of your VM or disrupt services. Always test on a VM first.



## Prerequisites

1. Install **VirtualBox** on your machine.
2. Create a new VM and install **Ubuntu 22.04 LTS**.
3. Ensure your VM has **network access**.
4. Use a snapshot before hardening so you can revert if needed.



## What is Linux System Hardening?

Linux hardening reduces security risk by:

* Removing unnecessary services
* Restricting access
* Applying least-privilege rules
* Enabling logging and monitoring
* Securing network and authentication settings

The Goal: **reduce your attack surface** and resist brute-force attacks.


## 1. Update and Upgrade the System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

Enable automatic security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

> **Why:** Keeps your system patched against known vulnerabilities.



## 2. Create a Non-Root Admin User

Never perform admin tasks as root.

```bash
sudo adduser devsecops
sudo usermod -aG sudo devsecops
```

Test the new user:

```bash
su - devsecops
sudo whoami  # should output 'root'
```



## 3. Harden SSH Access

### 3.1 Disable Root Login

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```
PermitRootLogin no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```



### 3.2 Change Default SSH Port

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Set a custom port above 1024:

```
Port 2222
```

Update firewall:

```bash
sudo ufw allow 2222/tcp
sudo systemctl restart ssh
```

Test:

```bash
ssh -p 2222 devsecops@<VM_IP>
```


## 4. Enable Key-Based Authentication

### Generate Key Pair (on local machine)

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# Or fallback to RSA 4096-bit:
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

### Copy Key to Server

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub devsecops@<VM_IP>
```

Verify permissions:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```


## 5. Disable Password Authentication

Once key login works, disable password login:

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```



## 6. Configure Firewall Rules

Restrict SSH to specific IPs:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 2222
sudo ufw enable
```

For iptables:

```bash
sudo iptables -A INPUT -p tcp -s 192.168.56.0/24 --dport 2222 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 2222 -j DROP
sudo iptables-save > /etc/iptables/rules.v4
```



## 7. Install and Configure Fail2ban

Install:

```bash
sudo apt install fail2ban -y
```

Copy and edit config:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Add:

```
[sshd]
enabled = true
maxretry = 3
bantime = 3600
findtime = 600
```

Start and enable:

```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
sudo fail2ban-client status sshd
```

Check failed logins:

```bash
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head
```



## 8. Enable Two-Factor Authentication (Optional)

Install Google Authenticator:

```bash
sudo apt install libpam-google-authenticator -y
```

Setup for user:

```bash
google-authenticator
```

Edit PAM for SSH:

```bash
sudo nano /etc/pam.d/sshd
```

Add at the top:

```
auth required pam_google_authenticator.so
```

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

```
ChallengeResponseAuthentication yes
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive
```

Restart SSH:

```bash
sudo systemctl restart sshd
```



## 9. Set Proper SSH Key Permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 755 ~
ls -la ~/.ssh
```



## 10. Configure Idle SSH Session Timeout

```bash
sudo nano /etc/ssh/sshd_config
```

Add:

```
ClientAliveInterval 300
ClientAliveCountMax 2
```

Restart SSH:

```bash
sudo systemctl restart ssh
```



## 11. Disable Empty Passwords

```bash
sudo nano /etc/ssh/sshd_config
```

Set:

```
PermitEmptyPasswords no
```

Audit for empty passwords:

```bash
sudo awk -F: '($2 == "") {print $1}' /etc/shadow
sudo passwd -l <username>
```


## 12. Enforce SSH Protocol Version 2

```bash
sudo nano /etc/ssh/sshd_config
```

```
Protocol 2
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

Verify:

```bash
ssh -v devsecops@<VM_IP> 2>&1 | grep "protocol"
```


## 13. Restrict SSH Access to Specific Users or Groups

Allow specific users:

```bash
sudo nano /etc/ssh/sshd_config
```

```
AllowUsers devsecops alice
```

Or restrict by group:

```bash
sudo groupadd sshusers
sudo usermod -aG sshusers devsecops
sudo nano /etc/ssh/sshd_config
```

```
AllowGroups sshusers
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

🎯 Conclusively, system hardening is **an ongoing process**. Practicing on a VirtualBox VM allows you to learn safely without risking production systems.

Regularly:

* Audit logs
* Test firewall and SSH rules
* Update and patch your VM
* Experiment with additional security tools


