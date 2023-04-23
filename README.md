# devops-DZ12.4-K8S-net1

# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 1»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к приложению, установленному в предыдущем ДЗ и состоящему из двух контейнеров, по разным портам в разные контейнеры как внутри кластера, так и снаружи.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым Git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Описание Service.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера

1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт.
2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 — multitool 8080.
3. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложения из п.1 по разным портам в разные контейнеры.
4. Продемонстрировать доступ с помощью `curl` по доменному имени сервиса.
5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

------

### Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера

1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort.
2. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.
3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2.

------

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------

# Ответ

Проверим установку `MicroK8S`

```bash
root@vkvm:/home/vk/12.3# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
vkvm   Ready    <none>   16d   v1.26.3
root@vkvm:/home/vk/12.3# kubectl getpod -A
error: unknown command "getpod" for "kubectl"
root@vkvm:/home/vk/12.3# kubectl get pod -A
NAMESPACE     NAME                                        READY   STATUS    RESTARTS       AGE
kube-system   calico-node-tl7rj                           1/1     Running   5 (4m5s ago)   16d
kube-system   dashboard-metrics-scraper-7bc864c59-7dspr   1/1     Running   5 (4m5s ago)   16d
kube-system   kubernetes-dashboard-dc96f9fc-m949s         1/1     Running   5 (4m5s ago)   16d
kube-system   coredns-6f5f9b5d74-c6l72                    1/1     Running   2 (4m5s ago)   4d13h
kube-system   calico-kube-controllers-79568db7f8-jk9ck    1/1     Running   5 (4m5s ago)   16d
kube-system   metrics-server-6f754f88d-qlllr              1/1     Running   5 (4m5s ago)   16d
```

## Задание 1

### 1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт

Создадим `deploy1.yml` с развёртыванием контейнеров `nginx` и `multitool` , кол-во  реплик = 3.

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy1
  name: deploy1
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: deploy1
  template:
    metadata:
      labels:
        app: deploy1
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: multitool
          image: wbitt/network-multitool
          ports:
            - name: http-8080
              containerPort: 8080
              protocol: TCP
          env:
            - name: HTTP_PORT
              value: "8080"
            - name: HTTPS_PORT
              value: "11443"
```

[Ссылка на deploy1.yml](deploy1.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk/12.4# kubectl create -f deploy1.yml
deployment.apps/deploy1 created
```

Проверим состояние подов

```bash
root@vkvm:/home/vk/12.4# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
deploy1-67c5999d57-454ft   2/2     Running   0          25s
deploy1-67c5999d57-rrpbn   2/2     Running   0          25s
deploy1-67c5999d57-t7zcv   2/2     Running   0          25s
```

### 2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 — multitool 8080

Создадим файл `service1.yml` с развёртыванием сервиса

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service1
spec:
  selector:
    app: deploy1
  ports:
    - name: nginx-http
      port: 9001
      protocol: TCP
      targetPort: 80
    - name: multitool-http
      port: 9002
      protocol: TCP
      targetPort: 8080
```

[Ссылка на service1.yml](service1.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk/12.4# kubectl create -f service1.yml
service/service1 created
```

Проверим состояние сервиса

```bash
root@vkvm:/home/vk/12.4# kubectl get service
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP             16d
service1     ClusterIP   10.152.183.148   <none>        9001/TCP,9002/TCP   21s
```

### 3. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложения из п.1 по разным портам в разные контейнеры

Создадим `multitool-pod.yml` для пода с `multitool`

```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: multitool-pod
  labels:
    app: multitool-pod
spec:
  containers:
    - name: multitool
      image: wbitt/network-multitool
      ports:
        - name: http-1080
          containerPort: 1080
          protocol: TCP
      env:
        - name: HTTP_PORT
          value: "1080"
        - name: HTTPS_PORT
          value: "10443"
```

