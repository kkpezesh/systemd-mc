### 411
This repo documents steps taken to run a Minecraft 1.21.4 server on an Ubuntu VPS

### Resources
* [FabricMC - Server](https://fabricmc.net/use/server)
* [WilhelmRoscher - MC systemd service file](https://github.com/WilhelmRoscher/minecraft-systemd-service-file)
* [brucethemoose - MC Performance Flags Benchmarks](https://github.com/brucethemoose/Minecraft-Performance-Flags-Benchmarks)
* [YouHaveTrouble - MC optimization](https://github.com/YouHaveTrouble/minecraft-optimization)
* [Minecraft Wiki](https://minecraft.wiki/w/Tutorials/Setting_up_a_server)
* [nftables Wiki](https://wiki.nftables.org/wiki-nftables/index.php)
* [nginx docs](https://docs.nginx.com/nginx/admin-guide)
* [Bluemap docs](https://bluemap.bluecolored.de)
* [Tiiffi - mcrcon](https://github.com/Tiiffi/mcrcon)

### Optional: SSH config
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

### SSHD config
Add SSHD configs to
* disconnect clients after 2 failed authentication attemps
* disable password based authentication
    * Password based authentication is fine. Beware of setting weak user passwords when this setting is enabled
* disable keyboard interactive authentication
    * Enable this feature if you setup alternative authentication methods like MFA via OTP (any sort of plugin authentication module/PAM)
* disable use of plugin authentication modules
    * see above
* disable SSH agent forwarding
    * a server with a client's SSH agent can authenticate as the client with other servers
* disable TCP forwarding
    * prevent clients from using this server as a TCP proxy
* disable X11 forwarding
    * we don't require graphical access to this server

```
MaxAuthTries 2
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM no
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
```

### Configure /etc/sysctl.conf
Run through your /etc/sysctl.conf and uncomment lines as you see fit
* ex: block ICMP/ping requests
* ex2: block ICMP redirects
* ex3: log martian packets
    * see [Martian packet](https://en.wikipedia.org/wiki/Martian_packet) for more information

Reload Linux kernel parameters
```
sysctl -p
```

### Install JRE
Install a Java runtime environment for running the server process
```
apt update && apt upgrade
apt install openjdk-21-jre-headless
```

Verify `java` is installed
```
java --version
```

### Setup System Group + User
Follow [PoLP](https://en.wikipedia.org/wiki/Principle_of_least_privilege) and set up a dedicated Linux user + group 😂
```
mkdir -p /var/minecraft
groupadd -f -r minecraft
useradd -g minecraft -s /sbin/nologin -d /var/minecraft minecraft
chown -R minecraft:minecraft /var/minecraft
```

Verify `minecraft` user creation
```
# username:password:UID:GID:GECOS:home_directory:shell
cat /etc/passwd | grep minecraft
```

Verify `minecraft` group creation
```
# group_name:password:GID:user_list
cat /etc/group | grep minecraft
```

### Install server JAR
Find desired FabricMC server versions [here](https://fabricmc.net/use/server)
```
curl -OJ https://meta.fabricmc.net/v2/versions/loader/1.21.6/0.16.14/1.0.3/server/jar
mv -i fabric-server-mc.1.21.6-loader.0.16.14-launcher.1.0.3.jar /var/minecraft
ln -siv /var/minecraft/fabric-server-mc.1.21.6-loader.0.16.14-launcher.1.0.3.jar /var/minecraft/server.jar
```

Verify `server.jar` checksum
```
sha256sum /var/minecraft/server.jar
chown -R minecraft:minecraft /var/minecraft
```

### Install Fabric API JAR
Find desired Fabric API versions [here](https://modrinth.com/mod/fabric-api)
```
cd /var/minecraft/mods
curl -OJL "https://cdn.modrinth.com/data/P7dR8mSH/versions/N3z6cNQv/fabric-api-0.127.1+1.21.6.jar"
chown -R minecraft:minecraft /var/minecraft
```

### Initial startup
Accept EULA after first startup
```
/usr/bin/java -jar /var/minecraft/server.jar --nogui
vim /var/minecraft/eula.txt
```

### JVM flags
Set JVM flags to tune server performance
* -Xmx sets max heap size
* -Xms sets initial heap size

```
/usr/bin/java -Xms12G -Xmx12G -XX:+UnlockExperimentalVMOptions -XX:+UseZGC -XX:+ZGenerational -jar server.jar --nogui
```

Configure server.properties
* use [Minecraft wiki - Server.properties](https://minecraft.wiki/w/Server.properties) for a thorough description of each field
* ex: set difficulty to hard, server-port to 7000

### systemd Service Setup
Setup minecraft systemd service
* copy/paste using the `minecraft.service` file in this repo
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

Validate nftables configuration
```
nft list ruleset
```

### Bluemap setup
Follow instructions at [Bluemap - Installation](https://bluemap.bluecolored.de/wiki/getting-started/Installation.html)

```
curl -L "https://github.com/BlueMap-Minecraft/BlueMap/releases/download/v5.9/bluemap-5.9-fabric.jar" > /var/minecraft/mods/bluemap-5.9-fabric.jar
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

### rcon setup
Setup RCON to enable remote execution of commands
* WARNING: Ensure nftables doesn't allow inbound traffic to rcon (subject to MITM). See [here](https://minecraft.wiki/w/RCON) for more info
* Install the `mcrcon` tool published at [Tiiffi/mcrcon](https://github.com/Tiiffi/mcrcon)

```
cd /root
git clone https://github.com/Tiiffi/mcrcon.git
cd /root/mcrcon
apt install make gcc
make
mv mcrcon /usr/bin
```

* Validate `mcrcon` tool is installed

```
mcrcon -h
```

* Enable RCON in server.properties and configure a port (ex: 25575)
* Optional: Update your bashrc with `mcrcon` inputs

```
vim /root/.bashrc
export MCRCON_HOST="localhost"
export MCRCON_PORT="25575"
export MCRCON_PASS="dont-expose-this-svc-to-inet-123"
```

* Restart your server and validate rcon is working

```
mcrcon
say "testing 123.. Hello Minecraft"
Q
```


### DNS
Setup DNS for your server
* Purchase domain from a domain registrar
* Setup A/AAAA record(s)
* Create NS delegations + complete zone transfer

Optional: Set your server's hostname locally

```
hostnamectl hostname my-hostname.example.com
```

### TLS certificates from Let's Encrypt
Setup a TLS certificate for your server
* Install `certbot` - `apt install certbot`
* Setup auth + cleanup hooks

Reference: https://community.hetzner.com/tutorials/letsencrypt-dns

```
certbot run -a manual -i nginx \
--register-unsafely-without-email \
--key-type ecdsa \
--elliptic-curve secp384r1 \
--preferred-challenges=dns \
--manual-auth-hook /usr/local/bin/certbot-hetzner-auth.sh \
--manual-cleanup-hook /usr/local/bin/certbot-hetzner-cleanup.sh \
-d <insert_fqdn> -d *.<insert_fqdn>
```
