# ee-ns-repro

Tests ee dns issue

## Set environment

export LOCALDNS1_IP=
export LOCALDNS2_IP=
export DOMAIN=
export EXTDNSSERVER1=
export EXTDNSSERVER2=
export CLOUDDNSIP=
export LOCALDNS1_IP=
export LOCALDNS2_IP=
export WEB_IP=
export TEECTRL_IP=
export TEEPRX_IP=
export TEE_LICENSE=

## Create ns configuration

>> On each dns server

```shell
echo 'DNSStubListener=no' | sudo tee -a /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
sudo resolvectl dns ens5 ${LOCALDNS1_IP} ${LOCALDNS2_IP}
sudo resolvectl domain ens5 ${DOMAIN}
sudo apt install dnsmasq -y
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
export TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
cat << EOF | sudo tee /etc/dnsmasq.conf
# dnsmasq options
domain-needed
bogus-priv
expand-hosts
domain=${DOMAIN}
# listening interface
listen-address=127.0.0.1
listen-address=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4)
bind-interfaces
# Use open source DNS servers
server=${EXTDNSSERVER1}
server=${EXTDNSSERVER2}
server=${CLOUDDNSIP}
EOF
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```

### add DNS entry

```shell
echo '${LOCALDNS1_IP} ns1' | sudo tee -a /etc/hosts
echo '${LOCALDNS2_IP} ns2' | sudo tee -a /etc/hosts
echo '${WEB_IP} web' | sudo tee -a /etc/hosts
echo '${TEECTRL_IP} ctr' | sudo tee -a /etc/hosts
echo '${TEEPRX_IP} prx' | sudo tee -a /etc/hosts
echo '${TEEPRX_IP} whoami' | sudo tee -a /etc/hosts
echo '${TEEPRX_IP} dashboard' | sudo tee -a /etc/hosts
```

## Setup EE controller

>> On the Controller VM

```shell
echo 'DNSStubListener=no' | sudo tee -a /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
sudo resolvectl dns ens5 ${DNS1_IP} ${DNS2_IP}
sudo resolvectl domain ens5 ${DOMAIN}
sudo groupadd -g 1500 traefikee
sudo useradd -g traefikee --no-user-group --home-dir="/opt/traefikee" --shell="/usr/sbin/nologin" --system --uid="1500" traefikee
curl -OL https://s3.amazonaws.com/traefikee/binaries/v2.12.7/traefikee_v2.12.7_linux_amd64.tar.gz && tar -xzf traefikee_v2.12.7_linux_amd64.tar.gz traefikee && rm -rf traefikee_v2.12.7_linux_amd64.tar.gz && sudo mv traefikee /usr/local/bin/. && sudo chown traefikee: /usr/local/bin/traefikee && sudo chmod +x /usr/local/bin/traefikee
export TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
sudo mkdir /opt/traefikee
sudo chown -R traefikee.traefikee /opt/traefikee
sudo chmod -R 700 /opt/traefikee
cat << EOF | sudo -u traefikee tee /opt/traefikee/controller.env
CONTROLLER_BIND_ADDRESS="$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4):4242"
TRAEFIKEE_LICENSE="${TEE_LICENSE}"
EOF
cat << EOF | sudo tee /etc/systemd/system/traefikee-controller.service
[Unit]
Description=Traefik Enterprise Controller
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
EnvironmentFile=-/opt/traefikee/controller.env
Restart=on-abnormal
User=traefikee
Group=traefikee
ExecStart=/usr/local/bin/traefikee controller --advertise=\${CONTROLLER_BIND_ADDRESS} --license=\${TRAEFIKEE_LICENSE} --api.socket=/opt/traefikee/run/teectl.sock --socket=/opt/traefikee/run/cluster.sock --statedir=/opt/traefikee/data --jointoken.file.path=/opt/traefikee/tokens --api.autocerts --configfile=/opt/traefikee/static.yaml --log.filepath=/var/log/traefikee/traefikee.log --log.format=json --log.level=DEBUG --name=ctr.${DOMAIN}
PrivateTmp=true
PrivateDevices=false
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/opt/traefikee
PermissionsStartOnly=true
ExecStartPre=/usr/bin/mkdir -p /opt/traefikee/run /opt/traefikee/data /opt/traefikee/tokens /opt/traefikee/dynamics /var/log/traefikee
ExecStartPre=-chown -R traefikee.traefikee /opt/traefikee
ExecStartPre=-chown -R traefikee.traefikee /var/log/traefikee
ExecStartPre=-chmod -R 700 /opt/traefikee

NoNewPrivileges=true
LimitNOFILE=16384

[Install]
WantedBy=multi-user.target
EOF
cat << EOF | sudo -u traefikee tee /opt/traefikee/static.yaml
entryPoints:
  web:
    address: ":80"
api:
  dashboard: true
log:
  level: DEBUG
  filePath: /var/log/traefikee/traefikee.log
accessLog:
  filePath: /var/log/traefikee/access.log
  format: json
  fields:
    defaultMode: keep
    headers:
      defaultMode: keep
providers:
  file:
    directory: /opt/traefikee/dynamics
EOF
cat << EOF | sudo -u traefikee tee /opt/traefikee/dynamics/dashboard.yaml
http:
  routers:
    dashboard:
      entryPoints:
        - "web"
      rule: Host(\`dashboard.${DOMAIN}\`)
      service: api@internal
EOF
cat << EOF | sudo -u traefikee tee /opt/traefikee/dynamics/whoami.yaml
http:
  routers:
    whoami:
      entryPoints:
        - "web"
      rule: Host(\`whoami.${DOMAIN}\`)
      service: whoami
  services:
    whoami:
      loadBalancer:
        servers:
          - url: "http://web.${DOMAIN}/"
        healthCheck:
          path: /health
          interval: "10s"
          timeout: "3s"
EOF
sudo systemctl daemon-reload
sudo systemctl start traefikee-controller
```

