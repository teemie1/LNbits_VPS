
# ติดตั้ง LNbits และทำ Clearnet บน VPS (LN Node Workshop)
<img src="tunnels-shanghai.jpg" />
ขั้นตอนต่างๆ ในการติดตั้งอ้างอิงจาก [TrezorHannes/vps-lnbits](https://github.com/TrezorHannes/vps-lnbits) แต่มีการปรับค่าพารามิเตอร์บางอย่างให้ถูกต้องและเหมาะสมกับการใช้งาน node ของเรา ผมจะขอจัดกลุ่มขั้นตอนการติดตั้งใหม่ดังนี้

- [เตรียมความพร้อม](#เตรียมความพร้อม)
  - [VPS: ติดตั้ง VPS บน Cloud](#vps-ติดตั้ง-vps-บน-cloud)
  - [VPS: hardening VPS ให้เหมาะสม](#vps-hardening-vps-ให้เหมาะสม)
- [ติดตั้งและ config OpenVPN บน VPS และ node](#ติดตั้งและ-config-openvpn-บน-vps-และ-node)
  - [VPS: ติดตั้ง OpenVPN Server บน VPS พร้อม config & certificate](#vps-ติดตั้ง-openvpn-server-บน-vps-พร้อม-config--certificate)
  - [Node: ติดตั้งและทดสอบ VPN Tunnel บน node](#node-ติดตั้งและทดสอบ-vpn-tunnel-บน-node)
  - [VPS: ทำ port forward ใน docker ของ OpenVPN](#vps-ทำ-port-forward-ใน-docker-ของ-openvpn)
- [แก้ไขพารามิเตอร์บน LND สำหรับรองรับ hybrid mode และ LNbits](#แก้ไขพารามิเตอร์บน-lnd-สำหรับรองรับ-hybrid-mode-และ-lnbits)
  - [Node: แก้ไขพารามิเตอร์บน LND](#node-แก้ไขพารามิเตอร์บน-lnd)
  - [Node: Restart LND](#node-restart-lnd)
- [ติดตั้ง LNbits บน VPS พร้อมทั้ง generate certificate](#ติดตั้ง-lnbits-บน-vps-พร้อมทั้ง-generate-certificate)
  - [VPS: ติดตั้ง LNbits บน VPS](#vps-ติดตั้ง-lnbits-บน-vps)
  - [VPS: ทำ Domain, Webserver และ SSL Certificate](#vps-ทำ-domain-webserver-และ-ssl-certificate)

## เตรียมความพร้อม
ในขั้นตอนแรก เราจำเป็นต้องสมัคร VPS บน Cloud ซึ่งมีให้เลือกหลากหลายโดยที่ผมเคยใช้คือ Lunanode หรือถ้าใครต้องการใช้ที่อื่นๆ เช่น Digital Ocean, Amazon Web Server เป็นต้น ใครสนใจ Lunanode สามารถใช้ [referal link](https://www.lunanode.com/?r=21167) ของผมได้ครับ

### VPS: ติดตั้ง VPS บน Cloud
เมื่อสมัคร cloud เรียบร้อยให้ทำการสร้าง VPS ขึ้นมาใหม่ โดยเลือก OS เป็น Ubuntu และจำเป็นต้องใช้ Public IP ให้จด Public IP ที่ใช้บน VPS ไว้เพราะ IP นี้จำเป็นต้องสำหรับการเชื่อมต่อของ LND และ LNbits 

### VPS: hardening VPS ให้เหมาะสม
ทำการอัพเกรด OS และ Config UFW
~~~
apt-get update
apt-get upgrade
apt-get install docker.io tmux
systemctl start docker.service
apt install ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow OpenSSH
ufw allow 80 comment 'Standard Webserver'
ufw allow 443 comment 'SSL Webserver'
ufw allow 9735 comment 'LND Main Node 1'
ufw enable
sudo apt install fail2ban
~~~

## ติดตั้งและ config OpenVPN บน VPS และ node
ขั้นตอนนี้เป็นการสร้าง tunnel ระหว่าง VPS และ node ให้สามารถเชื่อมต่อกันได้แบบปลอดภัย

### VPS: ติดตั้ง OpenVPN Server บน VPS พร้อม config & certificate
~~~
export OVPN_DATA="ovpn-data"
echo 'export OVPN_DATA="ovpn-data"' >> ~/.bashrc

sudo docker volume create --name $OVPN_DATA
sudo docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://<PUBLIC IP>
sudo docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
# Generate key for ca, use passphrase as usual

sudo docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp -p 9735:9735 -p 8080:8080 --cap-add=NET_ADMIN kylemanna/openvpn
sudo docker ps
# จดหมายเลข CONTAINER ID

sudo docker exec -it <CONTAINER ID> sh
ifconfig
# internal ip (docker): 172.17.0.2

exit

docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full node1 nopass
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient node1 > node1.ovpn
~~~

### Node: ติดตั้งและทดสอบ VPN Tunnel บน node
~~~
cd ~
mkdir VPNcert
scp ubuntu@<PUBLIC IP>:/home/ubuntu/node1.ovpn /home/admin/VPNcert/
chmod 600 /home/admin/VPNcert/node1.ovpn
sudo apt-get install openvpn
sudo cp -p /home/admin/VPNcert/node1.ovpn /etc/openvpn/CERT.conf
sudo systemctl enable openvpn@CERT
sudo systemctl start openvpn@CERT

~~~
หลังจาก node ของเรา connect ไปที่ VPS ด้วย OpenVPN สำเร็จ node และ VPS จะคุยกันได้ภายใน tunnel และจะมี Private IP ภายใน tunnel ดังนี้
~~~
VPS IP  : 192.168.255.1
Node IP : 192.168.255.6
~~~


### VPS: ทำ port forward ใน docker ของ OpenVPN
~~~
docker ps
sudo docker exec -it <CONTAINER ID> sh
iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9735 -j DNAT --to 192.168.255.6:9735
iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 8080 -j DNAT --to 192.168.255.6:8080
iptables -t nat -A POSTROUTING -d 192.168.255.0/24 -o tun0 -j MASQUERADE

cd /etc/openvpn
vi ovpn_env.sh
# Add the following line to the file
iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9735 -j DNAT --to 192.168.255.6:9735
iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 8080 -j DNAT --to 192.168.255.6:8080
iptables -t nat -A POSTROUTING -d 192.168.255.0/24 -o tun0 -j MASQUERADE

exit
~~~

## แก้ไขพารามิเตอร์บน LND สำหรับรองรับ hybrid mode และ LNbits
เพื่อให้ LND เราสามารถใช้ Hybrid Mode และ LNbits จำเป็นต้องแก้ไขไฟล์ lnd.conf และ restart lnd

### Node: แก้ไขพารามิเตอร์บน LND
~~~
sudo ufw allow 9735 comment 'allow LND from outside'
sudo ufw allow 8080 comment 'allow RestLNDWallet from outside'
sudo nano /mnt/hdd/lnd/lnd.conf
# แก้ไขพารามิเตอร์
[Application Options]
externalip=<PUBLIC IP>:9735
listen=0.0.0.0:9735
restlisten=0.0.0.0:8080
tlsextraip=192.168.255.1
nat=false

[tor]
tor.active=true
tor.v3=true
tor.streamisolation=false
tor.skip-proxy-for-clearnet-targets=true
~~~

### Node: Restart LND
~~~
sudo systemctl restart lnd.service
~~~

## ติดตั้ง LNbits บน VPS พร้อมทั้ง generate certificate


### VPS: ติดตั้ง LNbits บน VPS


### VPS: ทำ Domain, Webserver และ SSL Certificate

