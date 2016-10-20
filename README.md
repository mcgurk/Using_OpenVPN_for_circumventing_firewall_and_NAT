# Using OpenVPN for circumventing firewall and NAT when you want to make server accessible from Internet


## Server

Log in as root or give command `sudo su -`

### Install openvpn and easy-rsa
```
apt install openvpn easy-rsa
```
### Prepare keys and certificates
```
mkdir -p /etc/openvpn/easy-rsa
cp -R /usr/share/easy-rsa/* easy-rsa/
cd /etc/openvpn/easy-rsa
edit /etc/openvpn/easy-rsa/vars:
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="mail@domain"
export KEY_EMAIL=mail@domain

mkdir keys
touch keys/index.txt
echo 01 > keys/serial
. ./vars
./clean-all
```

### Make server keys and certificates
```
./build-ca
./build-key-server server
```
You can aswer just with enter every question but two last you have to give y.
```
./build-dh
```
### Make client keys and certificates
```
./build-key raspberrypi
```
You can aswer just with enter every question but two last you have to give y.

If you want password for client (if you want to connect automatically to server, don't do this):
`./build-key-pass raspberrypi`

### Tar keys and certificates so you can copy them as normal user (root-login may be disabled in ssh)
```
tar cvf /raspberrypi_keys.tar {keys/ca.crt,keys/raspberrypi.crt,keys/raspberrypi.key}
```
### Start openvpn server for testing
```
openvpn --dev tun1 --ifconfig 10.9.8.1 10.9.8.2 --tls-server --dh /etc/openvpn/easy-rsa/keys/dh2048.pem --ca /etc/openvpn/easy-rsa/keys/ca.crt --cert /etc/openvpn/easy-rsa/keys/server.crt --key /etc/openvpn/easy-rsa/keys/server.key --reneg-sec 60 --verb 5
```

## Client
### Install programs and get client keys and certificates
```
sudo apt install openvpn easy-rsa
sudo mkdir -p /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
sudo scp yourusernameinserver@your_server_ip_or_address:/raspberrypi_keys.tar .
sudo tar xvf raspberrypi_keys.tar
```
### Start openvpn client for testing
```
sudo openvpn --remote your_server_ip_or_address --dev tun1 --ifconfig 10.9.8.2 10.9.8.1 --tls-client --ca /etc/openvpn/easy-rsa/keys/ca.crt --cert /etc/openvpn/easy-rsa/keys/raspberrypi.crt --key /etc/openvpn/easy-rsa/keys/raspberrypi.key --reneg-sec 60 --verb 5
```
If you get in both ends "Initialization Sequence Completed", everything is ok.
you can test ping server from raspberrypi: `ping 10.9.8.1`
and ping raspberrypi from server: `ping 10.9.8.2`