## Setup EE proxy

>> On the Proxy VM

```shell
echo 'DNSStubListener=no' | sudo tee -a /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
sudo resolvectl dns ens5 ${DNS1_IP} ${DNS2_IP}
sudo resolvectl domain ens5 ${DOMAIN}
export PRXTOKEN="$(ssh ctrl_vm sudo -u traefikee cat /opt/traefikee/tokens/proxies)"
sudo groupadd -g 1500 traefikee
sudo useradd -g traefikee --no-user-group --home-dir="/opt/traefikee" --shell="/usr/sbin/nologin" --system --uid="1500" traefikee
curl -OL https://s3.amazonaws.com/traefikee/binaries/v2.12.7/traefikee_v2.12.7_linux_amd64.tar.gz && tar -xzf traefikee_v2.12.7_linux_amd64.tar.gz traefikee && rm -rf traefikee_v2.12.7_linux_amd64.tar.gz && sudo mv traefikee /usr/local/bin/. && sudo chown traefikee: /usr/local/bin/traefikee && sudo chmod +x /usr/local/bin/traefikee
export TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
sudo mkdir /opt/traefikee
sudo chown -R traefikee.traefikee /opt/traefikee
sudo chmod -R 700 /opt/traefikee
cat << EOF | sudo -u traefikee tee /opt/traefikee/proxies.env
CONTROLLER_PEERS="${TEECTRL_IP}:4242"
PROXY_NODE_TOKEN="${PRXTOKEN}"
EOF
cat << EOF | sudo tee /etc/systemd/system/traefikee-proxy.service
[Unit]
Description=Traefik Enterprise proxy
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
EnvironmentFile=-/opt/traefikee/proxies.env
Restart=on-abnormal
User=traefikee
Group=traefikee
ExecStart=/usr/local/bin/traefikee proxy --jointoken.value=\${PROXY_NODE_TOKEN} --discovery.static.peers=\${CONTROLLER_PEERS} --statedir=/opt/traefikee/data --log.filepath=/var/log/traefikee/traefikee.log --log.format=json --log.level=DEBUG
PrivateTmp=true
PrivateDevices=false
ProtectHome=true
ProtectSystem=full
ReadWritePaths=/opt/traefikee
PermissionsStartOnly=true
ExecStartPre=mkdir -p /opt/traefikee/data /var/log/traefikee
ExecStartPre=-chown -R traefikee.traefikee /opt/traefikee
ExecStartPre=-chown -R traefikee.traefikee /var/log/traefikee

LimitNOFILE=16384

; The following additional security directives only work with systemd v229 or later.
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl start traefikee-proxy
```

## Setup web server

>> On the Web server

```shell
echo 'DNSStubListener=no' | sudo tee -a /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
sudo resolvectl dns ens5 ${DNS1_IP} ${DNS2_IP}
sudo resolvectl domain ens5 ${DOMAIN}
sudo apt update
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo docker run -d -p 80:80 --name=whoami-dockervm traefik/whoami:latest -name whoami-dockervm
```
