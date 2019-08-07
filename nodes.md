Workers:
==========

Składowe:
=============

Non-Kubernetes:

* docker
* flanneld

Kubernetes:

* kublelet
* proxy

Krok 1: Przygotowanie
=================

* Zaktualizowany grub
* Reboot

```
echo '10.1.254.80 kube-master.anubisworks.local kube-master' >> /etc/hosts
echo '10.1.254.81 kube-node1.anubisworks.local kube-node1' >> /etc/hosts
echo '10.1.254.82 kube-node2.anubisworks.local kube-node2' >> /etc/hosts
```

```apt-get -fqqy install apt-transport-https ca-certificates curl```

Optymalizacja binarium - wypakowanie

```
curl \
  --progress-bar \
  --location \
  'https://github.com/kubernetes/kubernetes/releases/download/v1.3.4/kubernetes.tar.gz' \
    | tar --to-stdout -zxf - kubernetes/server/kubernetes-server-linux-amd64.tar.gz \
      | tar --to-stdout -zxf - kubernetes/server/bin/hyperkube > /usr/bin/hyperkube 
chmod a+x /usr/bin/hyperkube
cd /usr/bin/
/usr/bin/hyperkube --make-symlinks
```

```
mkdir -p /var/lib/k8s
groupadd kube
useradd kube -g kube -d /var/lib/k8s/ -s /bin/false
chown kube:kube /var/lib/k8s
```

Krok 2: Instalacja Docker
======================

```
apt-key adv \
  --keyserver hkp://p80.pool.sks-keyservers.net:80 \
  --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo 'deb https://apt.dockerproject.org/repo debian-jessie main' \
  > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get -fqqy install docker-engine
```

Krok 3: Instalacja Flanneld
========================

Pobranie + instalacja

```
cd /usr/local/src
curl \
  --silent \
  --location \
  'https://github.com/coreos/flannel/releases/download/v0.5.5/flannel-0.5.5-linux-amd64.tar.gz' \
    | tar -zvxf-
cd flannel-0.5.5
cp flanneld /usr/bin
mkdir -p /var/lib/k8s/flannel/networks
```

Konfiguracja

```
cat << EOF > /lib/systemd/system/flanneld.service
[Unit]
Description=Network fabric for containers
Documentation=https://github.com/coreos/flannel
After=etcd.service

[Service]
Type=notify
Restart=always
RestartSec=5
ExecStart=/usr/bin/flanneld \\
  -etcd-endpoints=http://10.1.254.80:4001 \\
  -logtostderr=true \\
  -subnet-dir=/var/lib/k8s/flannel/networks \\
  -subnet-file=/var/lib/k8s/flannel/subnet.env
[Install]
WantedBy=multi-user.target
EOF
```

Start

```
systemctl daemon-reload
systemctl enable flanneld
service flanneld start
```

Krok 4: Rekonfiguracja dockera do użycia FlannelD
==========================================

Rekonfiguracja

```
cat << EOF > /lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket flanneld.service
Requires=docker.socket

[Service]
Type=notify
EnvironmentFile=-/var/lib/k8s/flannel/subnet.env
ExecStart=/usr/bin/dockerd -H fd:// --bip=\${FLANNEL_SUBNET} --mtu=\${FLANNEL_MTU}
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
MountFlags=shared

[Install]
WantedBy=multi-user.target
EOF
```

Dostęp dla Kube

```gpasswd -a kube docker```

Reload
```
systemctl daemon-reload
service docker restart
```

Krok 5: Kubernetes Proxy
========================

Konfiguracja
```
cat << EOF > /lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=flanneld.service

[Service]
User=root
ExecStart=/usr/bin/proxy \\
  --master=http://10.1.254.80:8080 \\
  --logtostderr=true
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

# Włączenie

```
systemctl daemon-reload
systemctl enable kube-proxy
service kube-proxy start
```

Krok 6 Kubernetes Kubelet
=========================

Konfiguracja

```
cat << EOF > /lib/systemd/system/kube-kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=flanneld.service

[Service]
User=root
ExecStart=/usr/bin/kubelet \\
  --cert-dir=/var/lib/k8s/kubernetes/ \\
  --chaos-chance=0.0 \\
  --container-runtime=docker \\
  --address=0.0.0.0 \\
  --cpu-cfs-quota=false \\
  --api-servers=10.1.254.80:8080 \\
  --cluster-dns=8.8.8.8 \\
  --port=10250 \\
  --logtostderr=true
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

Włączenie

```
systemctl daemon-reload
systemctl enable kube-kubelet
service kube-kubelet start
```