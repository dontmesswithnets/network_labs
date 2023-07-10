# Minikube K8S cluster

### *Before all*

```
sudo apt update
sudo apt upgrade -y
```


### *Install Docker*

```
sudo apt install -y curl wget apt-transport-https
curl -fsSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo systemctl status docker
```

```
user@isangulovkuber:~$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-07-10 08:23:36 UTC; 46s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 79313 (dockerd)
      Tasks: 9
     Memory: 27.0M
     CGroup: /system.slice/docker.service
             └─79313 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Jul 10 08:23:36 isangulovkuber systemd[1]: Starting Docker Application Container Engine...
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.405220088Z" level=info msg="Starting up"
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.408719254Z" level=info msg="detected 127.0.0.53 nameserver, assuming systemd-resolved, so using resolv.conf: /run/s>
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.463193703Z" level=info msg="Loading containers: start."
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.643305480Z" level=info msg="Loading containers: done."
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.656399076Z" level=warning msg="WARNING: No swap limit support"
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.656430173Z" level=info msg="Docker daemon" commit=4ffc614 graphdriver=overlay2 version=24.0.4
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.656499541Z" level=info msg="Daemon has completed initialization"
Jul 10 08:23:36 isangulovkuber dockerd[79313]: time="2023-07-10T08:23:36.683330702Z" level=info msg="API listen on /run/docker.sock"
Jul 10 08:23:36 isangulovkuber systemd[1]: Started Docker Application Container Engine.
```

### *Install Minikube*

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### *Install Kubectl*

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### *Finally*

```
sudo usermod -aG docker $USER && newgrp docker
minikube start --driver=docker
```

```
user@isangulovkuber:~$ minikube start --driver=docker
* minikube v1.30.1 on Ubuntu 20.04
* Using the docker driver based on user configuration

X The requested memory allocation of 1983MiB does not leave room for system overhead (total system memory: 1983MiB). You may face stability issues.
* Suggestion: Start minikube with less memory allocated: 'minikube start --memory=1983mb'

* Using Docker driver with root privileges
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
* Downloading Kubernetes v1.26.3 preload ...
    > preloaded-images-k8s-v18-v1...:  397.02 MiB / 397.02 MiB  100.00% 8.22 Mi
    > gcr.io/k8s-minikube/kicbase...:  373.53 MiB / 373.53 MiB  100.00% 4.60 Mi
* Creating docker container (CPUs=2, Memory=1983MB) ...
* Preparing Kubernetes v1.26.3 on Docker 23.0.2 ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Verifying Kubernetes components...
* Enabled addons: storage-provisioner, default-storageclass
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### *As a result*

```
user@isangulovkuber:~$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```
user@isangulovkuber:~$ kubectl run kubia --image=luksa/kubia --port=8080
pod/kubia created
user@isangulovkuber:~$ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
kubia   1/1     Running   0          22m
```