[Ссылка на multitool-pod.yml](multitool-pod.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk/12.4# kubectl create -f multitool-pod.yml
pod/multitool-pod created
```

Проверим состояние подов

```bash
root@vkvm:/home/vk/12.4# kubectl get pod -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE   NOMINATED NODE   READINESS GATES
deploy1-67c5999d57-454ft   2/2     Running   0          2m57s   10.1.101.229   vkvm   <none>           <none>
deploy1-67c5999d57-rrpbn   2/2     Running   0          2m57s   10.1.101.228   vkvm   <none>           <none>
deploy1-67c5999d57-t7zcv   2/2     Running   0          2m57s   10.1.101.230   vkvm   <none>           <none>
multitool-pod              1/1     Running   0          27s     10.1.101.231   vkvm   <none>           <none>
```

Запустим проверки портов из пода `multitool-pod`

- Проверка доступа к подам

```bash
root@vkvm:/home/vk/12.4# kubectl exec -it multitool-pod -- curl --silent -i 10.1.101.230:80
HTTP/1.1 200 OK
Server: nginx/1.23.4
Date: Sun, 23 Apr 2023 06:20:38 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Mar 2023 15:01:54 GMT
Connection: keep-alive
ETag: "64230162-267"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

root@vkvm:/home/vk/12.4# kubectl exec -it multitool-pod -- curl --silent -i 10.1.101.229:8080
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Sun, 23 Apr 2023 06:23:09 GMT
Content-Type: text/html
Content-Length: 145
Last-Modified: Sun, 23 Apr 2023 06:14:51 GMT
Connection: keep-alive
ETag: "6444ccdb-91"
Accept-Ranges: bytes

WBITT Network MultiTool (with NGINX) - deploy1-67c5999d57-454ft - 10.1.101.229 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```

Убедились, что  доступ до подов работает из другого пода

- Проверка доступа к сервисам

```bash
root@vkvm:/home/vk/12.4# kubectl exec -it multitool-pod -- curl  -i 10.152.183.148:9001
HTTP/1.1 200 OK
Server: nginx/1.23.4
Date: Sun, 23 Apr 2023 06:26:01 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Mar 2023 15:01:54 GMT
Connection: keep-alive
ETag: "64230162-267"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
root@vkvm:/home/vk/12.4# kubectl exec -it multitool-pod -- curl  -i 10.152.183.148:9002
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Sun, 23 Apr 2023 06:26:16 GMT
Content-Type: text/html
Content-Length: 145
Last-Modified: Sun, 23 Apr 2023 06:14:53 GMT
Connection: keep-alive
ETag: "6444ccdd-91"
Accept-Ranges: bytes

WBITT Network MultiTool (with NGINX) - deploy1-67c5999d57-t7zcv - 10.1.101.230 - HTTP: 8080 , HTTPS: 11443 . (Formerly praqma/network-multitool)
```

Убедились, что  доступ до сервисов работает из другого пода

### 4. Продемонстрировать доступ с помощью `curl` по доменному имени сервиса

Проверим запуск `curl` из пода `multitool-pod` по относительному и абсолютному DNS имени сервиса

```bash
root@vkvm:/home/vk/12.4# kubectl exec -it multitool-pod -- curl --silent -i service1:9001 | grep Server
Server: nginx/1.23.4
root@vkvm:/home/vk/12.4# kubectl exec -it multitool-pod -- curl --silent -i service1.default.svc.cluster.local:9002 | grep Server
Server: nginx/1.20.2
```

Убедились, что доступ по относительному и абсолютному DNS имени сервиса работает из другого пода

### 5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4

Удалим развёрнутые ресурсы

```bash
root@vkvm:/home/vk/12.4# kubectl delete -f service1.yml 
service "service1" deleted
```

## Задание 2

### 1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort

Создадим файл `service2.yml` с развёртыванием сервиса

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service2
spec:
  type: NodePort
  selector:
    app: deploy1
  ports:
    - name: nginx-http
      port: 9001
      nodePort: 30000
      protocol: TCP
      targetPort: 80
```

[Ссылка на service2.yml](service2.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk/12.4# kubectl create -f service2.yml
service/service2 created
```

Проверим состояние сервиса

```bash
root@vkvm:/home/vk/12.4# kubectl get service
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP          16d
service2     NodePort    10.152.183.133   <none>        9001:30000/TCP   18s
```

### 2. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера

Проверим IP адрес нод

```bash
root@vkvm:/home/vk/12.4# kubectl get node -o yaml | grep IPv4Address
      projectcalico.org/IPv4Address: 192.168.121.135/24
```

Проверим запуск curl из локального компьютера

```bash
root@vkvm:/home/vk/12.4# curl --silent -i 192.168.121.135:30000 | grep Server
Server: nginx/1.23.4
```

Убедились, что доступ до сервиса по адресу ноды возможен c локального компьютера

### 3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2

Удалим ресурсы

```bash
root@vkvm:/home/vk/12.4# kubectl delete -f deploy1.yml -f multitool-pod.yml -f service2.yml
deployment.apps "deploy1" deleted
pod "multitool-pod" deleted
service "service2" deleted
```

