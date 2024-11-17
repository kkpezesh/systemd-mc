### 411
This repo roughly documents the steps I followed to setup a Minecraft server on an Ubuntu VPS for the first time
* **Heavily** influenced by [WilhelmRoscher/minecraft-systemd-service-file](https://github.com/WilhelmRoscher/minecraft-systemd-service-file)
* [Minecraft Wiki](https://minecraft.wiki/w/Tutorials/Setting_up_a_server) is another insightful reference

### Setup SSH config (optional)
Add VPS entry to your ssh config (~/.ssh/config)
```
Host vps
  HostName X.X.X.X
  User root
  Port 22
  IdentityFile ~/.ssh/id_ed25519
```

### Prepare VPS environment
We'll log in as root to simplify all remaining steps
```
ssh vps

su -
```

Install suitable Java runtime environment
```
apt update && apt upgrade
apt install openjdk-21-jre-headless
```

Setup Linux group + user to run server ([PoLP](https://en.wikipedia.org/wiki/Principle_of_least_privilege) 😂)
```
mkdir -p /var/minecraft
groupadd -f minecraft
useradd -g minecraft -s /bin/bash -d /var/minecraft minecraft
chown -R minecraft:minecraft /var/minecraft
```

Download server jar
* the [official download](https://www.minecraft.net/en-us/download/server) is limited to latest release
* for historical releases, consider using https://mcversions.net

```
curl "https://piston-data.mojang.com/v1/objects/59353fb40c36d304f2035d51e7d6e6baa98dc05c/server.jar" > /var/minecraft/server.jar
```

### Complete server config
Accept EULA after first startup attempt and start configuring your JVM settings
* -Xmx sets max heap size
* -Xms sets initial heap size
* -XX:+UnlockExperimentalVMOptions enables use of non-default [garbage collector (GC)](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)) settings
* -XX:+UseZGC to use the [Z Garbage Collector](https://docs.oracle.com/en/java/javase/21/gctuning/available-collectors.html)

```
/usr/bin/java -jar /var/minecraft/server.jar --nogui
vim /var/minecraft/eula.txt

/usr/bin/java -Xmx3072M -Xms1024M -XX:+UnlockExperimentalVMOptions -XX:+UseZGC -jar server.jar --nogui
```

Configure server.properties
* use Minecraft wiki [Server.properties](https://minecraft.wiki/w/Server.properties) for a thorough description of each field
* ex: set difficulty to hard and set server-port to 443

### Setup Systemd service
Setup minecraft systemd service
* copy/paste using the minecraft.service file in this repo
* set ExecStart to match your JVM tuning in the previous setps

```
touch /etc/systemd/system/minecraft.service
vim /etc/systemd/system/minecraft.service
```

### Misc systemd
[Arch](https://wiki.archlinux.org/title/Systemd) has helpful documentation here

Start a systemd service (.service is optional)
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
