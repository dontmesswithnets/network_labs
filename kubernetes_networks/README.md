# *Домашнее задание - Kubernetes-Intro*

## Minikube K8S cluster

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

### *Проверка minikube*

_После команды minikube start можем войти в сам конетйнер с помощью команды ***minikube ssh*** и посмотреть состояние контейнеров, набрав команду_ ***docker ps***

```
user@isangulovkuber:~$ minikube ssh
docker@minikube:~$ docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS       PORTS     NAMES
c7542527c892   luksa/kubia                 "node app.js"            3 hours ago   Up 3 hours             k8s_kubia_kubia_default_5e5fb59f-2008-4194-a35c-58534f984c0b_0
0980067dde83   registry.k8s.io/pause:3.9   "/pause"                 3 hours ago   Up 3 hours             k8s_POD_kubia_default_5e5fb59f-2008-4194-a35c-58534f984c0b_0
604e7b8bf0ac   6e38f40d628d                "/storage-provisioner"   5 hours ago   Up 5 hours             k8s_storage-provisioner_storage-provisioner_kube-system_dad5f0a2-3adc-49cf-a13b-869557d64d89_1
27e7be551d76   5185b96f0bec                "/coredns -conf /etc…"   5 hours ago   Up 5 hours             k8s_coredns_coredns-787d4945fb-xk266_kube-system_45d492ee-b831-4b6f-8d38-f444cfdb3bd8_0
04b2dba8e0e2   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_coredns-787d4945fb-xk266_kube-system_45d492ee-b831-4b6f-8d38-f444cfdb3bd8_0
861eda1e117b   92ed2bec97a6                "/usr/local/bin/kube…"   5 hours ago   Up 5 hours             k8s_kube-proxy_kube-proxy-xz2f2_kube-system_e9b17578-fee1-48b0-bf59-aa170ef604d2_0
1b9d8ce9ad08   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_storage-provisioner_kube-system_dad5f0a2-3adc-49cf-a13b-869557d64d89_0
b58fec3c5df1   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_kube-proxy-xz2f2_kube-system_e9b17578-fee1-48b0-bf59-aa170ef604d2_0
7bf26e44402d   5a7904736932                "kube-scheduler --au…"   5 hours ago   Up 5 hours             k8s_kube-scheduler_kube-scheduler-minikube_kube-system_0818f4b1a57de9c3f9c82667e7fcc870_0
bcfc0e30201b   ce8c2293ef09                "kube-controller-man…"   5 hours ago   Up 5 hours             k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_466b9e73e627277a8c24637c2fa6442d_0
46caeb2032df   1d9b3cbae03c                "kube-apiserver --ad…"   5 hours ago   Up 5 hours             k8s_kube-apiserver_kube-apiserver-minikube_kube-system_cdcbce216c62c4407ac9a51ac013e7d7_0
622e54c46ad2   fce326961ae2                "etcd --advertise-cl…"   5 hours ago   Up 5 hours             k8s_etcd_etcd-minikube_kube-system_a121e106627e5c6efa9ba48006cc43bf_0
df2d50abc93d   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_kube-scheduler-minikube_kube-system_0818f4b1a57de9c3f9c82667e7fcc870_0
afaae951421c   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_kube-controller-manager-minikube_kube-system_466b9e73e627277a8c24637c2fa6442d_0
045399fc8e38   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_kube-apiserver-minikube_kube-system_cdcbce216c62c4407ac9a51ac013e7d7_0
36f861326e9c   registry.k8s.io/pause:3.9   "/pause"                 5 hours ago   Up 5 hours             k8s_POD_etcd-minikube_kube-system_a121e106627e5c6efa9ba48006cc43bf_0
```

*Удалим все контейнеры из под minikube с помощью команды* ***docker rm -f $(docker ps -a -q)***

*Затем выйдем из minikube и проверим, что ничего не сломалось*

