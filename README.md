# OpenVPN-on-OpenWRT

This Guide is showing you how to install openvpn on openwrt as a openVPN server

Mostly is based on here
https://openwrt.org/docs/guide-user/services/tor/create-tor-openvpn
AND
http://blog.sina.com.cn/s/blog_723431cd0102yimb.html

If you are justing setting up a basic router, use the script install, Nice and Easy. 
If you are like me who has bunch of random stuff already added to the router, use the manual install


This demo is based on this version 
```
OpenVPN 2.4.5 powerpc-openwrt-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [MH/PKTINFO] [AEAD]
library versions: OpenSSL 1.0.2t  10 Sep 2019, LZO 2.10
Originally developed by James Yonan
Copyright (C) 2002-2018 OpenVPN Inc <sales@openvpn.net>
```

STEPS: 

**0. Make sure you have OpenWRT unit.**
I found out that MX60 is a very powerful Gateway to play around 

**1. Update and Install the neccessary packages.**
```
ssh root@192.168.1.1(your OpenWRT IP)
opkg update
opkg install openvpn-openssl openvpn-easy-rsa luci-app-openvpn
```
**2. Generate keys using easy-rsa.**
```
cd /etc/easy-rsa
```
2.1 Before you move on to the next sub step, make sure that you back up this configuration files
```
cp vars vars-bak
```
2.2 Modify vars file to fit YOUR settings 
```
vi vars 
```
Remove the # sign in front of the following lines
```
set_var EASYRSA                 "$PWD"
set_var EASYRSA_PKI             "$EASYRSA/pki"
set_var EASYRSA_DN              "cn_only"
set_var EASYRSA_REQ_COUNTRY     "US"
set_var EASYRSA_REQ_PROVINCE    "Iowa"
set_var EASYRSA_REQ_CITY        "Orange"
set_var EASYRSA_REQ_ORG         "OrangeGate Certificate Authority"
set_var EASYRSA_REQ_EMAIL       "fixme@Orangegate.net"
set_var EASYRSA_REQ_OU          "OpenVPN EASY CA"
set_var EASYRSA_KEY_SIZE        2048
set_var EASYRSA_ALGO            rsa
set_var EASYRSA_CA_EXPIRE       3650
set_var EASYRSA_CERT_EXPIRE     3650
set_var EASYRSA_NS_SUPPORT      "no"
set_var EASYRSA_NS_COMMENT      "OpenVPN Certificate Authority"
set_var EASYRSA_EXT_DIR         "$EASYRSA/x509-types"
set_var EASYRSA_SSL_CONF        "$EASYRSA/openssl-1.0.cnf"
set_var EASYRSA_DIGEST          "sha256"
```
2.3 Generate certs 

```
cd /etc/easy-rsa/
easyrsa init-pki
easyrsa build-ca nopass
```
With nopass switch, you don't need to input the password when the you use cert. 

this process will generate 2 files. 
```
/etc/easy-rsa/pki/ca.crt
/etc/easy-rsa/pki/private/ca.key
```
2.3 Generate Diffie-Hellman keys 
Some people said it will take a long time... Well, if you are generating a 4096bit could take a long time, in this case, we 2048 is good enough. With powerful MX60, feel free to do it locally  
```
openssl dhparam 2048 -out dh2048.pem
```
2.4 Generate server-side cert and sign
```
easyrsa build-server-full server nopass
```
This cert will show in here when it is done. 
```
Certificate created at: /etc/easy-rsa/pki/issued/server.crt
```

2.5 Generate client-side cert and sign. 

```
easyrsa build-client-full client nopass
```
This cert will show in here
```
Certificate created at: /etc/easy-rsa/pki/issued/client.crt
```
2.5.1 If you want your client have their own certs 
```
easyrsa build-clinet-full your-client-name nopass
```
2.6 Generate TLS transport level key, assume you are still at /etc/easy-rsa/ folder
```
openvpn --genkey --secret ./ta.key
```
2.7 File clean up 
Let's put our file in a certal location 
```
mkdir /etc/openvpn/certs
mkdir /etc/openvpn/client
```
So far, certs folder should have ca.crt dh2048.pem server.crt server.key ta.key
client folder should have ca.crt client.crt client.key ta.key 

**3. Config /etc/config/openvpn.**
Add this at the end of this file 
```
config openvpn 'Orange_remote'
        option dev 'tun'
        option port '1194'
        option proto 'udp'
        option cipher AES-256-GCM
        option keepalive '10 60'
        option ca '/etc/openvpn/certs/ca.crt'
        option cert '/etc/openvpn/certs/server.crt'
        option key '/etc/openvpn/certs/server.key'
        option dh '/etc/openvpn/certs/dh2048.pem'
        option tls_auth '/etc/openvpn/certs/ta.key 0'
        option server '10.8.0.0 255.255.255.0'
        option topology 'subnet'
        option push 'route 192.168.1.0 255.255.255.0'
        list push 'dhcp-option DNS 192.168.1.1'
        list push 'redirect-gateway def1'
        option tls_server '1'
        option auth_nocache '1'
        option verb '1'
        option client_to_client '1'
        option float '1'
        option comp_lzo adaptive
        option persist_tun '1'
        option persist_key '1'
        option duplicate_cn '1'
        option enabled '1'
```
1）option duplicate_cn ‘1’  Allows multiple user share one certs, for easy setup
2）option AES-256-GCM       AES-256-CBC is safe enought，GCM has better security
3）option client_to_client ‘1’ Allow client communicated with each other.
4）list push ‘redirect-gateway def1’ direct all vpn traffice to main gateway

**4. Client *.ovpn file Config**
```
client
dev tun
proto udp
remote YourGateWayPUblic--IP 1194
float
cipher AES-256-GCM
comp-lzo adaptive
keepalive 10 60
remote-cert-tls server
key-direction 1
auth-nocache
resolv-retry infinite
nobind
<ca>
*********Insert your ca.crt content here*********  
</ca>
<cert>
*********Insert your client.crt content here*********
</cert>
<key>
*********Insert your client.key content here*********
</key>
<tls-auth>
*********Insert your ta.key content here*********
</tls-auth>
```
**5. Add interface config at /etc/config/network**
```
config interface 'openvpn'
    option proto 'none'
    option auto '1'
    option ifname 'tun0'
```
**6. Add settings to Firewall file /etc/config/firewall**
```
config rule
    option name 'Allow-OpenVPN-Wan-Inbound'
    option src 'wan'
    option dest_port '1194'
    option target 'ACCEPT'
    option proto 'udp'
    option family 'ipv4'
config zone
    option name 'openvpn'
    option input 'ACCEPT'
    option forward 'ACCEPT'
    option output 'ACCEPT'
    option masq '1'
    option network 'openvpn'
config forwarding
    option dest 'wan'
    option src 'openvpn'
config forwarding
    option dest 'lan'
    option src 'openvpn'
```
**7. ALL DONE, enjoy! **
