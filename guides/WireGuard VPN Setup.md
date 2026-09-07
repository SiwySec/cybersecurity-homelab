WireGuard VPN Setup Guide for Ubuntu 26.04 LTS

This guide provides a simplified, streamlined installation of the **WireGuard** VPN server paired with the modern **WG-Easy v15** web dashboard on **Ubuntu 26.04 LTS**. It automatically resolves kernel module (`iptables`) and AppArmor restriction issues specific to newer Ubuntu releases in containerized environments.

## Prerequisites

- A VPS running **Ubuntu 26.04 LTS**.
- Basic SSH access to your server (IP address and SSH port).

---

## Step 1: One-Command System Preparation

Log in to your VPS via SSH and run the following combined command. It installs Docker, enables IP forwarding, loads necessary kernel modules, adjusts AppArmor profiles for WireGuard, and configures the UFW firewall:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && \
sudo usermod -aG docker $USER && \
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf && sudo sysctl -p && \
sudo modprobe ip_tables iptable_nat && \
sudo mkdir -p /etc/apparmor.d/disable && \
sudo ln -sf /etc/apparmor.d/wg /etc/apparmor.d/disable/ 2>/dev/null; \
sudo ln -sf /etc/apparmor.d/wg-quick /etc/apparmor.d/disable/ 2>/dev/null; \
sudo apparmor_parser -R /etc/apparmor.d/wg 2>/dev/null; \
sudo apparmor_parser -R /etc/apparmor.d/wg-quick 2>/dev/null; \
sudo ufw allow 51820/udp && sudo ufw reload
```

---

## Step 2: Deploy WG-Easy v15 via Docker Compose

1. Refresh Docker group permissions:
  
  ```bash
  newgrp docker
  ```
  
2. Create a working directory and navigate into it:
  
  ```bash
  mkdir -p ~/wg-easy && cd ~/wg-easy
  ```
  
3. Create the configuration file `docker-compose.yml`:
  
  ```bash
  nano docker-compose.yml
  ```
  
4. Paste the following configuration, save (`Ctrl + O`, `Enter`), and exit (`Ctrl + X`):
  
  ```yaml
  services:
    wg-easy:
      image: ghcr.io/wg-easy/wg-easy:15
      container_name: wg-easy
      privileged: true
      security_opt:
        - apparmor=unconfined
        - systempaths=unconfined
      environment:
        - LANG=en
        - PORT=51821
        - INSECURE=true
      volumes:
        - ./etc_wireguard:/etc/wireguard
        - /lib/modules:/lib/modules:ro
      ports:
        - "51820:51820/udp"
        - "127.0.0.1:51821:51821/tcp"
      restart: unless-stopped
      cap_add:
        - NET_ADMIN
        - SYS_MODULE
      sysctls:
        - net.ipv4.ip_forward=1
        - net.ipv4.conf.all.src_valid_mark=1
  ```
  
5. Start the WG-Easy container:
  
  ```bash
  docker compose up -d
  ```
  

---

## Step 3: Secure Web Dashboard Access (SSH Tunneling)

For security reasons, the Web Dashboard is bound only to `localhost` (`127.0.0.1`). Access it via an SSH tunnel from your **local computer** (Terminal on Linux/macOS or PowerShell/Command Prompt on Windows):

```bash
ssh -N -L 51821:127.0.0.1:51821 -p <SSH_PORT> <USER>@<VPS_PUBLIC_IP>
```

*Example:*

```bash
ssh -N -L 51821:127.0.0.1:51821 -p 2222 ubuntu@57.128.248.235
```

---

## Step 4: First-Time Web Setup

1. Open your browser on your local machine and go to: `http://127.0.0.1:51821`
2. Follow the **Setup Wizard**:
  - **Existing configuration?** Choose **No**.
  - **Admin Credentials:** Create your username and a strong admin password.
  - **Host Setup:** Enter your VPS public IP address and default WireGuard port (`51820`).
3. You can now generate `.conf` profiles and QR codes for mobile devices and PCs.