```
user@isangulovkuber:~$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS      AGE
coredns-787d4945fb-xk266           1/1     Running   1             5h23m
etcd-minikube                      1/1     Running   1             5h23m
kube-apiserver-minikube            1/1     Running   1             5h23m
kube-controller-manager-minikube   1/1     Running   1             5h23m
kube-proxy-xz2f2                   1/1     Running   1             5h23m
kube-scheduler-minikube            1/1     Running   1             5h23m
storage-provisioner                1/1     Running   3 (78s ago)   5h23m
```

*Проведем еще одну проверу, на этот раз с помощью команды ***kubectl delete pod --all -n kube-system***

```
user@isangulovkuber:~$ kubectl delete pod --all -n kube-system
pod "coredns-787d4945fb-xk266" deleted
pod "etcd-minikube" deleted
pod "kube-apiserver-minikube" deleted
pod "kube-controller-manager-minikube" deleted
pod "kube-proxy-xz2f2" deleted
pod "kube-scheduler-minikube" deleted
pod "storage-provisioner" deleted
user@isangulovkuber:~$
user@isangulovkuber:~$
user@isangulovkuber:~$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-787d4945fb-p8hsf           1/1     Running   0          17s
etcd-minikube                      1/1     Running   1          17s
kube-apiserver-minikube            1/1     Running   1          17s
kube-controller-manager-minikube   1/1     Running   1          17s
kube-proxy-dq4ct                   1/1     Running   0          15s
kube-scheduler-minikube            1/1     Running   1          16s
```

```
user@isangulovkuber:~$ kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
```

*Как видим, кластер пересобрался, self-healing отработал*

### *Dockerfile*

_Для дальнейшего выполнения ДЗ необходимо написать Dockerfile с web сервером на борту, работающим на 8000 tcp порте и отдающим содержимое директории **/app**_

```
FROM nginx
RUN mkdir -p /app && chown -R nginx:nginx /app
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8000
CMD ["nginx", "-g", "daemon off;"]
```

_Радом положим **nginx.conf** файл_

```
server {
    listen 8000;
    server_name localhost;

    location / {
        root /app;
        index index.html;
    }
}
```

_Соберем образ из Dockerfile_

