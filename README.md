# **Wireguard-VPN-Server-with-WebSocket-Tunneling**


## Wireguard VPN

---

On the VPN server:

Install the Wireguard management tools:

```
apt install wireguard
```


If necessary, install a generic kernel and remove the cloud kernel:
```
uname -r
apt install linux-image-amd64
apt remove linux-*-cloud-*
```


Enable IPv4 forwarding:

```
nano /etc/sysctl.d/99-sysctl.conf
```


Uncomment the following line by removing the leading # mark:

```
net.ipv4.ip_forward=1
```

Reboot the server to apply (or run `sysctl -w net.ipv4.ip_forward=1` if you didn't have to change your kernel):

```
systemctl reboot
```
--- 

On the VPN client:

Install the Wireguard management tools:

```
sudo apt install wireguard-tools
```
Generate the private and public keys:

```
sudo bash -c "umask 077 ; wg genkey > /etc/wireguard/client.key"
sudo bash -c "wg pubkey < /etc/wireguard/client.key > /etc/wireguard/client.pub"
sudo cat /etc/wireguard/client.key
```

On the VPN server:

Generate the private and public keys:

```
umask 077 ; wg genkey > /etc/wireguard/server.key
wg pubkey < /etc/wireguard/server.key > /etc/wireguard/server.pub
cat /etc/wireguard/server.pub
```


Create the WireGuard configuration file:

```
cp /etc/wireguard/server.key /etc/wireguard/wg0.conf
nano /etc/wireguard/wg0.conf
```


Enter the following configuration:


```
[Interface]
Address = 172.16.0.1/24
ListenPort = 51820
PrivateKey = (server's private key goes here)
# Firewall rules
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Client #1 details
PublicKey = (client's public key goes here)
# Traffic to route to this client
AllowedIPs = 172.16.0.2/32

```


Enable and start the WireGuard service:


On the VPN client:

Create the WireGuard configuration file:

```
sudo cp /etc/wireguard/server.key /etc/wireguard/wg0.conf sudo nano /etc/wireguard/wg0.conf
```

Enter the following configuration:

```
[Interface]
Address = 172.16.0.2/24
PrivateKey = (client's private key goes here)
# Set to your desired DNS server
DNS = 8.8.8.8

[Peer]
PublicKey = (server's public key goes here)
# Endpoint (server) can be a domain name or IP address
Endpoint = (server's IP address goes here):51820
# Traffic to route to server
AllowedIPs = 0.0.0.0/0, ::/0
# Keepalive (use if you're behind NAT)
PersistentKeepalive = 25
```

Start WireGuard:

```
sudo wg-quick up wg0
```

## WebSocket Tunnel




On the VPN server:

 Download and install wstunnel:


```
wget https://github.com/erebe/wstunnel/releases/download/v4.0/wstunnel-x64-linux
mv wstunnel-x64-linux /usr/local/bin/wstunnel
chmod uo+x /usr/local/bin/wstunnel
setcap CAP_NET_BIND_SERVICE=+eip /usr/local/bin/wstunnel
```

 Create the wstunnel unit file:

```
nano /etc/systemd/system/wstunnel.service
```

Enter the following configuration:


```
[Unit]
Description=Tunnel WireGuard UDP over websocket
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/wstunnel -v --server wss://0.0.0.0:443 --restrictTo=127.0.0.1:51820
Restart=no

[Install]
WantedBy=multi-user.target
```

Enable and start wstunnel:

```
systemctl enable --now wstunnel
```

On the VPN client:

Download and install wstunnel:

```
wget https://github.com/erebe/wstunnel/releases/download/v4.0/wstunnel-x64-linux
sudo mv wstunnel-x64-linux /usr/local/bin/wstunnel
sudo chmod +x /usr/local/bin/wstunnel
wget https://raw.githubusercontent.com/jnsgruk/wireguard-over-wss/master/wstunnel.sh
sudo mv wstunnel.sh /etc/wireguard/wstunnel.sh
sudo chmod +x /etc/wireguard/wstunnel.sh
```

Edit the wstunnel routing configuration script:

```
sudo nano /etc/wireguard/wstunnel.sh
```

In the `if [[ -z "${remote_ip}" ]]; then` statement, replace `exit 1` with `remote_ip=${remote}`.

 Create the wstunnel configuration file:

```
sudo nano /etc/wireguard/wg0.wstunnel
```

Enter the following configuration:

```
REMOTE_HOST=(server's IP address goes here)
REMOTE_PORT=51820
# Use the following line if you're connecting to your VPN server using a domain name.
#UPDATE_HOSTS='/etc/hosts'
```


 Edit the WireGuard configuration file:
```
sudo nano /etc/wireguard/wg0.conf
```

Change the `Endpoint` line to point to `127.0.0.1:51820` and add these four lines in the` [Interface]` section:

```
Table = off
PreUp = source /etc/wireguard/wstunnel.sh && pre_up %i
PostUp = source /etc/wireguard/wstunnel.sh && post_up %i
PostDown = source /etc/wireguard/wstunnel.sh && post_down %i
```

Start WireGuard again:

```
sudo wg-quick up wg0
```
 
