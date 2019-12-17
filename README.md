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

### Join-Token
Um einen Node in das Cluster zu integrieren benötigen wir einen sogenannten *Join-Token* mit dem Befehl ``sudo kubeadm init [...]`` haben wir ein solches Token generiert dieses ist jedoch nur 24 Stunden gültig. 

Mit:

```shell
$ sudo kubeadm token create --print-join-command | tee $HOME/.kube/join-token
```

erzeugen wir ein neues Join-Token.

### Kube-Worker
Auf den Worker-Nodes müssen wir lediglich die zuvor gespeicherte Anweisung ausführen:


```shell
$ sudo kubeadm join ${kube-master-1IP}:6443 --token ${joinToken} --discovery-token-ca-cert-hash ${tokenCaCertHash}
```

Nun sollten wir mit dem folgenden Befehl, auf dem *kube-master* ausgeführt unseren worker entdecken.

```shell
$ kubectl get nodes
```

### Einen Node aus dem Cluster entfernen

Als erstes müssen wir sicherstellen, dass Pods umgeitet werden und der Master diesem Node nicht noch mehr Workload zuweist. Anschließend weisen wir den Master an den Worker zu entfernen.


```shell
$ kubectl drain ${nodeName} --ignore-daemonsets --delete-local-data
$ kubectl delete node ${nodeName}
```

Nun reinigen wir den Worker indem wir auf dem entsprechenden Node folgenden Befehl ausführen:

```shell
$ kubeadm reset
```

## Dashboard
Das Dashboard dient dazu einen Überblick über den Zustand des Clusters zu gewährleisten. Hier können wir unter Anderem erkennen welche Nodes sich im Cluster befinden, welche Deployments auf ihnen betrieben werden und wie deren Status ist. Ebenfalls zuerkennen ist, wo die entsprechenden Container ausgeführt werden. Außerdem können Secrets, Konfigurationen, Jobs und gemeinsamer genutzter Speicher eingesehen werden.
Bei dem Dashboard handelt es sich, wie bei allem (im Zusammenhang mit K8s), um ein Deployment.

### Installation

Auf dem Master legen wir nun die Cluster

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml
```

### Zugriff
Um auf das Dashboard über den Browser zugreifen zu können ist es nötig folgende Schritte zu vollziehen.
Als erstes falls
