# Using OpenVPN for circumventing firewall and NAT when you want to make Raspberry Pi (or any Debian based device) accessible from Internet

##  Introduction
In this example, I have Raspberry Pi behind firewall and NAT and I can't/I don't have priviledges to change firewall/NAT-rules. Even so, I want to expose some Raspberry Pi services to Internet. I have cheap virtual server (cloudatcost) with Debian, which are directly connected to Internet with static public ip-address. I use OpenVPN to connect Raspberry Pi to server and redirect some traffic from server's public ip to Raspberry Pi. Server could be also just another computer at some place where you can open ports/traffic to Internet.

Connection works automatically even if you restart server or client and connection keeps up when client changes to another network (e.g. from broadband to mobile network).

Even if you can make changes to firewall/NAT-rules, VPN can be handy. You can move Raspberry Pi to any network and it always can be accessed through server. You don't have to do holes to your own router or broadband modem, which would be gone if you have to reset your router to factory settings or if you replace it. In top of that, you don't have to care about dynamic ip-address.

### Server

Log in as root or give command `sudo su -` or `su -`.

#### Install openvpn and easy-rsa
```
apt install openvpn easy-rsa
```
#### Prepare keys and certificates
```
mkdir -p /etc/openvpn/easy-rsa
cp -R /usr/share/easy-rsa/* easy-rsa/
cd /etc/openvpn/easy-rsa
```
edit `/etc/openvpn/easy-rsa/vars` with your own stuff if you want:
```
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="mail@domain"
export KEY_EMAIL=mail@domain
```

```
mkdir keys
touch keys/index.txt
echo 01 > keys/serial
. ./vars
./clean-all
```

#### Make server keys and certificates
```
./build-ca
./build-key-server server
```
You can aswer just with enter every question but two last you have to give y.
```
./build-dh
```
#### Make client keys and certificates
```
./build-key raspberrypi
```
You can aswer just with enter every question but two last you have to give y.

