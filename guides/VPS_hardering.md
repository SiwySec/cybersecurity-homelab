Implementing basic security steps (so-called *hardening*) on a new VPS server is a crucial step. The most important rule for an administrator during this process is: **never close an active, working SSH session until you verify in a new window that the new security settings and ports are working correctly.** This will prevent you from accidentally locking yourself out of the server.
Below is a safe, step-by-step chronological guide for **Ubuntu 26.04**.

---

### Step 1: System Update

Before proceeding with the configuration, make sure the operating system and all packages are updated to their latest stable versions:

```shell
sudo apt update && sudo apt upgrade -y
```

---

### Step 2: Firewall Configuration (UFW)

The Uncomplicated Firewall (UFW) is installed by default in Ubuntu, but it is inactive. Before enabling it, **you must open a port for the new SSH service**. For this example, let's assume your new SSH port will be **`2222`** (you can choose any free port in the range 1024–65535, e.g., 4829).

1. Set the default rules (deny all incoming traffic; allow all outgoing traffic):
  
  
  
  ```Shell
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  ```
  
2. Allow traffic on the new SSH port:
  
  Shell
  
  ```
  sudo ufw allow 2222/tcp
  ```
  
3. Enable the firewall:
  
  Shell
  
  ```
  sudo ufw enable
  ```
  
  *(When asked about potentially disrupting existing connections, answer `y` – you have already allowed port 2222, so you are safe).*
  
4. Check the status of the firewall rules:
  
  Shell
  
  ```
  sudo ufw status verbose
  ```
  

### Step 3: Changing the SSH Port (Ubuntu 24.04 / 26.04 Specifics)

In newer versions of Ubuntu, the OpenSSH service does not run as a continuous background process by default. Instead, systemd listens for connections on port 22 using **network sockets** (*socket activation*) and starts the SSH server only when a login attempt is detected.

Therefore, simply restarting the `ssh` service after modifying the configuration file will have no effect. We must instruct systemd to generate a new listening socket based on the configuration.

1. Open the SSH configuration file:
  
  Shell
  
  ```
  sudo nano /etc/ssh/sshd_config
  ```
  
2. Find the line `#Port 22`, uncomment it (remove the `#` symbol), and change the value to your chosen port:
  
  ```
  Port 2222
  ```
  
3. Save the file (`Ctrl + O`, `Enter`, `Ctrl + X`).
  
4. **Key step for Ubuntu 26.04:** Reload the systemd manager configuration (which triggers OpenSSH's internal socket generator) and restart the SSH socket:
  
  Shell
  
  ```
  sudo systemctl daemon-reload
  sudo systemctl restart ssh.socket
  ```
  
5. Check if the system is correctly listening on the new port:
  
  Shell
  
  ```
  sudo ss -tulpn | grep ssh
  ```
  
  In the output, you should see that the systemd service (PID 1) is listening on port `2222`.
  

### Step 4: The Big Test (Do NOT close the current window!)

Open a **completely new terminal** on your local machine (Linux Mint) and try to connect to the server using the new port:

Shell

```
ssh -p 2222 tester@server_IP_address
```

If logging in with your key succeeds, the SSH and firewall configurations are working properly. You can now safely close the previous, backup terminal window.

### Step 5: Fail2ban Installation and Configuration

Fail2ban scans system logs for suspicious activity (e.g., repeated login attempts with an incorrect key or port scanning attempts) and automatically bans those IP addresses in the firewall (UFW).

1. Install the package:
  
  Shell
  
  ```
  sudo apt install fail2ban -y
  ```
  
2. Copy the default configuration file to a `.local` file (so your changes are not overwritten during future package updates):
  
  Shell
  
  ```
  sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
  ```
  
3. Open the configuration file for editing:
  
  Shell
  
  ```
  sudo nano /etc/fail2ban/jail.local
  ```
  
4. Find the `[sshd]` section (you can use `Ctrl + W` in the nano editor and type `[sshd]`). Change the default port to your custom one (e.g., `2222`) and adjust the aggressiveness parameters according to your preference:
  
  Ini, TOML
  
  ```
  [sshd]
  enabled = true
  port    = 2222
  logpath = %(sshd_log)s
  backend = %(sshd_backend)s
  maxretry = 5
  findtime = 10m
  bantime = 1h
  ```
  
  - `maxretry`: The number of failed login attempts before a ban is issued.
    
  - `findtime`: The time window during which failed attempts are counted (e.g., 5 attempts within 10 minutes).
    
  - `bantime`: The duration for which the attacker is banned (e.g., `1h` – one hour, `1d` – one day).
    
5. Save the file and restart the Fail2ban service to load the new configuration:
  
  Shell
  
  ```
  sudo systemctl enable --now fail2ban
  sudo systemctl restart fail2ban
  ```
  
6. You can check the status of your SSH protection at any time with the following command:
  
  Shell
  
  ```
  sudo fail2ban-client status sshd
  ```
  
  *(It will show information such as the number of currently banned IP addresses).*
  

After completing these steps, your VPS server is protected against the most common automated attacks and port scanning attempts.

### Additional Security and Administration Suggestions

#### 1. Automatic Security Updates (`unattended-upgrades`)

As an administrator, you might not always remember to log in daily and run `apt upgrade`. The `unattended-upgrades` tool automatically downloads and installs security patches (and only security patches, without the risk of breaking running applications).

- **Installation:**
  
  Shell
  
  ```
  sudo apt install unattended-upgrades -y
  ```
  
- **Enable:**
  
  Shell
  
  ```
  sudo dpkg-reconfigure -plow unattended-upgrades
  ```
  
  *(Select "Yes" in the window that appears).*
  

#### 2. Important Note: Docker and UFW Firewall (A Common Trap)

If you plan to deploy applications using Docker, you need to be aware of a major Linux quirk.

- **The Issue:** Docker manipulates `iptables` rules directly at the kernel level. This means **Docker ignores UFW rules by default**. If you run a container with `docker run -p 8080:80 ...`, port 8080 will be publicly accessible to the world, even if you did not explicitly allow it in UFW.
  
- **The Solution:** When running containers, always bind ports locally to the loopback interface, for example: `-p 127.0.0.1:8080:80`. External traffic will then be blocked, and you can safely access the container through a VPN or a Reverse Proxy (like Nginx Proxy Manager) exposed on ports 80/443.
  

#### 3. Monitoring Login Attempts

It is good practice to occasionally check if anyone is attempting to scan your new port. You can do this with:

Shell

```
sudo grep "Failed password" /var/log/auth.log
```

Or check Fail2ban activity:

Shell

```
sudo fail2ban-client status sshd
```

"""

with open("vps-server-hardening.md", "w", encoding="utf-8") as f:
f.write(content)

print("File generated successfully: vps-server-hardening.md")

````
```text?code_stdout&code_event_index=1
File generated successfully: vps-server-hardening.md
````