```
user@isangulovkuber:~$ docker build -t kubernetes-intro .
[+] Building 10.8s (9/9) FINISHED                                                                                                                                             docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                    0.0s
 => => transferring dockerfile: 191B                                                                                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                         0.0s
 => [internal] load metadata for docker.io/library/nginx:latest                                                                                                                         2.0s
 => [auth] library/nginx:pull token for registry-1.docker.io                                                                                                                            0.0s
 => [1/3] FROM docker.io/library/nginx@sha256:08bc36ad52474e528cc1ea3426b5e3f4bad8a130318e3140d6cfe29c8892c7ef                                                                          8.3s
 => => resolve docker.io/library/nginx@sha256:08bc36ad52474e528cc1ea3426b5e3f4bad8a130318e3140d6cfe29c8892c7ef                                                                          0.0s
 => => sha256:cf707e2339551222cafe3bf835fddfb859f26bf59058b3487de2b7659309b6b7 625B / 625B                                                                                              0.2s
 => => sha256:08bc36ad52474e528cc1ea3426b5e3f4bad8a130318e3140d6cfe29c8892c7ef 1.86kB / 1.86kB                                                                                          0.0s
 => => sha256:76579e9ed380849b4d22696f292770963b5ea917a45ef77336a1a0a782b410b5 41.46MB / 41.46MB                                                                                        6.1s
 => => sha256:faef57eae888cbe4a5613eca6741b5e48d768b83f6088858aee9a5a2834f8151 29.12MB / 29.12MB                                                                                        5.5s
 => => sha256:1bb5c4b86cb7c1e9f0209611dc2135d8a2c1c3a6436163970c99193787d067ea 1.78kB / 1.78kB                                                                                          0.0s
 => => sha256:021283c8eb95be02b23db0de7f609d603553c6714785e7a673c6594a624ffbda 8.15kB / 8.15kB                                                                                          0.0s
 => => sha256:91bb7937700d7d3496cf43cb0012e5f818064fecb766bd01041db23c127ab219 959B / 959B                                                                                              0.3s
 => => sha256:4b962717ba558b7dfabe88c40e20ac86844b132015b66002deac49010cc96be1 367B / 367B                                                                                              0.6s
 => => sha256:f46d7b05649a846d7e24418b6ecea3b1efbdac88d361631e849e9c41917ba776 1.21kB / 1.21kB                                                                                          0.8s
 => => sha256:103501419a0aecf94398ffcc7404f22931d9b89bbb6021391c2cd4a286f37ca9 1.41kB / 1.41kB                                                                                          1.0s
 => => extracting sha256:faef57eae888cbe4a5613eca6741b5e48d768b83f6088858aee9a5a2834f8151                                                                                               1.3s
 => => extracting sha256:76579e9ed380849b4d22696f292770963b5ea917a45ef77336a1a0a782b410b5                                                                                               1.2s
 => => extracting sha256:cf707e2339551222cafe3bf835fddfb859f26bf59058b3487de2b7659309b6b7                                                                                               0.0s
 => => extracting sha256:91bb7937700d7d3496cf43cb0012e5f818064fecb766bd01041db23c127ab219                                                                                               0.0s
 => => extracting sha256:4b962717ba558b7dfabe88c40e20ac86844b132015b66002deac49010cc96be1                                                                                               0.0s
 => => extracting sha256:f46d7b05649a846d7e24418b6ecea3b1efbdac88d361631e849e9c41917ba776                                                                                               0.0s
 => => extracting sha256:103501419a0aecf94398ffcc7404f22931d9b89bbb6021391c2cd4a286f37ca9                                                                                               0.0s
 => [internal] load build context                                                                                                                                                       0.0s
 => => transferring context: 31B                                                                                                                                                        0.0s
 => [2/3] RUN mkdir -p /app && chown -R nginx:nginx /app                                                                                                                                0.3s
 => [3/3] COPY nginx.conf /etc/nginx/conf.d/default.conf                                                                                                                                0.0s
 => exporting to image                                                                                                                                                                  0.1s
 => => exporting layers                                                                                                                                                                 0.0s
 => => writing image sha256:dd3eb13f87bf055b9baf9645764746bc495679db39a3640db3f2da41b8db3e51                                                                                            0.0s
 => => naming to docker.io/library/kubernetes-intro                                                                                                                                     0.0s
```

_После этого нужно задать **tag** и запушить готовый образ в Docker Hub_

```
user@isangulovkuber:~$ docker tag kubernetes-intro ildarvildanovich/kubernetes-intro:1
user@isangulovkuber:~$ docker push ildarvildanovich/kubernetes-intro:1
The push refers to repository [docker.io/ildarvildanovich/kubernetes-intro]
b96d141f696c: Pushed
9e2394ffabab: Pushed
3c9d04c9ebd5: Mounted from library/nginx
434c6a715c30: Mounted from library/nginx
9fdfd12bc85b: Mounted from library/nginx
f36897eea34d: Mounted from library/nginx
1998c5cd2230: Mounted from library/nginx
b821d93f6666: Mounted from library/nginx
24839d45ca45: Mounted from library/nginx
1: digest: sha256:3e6ec89d02b4d29e9686caa87ad7d1620595c721f26354287cc5b473b964bba5 size: 2192
```

### _Manifest pod_

_Сейчас нам нужно написать манифест, в котором опишем создание нужного нам пода и назовем его **web-pod.yaml**_

```
apiVersion: v1
kind: Pod
metadata:
  name: my-web
  labels:
    app: web
spec:
  containers:
    - name: my-container
      image: ildarvildanovich/kubernetes-intro:1
```