If you want password for client (if you want to connect automatically to server, don't do this):
`./build-key-pass raspberrypi`

#### Assign permanent ip for client and add it to hosts
We need pemanent ip for client so we can make iptable-rules for redirecting later. Additionally we add ip and name to hosts-file, so we can later use name raspberrypi instead of 10.9.8.20.
```
mkdir /etc/openvpn/ccd
echo ifconfig-push 10.9.8.20 10.9.8.1 > /etc/openvpn/ccd/raspberrypi
echo 10.9.8.20 raspberrypi >> /etc/hosts
```

#### Tar keys and certificates so you can get them with client as normal user (root-login may be disabled in ssh)
```
tar cvf /raspberrypi_keys.tar {keys/ca.crt,keys/raspberrypi.crt,keys/raspberrypi.key}
```
#### Start openvpn server for testing
```
openvpn --dev tun1 --ifconfig 10.9.8.1 10.9.8.20 --tls-server --dh /etc/openvpn/easy-rsa/keys/dh2048.pem --ca /etc/openvpn/easy-rsa/keys/ca.crt --cert /etc/openvpn/easy-rsa/keys/server.crt --key /etc/openvpn/easy-rsa/keys/server.key --reneg-sec 60 --verb 5
```

### Client
#### Install programs and get client keys and certificates
```
sudo apt install openvpn easy-rsa
sudo mkdir /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
sudo scp yourusernameinserver@your_server_ip_or_address:/raspberrypi_keys.tar .
sudo tar xvf raspberrypi_keys.tar
```
#### Start openvpn client for testing
```
sudo openvpn --remote your_server_ip_or_address --dev tun1 --ifconfig 10.9.8.20 10.9.8.1 --tls-client --ca /etc/openvpn/easy-rsa/keys/ca.crt --cert /etc/openvpn/easy-rsa/keys/raspberrypi.crt --key /etc/openvpn/easy-rsa/keys/raspberrypi.key --reneg-sec 60 --verb 5
```
## First test
If you get in both ends "Initialization Sequence Completed", everything is ok.
You can test ping server from raspberrypi: `ping 10.9.8.1`
and ping raspberrypi from server: `ping 10.9.8.20`

## Making VPN permanent
### Server
#### /etc/openvpn/server.conf
```
port 1194
proto udp
dev tun

ca      /etc/openvpn/easy-rsa/keys/ca.crt
cert    /etc/openvpn/easy-rsa/keys/server.crt
key     /etc/openvpn/easy-rsa/keys/server.key
dh      /etc/openvpn/easy-rsa/keys/dh2048.pem

server 10.9.8.0 255.255.255.0
ifconfig-pool-persist ipp.txt
client-config-dir /etc/openvpn/ccd

keepalive 10 120

comp-lzo
persist-key
persist-tun

status log/openvpn-status.log

verb 3
client-to-client
```
#### Create log file
```
sudo mkdir /etc/openvpn/log/
sudo touch /etc/openvpn/log/openvpn-status.log
```

#### Start server
```
sudo systemctl daemon-reload
sudo systemctl restart openvpn
```

### Client
#### /etc/openvpn/client.conf
```
client
dev tun
port 1194
proto udp

remote your_server_ip_or_address 1194
nobind

ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/raspberrypi.crt
key /etc/openvpn/easy-rsa/keys/raspberrypi.key

comp-lzo
persist-key
persist-tun

verb 3
```
#### Start client
```
sudo systemctl daemon-reload
sudo systemctl restart openvpn
```
## Second test
You can test ping server from raspberrypi: `ping 10.9.8.1`.
If ping works, one big milestone is achieved - you can ssh to server and ssh from there to your client!

## Making redirections
Add this to /etc/rc.local
```
sysctl -w net.ipv4.ip_forward=1
```
Add also redirections.
### Redirection examples
#### One port from public network to VPN-client (same port in server and client)
```
# raspberrypi, node-red
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 1880 -j DNAT --to-dest 10.9.8.20:1880
iptables -t nat -A POSTROUTING -d 10.9.8.20 -p tcp --dport 1880 -j SNAT --to-source 10.9.8.1
```
#### One port from public network to VPN-client (different port in server than in client)
```
# raspberrypi, ssh accessible from server port 2222
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 2222 -j DNAT --to-dest 10.9.8.20:22
iptables -t nat -A POSTROUTING -d 10.9.8.20 -p tcp --dport 22 -j SNAT --to-source 10.8.0.1
```
#### All traffic from public network to VPN-client (Be carefull! if you redirect your prime ip, you cannot any more directly ssh to server)
```
#raspberrypi, all, server have extra ip x.x.x.x
iptables -t nat -A PREROUTING -d x.x.x.x -p tcp -j DNAT --to-dest 10.9.8.20
iptables -t nat -A POSTROUTING -d 10.9.8.20 -p tcp -j SNAT --to-source 10.9.8.1
```
## Flush
When you test different iptables rules, flush rules between tests.
/usr/local/bin/flush:
```
echo "Flushing Tables ..."

IPT="/sbin/iptables"

# Reset Default Policies
$IPT -P INPUT ACCEPT
$IPT -P FORWARD ACCEPT
$IPT -P OUTPUT ACCEPT
$IPT -t nat -P PREROUTING ACCEPT
$IPT -t nat -P POSTROUTING ACCEPT
$IPT -t nat -P OUTPUT ACCEPT
$IPT -t mangle -P PREROUTING ACCEPT
$IPT -t mangle -P OUTPUT ACCEPT

# Flush all rules
$IPT -F
$IPT -t nat -F
$IPT -t mangle -F

# Erase all non-default chains
$IPT -X
$IPT -t nat -X
$IPT -t mangle -X
```

## Multiple clients
When `client-to-client` is defined in server's config, clients see each others (e.g. you can ping 10.9.8.20 from another client). If you want isolate clients leave it out.

## Quick guide to add client
Server (log as root):
```
cd /etc/openvpn/easy-rsa
. ./vars
./build-key new_client_name
tar cvf /new_client_name_keys.tar {keys/ca.crt,keys/new_client_name.crt,keys/new_client_name.key}
echo ifconfig-push 10.9.8.x 10.9.8.1 > /etc/openvpn/ccd/new_client_name
echo 10.9.8.x new_client_name >> /etc/hosts
```
Client:
```
sudo apt install openvpn easy-rsa
sudo mkdir -p /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
sudo scp yourusernameinserver@your_server_ip_or_address:/new_client_name_keys.tar .
sudo tar xvf new_client_name_keys.tar
```
Client /etc/openvpn/client.conf:
```
client
dev tun
port 1194
proto udp

remote your_server_ip_or_address 1194
nobind

ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/new_client_name.crt
key /etc/openvpn/easy-rsa/keys/new_client_name.key

comp-lzo
persist-key
persist-tun

verb 3
```
Start client:
```
sudo systemctl daemon-reload
sudo systemctl restart openvpn
```

## Links
https://wiki.debian.org/OpenVPN

