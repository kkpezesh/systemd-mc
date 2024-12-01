### 411
This repo documents steps taken to run a Minecraft 1.21.3 server on an Ubuntu VPS

### Resources
* [Paper MC](https://docs.papermc.io/paper/getting-started)
* [WilhelmRoscher - MC systemd service file](https://github.com/WilhelmRoscher/minecraft-systemd-service-file)
* [brucethemoose - MC Performance Flags Benchmarks](https://github.com/brucethemoose/Minecraft-Performance-Flags-Benchmarks)
* [YouHaveTrouble - MC optimization](https://github.com/YouHaveTrouble/minecraft-optimization)
* [Minecraft Wiki](https://minecraft.wiki/w/Tutorials/Setting_up_a_server)
* [nftables Wiki](https://wiki.nftables.org/wiki-nftables/index.php)
* [nginx docs](https://docs.nginx.com/nginx/admin-guide)
* [Bluemap docs](https://bluemap.bluecolored.de)

### SSH config (optional)
Add VPS entry to your ssh config (~/.ssh/config)
```
Host vps
  HostName X.X.X.X
  User root
  Port 22
  IdentityFile ~/.ssh/id_ed25519
```

### SSH to VPS
This README assumes root is used to setup the server
```
ssh vps

su -
```

### Install JRE
```
apt update && apt upgrade
apt install openjdk-21-jre-headless
```

### Setup System Group + User
[PoLP](https://en.wikipedia.org/wiki/Principle_of_least_privilege) 😂
```
mkdir -p /var/minecraft
groupadd -f -r minecraft
useradd -g minecraft -s /sbin/nologin -d /var/minecraft minecraft
chown -R minecraft:minecraft /var/minecraft
```

### Install server JAR
Find desired PaperMC server versions [here](https://papermc.io/downloads/all)
```
curl "https://api.papermc.io/v2/projects/paper/versions/1.21.3/builds/60/downloads/paper-1.21.3-60.jar" > /var/minecraft/server.jar
```

Verify checksum
```
sha256sum /var/minecraft/server
chown -R minecraft:minecraft /var/minecraft
```

### Initial startup
Accept EULA after first startup
```
/usr/bin/java -jar /var/minecraft/server.jar --nogui
vim /var/minecraft/eula.txt
```

### JVM flags
* -Xmx sets max heap size
* -Xms sets initial heap size

```
/usr/bin/java -Xms12G -Xmx12G -XX:+UnlockExperimentalVMOptions -XX:+UseZGC -XX:+ZGenerational -jar server.jar --nogui
```

Configure server.properties
* use [Minecraft wiki - Server.properties](https://minecraft.wiki/w/Server.properties) for a thorough description of each field
* ex: set difficulty to hard, server-port to 444

### systemd Service Setup
Setup minecraft systemd service
* copy/paste using the minecraft.service file in this repo
* set ExecStart to match your JVM tuning in the previous setps

```
touch /etc/systemd/system/minecraft.service
vim /etc/systemd/system/minecraft.service
```

### systemd misc commands
[Arch - systemd](https://wiki.archlinux.org/title/Systemd) has helpful documentation here

Start a systemd service
```
systemctl start minecraft.service
```

Check service status
```
systemctl status minecraft
```

Stop service
```
systemctl stop minecraft
```

Restart service
```
systemctl restart minecraft
```

Start service on startup
```
systemctl enable minecraft
```

Stop service from starting on startup
```
systemctl disable minecraft
```

Tail logs
```
journalctl -u minecraft -f
```

Setting AmbientCapabilities to CAP_NET_BIND_SERVICE allows one to start a server on a [priviledged port](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html)

### nftables setup
[nftables - Quick reference-nftables in 10 minutes](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)

Setup nftables config
* drop all inbound, forward traffic by default
    * accept traffic on established/related connections, drop invalid connections
    * accept traffic on loopback interface
    * drop ipv6 traffic
    * accept TCP traffic on ports 22, 443, 444
* accept outbound traffic by default

```
vim /etc/nftables.conf
```

Start nftables on startup
```
systemctl enable nftables
```

Start nftables
```
systemctl start nftables
```

### Bluemap setup
Follow intructions at [Bluemap - Installation](https://bluemap.bluecolored.de/wiki/getting-started/Installation.html)

```
curl -L "https://github.com/BlueMap-Minecraft/BlueMap/releases/download/v5.5/bluemap-5.5-paper.jar" > /var/minecraft/plugins/bluemap-5.5-paper.jar
chown -R minecraft:minecraft /var/minecraft
```

Restart server
```
systemctl restart minecraft.service
```

Accept required downloads
```
sudo -u minecraft vim /var/minecraft/plugins/BlueMap/core.conf
```

### nginx setup
Follow [nginx - linux packages](https://nginx.org/en/linux_packages.html#Ubuntu) installation instructions


Start nginx on startup
```
systemctl enable nginx
```

Start nginx
```
systemctl start nginx
```