_Теперь применим его и проверим состояние_

```
user@isangulovkuber:~$ kubectl apply -f web-pod.yaml
pod/my-web created
user@isangulovkuber:~$ kubectl get pods my-web
NAME     READY   STATUS    RESTARTS   AGE
my-web   1/1     Running   0          17m
```

### _Init containers_

_Добавляем в наш под Init контейнер, который должен генерировать страницу index.html, а также добавим Volumes. В итоге наш манифест приобретает следующий вид:_

```
apiVersion: v1
kind: Pod
metadata:
  name: my-web
  labels:
    app: web
spec:
  volumes:
    - name: app
      emptyDir: {}
  containers:
    - name: my-container
      image: ildarvildanovich/kubernetes-intro:1
      volumeMounts:
        - name: app
          mountPath: /app
  initContainers:
    - name: init-container-1
      image: busybox:latest
      volumeMounts:
        - name: app
          mountPath: /app
      command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh']
```

_После этого нужно удалить прежний под командой ***kubectl delete pod my-web*** и создать новый на основе обновленного yaml файла командой ***kubectl apply -f web-pod.yaml***_

_Теперь, если мы введем команду ***kubectl port-forward --address 0.0.0.0 pod/my-web 8000:8000*** для временной трансляции портов и пройдем по IP адресу хоста и 8000 TCP порту мы увидим наш index.html_

## _Hipster Shop_

_Для дальнейшей работы необходимо запустить микросервис frontend внутри нашего пода_

_Для начала я форкну (да, я знаю, что это не обязательно) к себе этот репозиторий и склонирую его к себе на ВМ. После этого я строю образ и пушу его к себе в Docker Hub_

```
user@isangulovkuber:~$ git clone https://github.com/dontmesswithnets/microservices-demo.git
Cloning into 'microservices-demo'...
remote: Enumerating objects: 11976, done.
remote: Counting objects: 100% (4402/4402), done.
remote: Compressing objects: 100% (411/411), done.
remote: Total 11976 (delta 4237), reused 4010 (delta 3990), pack-reused 7574
Receiving objects: 100% (11976/11976), 31.69 MiB | 6.03 MiB/s, done.
Resolving deltas: 100% (9053/9053), done.
```

