Step-by-step guide on installing a Luanti (Minecraft-like) server with the **VoxelLibre** game engine on Ubuntu 26.04 using Docker.

## Step 1: System Update & Tools Installation

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-v2 git unzip
```
## Step 2: Granting Permissions to Run Docker (Without Sudo)


By default, Docker requires administrator privileges. Add your user (e.g., `ubuntu`) to the `docker` group:

Bash

```
sudo usermod -aG docker $USER
```

To apply the changes to the current terminal session (without logging out), run:

Bash

```
newgrp docker
```

## Step 3: Preparing Directories & Downloading VoxelLibre

Create a folder structure for the server, then clone the VoxelLibre game files from the official GitHub repository:

Bash

```
mkdir -p ~/luanti/data ~/luanti/config ~/luanti/games ~/luanti/mods
cd ~/luanti

git clone [https://github.com/VoxelLibre/VoxelLibre.git](https://github.com/VoxelLibre/VoxelLibre.git) ~/luanti/games/voxelibre
```

## Step 4: Server Configuration (`minetest.conf`)

Create a configuration file to define game settings (creative mode, disabled damage, and administrator name):

Bash

```
nano ~/luanti/config/minetest.conf
```

Paste the configuration below (replace `TwojNickWGrze` with your actual login):

Ini, TOML

```
# Podstawowe parametry serwera
server_name = Kreatywny Swiat Jakuba Wymiatacza
server_description = Serwer demonstracyjny Luanti z gry VoxeLibre
max_users = 15

# Ustawienia rozgrywki (Tryb Kreatywny)
creative_mode = true
enable_damage = false

# Nazwa konta z pełnymi uprawnieniami administratora
name = TwojNickWGrze
```

Save the file (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

## Step 5: Container Configuration (`compose.yml`)

Create a Docker Compose configuration file:

Bash

```
nano compose.yml
```

Paste the following content:

YAML

```
services:
  luanti:
    image: ghcr.io/luanti-org/luanti:latest
    container_name: luanti_server
    restart: unless-stopped
    volumes:
      - ./config:/etc/minetest
      - ./data:/var/lib/minetest
      - ./games:/var/lib/minetest/.minetest/games
      - ./mods:/var/lib/minetest/.minetest/mods
    ports:
      - "30000:30000/udp"
    command: --gameid voxelibre --worldname swiat_kreatywny
```

Save the file and exit.

## Step 6: Securing Container Permissions (UID 30000)

For security reasons, the Luanti container runs as an unprivileged user with UID 30000. Change ownership of the `data` directory on the host so the container can write logs and world files:

Bash

```
sudo chown -R 30000:30000 ~/luanti/data
```

## Step 7: Unblocking Port in Firewall

The Luanti server communicates using the UDP protocol on port 30000. Open this port in UFW:

Bash

```
sudo ufw allow 30000/udp
```

## Step 8: Startup & Verification

Start the server in the background:

Bash

```
docker compose up -d
```

Monitor server activity and map generation in real time:

Bash

```
docker logs -f luanti_server
```

*(To exit log view, press `Ctrl+C`)*

## Step 9: Connecting to the Server

1. Download the free game client from [luanti.org](https://www.luanti.org/).
  
2. Navigate to the **Join Game** tab.
  
3. Enter your VPS public IP address in the **Address** field.
  
4. Set the **Port** field to `30000`.
  
5. Enter your username (matching the `name` parameter in `minetest.conf`).
  
6. Upon first login, set any password (it will be associated with your account).
  
7. After joining, open chat and type `/privs` to confirm administrator privileges.
