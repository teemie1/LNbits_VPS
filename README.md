
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
- [ติดตั้ง LNbits บน VPS](#ติดตั้ง-lnbits-บน-vps)
  - [Node: การคัดลอกสิทธิ์ LNbits เพื่อใช้เชื่อมต่อกับ LND](#node-การคัดลอกสิทธิ์-lnbits-เพื่อใช้เชื่อมต่อกับ-lnd)
  - [VPS: ติดตั้ง LNbits บน VPS](#vps-ติดตั้ง-lnbits-บน-vps)
  - [VPS: ทำ Domain, Webserver และ SSL Certificate](#vps-ทำ-domain-webserver-และ-ssl-certificate)

## เตรียมความพร้อม
ในขั้นตอนแรก เราจำเป็นต้องสมัคร VPS บน Cloud ซึ่งมีให้เลือกหลากหลายโดยที่ผมใช้คือ Lunanode หรือถ้าใครต้องการที่อื่นๆ ก็สามารถใช้ได้เช่น Digital Ocean, Amazon Web Server เป็นต้น ใครสนใจ Lunanode สามารถใช้ [referal link](https://www.lunanode.com/?r=21167) ของผมได้ครับ

### VPS: ติดตั้ง VPS บน Cloud
เมื่อสมัคร cloud เรียบร้อยให้ทำการสร้าง VPS ขึ้นมาใหม่ โดยเลือก OS เป็น Ubuntu และจำเป็นต้องใช้ Public IP ให้จด Public IP ที่ใช้บน VPS ไว้เพราะ IP นี้จำเป็นต้องสำหรับการเชื่อมต่อของ LND และ LNbits 

### VPS: hardening VPS ให้เหมาะสม
ทำการอัพเกรด OS และ Config UFW
~~~
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install docker.io tmux
sudo systemctl start docker.service
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80 comment 'Standard Webserver'
sudo ufw allow 443 comment 'SSL Webserver'
sudo ufw allow 9735 comment 'LND Main Node 1'
sudo ufw allow 1194 comment 'OpenVPN'
sudo ufw enable
sudo apt install fail2ban
sudo timedatectl set-timezone Asia/Bangkok
~~~

## ติดตั้งและ config OpenVPN บน VPS และ node
ขั้นตอนนี้เป็นการสร้าง tunnel ระหว่าง VPS และ node ให้สามารถเชื่อมต่อกันได้แบบปลอดภัย

### VPS: ติดตั้ง OpenVPN Server บน VPS พร้อม config & certificate
~~~
export OVPN_DATA="ovpn-data"
nano .bashrc
# เพิ่มบันทัดล่างสุด export OVPN_DATA="ovpn-data"

sudo docker volume create --name $OVPN_DATA
sudo docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://<PUBLIC IP>
sudo docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
# Generate key for ca, use passphrase as usual

sudo docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp -p 9735:9735 -p 8080:8080 --cap-add=NET_ADMIN kylemanna/openvpn
sudo docker ps
# จดหมายเลข CONTAINER ID

sudo docker exec -it $(sudo docker ps -q) sh
ifconfig
# internal ip (docker): 172.17.0.2

exit

sudo docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full node1 nopass
sudo docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient node1 > node1.ovpn
~~~

### Node: ติดตั้งและทดสอบ VPN Tunnel บน node
คัดลอกไฟล์ node1.ovpn ซึ่งเป็น client openvpn config ใช้สำหรับการเชื่อมต่อระหว่าง VPN Client และ VPN Server เพื่อสร้าง tunnel เชื่อมต่อถึงกัน
~~~
cd ~
mkdir VPNcert
scp <VPS User>@<PUBLIC IP>:node1.ovpn ~/VPNcert
chmod 600 ~/VPNcert/node1.ovpn
sudo apt-get install openvpn
sudo cp -p ~/VPNcert/node1.ovpn /etc/openvpn/CERT.conf
sudo systemctl enable openvpn@CERT
sudo systemctl start openvpn@CERT

~~~
* VPS User สำหรับ Lunanode คือ ubuntu ส่วน Digital Ocean คือ root

หลังจาก node ของเรา connect ไปที่ VPS ด้วย OpenVPN สำเร็จ node และ VPS จะคุยกันได้ภายใน tunnel และจะมี Private IP ภายใน tunnel ดังนี้
~~~
VPS IP  : 192.168.255.1
Node IP : 192.168.255.6
~~~


### VPS: ทำ port forward ใน docker ของ OpenVPN
~~~
sudo docker ps
sudo docker update --restart unless-stopped $(sudo docker ps -q)
sudo docker exec -it $(sudo docker ps -q) sh
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
allow-circular-route=1
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

## ติดตั้ง LNbits บน VPS
ขั้นตอนนี้เป็นส่วนของการติดตั้ง LNbits บน VPS เพื่อให้สามารถใช้งานได้จากภายนอกบ้าน (node เรารันในบ้าน) ทำให้สามารถใช้จ่าย bitcoin นอกสถานที่แต่ยังคงจ่ายผ่าน node ของเราเองที่รันอยู่ภายในบ้านได้

### Node: การคัดลอกสิทธิ์ LNbits เพื่อใช้เชื่อมต่อกับ LND
~~~
scp /data/lnd/tls.cert ubuntu@[PUBLIC IP]:/home/ubuntu
scp /data/lnd/data/chain/bitcoin/mainnet/admin.macaroon ubuntu@[PUBLIC IP]:/home/ubuntu
~~~

### VPS: ติดตั้ง LNbits บน VPS
~~~
sudo apt-get install git
git clone https://github.com/lnbits/lnbits-legend
sudo apt update
sudo apt install python3-venv
cd lnbits-legend
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
mkdir ~/lnbits-legend/data
cp .env.example .env
sudo nano .env
~~~
แก้ไขไฟล์ .env ดังนี้
~~~
LNBITS_DATA_FOLDER="/home/ubuntu/lnbits-legend/data"
LNBITS_BACKEND_WALLET_CLASS=LndRestWallet
LND_REST_ENDPOINT="https://172.17.0.1:8080"
LND_REST_CERT="/home/ubuntu/tls.cert"
LND_REST_MACAROON="/home/ubuntu/admin.macaroon"
~~~
ทำการ build static file และเริ่มทดสอบ start 
~~~
./venv/bin/python build.py
tmux new -s lnbits
./venv/bin/uvicorn lnbits.__main__:app --port 5000 --host 0.0.0.0
~~~

หลังจากติดตั้งเสร็จแล้ว เราจำเป็นต้องทำ lnbits services เพื่อให้ start ทุกครั้งที่ reboot เครื่อง

~~~
sudo nano /etc/systemd/system/lnbits.service
~~~
ใส่รายละเอียดดังนี้
~~~
# Systemd unit for lnbits
# /etc/systemd/system/lnbits.service

[Unit]
Description=LNbits

[Service]
WorkingDirectory=/home/ubuntu/lnbits-legend
ExecStart=/home/ubuntu/lnbits-legend/venv/bin/uvicorn lnbits.__main__:app --port 5000 --host 0.0.0.0
User=ubuntu
Restart=always
TimeoutSec=120
RestartSec=30
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
~~~

หลังจากได้ไฟล์ lnbits.service แล้ว เราต้อง enable และ start ขึ้นมาดังนี้
~~~
sudo systemctl enable lnbits.service
sudo systemctl start lnbits.service
~~~


### VPS: ทำ Domain, Webserver และ SSL Certificate

#### Domain
เราจำเป็นต้อง register domain ของเราเองขึ้นมา ซึ่งเราสามารถใช้ free domain จาก duckdns.org ได้ดังนี้
 - เข้าไปที่เว็บ https://duckdns.org
 - login ด้วย gmail หรือ github หรืออะไรก็แล้วแต่เราได้เลย
 - หลังจากนั้นให้สร้าง domain ของเราขึ้นมา โดยใส่ ip เป็น Public IP ของ VPS
 - จด token ในหน้า duckdns.org ของเราไว้และเก็บเป็นความลับ เพราะ token เปรียบเสมือน private key เพื่อแสดงความเป็นเจ้าของ domain
 
#### SSL Certificate
ใช้คำสั่งเพื่อ generate SSL Certificate สำหรับใช้งานกับ LNbits บน domain ของเราเอง 
~~~
sudo apt update
sudo apt install nginx certbot
sudo certbot certonly --manual --preferred-challenges dns
~~~
หลังจากนั้น certbot จะถามหา domain เราให้ใส่ domain ที่เราสร้างในขั้นตอนก่อนหน้า แล้วมันจะให้เราแสดงความเป็นเจ้าของด้วยการกำหนดค่า TXT Record ใน domain ของเรา ซึ่งทำได้ดังนี้
 - เปิด notepad ขึ้นมา และพิมพ์ค่า `https://www.duckdns.org/update?domains={YOURVALUE}&token={YOURVALUE}&txt={YOURVALUE}&verbose=true` โดยแทนที่ค่าต่าง ๆ ดังนี้
     - `domains={YOURVALUE}` ใส่ค่า subdomain ไม่ต้องใส่ duckdns.org เช่น `domains=teemie`
     - `token={YOURVALUE}` ใส่ค่า token ในหน้า duckdns.org 
     - `txt={YOURVALUE}` ใส่ค่าที่ได้จาก cerbot
 - ตัวอย่าง `https://www.duckdns.org/update?domains=teemie&token=a123x4bc-e221-2352-wx15-57832e1h423g&txt=_acme-challenge.teemie.duckdns.org&verbose=true` ให้ copy URL ทั้งหมดไปใส่ใน Browser และผลที่ได้จะลงท้ายด้วย OK
 - เปิดหน้าจอ terminal อีกจอ ติดตั้ง dig `sudo apt-get install dnsutils` เพื่อใช้สำหรับตรวจสอบโดยใช้คำสั่ง `dig -t txt _acme-challenge.teemie.duckdns.org` เปรียบเทียบ TXT Record ที่ได้กับ certbot ถ้าทุกอย่างตรงกัน กลับไปที่หน้าจอ certbot กด enter
 - certbot จาก generate certificate สำหรับ domain ที่เราสร้างขึ้นอยู่ใน `/etc/letsencrypt/live/teemie.duckdns.org/fullchain.pem` และ `/etc/letsencrypt/live/teemie.duckdns.org/privkey.pem`

#### Webserver NGINX
สร้างไฟล์ config สำหรับ LNbits ใน nginx
~~~
sudo nano /etc/nginx/sites-available/lnbits.conf
~~~
ใส่ตามนี้
~~~
server {
# Binds the TCP port 80
listen 80;
# Defines the domain or subdomain name
server_name <DOMAIN>.duckdns.org;
# Redirect the traffic to the corresponding
# HTTPS server block with status code 301
return 301 https://$host$request_uri;
}
server {
listen 443 ssl; # tell nginx to listen on port 443 for SSL connections
server_name <DOMAIN>.duckdns.org; # tell nginx the expected domain for requests
access_log /var/log/nginx/lnbits-access.log; # Your first go-to for troubleshooting
error_log /var/log/nginx/lnbits-error.log; # Same as above
location / {
proxy_pass http://127.0.0.1:5000; # This is your uvicorn LNbits local host IP and port
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection 'upgrade';
proxy_set_header X-Forwarded-Proto https;
proxy_set_header Host $host;
proxy_http_version 1.1; # headers to ensure replies are coming back and forth through your domain
}
ssl_certificate /etc/letsencrypt/live/teemie.duckdns.org/fullchain.pem; # Point to the fullchain.pem from Certbot
ssl_certificate_key /etc/letsencrypt/live/teemie.duckdns.org/privkey.pem; # Point to the private key from Certbot
}
~~~
ใช้คำสั่งเพื่อตรวจสอบความถูกต้องและเริ่ม start ใช้งานได้เลย
~~~
sudo nginx -t
sudo ln -s /etc/nginx/sites-available/lnbits.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx
~~~

เสร็จสิ้นทุกขั้นตอน เราจะสามารถเข้าหน้าเว็บของ LNbits ด้วย `https://teemie.duckdns.org` ซึ่งเชื่อมต่อกับ node ของเราผ่าน tunnel ที่เป็นส่วนตัวและปลอดภัย ได้จากภายนอกบ้าน ทุกที่ทั่วโลกครับ