```
user@isangulovkuber:~$ cd microservices-demo/src/frontend

user@isangulovkuber:~/microservices-demo/src/frontend$ docker build -t microservices-demo .
[+] Building 98.3s (22/22) FINISHED                                                                                                                                                                                                                                   docker:default
 => [internal] load .dockerignore                                                                                                                                                                                                                                               0.0s
 => => transferring context: 48B                                                                                                                                                                                                                                                0.0s
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                                            0.0s
 => => transferring dockerfile: 1.59kB                                                                                                                                                                                                                                          0.0s
 => [internal] load metadata for docker.io/library/golang:1.20.6-alpine@sha256:5bb5c69803a486fd89d5284746869880fc2af89db8655a79f4a459994bba141b                                                                                                                                 1.6s
 => [internal] load metadata for docker.io/library/alpine:3.18.2@sha256:82d1e9d7ed48a7523bdebc18cf6290bdb97b82302a8a9c27d4fe885949ea94d1                                                                                                                                        1.9s
 => [auth] library/alpine:pull token for registry-1.docker.io                                                                                                                                                                                                                   0.0s
 => [auth] library/golang:pull token for registry-1.docker.io                                                                                                                                                                                                                   0.0s
 => [builder 1/8] FROM docker.io/library/golang:1.20.6-alpine@sha256:5bb5c69803a486fd89d5284746869880fc2af89db8655a79f4a459994bba141b                                                                                                                                          15.2s
 => => resolve docker.io/library/golang:1.20.6-alpine@sha256:5bb5c69803a486fd89d5284746869880fc2af89db8655a79f4a459994bba141b                                                                                                                                                   0.0s
 => => sha256:31e352740f534f9ad170f75378a84fe453d6156e40700b882d737a8f4a6988a3 3.40MB / 3.40MB                                                                                                                                                                                  0.4s
 => => sha256:7f9bcf943fa5571df036dca6da19434d38edf546ef8bb04ddbc803634cc9a3b8 284.71kB / 284.71kB                                                                                                                                                                              0.4s
 => => sha256:08174ed119b8a06a34eb4623c0d85ef6886ebbc99a5a224ea1dc304f4d5bb8c9 100.94MB / 100.94MB                                                                                                                                                                              9.7s
 => => sha256:5bb5c69803a486fd89d5284746869880fc2af89db8655a79f4a459994bba141b 1.16kB / 1.16kB                                                                                                                                                                                  0.0s
 => => sha256:edad107142b084141ba2a7f18022c8c6c0b1763cc8bd751d99a4a26799cd0d75 5.11kB / 5.11kB                                                                                                                                                                                  0.0s
 => => extracting sha256:31e352740f534f9ad170f75378a84fe453d6156e40700b882d737a8f4a6988a3                                                                                                                                                                                       0.1s
 => => sha256:2340748fe533e40df8b2a7786972c349568e39e8b3eff95d0fd1a583bf67e32a 0B / 157B                                                                                                                                                                                       96.3s
 => => extracting sha256:7f9bcf943fa5571df036dca6da19434d38edf546ef8bb04ddbc803634cc9a3b8                                                                                                                                                                                       0.0s
 => => extracting sha256:08174ed119b8a06a34eb4623c0d85ef6886ebbc99a5a224ea1dc304f4d5bb8c9                                                                                                                                                                                       5.2s
 => => extracting sha256:2340748fe533e40df8b2a7786972c349568e39e8b3eff95d0fd1a583bf67e32a                                                                                                                                                                                       0.0s
 => [internal] load build context                                                                                                                                                                                                                                               0.1s
 => => transferring context: 4.40MB                                                                                                                                                                                                                                             0.1s
 => [release 1/6] FROM docker.io/library/alpine:3.18.2@sha256:82d1e9d7ed48a7523bdebc18cf6290bdb97b82302a8a9c27d4fe885949ea94d1                                                                                                                                                  0.6s
 => => resolve docker.io/library/alpine:3.18.2@sha256:82d1e9d7ed48a7523bdebc18cf6290bdb97b82302a8a9c27d4fe885949ea94d1                                                                                                                                                          0.0s
 => => sha256:25fad2a32ad1f6f510e528448ae1ec69a28ef81916a004d3629874104f8a7f70 528B / 528B                                                                                                                                                                                      0.0s
 => => sha256:c1aabb73d2339c5ebaa3681de2e9d9c18d57485045a4e311d9f8004bec208d67 1.47kB / 1.47kB                                                                                                                                                                                  0.0s
 => => sha256:31e352740f534f9ad170f75378a84fe453d6156e40700b882d737a8f4a6988a3 3.40MB / 3.40MB                                                                                                                                                                                  0.3s
 => => sha256:82d1e9d7ed48a7523bdebc18cf6290bdb97b82302a8a9c27d4fe885949ea94d1 1.64kB / 1.64kB                                                                                                                                                                                  0.0s
 => [release 2/6] RUN apk add --no-cache ca-certificates     busybox-extras net-tools bind-tools                                                                                                                                                                                9.4s
 => [release 3/6] WORKDIR /src                                                                                                                                                                                                                                                  0.1s
 => [builder 2/8] RUN apk add --no-cache ca-certificates git                                                                                                                                                                                                                    2.1s
 => [builder 3/8] RUN apk add build-base                                                                                                                                                                                                                                        8.8s
 => [builder 4/8] WORKDIR /src                                                                                                                                                                                                                                                  0.0s
 => [builder 5/8] COPY go.mod go.sum ./                                                                                                                                                                                                                                         0.1s
 => [builder 6/8] RUN go mod download                                                                                                                                                                                                                                          29.7s
 => [builder 7/8] COPY . .                                                                                                                                                                                                                                                      0.2s
 => [builder 8/8] RUN go build -gcflags="${SKAFFOLD_GO_GCFLAGS}" -o /go/bin/frontend .                                                                                                                                                                                         39.5s
 => [release 4/6] COPY --from=builder /go/bin/frontend /src/server                                                                                                                                                                                                              0.1s
 => [release 5/6] COPY ./templates ./templates                                                                                                                                                                                                                                  0.0s
 => [release 6/6] COPY ./static ./static                                                                                                                                                                                                                                        0.1s
 => exporting to image                                                                                                                                                                                                                                                          0.3s
 => => exporting layers                                                                                                                                                                                                                                                         0.3s
 => => writing image sha256:17a8c9aa44603b3a19cacf1180181aab04268bdb880722ae95a8adf3209170f2                                                                                                                                                                                    0.0s
 => => naming to docker.io/library/microservices-demo                       
```

