# K8s-Schulung
## Cluster aufsetzen
### User anlegen

```shell
$ sudo su -
$ adduser k8s-admin
```

Nun wird nach dem Passwort für den neuen User verlangt, bitte eingeben und bestätigen. Die restlichen Eingaben können jeweils mit <kbd>&#9166;</kbd>. Im Anschluß fügen wir diesen Nutzer der *sudo-gruppe* hinzu.

```shell
$ usermod -aG sudo k8s-admin
```

Nun können wir uns als **k8s-admin** anmelden und den Erfolg testen.

```shell
$ su - k8s-admin
$ sudo ls -la /root
```
### Installieren von Docker
Folgende Befehle installieren *Docker* und konfigurieren es mit dem System zu starten.

```shell
$ sudo apt install -y docker.io
$ sudo systemctl enable docker
$ echo "{\"insecure-registries\":[\"${urlDerDockerRegistry}\"]}" | sudo  tee /etc/docker/daemon.json
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
$ sudo reboot
```
### Auf jedem Node ausführen
Jetzt kümmern wir uns um Kubernetes(K8s).

```shell
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
$ sudo apt install -y kubeadm
```

Nodes die sich innerhalb eine K8s-Clusters befinden dürfen keinen *Swap-Memory* nutzen, daher deaktivieren wir dieses Feature:


```shell
$ sudo swapoff -a
```

Sollte die Zeitzone nicht korrekt konfiguriert sein so kann es zu Problemen bei der Validierung von TLS-Zertifikaten kommen. Um dies zu vermeiden verwenden wir folgenden Befehl:

```shell
$ sudo timedatectl set-timezone Europe/Berlin
```

Setzen des Hostnamen:

```shell
$ sudo hostnamectl set-hostname kube-[master/worker]-${nummer}
$ sudo reboot
```

#### Troubleshooting
Solltest du einen Fehler wie z.B.: "CGROUPS_MEMORY: missing" in einem der späteren Schritte erhalten so müssen noch einige Feature aktiviert werden editiere dafür bitte die Datei */boot/firmware/cmdline.txt* und fügen folgendes and den Anfang der Datei an: ** cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 **
Nach einem Neustart sollte die vorgenommene Änderung greifen.

### Kube-Master
Nun können wir den Kubernetes-Master initializieren:

```shell
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Die Ausführung des obigen Befehls kann einige Zeit dauern, im Anschluß werden wir den *Join-Token* in einer Datei speichern und den Master-Node so konfigurieren, dass wir unser Kubernetes-Cluster nutzen können.

```shell
$ mkdir -p $HOME/.kube
```

Kopiere die letzten zwei Zeilen der Ausgabe des *kubeadm*-Befehls und führe folgendes aus:

```shell
$ echo "${kopierteAusgabe}" > $HOME/.kube/join-token
```

```shell
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Für die Kommunikation unserer Pods wird ein *Overlay-Netz* genutzt:

```shell
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```


### Kube-Worker
Auf den Worker-Nodes müssen wir lediglich die zuvor gespeicherte Anweisung ausführen:


```shell
$ sudo kubeadm join ${kube-master-1IP}:6443 --token ${joinToken} --discovery-token-ca-cert-hash ${tokenCaCertHash}
```

Nun sollten wir mit dem folgenden Befehl, auf dem *kube-master* ausgeführt unseren worker entdecken.

```shell
$ kubectl get nodes
```
