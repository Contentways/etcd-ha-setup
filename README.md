# Installieren eines Hochverfügbaren ETCD Clusters für K8S

### Vorwort
In diesem Howto wird gezeigt wie man ein Hochverfügbares Cluster für den ETCD Key-Value Store Daemon erstellt. ETCD ist ein wichtiger Bestandteil von Kubernetes, welcher genutzt wird um Zertifikate und andere Cluster Daten. Mehr zu dem Thema unter [[EN] Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/http:// "[EN] Operating etcd clusters for Kubernetes"). Ich werde hier nur auf die Basis Installation eingehen und nicht auf das Thema Sicherheit, Backup und Disaster Recovery etc. Ebenfalls werde ich auch nicht auf die Basis Installation eines Serversystems eingehen, dies sollte im Umgang mit Servern bekannt sein. Dazu findet man genug im Internet. Wie immer werde ich für die Anleitung auch nicht für evebtuellen Datenverlust und-/oder andere Schäden am System aufkommen.

### Voraussetzungen
- Hetzner Cloud Account (Diesen kann man sich kostenlos erstellen unter https://accounts.hetzner.com/, achtung es enstehen beim Anlegen von Ressourcen Kosten)
- Basiswissen im Umgang mit Linux und der CLI, insbesondere Debian (welches ich für diese Anleitung nutze.)
- Minimum 3 Server (am besten wären 5 )
- Cloud Network mit aktivem Subnet (in diesem Howto 192.168.1.0/24)
- Optional Key Based SSH Login um die Sichehreit zu steigern.

### Repositorys einrichten und Basis Software Installieren

Auf allen Servern wird zunächst das Repository für Docker bzw. Containerd.io und Kubernetes eingerichtet, diese Schritte sind auf allen Servern gleich. Zunächst werden wir unser System auf den aktuellsten Stand bringen mittels:

```shell
apt-get update
apt-get upgrade
```

Einrichten der Repositorys für Kubernetes und Docker bzw. Containerd.io
```shell
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release ntp
```

Hinzufügen der Docker Repository Keys
```shell
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
Hinzufügen des Docker APT Repositorys
```shell
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Hinzufügen der Kubernetes Repository Keys
```shell
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Hinzufügen des Kubernetes APT Repositorys
```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Repositorys updaten und Software installieren
```shell
sudo apt-get update
sudo apt-get -y install docker-ce docker-ce-cli containerd.io kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
### Anpassungen an der Sysctl Konfiguration

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
### Installieren des ETCD Clusters mittels Kubeadm
Kubeadm bringt Standardmäßig alles für die erstellung der Zertifikate "Out of the Box" mit, wir werden in dieser Anleitung alle relevanten Zertifikate auf dem Master Server erstllen und diese später mittels SCP auf die anderen Nodes verteilen.
Dazu werden wir uns erstmal einige Aliase auf der Shell anlegen für Hostname, IP Adresse etc. Zuerst müssen wir aber noch ein UNIT File für systemd anlegen, dies muss eine höhere Priorität haben als das was mit kubeadm ausgeliefert wird. Dies erledigen wir ebenfalls auf der Shell mit folgendem Befehl. Aber zuerst müssen wir einige Ordner und Dateien für Conatinerd anlegen, andernfalls kommt es zu fehlern

**(1) Anlegen der Ordner und Configfiles für Containerd**

```shell
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
systemctl restart containerd
```

**(2) Anlegen des Systemd Unit Files und neustarten des Systemd Daemons**

```shell
cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
# Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
# Replace the value of "--container-runtime-endpoint" for a different container runtime if needed.
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
Restart=always
``` 
```shell
systemctl daemon-reload
systemctl restart kubelet
```
**(3) Status des kubelet Daemons überprüfen mittels:**

```shell
systemctl status kubelet
```
**(4) Anlegen von Aliasen und Konfigurationsdateien für Kubeadm**

```shell
# Update HOST0, HOST1 and HOST2 with the IPs of your hosts
export HOST0=IP ADRESS
export HOST1=IP ADRESS
export HOST2=IP ADRESS

# Update NAME0, NAME1 and NAME2 with the hostnames of your hosts
export NAME0="HOSTNAME"
export NAME1="HOSTNAME"
export NAME2="HOSTNAME"

# Create temp directories to store files that will end up on other hosts.
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

HOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=(${NAME0} ${NAME1} ${NAME2})

for i in "${!HOSTS[@]}"; do
HOST=${HOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: InitConfiguration
nodeRegistration:
    name: ${NAME}
localAPIEndpoint:
    advertiseAddress: ${HOST}
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```

Bitte in den oberen Variablen HOST0-2 und NAME0-2 die hostnamen und der IP Adressen auf die richtigen Daten der eingesetzten Server ändern, andernfalls kommt es im späterem Verlauf zu Fehlern.

**(5) Erstellen der CertificateAuthority für das ETCD Cluster**

```shell
kubeadm init phase certs etcd-ca
```
Danach erhalten wir zwei Dateien
- `/etc/kubernetes/pki/etcd/ca.crt`
- `/etc/kubernetes/pki/etcd/ca.key`

**(6) Zertifikate für jedes ETCD Cluster Mitglied erstellen**

```shell
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# cleanup non-reusable certificates
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# No need to move the certs because they are for HOST0

# clean up certs that should not be copied off this host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

**(7) Kopieren der Zertifikate und Konfigurationsdateien auf die jeweiligen Server**
- HOST1
```shell
USER=debian
HOST=${HOST1}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
sudo -Es
chown -R root:root pki
mv pki /etc/kubernetes/
```

- HOST2
```shell
USER=debian
HOST=${HOST2}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
sudo -Es
chown -R root:root pki
mv pki /etc/kubernetes/
```

Bitte den "USER" anpassen mit dem man sich einloggen kann auf dem Remotesystem

**(8) Überischt der vorhandenen Ordner und Files**

**HOST0**
```shell
/tmp/${HOST0}
-- kubeadmcfg.yaml

/etc/kubernetes/pki
-- apiserver-etcd-client.crt
-- apiserver-etcd-client.key
-- etcd/
    -- ca.crt
    -- ca.key
    -- healthcheck-client.crt
    -- healthcheck-client.key
    -- peer.crt
    -- peer.key
    -- server.crt
    -- server.key
```

**HOST1**
```shell
$HOME
-- kubeadmcfg.yaml
---
/etc/kubernetes/pki
-- apiserver-etcd-client.crt
-- apiserver-etcd-client.key
-- etcd/
    -- ca.crt
    -- healthcheck-client.crt
    -- healthcheck-client.key
    -- peer.crt
    -- peer.key
    -- server.crt
    -- server.key
```

**HOST2**
```shell
$HOME
-- kubeadmcfg.yaml
---
/etc/kubernetes/pki
-- apiserver-etcd-client.crt
-- apiserver-etcd-client.key
-- etcd/
    -- ca.crt
    -- healthcheck-client.crt
    -- healthcheck-client.key
    -- peer.crt
    -- peer.key
    -- server.crt
    -- server.key
```
**(9) Pod Manifeste und Konfigurationen erstellen**
Wir haben nun die Zertifikate und Konfigurationen erstellet und auf die jeweilligen Server verteilt, nun wird es Zeit die Manifeste für Kubeadm zu erstellen und unsere Pods mit dem ETCD Daemon zu starten.

```shell
root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
root@HOST1 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
root@HOST2 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
```

**(10) Docker Status und ETCD überprüfen**
```shell
docker run --rm -it \
--net host \
-v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--endpoints https://${HOST0}:2379 endpoint health --cluster
```
- Der ${ETCD_TAG} muss auf die Version gesetzt werden welche kubeadm nutzt, dies können wir ganz einfach mittels `kubeadm config images list --kubernetes-version ${K8S_VERSION}` herrausfinden. Bitte die Variable ${K8S_VERSION} durch die aktuell installierte kubeadm version ersetzen.
- Zum zeitpunkt der Anleitung liegt kubeadm in der Version 1.23.5 vor und das ETCD Image welches genutzt wird etcd:3.5.1-0

Wenn alles richtig funktioniert solltet Ihr folgende Ausgabe erhalten.
```shell
https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
```

### Schlusswort
Man hat nun ein hochverfügbares ETCD Cluster für Kubernetes, sicherlich gibt es noch viele andere Wege dieses zu realisieren und z.B. einen Loadbalancer vorzuschalten etc. Ich habe mich mit dieser Anleitung an die Offizielle Dokumentation des Kubernetes Projekt gehalten bis auf wenige minimale Anpassungen. In der nächsten Anleitung werde ich dann drauf eingehen, wie man ein hochverfügbares Kubernetes Controlplane Setup aufsetzt in Verbindung mit unserem hier erstelltem etcd Cluster.
Im laufe der Zeit werde ich diese Anleitung auch noch ins englischsprachige übersetzen.

### Referenzen
- Debian -- https://www.debian.org/
- Docker -- https://docker.com/
- Kubernetes -- https://kubernetes.io/
- Kubernetes Doku -- https://kubernetes.io/docs/home/
- Hetzner Online GmbH -- https://www.hetzner.com/

### Lizenz
**MIT License**