```
user@isangulovkuber:~/microservices-demo/src/frontend$ docker tag microservices-demo ildarvildanovich/microservices-demo:1
user@isangulovkuber:~/microservices-demo/src/frontend$ docker push ildarvildanovich/microservices-demo:1
The push refers to repository [docker.io/ildarvildanovich/microservices-demo]
d2b1247efbec: Pushed
0d984545a0df: Pushed
2f6d6bd60ec0: Pushed
f06ca7d63808: Pushed
6ecc59c5c6dd: Pushed
78a822fe2a2d: Mounted from library/golang
1: digest: sha256:167ada6ad5ecdc95618c21106b83feea05d74a23355c14a87ff15021fcf755d8 size: 1577
```

### _Запуск пода через ***kubectl run***_

```
user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl run frontend --image ildarvildanovich/microservices-demo:1 --restart=Never --dry-run=client -o yaml > frontend-pod.yaml

user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl apply -f frontend-pod.yaml
pod/frontend created

user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl get pods
NAME       READY   STATUS   RESTARTS   AGE
frontend   0/1     Error    0          26s
```

_Видим, что под в статусе ***Error***_

_Для того, чтобы выяснить, почему так получилось, можно воспользоваться логами_

```
user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl logs frontend
{"message":"Tracing disabled.","severity":"info","timestamp":"2023-07-12T11:42:47.804154668Z"}
{"message":"Profiling disabled.","severity":"info","timestamp":"2023-07-12T11:42:47.804220576Z"}
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set

goroutine 1 [running]:
main.mustMapEnv(0xc000160000, {0xc07754, 0x1c})
        /src/main.go:208 +0xb9
main.main()
        /src/main.go:124 +0x5be
```

_Видим ошибку вида ***panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set***_

_Лезем в логи_

```
user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl logs frontend
{"message":"Tracing disabled.","severity":"info","timestamp":"2023-07-12T11:42:47.804154668Z"}
{"message":"Profiling disabled.","severity":"info","timestamp":"2023-07-12T11:42:47.804220576Z"}
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set

goroutine 1 [running]:
main.mustMapEnv(0xc000160000, {0xc07754, 0x1c})
        /src/main.go:208 +0xb9
main.main()
        /src/main.go:124 +0x5be
```

_Безуспешно попытавшись понять причину, смотрю подсказу в виде готового манифеста и вижу недостающие envirinment переменные_

_Добавляю все перечисленные путем редактирования манифеста через ***vim***_

_Удаляю неправильный под и запускаю заново, уже с исправленного манифеста_

```
user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl delete pods frontend
pod "frontend" deleted

user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl apply -f frontend-pod.yaml
pod/frontend created

user@isangulovkuber:~/microservices-demo/src/frontend$ kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
frontend   1/1     Running   0          7s
```

_Таким образом, причина нахождения пода в статусе ***Error*** была в недостающих переменных Environment_